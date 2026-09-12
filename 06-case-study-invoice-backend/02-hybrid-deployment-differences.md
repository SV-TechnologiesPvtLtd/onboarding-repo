# Deployment Differences for invoice-backend — Hybrid Model

Same headline as the general note: **almost nothing changes**, and it's even more true here than in the generic case, because this repo's actual deployment setup (`docker-compose.yml` + three Dockerfiles) is simpler than the AWS ECS/ALB example from earlier — there's no load balancer or HTTP/2-aware infrastructure in play yet at all. Let's go through exactly what's here and confirm what does and doesn't move.

---

## 1. `docker-compose.yml` — no changes needed

```yaml
gateway:
  build:
    context: .
    dockerfile: services/gateway/Dockerfile
  environment:
    PORT: "8080"
    COMPANY_SERVICE_URL: "company-service:50051"
    INVOICE_SERVICE_URL: "invoice-service:50052"
    ALLOWED_ORIGINS: ${ALLOWED_ORIGINS:-http://localhost:3000,http://localhost:5173}
  ports:
    - "8080:8080"
```

- `PORT: "8080"` and the exposed port mapping — unchanged. The gateway still speaks plain HTTP/1.1 on 8080; only the body encoding of what flows over it changes.
- `COMPANY_SERVICE_URL` / `INVOICE_SERVICE_URL` — unchanged. These point at the internal gRPC services, and the migration plan from the previous note explicitly doesn't touch that leg at all.
- `ALLOWED_ORIGINS` — unchanged. CORS config doesn't need new entries for this hybrid (confirmed in Step 6 of the migration plan: `Content-Type` was already an allowed header).
- `company-service` / `invoice-service` blocks — completely untouched. They don't know or care that the gateway's edge protocol changed; from their point of view, they're still receiving the exact same gRPC calls they always have.

**The one thing worth double-checking, not changing:** there's no `healthcheck:` block defined on the `gateway` service here (unlike `postgres`, which has one). If you ever add one as part of this work, it would just be a plain HTTP health check against a route like `GET /health` — that route's behavior has nothing to do with protobuf vs JSON, so it isn't a "hybrid deployment change," just an unrelated gap worth being aware of if you're touching this file anyway.

---

## 2. `services/gateway/Dockerfile` — no changes needed

```dockerfile
FROM golang:alpine AS builder
ENV GOTOOLCHAIN=auto
WORKDIR /app
COPY go.mod go.sum ./
COPY vendor/ vendor/
COPY pkg/ pkg/
COPY services/gateway/ services/gateway/
RUN CGO_ENABLED=0 GOOS=linux go build -mod=vendor -ldflags="-w -s" -o /gateway ./services/gateway/cmd/main.go

FROM alpine:3.19
RUN apk --no-cache add ca-certificates tzdata
WORKDIR /app
COPY --from=builder /gateway ./gateway
EXPOSE 8080
CMD ["./gateway"]
```

- `google.golang.org/protobuf` (needed for `proto.Marshal`/`Unmarshal` per the migration plan) is **already an indirect dependency** of this project today — the gateway already imports `google.golang.org/protobuf/encoding/protojson` in `response.go`. So `go.mod`/`go.sum`/`vendor/` don't need any *new* dependency added; you're using a package that's already vendored, just calling a different function from the same module (`proto.Marshal` instead of `protojson.Marshal`).
- `EXPOSE 8080` stays — still plain HTTP, not gRPC's usual `50051`-style convention, because this port never becomes gRPC; it's still REST-shaped traffic with a different body format.
- The multi-stage build itself (`golang:alpine` → `alpine:3.19`, vendored deps, static binary) needs zero structural changes.

**Important nuance specific to this project, versus the generic gRPC deployment note:** the general note assumed a CI/CD pipeline step that runs `protoc` to generate Go code from `.proto` files as part of the build. **This repo doesn't do that.** Per the `README.md`, `repo-proto`'s *already-generated* Go stubs are pulled in as a versioned Go module (`go get github.com/SV-TechnologiesPvtLtd/repo-proto/go@generated_proto`) — codegen happens once, centrally, in the `repo-proto` repository itself, not on every build here. So this hybrid migration needs **no new build step in this Dockerfile or any CI pipeline** — you're not generating anything new; `companyv1.CreateCompanyRequest` etc. are already compiled Go structs sitting in `vendor/` today.

---

## 3. `company-service` / `invoice-service` Dockerfiles and configs — fully untouched

Same reasoning as their `docker-compose.yml` blocks: these two services still run exactly as they do today — same `EXPOSE 50051` / `50052`, same `DATABASE_URL`, same multi-stage build. They were already gRPC-only, with no HTTP-facing surface at all, and this migration doesn't add one. There is nothing in this migration that touches `services/company-service/` or `services/invoice-service/` in any way.

---

## 4. CI/CD — one nuance worth flagging

The `README.md` describes a `.github/workflows/ci.yml` pipeline (not present in what was uploaded, but described as: authenticate against `repo-proto` → vendor deps → run tests → validate Docker builds). If/when that workflow exists:

- **No new step needed.** As established in Step 2, there's no local protoc/codegen step in this project's build process to begin with — the existing "vendor dependencies" step already pulls in everything needed (`google.golang.org/protobuf` was already vendored before this migration).
- **Test step (`go test -v ./...`) picks up new tests automatically** once you add the unit tests for `DecodeProto`/`WriteProto`/`HandleGRPCErrorProto` from Step 8 of the migration plan — no pipeline configuration change, just more tests running under the same existing command.
- **Docker Buildx validation** — unaffected, since the Dockerfile itself doesn't change (Step 2 above).

---

## 5. If/when this moves beyond docker-compose to a real load-balanced deployment

Worth noting explicitly, since the earlier general notes assumed AWS ECS + ALB: **that infrastructure doesn't exist in this repo yet** — today it's `docker-compose up` on a single host, with each service getting its own container and a bridge network (`receipt-net`). If this project is later deployed behind a real load balancer (ALB, Nginx, etc.), the same conclusion from the general hybrid deployment note applies directly:

- The gateway's target group would stay a standard **`HTTP1`** protocol type, not `GRPC` — because the gateway's public-facing side is still plain HTTP/1.1 REST, regardless of whether the body is JSON or protobuf.
- No Envoy or grpc-web proxy needed at the edge for this reason — that machinery is only required if the *browser* were speaking real gRPC/gRPC-Web directly, which it isn't here; it's speaking REST verbs with a protobuf body, same as the general hybrid case.
- The **internal** `company-service`/`invoice-service` connections (gateway → 50051/50052) are the only genuinely gRPC connections in this system, and they're both **internal**, not exposed through any public load balancer today (`docker-compose.yml` does map `50051`/`50052` to the host, but that's a local-dev convenience, not how you'd expose them in a real deployment — in production those ports would typically stay inside the private network, reachable only by the gateway).

So the one place this project *would* eventually need the HTTP/2-aware load-balancing considerations from the earlier gRPC deployment note is if you ever exposed `company-service` or `invoice-service` gRPC endpoints directly to something outside the gateway (e.g., another team's service calling in over gRPC from outside this docker-compose network) — not because of anything in this hybrid migration itself.

---

## Summary

| File / concern | Changes needed? |
|---|---|
| `docker-compose.yml` | None |
| `services/gateway/Dockerfile` | None |
| `services/company-service/Dockerfile`, `services/invoice-service/Dockerfile` | None |
| `go.mod` / `go.sum` / `vendor/` | None — `google.golang.org/protobuf` already present |
| CI pipeline (`ci.yml`, per README) | None — no protoc step exists or is needed; existing `go test ./...` picks up new tests automatically |
| Environment variables (`COMPANY_SERVICE_URL`, `INVOICE_SERVICE_URL`, `ALLOWED_ORIGINS`, `PORT`) | None |
| Ports / networking (`receipt-net`, exposed ports) | None |
| A future load balancer, if/when one is added | Would use standard `HTTP1` target group, same as plain REST — no `GRPC` protocol version or Envoy needed, since the public-facing side never becomes real gRPC/HTTP2 |

**Bottom line for this specific project:** because `company-service` and `invoice-service` were already gRPC-only and internal, and because `repo-proto`'s codegen already happens externally rather than in this repo's build, this hybrid migration is even more deployment-neutral here than the generic case — it's genuinely just an application-code change inside `services/gateway`, with zero Docker, Compose, environment, or CI/CD changes required.
