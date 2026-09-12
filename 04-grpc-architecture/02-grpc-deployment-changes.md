# How Deployment Changes in a gRPC Architecture

This note builds directly on the previous REST deployment guide (Docker → ECS Fargate → ALB → Route 53) and explains what changes — and why — when the API is gRPC instead of REST. The short version: gRPC is faster and more efficient, but it breaks a lot of assumptions that simple load balancers and REST tooling are built on, so the infrastructure has to compensate.

---

## 1. The core difference that causes everything else

REST typically runs over **HTTP/1.1**: each request opens a connection (or reuses a pooled one), gets a response, and the connection is often short-lived or the exchange is "one request, one response" per stream.

gRPC runs over **HTTP/2**, and encodes messages using **Protocol Buffers (protobuf)** instead of JSON. HTTP/2 gives gRPC:
- **Multiplexing** — many requests/responses can flow over a *single* long-lived TCP connection at the same time.
- **Streaming** — a client and server can send a continuous stream of messages back and forth on that same connection (not just one request → one response).
- **Binary framing** — messages are compact binary, not human-readable text, making them faster to serialize/parse and smaller over the wire.

**Why this matters for deployment:** almost every infrastructure decision in the REST guide (load balancing, health checks, connection handling) assumed short, independent HTTP/1.1 requests. gRPC's long-lived, multiplexed HTTP/2 connections need infrastructure that understands that model — otherwise things silently break or become unbalanced.

---

## 2. Load balancing — the biggest change

### The problem: connection-level load balancing doesn't work well for gRPC

A traditional Application Load Balancer (or any L4/basic L7 balancer) typically distributes **connections**, not individual **requests**. With REST/HTTP1.1, that's mostly fine, because connections are numerous and short-lived — traffic naturally spreads across your containers.

With gRPC, a client opens **one long-lived HTTP/2 connection** and sends potentially thousands of requests over it. If your load balancer just balances *connections*, one container could end up handling nearly all of one client's traffic for the connection's entire lifetime, while other containers sit idle — the load balancer only gets a chance to "rebalance" when a new connection is made, which might be rare.

```
REST (HTTP/1.1):                    gRPC (HTTP/2, naive LB):
Client → LB → many short conns      Client → LB → ONE long connection
         → spread across A, B, C             → pinned to Container A only
```

### The fix: you need a load balancer that understands HTTP/2 / gRPC at the request level

Two common approaches:

**a) A proxy that load-balances at the request level, not the connection level**
Tools like **Envoy**, **NGINX (with gRPC support)**, or a **service mesh** (Istio, Linkerd) sit between clients and your containers, terminate the HTTP/2 connection from the client, and open their own pool of connections to the backend containers — distributing individual gRPC calls across that pool rather than pinning a whole client connection to one backend.

**b) Client-side load balancing**
Instead of relying on a proxy, the gRPC client itself is configured with a list of backend addresses (often via DNS or a service discovery system) and balances requests across them directly — common in internal service-to-service gRPC calls (e.g., inside a Kubernetes cluster using headless services + client-side round robin).

### What this means for the AWS setup from the previous guide

- A plain **Application Load Balancer (ALB)** *does* support gRPC (AWS added this), but you must explicitly configure the ALB listener with protocol `GRPC` — using a regular HTTP listener will not correctly multiplex gRPC traffic.
```bash
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:...:loadbalancer/... \
  --protocol HTTPS \
  --port 443 \
  --certificates CertificateArn=arn:aws:acm:...:certificate/... \
  --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:...:targetgroup/... \
  --alpn-policy HTTP2Preferred
```
- The **target group protocol version** must also be set to `GRPC` (not `HTTP1`), so the ALB knows to parse gRPC status codes (which are embedded in HTTP/2 trailers) rather than plain HTTP status codes:
```bash
aws elbv2 create-target-group \
  --name my-grpc-targets \
  --protocol HTTPS \
  --protocol-version GRPC \
  --port 8080 \
  --vpc-id vpc-0abc123 \
  --target-type ip \
  --health-check-protocol HTTP2 \
  --health-check-path /grpc.health.v1.Health/Check
```
- Many teams instead put **Envoy** in front of their gRPC services (even on AWS, behind a network load balancer) specifically because it gives finer control over HTTP/2 connection pooling, retries, and circuit breaking than the ALB alone provides. This is the single biggest infrastructural addition compared to the REST setup.

---

## 3. Health checks look different

In the REST guide, the ALB just called a plain HTTP endpoint (`GET /health`) and checked for a `200`.

gRPC has a **standardized health-checking protocol** (`grpc.health.v1.Health`) instead of a plain REST endpoint. Your Go server needs to implement this service explicitly:

```go
import (
	"google.golang.org/grpc/health"
	"google.golang.org/grpc/health/grpc_health_v1"
)

healthServer := health.NewServer()
grpc_health_v1.RegisterHealthServer(grpcServer, healthServer)
healthServer.SetServingStatus("", grpc_health_v1.HealthCheckResponse_SERVING)
```

The load balancer or orchestrator (ECS, Kubernetes) then calls this gRPC method instead of an HTTP GET — which is why the ALB target group above sets `--health-check-path /grpc.health.v1.Health/Check` rather than `/health`.

---

## 4. TLS is effectively mandatory, and ALPN matters

REST over plain HTTP is common in internal networks, and TLS termination at the load balancer was a nice-to-have for security. With gRPC, most client libraries **require HTTP/2**, and many environments (including browsers, if you're using grpc-web) require HTTP/2 to be negotiated via **ALPN** (Application-Layer Protocol Negotiation) during the TLS handshake — there's no reliable "plaintext HTTP/2" path across most real infrastructure the way there was with HTTP/1.1.

Practically: your ALB listener needs `--alpn-policy HTTP2Preferred` (shown above), and if you're terminating TLS at a proxy like Envoy instead, that proxy needs to be explicitly configured to speak HTTP/2 to your backend containers too — otherwise you get a working connection to the edge but broken calls behind it (a common gRPC deployment bug: TLS terminates fine at the LB, but the LB→backend hop silently downgrades to HTTP/1.1 and gRPC calls fail).

---

## 5. Service discovery matters more, especially internally

For **service-to-service** gRPC calls (e.g., your `orders` service calling your `inventory` service internally), teams often skip the load balancer entirely and use:
- **Kubernetes headless services + client-side load balancing** — the gRPC client resolves DNS to get *all* pod IPs (not just one), and balances requests across them itself.
- **A service mesh sidecar** (Istio/Linkerd via Envoy) — handles load balancing, retries, mTLS, and observability transparently, without your application code needing to know about any of it.

This is a bigger architectural shift than just "swap REST for gRPC" — it often nudges teams toward adopting a service mesh specifically *because* gRPC's connection model makes naive load balancing unreliable at scale.

---

## 6. Docker / ECS Fargate setup — mostly the same, with small but important changes

The Dockerfile pattern from the REST guide barely changes:

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
EXPOSE 50051
CMD ["./server"]
```

**What's different:**
- The exposed port is conventionally `50051` for gRPC (not a hard rule, just a common convention), rather than `8080`.
- The ECS **target group protocol version** must be `GRPC`, as shown earlier — this is set at the AWS infrastructure layer, not in the Dockerfile, but it's a required change from the REST setup.
- If you're using **gRPC reflection** or **gRPC-Web** (to let browser clients call gRPC, since browsers can't natively speak raw HTTP/2 gRPC), you may need an additional proxy layer (e.g., Envoy configured with the grpc-web filter) between browser clients and your gRPC backend — this is a whole extra hop that a REST/JSON API never needed, since REST already works natively with `fetch()` in the browser.

---

## 7. CI/CD pipeline — one real addition: generating code from `.proto` files

With REST, your Go structs and your JSON contract were defined directly in your application code (like the `User` struct in the earlier examples). With gRPC, the contract is defined separately in a `.proto` file, and both the client and server generate code from it:

```proto
syntax = "proto3";

service UserService {
  rpc GetUser (GetUserRequest) returns (User);
  rpc CreateUser (CreateUserRequest) returns (User);
}

message User {
  int32 id = 1;
  string name = 2;
  string email = 3;
}

message GetUserRequest { int32 id = 1; }
message CreateUserRequest { string name = 1; string email = 2; }
```

This means your CI/CD pipeline gets a new step before the build:

```yaml
- name: Generate gRPC code from proto files
  run: |
    protoc --go_out=. --go-grpc_out=. proto/user.proto

- name: Run tests
  run: go test ./...

- name: Build, tag, and push image
  run: |
    docker build -t my-grpc-api .
    # ...same ECR push steps as before
```

**Why this matters operationally:** the `.proto` file becomes the single source of truth for the API contract, shared between whatever languages your client and server are written in (Go server, but maybe a Python or Java client) — this is a meaningful process change, since previously your "contract" with REST clients was often just documentation (like an OpenAPI/Swagger spec) rather than something that directly generates working client code.

---

## 8. What stays exactly the same

It's worth being clear about what *doesn't* change, so this doesn't feel like a total rebuild:

- **Docker multi-stage builds** — identical approach.
- **ECR** — same registry, same push/pull mechanics.
- **ECS Fargate task definitions, secrets management, auto scaling** — essentially unchanged; you're still running containers with CPU/memory limits, pulling secrets from Secrets Manager, and scaling on CPU/request metrics.
- **Route 53 / domain setup** — unchanged; you still point a domain at your load balancer.
- **Monitoring via CloudWatch** — still applies, though you'll want gRPC-aware metrics (e.g., per-RPC-method latency) in addition to generic container CPU/memory.

---

## Summary: REST vs gRPC deployment differences

| Concern | REST | gRPC |
|---|---|---|
| Transport | HTTP/1.1 (usually) | HTTP/2 (required) |
| Message format | JSON (text) | Protobuf (binary) |
| Load balancing | Connection-level LB works fine | Needs request-level LB (Envoy, mesh, or gRPC-aware ALB) |
| Health checks | Plain `GET /health` → `200` | `grpc.health.v1.Health` service, checked via gRPC call |
| TLS | Optional, often terminated at LB | Effectively required; ALPN negotiation matters end-to-end |
| Browser support | Native, via `fetch()` | Needs a grpc-web proxy layer |
| API contract | Often just documentation (OpenAPI) | `.proto` file generates real client/server code |
| ALB config | Standard HTTP target group | Target group protocol version must be `GRPC` |
| Internal service calls | Usually still via LB | Often client-side load balancing or service mesh |
| Docker/ECS basics | — | Unchanged |

**Bottom line:** the container-building and AWS scaffolding from the REST guide barely change. What changes is everything *between* the load balancer and your containers — you need infrastructure that actually understands HTTP/2's connection multiplexing, or you'll get containers with wildly uneven load despite "correct" load balancer configuration.
