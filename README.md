# Backend Architecture Onboarding Notes

A guided set of notes covering REST APIs, deployment, gRPC, and a hybrid REST+protobuf model — built up progressively, then applied to a real case study (`invoice-backend`).

Read these in order. Each one builds on the concepts introduced before it, and later notes assume you've read the earlier ones.

---

## Reading order

### 1. Frontend architecture
| # | Note | What it covers |
|---|---|---|
| 1.1 | [`01-frontend-architecture/mvvm-in-react.md`](01-frontend-architecture/mvvm-in-react.md) | MVVM as an architecture pattern applied to React — advantages, disadvantages, and when it's (and isn't) a good fit. Standalone; not required for the API/deployment track below, but useful frontend architecture context. |

### 2. REST API fundamentals
| # | Note | What it covers |
|---|---|---|
| 2.1 | [`02-rest-fundamentals/rest-api-explained.md`](02-rest-fundamentals/rest-api-explained.md) | The core building blocks of REST: HTTP methods (with a worked Go + React example for every verb — GET/POST/PUT/PATCH/DELETE/OPTIONS), headers, request/response bodies, status codes, auth tokens, CORS, path vs query params, statelessness, and content negotiation. **Start here if you're new to REST.** |

### 3. Deploying a REST API
| # | Note | What it covers |
|---|---|---|
| 3.1 | [`03-rest-deployment/rest-api-deployment-guide.md`](03-rest-deployment/rest-api-deployment-guide.md) | Taking a REST API to production: Docker (multi-stage builds), pushing to a container registry (ECR), running on ECS Fargate, load balancing (ALB — target groups, health checks, TLS termination), DNS (Route 53), secrets management, auto scaling, CI/CD (GitHub Actions), monitoring, and simpler alternatives if this feels like a lot. |

### 4. gRPC — how it differs from REST
| # | Note | What it covers |
|---|---|---|
| 4.1 | [`04-grpc-architecture/01-rest-vs-grpc-and-frontend-impact.md`](04-grpc-architecture/01-rest-vs-grpc-and-frontend-impact.md) | The fundamental differences between REST and gRPC (HTTP/2, protobuf, schema-first contracts), what changes in backend code to move to full gRPC, **why browsers can't call gRPC directly**, what the frontend needs instead (gRPC-Web + a proxy), and — critically — the **hybrid alternative**: keeping REST's shape but swapping JSON for protobuf, with a full backend example. |
| 4.2 | [`04-grpc-architecture/02-grpc-deployment-changes.md`](04-grpc-architecture/02-grpc-deployment-changes.md) | What deployment looks like for **full gRPC** specifically: why connection-level load balancers don't distribute gRPC calls well, gRPC-aware ALB target groups, the gRPC health-checking protocol, mandatory TLS/ALPN, and what stays the same (Docker, ECS, secrets, scaling). |

### 5. The hybrid model (REST + protobuf) — general guidance
| # | Note | What it covers |
|---|---|---|
| 5.1 | [`05-hybrid-model/01-hybrid-backend-changes-detailed.md`](05-hybrid-model/01-hybrid-backend-changes-detailed.md) | A detailed, checklist-style breakdown of every backend code change needed to move from JSON-over-REST to protobuf-over-REST: dependencies, project structure, message/schema definition (including errors), field-numbering discipline, content-type handling, request/response encode-decode changes, testing, and logging/observability adjustments. Ends with a full checklist. |
| 5.2 | [`05-hybrid-model/02-hybrid-protobuf-rest-deployment.md`](05-hybrid-model/02-hybrid-protobuf-rest-deployment.md) | What deployment looks like for the **hybrid model** — and why it's much closer to plain REST deployment (3) than to full gRPC deployment (4.2): no HTTP/2-aware load balancer needed, no Envoy/grpc-web proxy, standard health checks and TLS, one new CI step (proto codegen), and the debugging/schema-evolution trade-offs you do inherit from protobuf. |

### 6. Case study — applying this to a real project (`invoice-backend`)
| # | Note | What it covers |
|---|---|---|
| 6.1 | [`06-case-study-invoice-backend/01-hybrid-migration-plan.md`](06-case-study-invoice-backend/01-hybrid-migration-plan.md) | A concrete, file-by-file, step-by-step plan for migrating a real Go microservices project (gateway + two internal gRPC services) to the hybrid model — frontend calling REST with protobuf bodies, while the gateway keeps talking gRPC internally to its backend services. References actual file paths, functions, and existing dependencies in the codebase. |
| 6.2 | [`06-case-study-invoice-backend/02-hybrid-deployment-differences.md`](06-case-study-invoice-backend/02-hybrid-deployment-differences.md) | What deployment changes (spoiler: almost nothing) once that migration ships — walked through against the project's actual `docker-compose.yml` and Dockerfiles, including a nuance specific to this project: proto codegen already happens externally, so there's no new CI step at all. |

---

## Suggested reading paths

- **New to REST APIs entirely?** Read 2 → 3 in order before anything else.
- **Evaluating whether to adopt gRPC?** Read 2 → 4.1 → 4.2, then decide between full gRPC and the hybrid model.
- **Already decided on the hybrid model?** Read 4.1 (for the "why") → 5.1 → 5.2.
- **Want to see it applied to a real codebase?** Read 5.1 → 5.2 → 6.1 → 6.2 as a complete worked example.
- **Just want the frontend architecture note?** 1 is standalone — read it any time.

---

## How this repo was put together

These notes were developed progressively in conversation, moving from foundational concepts (what is a REST call, how do you deploy one) through comparative architecture decisions (REST vs gRPC, and a hybrid middle ground) to a concrete application against a real private codebase. Later notes assume the concepts from earlier ones — if something references "the earlier REST deployment guide" or "the general hybrid deployment note," it's referring to an earlier file in this same reading order.
