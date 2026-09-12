# Hybrid Model Migration Plan — invoice-backend

**Goal:** Frontend keeps calling the gateway's existing REST routes (`/api/v1/companies`, `/api/v1/invoices`, etc.) with the same verbs, URLs, and status codes — but the **body** becomes binary protobuf instead of JSON. Internal service-to-service calls (`gateway → company-service`, `gateway → invoice-service`) **stay exactly as they are today: real gRPC.** No change is needed there — this project already has that half of the hybrid built.

This plan is scoped entirely to `services/gateway`. `company-service`, `invoice-service`, and `repo-proto`'s Go stubs are untouched — mirroring the same "out of scope" boundary the existing `connect-rpc-gateway-bridge.md` plan already established for a similar reason.

---

## 0. Why this project is a particularly good fit for this hybrid

Two things about the current codebase make this easier than a from-scratch implementation:

1. **The request/response types are already real protobuf messages.** Look at `handler.go`:
   ```go
   var protoReq companyv1.CreateCompanyRequest
   ```
   `companyv1.CreateCompanyRequest` is generated from `repo-proto` and already implements `proto.Message`. Today it's being decoded via plain `encoding/json` (`DecodeJSON`) and encoded via `protojson` (`WriteJSON`) — both of which serialize the *same* underlying proto struct, just as JSON text. Switching to `proto.Marshal`/`proto.Unmarshal` doesn't require defining any new message types — you're reusing the exact same generated structs, just picking a different serializer for them.

2. **The frontend likely already has the matching TypeScript types generated.** Per the `connect-rpc-gateway-bridge.md` investigation, `repo-proto` already generates `protobuf-es` v2 stubs for the frontend (used today for the Connect-RPC bridge). Those same generated classes have `.toBinary()` / `.fromBinary()` methods built in — the frontend doesn't need new codegen for this either, just a different way of calling `fetch()`.

This means the entire migration is really: **change the serializer at the gateway's HTTP boundary, on both ends.** Nothing about routing, business logic, or the internal gRPC calls needs to move.

---

## Step 1 — Decide the error-response format, using what's already imported

Before touching the handlers, resolve one design question: **how do errors get encoded once responses are binary protobuf?**

The good news: you don't need to invent a new `ErrorResponse` proto message (which would violate the project's own rule — "Keep Protobuf Contracts Centralized... do not modify proto definitions locally"). `httputil/error.go` already imports `google.golang.org/grpc/status`, and a gRPC `status.Status` has a `.Proto()` method that returns `*spb.Status` — a real, standard protobuf message (`google.rpc.Status`) that already carries a code, a message, and a `details` field (which is exactly where `errdetails.BadRequest` field violations — already used in this codebase — plug in natively).

**Decision: reuse `status.Status.Proto()` directly as the wire format for protobuf error responses**, instead of the current custom `APIError`/`ErrorResponse` Go structs (which only exist for JSON encoding today). This keeps you inside "centralized contracts" — `google.rpc.Status` is a well-known, externally-defined proto, not something you're adding to `repo-proto` or hand-rolling locally.

---

## Step 2 — Add protobuf encode/decode helpers to `httputil`

Add two new functions alongside the existing `DecodeJSON` and `WriteJSON` — don't replace them yet (see Step 6 on content negotiation for why).

**`httputil/decode.go`** — add `DecodeProto`:
```go
import "google.golang.org/protobuf/proto"

// DecodeProto reads the raw request body and unmarshals it as a binary protobuf message.
func DecodeProto(r *http.Request, target proto.Message) error {
	defer r.Body.Close()

	body, err := io.ReadAll(io.LimitReader(r.Body, 1<<20)) // 1MB cap, matches the project's existing timeout-conscious defaults
	if err != nil {
		return status.Errorf(codes.InvalidArgument, "failed to read request body: %v", err)
	}
	if len(body) == 0 {
		return status.Error(codes.InvalidArgument, "request body cannot be empty")
	}

	if err := proto.Unmarshal(body, target); err != nil {
		return status.Errorf(codes.InvalidArgument, "invalid protobuf payload: %v", err)
	}
	return nil
}
```

**Note on field-level error detail parity:** `DecodeJSON`'s richest behavior — catching `json.UnmarshalTypeError` to report exactly which field had the wrong type — has no equivalent in `proto.Unmarshal`. Binary protobuf either parses or it doesn't; there's no per-field type mismatch to report the way malformed JSON has. This is a real, permanent trade-off of this hybrid approach worth flagging to the team: protobuf request validation errors will be less specific than the current JSON path's `errdetails.BadRequest` field violations for malformed payloads (though your existing *business-logic* validation errors coming back from `company-service`/`invoice-service` over gRPC are unaffected — those already carry structured field violations independent of the wire format at the edge).

**`httputil/response.go`** — add `WriteProto`:
```go
// WriteProto serializes a protobuf message to binary and writes it to the response.
func WriteProto(w http.ResponseWriter, statusCode int, msg proto.Message) error {
	body, err := proto.Marshal(msg)
	if err != nil {
		return fmt.Errorf("failed to marshal proto response: %w", err)
	}

	w.Header().Set("Content-Type", "application/x-protobuf")
	w.WriteHeader(statusCode)
	_, _ = w.Write(body)
	return nil
}
```

**`httputil/error.go`** — add a protobuf-encoding path for errors, reusing `status.Status.Proto()` per Step 1:
```go
// HandleGRPCErrorProto mirrors HandleGRPCError but writes google.rpc.Status as binary protobuf.
func HandleGRPCErrorProto(w http.ResponseWriter, err error) {
	st, ok := status.FromError(err)
	if !ok {
		log.Printf("[ERROR] Non-gRPC error in handler: %v", err)
		st = status.New(codes.Internal, "an internal error occurred")
	}

	httpCode := grpcCodeToHTTP(st.Code()) // extract the existing switch in HandleGRPCError into a shared helper — see note below

	body, marshalErr := proto.Marshal(st.Proto())
	if marshalErr != nil {
		w.WriteHeader(http.StatusInternalServerError)
		return
	}

	w.Header().Set("Content-Type", "application/x-protobuf")
	w.WriteHeader(httpCode)
	_, _ = w.Write(body)
}
```
This requires one small refactor: pull the `codes.NotFound → http.StatusNotFound` etc. `switch` block currently inline in `HandleGRPCError` out into a standalone `grpcCodeToHTTP(code codes.Code) int` function, so both the JSON and protobuf error paths share the exact same status-code mapping instead of duplicating (and risking drifting) that logic.

---

## Step 3 — Content negotiation: let `Wrap` and handlers pick the serializer per request

Rather than a hard cutover, use the `Content-Type` / `Accept` headers the client already sends to decide which serializer to use — this lets you migrate route-by-route or client-by-client without a breaking change.

Modify `httputil/response.go`'s `Wrap` and add a small content-type helper:

```go
// IsProtoRequest reports whether the request body/response should use binary protobuf,
// based on the client's declared Content-Type.
func IsProtoRequest(r *http.Request) bool {
	return r.Header.Get("Content-Type") == "application/x-protobuf"
}
```

Then in each handler, branch once at the top instead of hardcoding `DecodeJSON`/`WriteJSON`:

```go
// HandleCreateCompany: POST /api/v1/companies
func (h *Handler) HandleCreateCompany(w http.ResponseWriter, req *http.Request) error {
	var protoReq companyv1.CreateCompanyRequest

	if httputil.IsProtoRequest(req) {
		if err := httputil.DecodeProto(req, &protoReq); err != nil {
			return err
		}
	} else {
		if err := httputil.DecodeJSON(req, &protoReq); err != nil {
			return err
		}
	}

	res, err := h.companyClient.CreateCompany(req.Context(), &protoReq)
	if err != nil {
		return err
	}

	if httputil.IsProtoRequest(req) {
		return httputil.WriteProto(w, http.StatusCreated, res.GetCompany())
	}
	return httputil.WriteJSON(w, http.StatusCreated, res.GetCompany())
}
```

**Why branch on request `Content-Type` for the response format too, rather than `Accept`:** it's simpler and matches how most protobuf-over-REST setups behave in practice — a client sending binary protobuf almost always wants binary protobuf back. If you want stricter REST semantics later, swap the response branch to check `Accept` instead, but that's an enhancement, not a requirement to ship this.

Also update `Wrap` itself so error responses match the request's format:
```go
func Wrap(h AppHandler) http.HandlerFunc {
	return func(w http.ResponseWriter, req *http.Request) {
		if err := h(w, req); err != nil {
			if IsProtoRequest(req) {
				HandleGRPCErrorProto(w, err)
			} else {
				HandleGRPCError(w, err)
			}
		}
	}
}
```

This one change in `Wrap` means every handler registered via `httputil.Wrap(...)` in `routes.go` automatically gets protobuf-aware error handling for free — you don't need to touch `RecoveryMiddleware`'s panic path immediately (it can stay JSON-only initially, since panics are rare and not the primary path being migrated), though for full parity you'd eventually apply the same `IsProtoRequest` branch there too.

---

## Step 4 — Apply the Step 3 pattern to every handler

Repeat the same branch in each of the six other handlers (`HandleGetCompany`, `HandleCompanyInvoices` in `company/handler.go`; `HandleCreateInvoice`, `HandleGetInvoice`, `HandleListInvoices`, `HandleUpdateInvoice`, `HandleDeleteInvoice` in `invoice/handler.go`). This is mechanical once the pattern above is established — each handler just needs its existing `httputil.DecodeJSON(...)` / `httputil.WriteJSON(...)` calls wrapped in the same `if httputil.IsProtoRequest(req) { ... } else { ... }` shape.

**No changes needed to `routes.go` in either package** — this is the entire point of the hybrid approach. `POST /api/v1/companies`, `GET /api/v1/invoices/{id}`, etc. stay registered exactly as they are; only what happens *inside* the handler body changes.

---

## Step 5 — Confirm the gRPC calls inside the handlers need zero changes

This is worth stating explicitly since it's easy to assume more moved than actually did: `h.companyClient.CreateCompany(req.Context(), &protoReq)` and every other call in `clients.go` / the handler files is **completely unaffected**. The gateway already talks to `company-service` and `invoice-service` over real gRPC (`grpc.NewClient`, `companyv1.NewCompanyServiceClient`, etc.) — that internal leg of the hybrid architecture you asked about is already built and doesn't need touching. This migration only changes the *outer* edge — frontend ↔ gateway — not the *inner* edge — gateway ↔ services.

---

## Step 6 — Middleware: CORS header check

Look at `middleware.go`'s `CORSMiddleware`:
```go
AllowedHeaders: []string{"Accept", "Authorization", "Content-Type", "X-CSRF-Token"},
```
`Content-Type` is already allowed — no change needed here. Since `application/x-protobuf` is, like `application/json`, a "non-simple" content type, the browser will still send a CORS preflight (`OPTIONS`) request either way, and it'll pass for the same reason JSON requests already do today. **No middleware changes required for this step**, unlike the Connect-RPC bridge plan which needed new headers (`Connect-Protocol-Version`) — this hybrid doesn't introduce any new headers the frontend needs to send.

---

## Step 7 — Frontend changes

Since `repo-proto` already generates `protobuf-es` v2 TypeScript classes (confirmed by the Connect-RPC bridge doc), the frontend already has what it needs — no new codegen step required.

```ts
import { CreateCompanyRequest, Company } from "@sv-technologies/repo-proto/company/v1";

async function createCompany(userId: string, name: string, email: string) {
  const reqMsg = new CreateCompanyRequest({ userId, name, email });

  const res = await fetch("http://localhost:8080/api/v1/companies", {
    method: "POST",
    headers: {
      "Content-Type": "application/x-protobuf",
      "Accept": "application/x-protobuf",
    },
    body: reqMsg.toBinary(), // protobuf-es serialization, not JSON.stringify
  });

  if (!res.ok) {
    const errBuffer = await res.arrayBuffer();
    // Decode as google.rpc.Status, per Step 1's decision to reuse it for errors
    const errStatus = Status.fromBinary(new Uint8Array(errBuffer));
    throw new Error(errStatus.message);
  }

  const buffer = await res.arrayBuffer();
  return Company.fromBinary(new Uint8Array(buffer));
}
```

This is a genuinely small change per call site: swap `JSON.stringify(...)` → `.toBinary()`, `res.json()` → `res.arrayBuffer()` + `.fromBinary()`, and the two headers. Routing, method, and status-code handling in the frontend's existing REST client code don't change at all.

---

## Step 8 — Testing

- Unit test `DecodeProto`/`WriteProto`/`HandleGRPCErrorProto` directly (small, pure-function-style tests, matching the "keep this minimal" philosophy the Connect-RPC bridge doc already used for its own `errors_test.go` — this repo currently has no gateway tests, so don't over-invest beyond covering the new serialization logic itself).
- Add one integration-style test per resource that posts binary protobuf to `/api/v1/companies` and asserts a binary protobuf response decodes correctly — mirroring the existing `curl` verification steps in `connect-rpc-gateway-bridge.md`, but with `Content-Type: application/x-protobuf` and a raw binary body instead of JSON.
- Explicitly test the **content-negotiation branch**: the same route, called once with `Content-Type: application/json` and once with `application/x-protobuf`, should both keep working — this is the regression risk of Step 3's approach, since you're now maintaining two code paths per handler instead of one.

---

## Step 9 — Rollout order

1. Ship Steps 1–2 (helpers) with no handler changes yet — purely additive, zero risk.
2. Migrate one low-traffic route first (e.g. `GET /api/v1/companies/{id}`) end-to-end (Steps 3–4 for that route, plus the matching frontend call site from Step 7) to validate the pattern against a real request in your dev environment.
3. Roll the same pattern out to the remaining company and invoice handlers.
4. Once all frontend call sites have migrated to protobuf, the JSON branches (`DecodeJSON`/`WriteJSON`/`HandleGRPCError`) can either stay indefinitely (for any third-party consumers still on JSON) or be removed — that's a product decision, not a technical one, since Step 3's design supports both indefinitely at low cost.

---

## What does not change (summary)

- `services/company-service`, `services/invoice-service` — untouched, exactly like the Connect-RPC bridge plan already scoped.
- `repo-proto` — untouched; no new proto messages needed (reusing `google.rpc.Status` for errors keeps you within "centralized contracts").
- `routes.go` in both `handlers/company` and `handlers/invoice` — untouched; same URLs, same verbs.
- `clients.go` and every `h.companyClient.X(...)` / `h.invoiceClient.X(...)` call — untouched; internal gRPC calls are unaffected by what format the edge speaks to the browser.
- `docker-compose.yml`, both service `Dockerfile`s, `config.go` — untouched; no new ports, no new environment variables, no load-balancer-relevant changes, consistent with the earlier general finding that this hybrid has near-zero deployment impact.
- `CORSMiddleware`'s allowed headers — untouched; `Content-Type` was already permitted.

**What changes, in total:** `httputil/decode.go`, `httputil/response.go`, `httputil/error.go` (new functions, plus one small refactor to share the code→HTTP-status mapping), every handler function in `handlers/company/handler.go` and `handlers/invoice/handler.go` (each gains a content-type branch), and the frontend's `fetch()` call sites (swap serializer, same URLs/methods).
