# Deploying a REST API — Docker, Load Balancer, and AWS, Step by Step

This note walks through taking the Go REST API from earlier and deploying it properly — containerizing it, putting it behind a load balancer, and running it on AWS. Each step explains *why* it exists, not just *how* to do it.

---

## 0. The big picture first

Before diving into commands, here's the mental model of what we're building:

```
Internet
   │
   ▼
Domain (api.yourapp.com) — DNS
   │
   ▼
Load Balancer (distributes traffic, handles HTTPS)
   │
   ├──▶ Container instance 1 (your Go app)
   ├──▶ Container instance 2 (your Go app)
   └──▶ Container instance 3 (your Go app)
   │
   ▼
Database (separate, persistent, not inside a container that gets replaced)
```

**Why not just run `go run server.go` on one server?**
- If that one server crashes, your API is down — no redundancy.
- If traffic spikes, one server can't handle it — no scaling.
- Every time you deploy an update, you'd have to manually stop/restart the process, causing downtime.
- The load balancer + multiple instances solves all three: redundancy, scaling, and zero-downtime deploys.

---

## 1. Containerize the app with Docker

**Why Docker at all?** Your Go binary might run fine on your laptop, but the production server needs the same OS libraries, environment variables, and file layout to guarantee it behaves identically. Docker packages your app plus everything it needs into one portable image — "it works on my machine" becomes "it works in this exact container, everywhere."

### Dockerfile

```dockerfile
# --- Stage 1: build the Go binary ---
FROM golang:1.22-alpine AS builder

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o server .

# --- Stage 2: run it in a minimal image ---
FROM alpine:3.19

WORKDIR /app
COPY --from=builder /app/server .

EXPOSE 8080
CMD ["./server"]
```

**Why two stages ("multi-stage build")?**
- Stage 1 uses the full `golang` image (huge, has compilers, build tools) just to *compile* the binary.
- Stage 2 copies only the compiled binary into a tiny `alpine` image (a few MB) that has none of the build tooling.
- Result: your final image is small (faster to push/pull/deploy) and has a much smaller attack surface (fewer things that could have security vulnerabilities), since it doesn't ship the Go compiler in production.

### Build and run it locally to test

```bash
docker build -t my-rest-api .
docker run -p 8080:8080 my-rest-api
```

- `docker build -t my-rest-api .` — builds the image from the Dockerfile in the current directory, tagging it `my-rest-api`.
- `docker run -p 8080:8080 my-rest-api` — runs a container from that image, mapping port 8080 on your machine to port 8080 inside the container (`-p host:container`).

At this point you can `curl http://localhost:8080/users` and get the same response you got running it natively — except now it's running the exact same way it will in production.

---

## 2. Push the image to a container registry

A registry is just a storage service for Docker images — your production servers pull the image from here rather than you copying files around manually.

### AWS ECR (Elastic Container Registry)

```bash
# Authenticate Docker to your AWS account's registry
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com

# Create a repository (one-time setup)
aws ecr create-repository --repository-name my-rest-api

# Tag your local image with the registry's address
docker tag my-rest-api:latest 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-rest-api:latest

# Push it
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-rest-api:latest
```

**What's happening:** `docker tag` doesn't copy anything — it just gives your local image an additional name that includes the registry's address, the same way a file can have two names via a symlink. `docker push` then uploads the image layers to ECR so AWS services can pull it from there.

---

## 3. Choose where it runs: ECS Fargate (containers without managing servers)

You have several ways to run containers on AWS. **Fargate** is a good starting point because you don't manage any underlying EC2 servers — you just say "run this container image, with this much CPU/memory," and AWS handles the actual machine.

### Task Definition — describes *how* to run your container

```json
{
  "family": "my-rest-api-task",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "containerDefinitions": [
    {
      "name": "my-rest-api",
      "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/my-rest-api:latest",
      "portMappings": [{ "containerPort": 8080, "protocol": "tcp" }],
      "environment": [
        { "name": "ENV", "value": "production" }
      ],
      "secrets": [
        {
          "name": "DB_PASSWORD",
          "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789012:secret:db-password"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/my-rest-api",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

**Key parts explained:**
- `cpu` / `memory` — how much compute each container instance gets. `256` CPU units = 0.25 vCPU here; small APIs often start here and scale up based on real usage.
- `environment` — plain, non-sensitive config values (like which environment it's running in).
- `secrets` — this is the important one: instead of hardcoding `DB_PASSWORD` in your code or an env var visible in plaintext, it's pulled from **AWS Secrets Manager** at container startup. This is exactly the credential-handling problem from the earlier `Authorization: Bearer secret123` example — in real production, that kind of secret should live in a secrets manager, not in code.
- `logConfiguration` — sends your container's stdout/stderr logs to CloudWatch, so you can actually debug it once it's running remotely (you can't SSH into a Fargate container the way you might into a regular server).

### Service — keeps the desired number of containers running

```bash
aws ecs create-service \
  --cluster my-cluster \
  --service-name my-rest-api-service \
  --task-definition my-rest-api-task \
  --desired-count 3 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-abc,subnet-def],securityGroups=[sg-123],assignPublicIp=DISABLED}" \
  --load-balancers "targetGroupArn=arn:aws:elasticloadbalancing:...,containerName=my-rest-api,containerPort=8080"
```

**What's happening:** a "task" is one running container; a "service" is the thing that keeps `desired-count` (here, 3) of them running at all times. If one crashes or the underlying hardware fails, ECS automatically starts a replacement — this is where your redundancy actually comes from.

---

## 4. Put a load balancer in front of the containers

**Why you need this:** you now have 3 separate running containers, each with their own private IP inside AWS's network. Clients shouldn't need to know about any of them individually or pick which one to call — the load balancer gives you one stable address and spreads incoming requests across all healthy containers.

### Application Load Balancer (ALB) — good fit for HTTP/REST APIs

Conceptually, the ALB setup has three pieces:

1. **Target group** — the pool of containers that can receive traffic.
```bash
aws elbv2 create-target-group \
  --name my-rest-api-targets \
  --protocol HTTP \
  --port 8080 \
  --vpc-id vpc-0abc123 \
  --target-type ip \
  --health-check-path /health
```
- `--health-check-path /health` — the ALB will periodically call this endpoint on each container. If a container stops responding correctly, the ALB automatically stops sending it traffic — this is how a crashed or stuck container gets removed from rotation without you doing anything manually. **You need to add a simple `/health` route to your Go app that just returns `200 OK`** for this to work.

2. **Load balancer** — the actual entry point that receives all incoming traffic.
```bash
aws elbv2 create-load-balancer \
  --name my-rest-api-lb \
  --subnets subnet-abc subnet-def \
  --security-groups sg-456 \
  --scheme internet-facing
```

3. **Listener** — the rule that says "traffic arriving on port 443 (HTTPS) gets forwarded to the target group."
```bash
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:...:loadbalancer/... \
  --protocol HTTPS \
  --port 443 \
  --certificates CertificateArn=arn:aws:acm:...:certificate/... \
  --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:...:targetgroup/...
```
**Why the listener uses HTTPS/443, and cites a certificate:** this is where **TLS termination** happens — the load balancer handles the encryption/decryption for HTTPS, so your Go containers only ever receive plain HTTP internally. This simplifies your app code (no TLS cert management inside the app itself) and centralizes certificate renewal in one place (AWS Certificate Manager, referenced via `CertificateArn`, handles automatic renewal).

**End-to-end request path once this is live:**
```
Client → HTTPS (443) → ALB (decrypts TLS, picks a healthy container) → HTTP (8080) → Container
```

---

## 5. Point a domain at the load balancer

The load balancer gets an auto-generated AWS hostname (like `my-rest-api-lb-123456.us-east-1.elb.amazonaws.com`), but you want `api.yourapp.com` instead.

Using **Route 53** (AWS's DNS service):
```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123456789 \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "api.yourapp.com",
        "Type": "A",
        "AliasTarget": {
          "HostedZoneId": "Z35SXDOTRQ7X7K",
          "DNSName": "my-rest-api-lb-123456.us-east-1.elb.amazonaws.com",
          "EvaluateTargetHealth": true
        }
      }
    }]
  }'
```
**What's happening:** this creates a DNS "alias" record — when someone's browser resolves `api.yourapp.com`, it's redirected to the load balancer's actual address. `EvaluateTargetHealth: true` means Route 53 will stop directing traffic there if the load balancer itself reports as unhealthy.

---

## 6. Secrets and environment configuration

Never bake secrets (DB passwords, API keys, JWT signing keys) directly into your Docker image or source code — anyone who can pull the image can read them.

- **AWS Secrets Manager** or **Parameter Store** hold the actual values.
- The ECS task definition references them (as shown in step 3's `secrets` block) and AWS injects them as environment variables *only inside the running container*, never visible in the image itself or in your Git history.

In your Go code, you'd read them normally:
```go
dbPassword := os.Getenv("DB_PASSWORD")
```
The difference is *where that value came from* — injected securely at runtime by ECS, not hardcoded.

---

## 7. Auto scaling — handling traffic changes automatically

```bash
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --resource-id service/my-cluster/my-rest-api-service \
  --scalable-dimension ecs:service:DesiredCount \
  --min-capacity 2 \
  --max-capacity 10

aws application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --resource-id service/my-cluster/my-rest-api-service \
  --scalable-dimension ecs:service:DesiredCount \
  --policy-name cpu-scaling \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 60.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ECSServiceAverageCPUUtilization"
    }
  }'
```

**What's happening:** this tells AWS to automatically add more running containers (up to 10) when average CPU usage across them goes above 60%, and scale back down (to as few as 2) when traffic drops. The load balancer automatically starts sending traffic to new containers as they come online, with zero manual intervention.

---

## 8. CI/CD — automating the whole pipeline

Doing steps 1–2 manually every time you change code doesn't scale. A CI/CD pipeline automates: test → build image → push to ECR → deploy new version.

### Example: GitHub Actions workflow

```yaml
name: Deploy to ECS

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run tests
        run: go test ./...

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - name: Build, tag, and push image
        run: |
          aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com
          docker build -t my-rest-api .
          docker tag my-rest-api:latest 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-rest-api:latest
          docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-rest-api:latest

      - name: Deploy new version to ECS
        run: |
          aws ecs update-service --cluster my-cluster --service my-rest-api-service --force-new-deployment
```

**What's happening:**
- This runs automatically every time you push to `main`.
- `go test ./...` — the pipeline refuses to deploy broken code; tests must pass first.
- The AWS credentials are stored as encrypted **GitHub Secrets**, not written in the workflow file — same principle as step 6, just at the CI level instead of the runtime level.
- `aws ecs update-service --force-new-deployment` tells ECS to pull the freshly pushed image and gradually replace the old running containers with new ones — this is a **rolling deployment**: ECS starts new containers, waits for them to pass health checks, *then* stops the old ones, so there's no gap in availability.

---

## 9. Monitoring — knowing when something's wrong

Once it's live, you need visibility, since you can't just watch a terminal anymore.

- **CloudWatch Logs** — collects the stdout/stderr from every container (wired up in step 3's `logConfiguration`), so you can search logs across all instances in one place.
- **CloudWatch Alarms** — can page you or trigger automation when, say, the ALB's 5xx error rate crosses a threshold, or CPU stays pinned at 100% for 5 minutes.
- **ALB access logs / target health** — tells you if containers are failing health checks and getting pulled out of rotation.

---

## Putting the whole pipeline together

```
Developer pushes code
        │
        ▼
GitHub Actions: run tests → build Docker image → push to ECR
        │
        ▼
ECS pulls new image → starts new containers → health-checks them
        │
        ▼
ALB starts routing traffic to new containers, stops routing to old ones
        │
        ▼
Old containers are terminated (zero-downtime rolling deploy complete)
        │
        ▼
Auto Scaling watches CPU/traffic → adds/removes containers as needed
        │
        ▼
CloudWatch collects logs + metrics → alarms fire if something's wrong
```

---

## Simpler alternatives, if ECS/ALB feels like a lot

You don't have to start here. Depending on scale and how much control you need:

| Option | What it gives you | Trade-off |
|---|---|---|
| **Fly.io / Render / Railway** | Push code, get a running container with a load balancer and HTTPS automatically | Less fine-grained control, but dramatically less setup |
| **AWS App Runner** | Similar simplicity to the above, but native AWS, connects directly to ECR | Fewer configuration knobs than raw ECS |
| **A single EC2 instance + Docker + Nginx** | Full manual control, cheapest for small/hobby projects | You manage OS patches, scaling, and failover yourself |
| **ECS Fargate + ALB** (this note) | Production-grade scaling, zero-downtime deploys, managed infrastructure | More moving pieces to understand and configure |
| **Kubernetes (EKS)** | Maximum flexibility, industry-standard for large/complex systems | Significant operational complexity — usually overkill until you actually need it |

**Rule of thumb:** start with the simplest option that meets your needs, and move down this table only when you hit an actual limitation (not because it seems more "production-grade" in the abstract).
