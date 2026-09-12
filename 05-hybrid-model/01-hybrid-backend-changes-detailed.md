# Backend Changes Needed for the Hybrid Model (REST + Protobuf) — In Detail

This is a complete, detailed breakdown of everything that needs to change in the backend to move from JSON-over-REST to protobuf-over-REST, organized as a checklist you can work through in order. Each item explains *what* changes, *why*, and *how*.

---

## 1. Dependencies — add the protobuf library

```bash
go get google.golang.org/protobuf
```

Your `go.mod` gains this as a direct dependency. This is the library that provides `proto.Marshal` / `proto.Unmarshal` — the binary encode/decode functions that replace `encoding/json`'s `json.Marshal` / `json.Unmarshal`.

You also need the `protoc` compiler and its Go plugin installed wherever you generate code (locally and in CI):
```bash
# protoc itself (the compiler)
brew install protobuf   # or apt-get install protobuf-compiler, etc.

# the Go plugin that protoc calls to generate Go-specific code
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
```

---

## 2. Project structure — schema moves into its own layer

**Before:**
```
main.go   ← User struct + all handler logic in one file
```

**After:**
```
proto/
  user.proto      ← schema definitions (source of truth)
pb/
  user.pb.go      ← generated Go code (never hand-edited)
main.go           ← handlers, now using pb.User instead of a hand-written struct
```

This is a real structural change to how you organize code: your data model is no longer "just a Go struct" — it's defined once in `.proto`, and Go code is a *generated artifact* of that schema. Any other service (a Python service, a mobile app) can generate its own client code from the exact same `.proto` file, guaranteeing everyone agrees on the shape of `User`.

---

## 3. Define every message type you send or receive — including errors

This is the item people most often miss. In JSON, you could always improvise:
```go
json.NewEncoder(w).Encode(map[string]string{"error": "not found"})
```
No schema required — any map or struct can become JSON. **Protobuf doesn't allow this.** Every single thing you marshal needs to be a predefined message type. So you need to define messages for:

- Every resource (`User`, `Order`, etc.)
- List/collection wrappers, since protobuf doesn't have a bare "array" as a top-level message — you wrap it:
```proto
message UserList {
  repeated User users = 1;
}
```
- Every distinct request shape, if it differs from the resource itself (e.g., a `CreateUserRequest` if it doesn't need an `id` field yet):
```proto
message CreateUserRequest {
  string name = 1;
  string email = 2;
}
```
- **A generic error message**, since you'll need one for every failure path:
```proto
message ErrorResponse {
  string error = 1;
  string code = 2;   // optional: a machine-readable error code, since HTTP status alone is sometimes not granular enough
}
```

**Action item:** audit every JSON response your API currently sends — success and error paths both — and make sure each one has a corresponding `.proto` message defined before you touch any handler code.

---

## 4. Field numbering rules — get this right from day one

Every field in a protobuf message needs a unique number (`= 1`, `= 2`, etc.). This has no equivalent in JSON and is the source of most protobuf mistakes:

```proto
message User {
  int32 id = 1;
  string name = 2;
  string email = 3;
}
```

Rules that matter for a backend team:
- **Never reuse a field number**, even for a field you removed. If you delete `email = 3` later, don't give a new field `= 3` — retire that number permanently (you can mark it `reserved 3;` to prevent accidental reuse).
- **Never renumber existing fields.** The wire format is based on field *numbers*, not names — changing `email`'s number from `3` to `4` breaks every client that hasn't regenerated and redeployed with the new schema.
- **New fields must be added with new numbers**, and should generally be treated as optional in proto3 (proto3 fields are optional-by-default, with a zero-value default when absent) so that old and new code can coexist during a rolling deploy.

This is a genuinely new discipline for a backend team used to JSON, where adding/removing/renaming fields rarely broke anything as long as consumers used sensible defaults.

---

## 5. Content negotiation — Content-Type handling

**Change every response header:**
```go
// Before
w.Header().Set("Content-Type", "application/json")

// After
w.Header().Set("Content-Type", "application/x-protobuf")
```

**Change how you read the incoming Content-Type**, if your API needs to validate it (recommended, so you reject mismatched clients clearly instead of failing confusingly deep in unmarshal logic):
```go
if r.Header.Get("Content-Type") != "application/x-protobuf" {
	writeProtoError(w, http.StatusUnsupportedMediaType, "expected application/x-protobuf")
	return
}
```

**If you need to support both JSON and protobuf simultaneously** (a common transitional step — e.g., during a migration, or for third parties who aren't ready for protobuf), you'd branch on `Content-Type`/`Accept` headers and marshal accordingly. This roughly doubles your serialization code paths, so most teams either commit fully to protobuf per-endpoint, or version the API (`/v2/users` = protobuf, `/v1/users` = JSON) rather than trying to content-negotiate the same endpoint both ways indefinitely.

---

## 6. Request body reading — no more streaming decoder

**Before**, `encoding/json` could decode directly from the `io.Reader` (the request body stream) without you manually buffering it:
```go
json.NewDecoder(r.Body).Decode(&newUser)
```

**After**, `proto.Unmarshal` requires a `[]byte`, not a stream — so you read the whole body into memory first, then unmarshal:
```go
body, err := io.ReadAll(r.Body)
if err != nil {
	writeProtoError(w, http.StatusBadRequest, "could not read body")
	return
}

var newUser pb.User
if err := proto.Unmarshal(body, &newUser); err != nil {
	writeProtoError(w, http.StatusBadRequest, "invalid protobuf body")
	return
}
```

**Practical implication:** for very large request bodies, this means the entire payload sits in memory before you can start processing it, whereas a streaming JSON decoder could in principle start processing before the whole body arrived. For typical API payloads (a user object, an order) this doesn't matter; it's worth knowing about if any endpoint accepts large payloads (bulk imports, file-like data).

**You should also enforce a body size limit**, which is good practice for JSON too but becomes more important here since you're fully buffering:
```go
r.Body = http.MaxBytesReader(w, r.Body, 1<<20) // cap at 1MB, adjust as needed
```

---

## 7. Response writing — marshal-then-write instead of encode-to-stream

**Before:**
```go
json.NewEncoder(w).Encode(newUser)
```

**After:**
```go
respBytes, err := proto.Marshal(&newUser)
if err != nil {
	writeProtoError(w, http.StatusInternalServerError, "failed to encode response")
	return
}
w.Write(respBytes)
```

Two things to notice:
- You now handle a marshal error explicitly (`json.NewEncoder(w).Encode(...)` in Go rarely fails for ordinary structs, but it's more idiomatic to check the error with `proto.Marshal`).
- Order matters: set headers and call `w.WriteHeader(status)` **before** `w.Write(...)`, same as with JSON — this doesn't change, but it's easy to get wrong if you're restructuring handler code at the same time.

---

## 8. Centralize error responses — you'll want a helper

Since every error path now needs to marshal a real message type instead of throwing together a map, it's worth writing one helper used everywhere, rather than repeating this in every handler:

```go
func writeProtoError(w http.ResponseWriter, status int, message string) {
	w.Header().Set("Content-Type", "application/x-protobuf")
	w.WriteHeader(status)
	errBytes, _ := proto.Marshal(&pb.ErrorResponse{Error: message})
	w.Write(errBytes)
}
```

This is a good moment to also standardize on a `code` field (a stable machine-readable string like `"USER_NOT_FOUND"`) in `ErrorResponse`, separate from the free-text `message` — frontend error handling logic can then switch on `code` reliably instead of string-matching on `message`, which is a nice byproduct of formalizing your error schema.

---

## 9. Routing, verbs, and status codes — confirm what does NOT change

Worth stating explicitly, since it's easy to assume more changed than actually did:
- `http.HandleFunc`, `r.Method` switches, path param parsing via `r.URL.Path` — **all unchanged**.
- Query param parsing via `r.URL.Query()` — **unchanged**.
- HTTP status codes (200, 201, 400, 404, 422, etc.) — **unchanged**, still set via `w.WriteHeader(...)`.
- `Authorization` header checking — **unchanged**, still just reading a header string.
- CORS headers (`Access-Control-Allow-Origin`, etc.) — **unchanged**.

None of this needs to be touched. If you find yourself rewriting routing logic while doing this migration, that's scope creep beyond what protobuf-over-REST actually requires.

---

## 10. Testing — update fixtures and assertions

Any existing tests that build request bodies with `json.Marshal(...)` or assert on response bodies with `json.Unmarshal(...)` need to switch to the protobuf equivalents:

```go
// Before
body, _ := json.Marshal(map[string]string{"name": "Alex", "email": "alex@example.com"})

// After
body, _ := proto.Marshal(&pb.CreateUserRequest{Name: "Alex", Email: "alex@example.com"})
```

And for asserting on responses:
```go
// Before
var got User
json.Unmarshal(w.Body.Bytes(), &got)

// After
var got pb.User
proto.Unmarshal(w.Body.Bytes(), &got)
```

If you have integration tests that hit the API with raw `curl`/HTTP calls and inspect JSON text directly, those need rewriting to construct/parse protobuf bytes instead — a plain string comparison against a JSON body no longer works.

---

## 11. Logging and observability — decide what you log

Since request/response bodies are now binary, **don't log raw bodies as if they were readable text** (a common leftover habit from JSON debugging, like `log.Println(string(body))` — this now prints binary garbage, or worse, could include non-printable bytes that mangle your log output).

Instead:
- Log structured fields you extract *after* unmarshaling (e.g., `log.Printf("created user id=%d", newUser.Id)`), not the raw body.
- If you need full-body debugging occasionally, use `protojson` (part of the same protobuf library) to convert a message to a JSON *representation* purely for logging/debugging purposes:
```go
import "google.golang.org/protobuf/encoding/protojson"

debugJSON, _ := protojson.Marshal(&newUser)
log.Printf("debug: %s", debugJSON)
```
This doesn't change what you send over the wire (still real protobuf binary) — it's purely a debugging convenience so you're not staring at raw bytes in your logs.

---

## Checklist summary

- [ ] Add `google.golang.org/protobuf` dependency; install `protoc` + Go plugin in local/CI environments
- [ ] Create `proto/` directory; move every data shape (including errors) into `.proto` message definitions
- [ ] Assign and document field numbers; never reuse or renumber them
- [ ] Set up code generation (`protoc --go_out=...`) producing `pb/*.pb.go`
- [ ] Update every response header from `application/json` to `application/x-protobuf`
- [ ] Replace `json.NewDecoder(r.Body).Decode(...)` with `io.ReadAll` + `proto.Unmarshal`
- [ ] Replace `json.NewEncoder(w).Encode(...)` with `proto.Marshal` + `w.Write`
- [ ] Add a body size limit (`http.MaxBytesReader`)
- [ ] Write a centralized `writeProtoError` helper; define `ErrorResponse` with a `code` field
- [ ] Confirm routing, verbs, status codes, auth header checks, and CORS are untouched
- [ ] Update tests to build/assert protobuf bytes instead of JSON strings
- [ ] Remove any raw-body logging; use `protojson` for debug-only JSON representations if needed
- [ ] Establish a process for regenerating `pb/*.pb.go` whenever `.proto` files change, and for coordinating schema changes with frontend/other consumers
