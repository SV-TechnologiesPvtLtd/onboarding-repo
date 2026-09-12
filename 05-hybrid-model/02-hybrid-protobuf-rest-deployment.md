# Deployment Differences for the Hybrid Model (REST + Protobuf)

Short answer up front: **almost nothing changes.** Because this hybrid keeps plain HTTP/1.1, normal REST verbs, and normal status codes — and only swaps the body's encoding — it deploys exactly like the original REST guide (Docker → ECR → ECS Fargate → ALB → Route 53). None of the HTTP/2-specific, load-balancer-specific, or proxy-specific changes from the full gRPC deployment note apply here. This note walks through why, and calls out the handful of things that genuinely do change.

---

## 1. Docker — unchanged

The Dockerfile is identical to the original REST guide. You're still compiling a Go binary and running it in a minimal image; the only difference is your `go.mod` now includes the protobuf library (`google.golang.org/protobuf`) as a dependency, which gets compiled into the same binary like any other package.

```dockerfile
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o server .

FROM alpine:3.19
WORKDIR /app
COPY --from=builder /app/server .
EXPOSE 8080
CMD ["./server"]
```

No new base images, no new exposed ports by convention (still `8080`, not gRPC's usual `50051`), no protocol-specific runtime requirements.

---

## 2. Load balancer — unchanged (this is the big one)

This is the single biggest difference from the full gRPC deployment. Recall from the gRPC note: the reason deployment got complicated there was that a **connection-level load balancer doesn't distribute individual gRPC calls well**, because gRPC multiplexes many calls over one long-lived HTTP/2 connection.

**None of that applies here.** This hybrid is still plain HTTP/1.1 request/response — each `fetch()` call from the frontend is its own independent HTTP request, exactly like the JSON version. So:

- **No Envoy or grpc-web proxy needed.** The ALB talks directly to your containers, same as before.
- **Target group protocol stays `HTTP1`**, not `GRPC`:
```bash
aws elbv2 create-target-group \
  --name my-rest-api-targets \
  --protocol HTTP \
  --port 8080 \
  --vpc-id vpc-0abc123 \
  --target-type ip \
  --health-check-path /health
```
- **No `--alpn-policy HTTP2Preferred` requirement** on the listener — plain HTTPS works fine, same as the original REST guide.
- **Connection-level load balancing works fine**, because there's no long-lived multiplexed connection pinning traffic to one container the way gRPC's HTTP/2 connections do.

In short: the ALB has no idea, and doesn't need to know, that the body happens to be protobuf instead of JSON. As far as HTTP is concerned, it's just bytes in a request/response body — the load balancer only cares about the HTTP-level envelope (method, path, headers, status code), which is completely unchanged.

---

## 3. Health checks — unchanged

Your `/health` endpoint can stay exactly as it was — a plain `GET` that returns `200 OK` with an empty or simple body. You do **not** need to implement the gRPC health-checking protocol (`grpc.health.v1.Health`) that the full gRPC deployment required, since there's no gRPC framework involved here at all.

```go
http.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(http.StatusOK)
})
```

The ALB's health check configuration is the same as the original guide too — no `--health-check-protocol HTTP2` or gRPC-specific path.

---

## 4. TLS — unchanged

TLS termination at the ALB works exactly as before. There's no ALPN negotiation requirement forcing HTTP/2 end-to-end, because nothing in this setup requires HTTP/2 — plain HTTPS (which is just HTTP/1.1 over TLS, from the ALB's perspective) is entirely sufficient. You keep the same ACM certificate setup from the original guide.

---

## 5. ECS task definition, secrets, auto scaling — unchanged

All of this carries over exactly as written in the original REST deployment guide:
- Task definition with `cpu`/`memory` limits — same.
- Secrets pulled from AWS Secrets Manager into environment variables — same.
- Auto scaling based on CPU/request count — same, since request patterns (short request/response cycles) look the same to the auto scaler regardless of whether the body is JSON or protobuf.

---

## 6. What actually does change

### a) CI/CD gets one new step: generating code from `.proto` files

This is the one real addition, and it's the same step the full gRPC pipeline needed:

```yaml
- name: Generate Go code from proto files
  run: |
    protoc --go_out=. --go_opt=paths=source_relative proto/user.proto

- name: Run tests
  run: go test ./...

- name: Build, tag, and push image
  run: |
    docker build -t my-rest-api .
    # ...same ECR push/tag steps as before
```

You need `protoc` (and the Go protobuf plugin) available in your CI runner, and the generated `pb/user.pb.go` file needs to be regenerated any time `user.proto` changes, before the Go build runs. This is a build-time dependency, not a deployment-time or infrastructure one — it doesn't touch Docker, ECS, or the ALB at all.

### b) Monitoring and debugging get slightly harder

- **CloudWatch logs** still work exactly the same for your application's own log lines (`log.Println(...)`, etc.) — that's just text, unaffected by protobuf.
- **But if you log or inspect request/response bodies for debugging**, they're now binary and unreadable as plain text in the CloudWatch console — you'd need to decode them with the `.proto` schema (e.g., using `protoc --decode` or a small script) rather than eyeballing raw JSON in a log line.
- **`curl`-based manual debugging** is less convenient — `curl https://api.yourapp.com/users` now returns binary garbage in your terminal instead of readable JSON, unless you pipe it through a protobuf decoder that has the schema loaded.
- **API Gateway / WAF rules that inspect JSON bodies** (e.g., a WAF rule that blocks requests containing certain JSON field values) generally can't do the same body inspection against protobuf's binary format — if you rely on this kind of body-aware filtering, it needs to be reworked or dropped for protobuf endpoints.

### c) Schema evolution needs discipline during rollouts

Because both the frontend and backend now depend on the same generated code from a shared `.proto` file, deploying a backend change (e.g., adding a new field) needs to follow protobuf's backward-compatibility rules — never reuse or renumber a field number, add new fields as optional, etc. — so that an old frontend build calling a newly-deployed backend (or vice versa, during a rolling deploy where old and new containers briefly coexist) doesn't break. This is a coordination discipline change, not an infrastructure change, but it's worth calling out because JSON was much more forgiving here (an old client ignoring an unexpected new JSON field never caused a problem; a poorly evolved protobuf schema can).

---

## Summary table: three deployments side by side

| Concern | Plain REST (JSON) | Hybrid (REST + protobuf) | Full gRPC |
|---|---|---|---|
| Dockerfile | Standard | Standard (protobuf lib added as a dependency) | Standard (different exposed port convention) |
| Load balancer | ALB, `HTTP1` target group | ALB, `HTTP1` target group — **unchanged** | ALB `GRPC` target group, or Envoy required |
| Connection-level LB works? | Yes | Yes | No — needs request-level LB |
| Health check | Plain `GET /health` | Plain `GET /health` — **unchanged** | `grpc.health.v1.Health` service |
| TLS/ALPN | Standard HTTPS | Standard HTTPS — **unchanged** | HTTP/2 ALPN negotiation required |
| Browser proxy needed? | No | No | Yes — grpc-web + Envoy |
| CI/CD | Test → build → push | Test → **generate proto code** → build → push | Test → generate proto + gRPC stubs → build → push |
| Log/debug readability | Human-readable JSON | Binary — needs schema to decode | Binary — needs schema to decode |
| Schema evolution discipline | Loose (JSON is forgiving) | Strict (protobuf field rules matter) | Strict (protobuf field rules matter) |

**Bottom line:** the hybrid model gets you protobuf's efficiency and schema discipline with essentially zero infrastructure cost beyond the original REST deployment — you inherit the debugging/observability friction and schema-evolution discipline that protobuf always brings, but you skip every load-balancer, health-check, TLS, and browser-compatibility change that full gRPC requires.
