# Production Deployment Guidelines: Invoice & Receipt Generation Platform

**Document Version:** 2.0.0 (Architecture & Implementation Guidelines)  
**Target Markets:** Nepal 🇳🇵, Thailand 🇹🇭, India 🇮🇳  
**Architecture:** React/Vite SPA (Frontend) + Go REST Gateway + Go gRPC Microservices + Managed PostgreSQL (AWS RDS)

---

## 1. Executive Summary & Architecture Strategy

### 1.1 Core Objectives
This guide outlines the production deployment architecture, security standards, and operational guidelines for the **Invoice & Receipt Generation Platform**. It is structured to ensure high availability (99.9% SLA), sub-50ms regional latency across Nepal, India, and Thailand, automated zero-downtime releases, and robust financial audit compliance.

### 1.2 Multi-Tier System Topology
The system is divided into four cleanly isolated layers:

1. **Frontend Distribution Tier (`invoice-frontend`):**
   * **Hosting Platform:** Cloudflare Pages (Global Edge Network).
   * **Mechanism:** The React/Vite application is built into static HTML, JavaScript, and CSS bundles.
   * **Key Advantage:** Static assets are served from local edge Points of Presence (Kathmandu, Bangkok, Mumbai, Delhi) with zero server load on our backend compute.

2. **Edge Security & Routing Tier:**
   * **Provider:** Cloudflare Anycast DNS & Web Application Firewall (WAF).
   * **Responsibilities:** Full Strict TLS 1.3 encryption, automatic DDoS protection, Bot Fight Mode, and secure reverse proxy routing to `api.yourdomain.com`.

3. **Backend Compute Tier (`receipt-generator`):**
   * **Compute Infrastructure:** AWS Lightsail or EC2 instance (2 vCPU / 4 GB RAM / Ubuntu 24.04 LTS) located in the **AWS Mumbai (`ap-south-1`)** region.
   * **Ingress Controller (Traefik v3):** Manages automated SSL certificate provisioning via Let's Encrypt, security headers (HSTS, CSP), and ingress rate limiting (100 req/s).
   * **API Gateway (`gateway`):** Serves public HTTP/JSON traffic on port 8080, validates incoming requests, attaches tracking correlation IDs (`X-Correlation-ID`), and routes calls internally to microservices.
   * **Internal Microservices (`company-service` & `invoice-service`):** High-speed gRPC services communicating over an isolated, internal-only Docker network with no direct exposure to the public Internet.

4. **Persistence & Disaster Recovery Tier:**
   * **Database Engine:** Managed PostgreSQL 16 on **AWS RDS** located within a private VPC.
   * **Connection Strategy:** Persistent connection pooling using `pgxpool`.
   * **Disaster Recovery:** Continuous Write-Ahead Logging (WAL) with a 7-day Point-in-Time Recovery (PITR) window and weekly encrypted cold backups archived to AWS S3/Glacier for 7-year regulatory compliance.

---

## 2. Infrastructure & Cloud Resources Breakdown

### 2.1 Cloud Resource Allocation

| Cloud Component | Target Service | Sizing & Region | Architectural Purpose |
| :--- | :--- | :--- | :--- |
| **Compute Host** | AWS Lightsail / EC2 (`t4g.medium`) | 2 vCPU, 4 GB RAM<br>`ap-south-1` (Mumbai) | Hosts all application Docker containers and ingress reverse proxy. |
| **Relational Database** | AWS RDS PostgreSQL 16 | `db.t4g.micro` / `db.t4g.small`<br>20–50 GB NVMe SSD | Secure, managed persistence layer with strict ACID guarantees. |
| **Disaster Recovery** | AWS RDS Automated Backups | 7-Day Continuous WAL | Enables point-in-time database restoration to any second within 7 days. |
| **Cold Audit Archive** | AWS S3 Standard to S3 Glacier | Encrypted S3 Bucket | 7-year immutable storage for weekly database dumps (`pg_dump`). |
| **Edge & Static Hosting**| Cloudflare Pages & Anycast DNS | Global Edge PoPs | Global static asset delivery, Anycast DNS resolution (< 5ms), and WAF. |
| **Container Registry** | GitHub Container Registry (GHCR) | `ghcr.io` | Versioned storage of SHA-tagged production Docker container images. |
| **CI/CD Automation** | GitHub Actions | Ubuntu Hosted Runners | Automated linting, race-condition testing, image builds, and SSH deployments. |
| **Synthetic Monitoring**| Better Stack / UptimeRobot | Global Ping Probes | 60-second synthetic health probe monitoring on `api.yourdomain.com/healthz`. |

---

## 3. Host Provisioning & Security Hardening Guidelines

When setting up the production Linux host on AWS, follow these standard operational guidelines:

### 3.1 Operating System & Package Management
* Provision **Ubuntu 24.04 LTS (x86_64 or ARM64/Graviton)**.
* Enable automated unattended security upgrades (`unattended-upgrades`) so critical OS patches are applied without manual intervention.
* Install Docker Engine and the Docker Compose plugin following official Docker repository guidelines.

### 3.2 Network Firewall Guidelines (UFW)
* Apply a **Default Deny** policy for all incoming traffic.
* Allow only the minimum required ports:
  * **Port 22 (SSH):** Restricted to key-based authentication only (disable password logins).
  * **Port 80 (HTTP):** Required for Let's Encrypt ACME HTTP challenge validation.
  * **Port 443 (HTTPS):** Production TLS traffic routed to Traefik.
* Keep all backend microservice ports (8080, 50051, 50052) and database ports (5432) closed to the public Internet.

### 3.3 Linux Kernel Socket Tuning Guidelines
* Optimize the host kernel (`/etc/sysctl.d/99-network.conf`) to support high concurrent connection volumes:
  * Increase the maximum system file descriptor limit (`fs.file-max`).
  * Enable TCP TIME_WAIT socket reuse (`net.ipv4.tcp_tw_reuse`).
  * Increase the TCP listen backlog queue (`somaxconn` and `tcp_max_syn_backlog`).
  * Enable the BBR congestion control algorithm for reduced network latency.

---

## 4. Container Orchestration & Ingress Guidelines

### 4.1 Ingress Proxy Guidelines (Traefik v3)
* **TLS Automation:** Configure Traefik to request and auto-renew TLS 1.3 certificates via Let's Encrypt ACME.
* **HTTP to HTTPS Redirection:** Automatically redirect all plaintext HTTP (port 80) traffic to encrypted HTTPS (port 443).
* **Security Headers:** Enforce strict HTTP security headers:
  * `Strict-Transport-Security` (HSTS with 1-year duration and preload).
  * `X-Content-Type-Options: nosniff`.
  * `X-Frame-Options: SAMEORIGIN`.
  * `X-XSS-Protection: 1; mode=block`.
* **Rate Limiting:** Enforce a global baseline rate limit of 100 requests/second with a burst allowance of 50 requests to prevent abuse.

### 4.2 Network Isolation Guidelines
* **Public Bridge Network (`public-net`):** Connects Traefik and the API Gateway.
* **Internal Backend Network (`backend-net`):** An internal-only bridge network connecting the API Gateway, `company-service`, and `invoice-service`.
* **Zero External Ports for Core Services:** The gRPC microservices must never expose ports to the host OS. They are accessible solely within the internal Docker network.

### 4.3 Container Resource Allocation Guidelines
To prevent any single service from exhausting host memory or CPU:
* **API Gateway:** Limit to 1.0 CPU and 512 MB RAM (Reserve 0.15 CPU, 128 MB RAM).
* **Company Microservice:** Limit to 0.75 CPU and 512 MB RAM (Reserve 0.10 CPU, 64 MB RAM).
* **Invoice Microservice:** Limit to 0.75 CPU and 512 MB RAM (Reserve 0.10 CPU, 64 MB RAM).
* **Traefik Proxy:** Limit to 0.50 CPU and 256 MB RAM.
* **Log Rotation:** Configure Docker `json-file` log driver on all containers with a max file size of 50 MB and a 5-file retention limit to prevent disk exhaustion.

---

## 5. Configuration & Secrets Management Guidelines

### 5.1 Twelve-Factor Configuration Principles
* No credentials, database connection strings, or API tokens may be stored in source code.
* All configuration must be injected dynamically via environment variables on the host.

### 5.2 Required Production Environment Variables
* `IMAGE_PREFIX`: Path to the GitHub Container Registry repository.
* `IMAGE_TAG`: Git commit SHA deployed by the CI/CD pipeline.
* `API_DOMAIN`: Public domain name for the API (e.g., `api.yourdomain.com`).
* `ACME_EMAIL`: Administrator email for Let's Encrypt certificate expiry notices.
* `ALLOWED_ORIGINS`: Comma-separated list of allowed web frontend domains for CORS.
* `MANAGED_DATABASE_URL`: Full AWS RDS PostgreSQL connection string with SSL mode enforced (`sslmode=require`).
* `ENVIRONMENT`: Set to `production`.
* `LOG_LEVEL`: Set to `info`.

---

## 6. Continuous Integration & Delivery (CI/CD) Guidelines

### 6.1 Backend Pipeline Lifecycle
The backend pipeline triggers automatically whenever code is pushed to the `main` branch:

1. **Stage 1: Quality Gate & Automated Testing:**
   * Checks out code and provisions Go 1.22.
   * Runs the entire test suite using the Go race detector (`go test -race`).
   * Fails the build immediately if any test fails or data races are detected.

2. **Stage 2: Container Image Build & Push:**
   * Builds production Docker images for `gateway`, `company-service`, and `invoice-service`.
   * Tags images with both the unique Git commit SHA (`${{ github.sha }}`) and `latest`.
   * Pushes the images securely to GitHub Container Registry (`ghcr.io`).

3. **Stage 3: Zero-Downtime Production Deployment:**
   * Connects to the AWS compute host via secure SSH.
   * Pulls the newly built container images by SHA tag.
   * Executes database migrations in an isolated ephemeral container using `golang-migrate`.
   * Performs a rolling update of the service containers using Docker Compose.
   * Executes a post-deployment health check against `https://api.yourdomain.com/healthz`.
   * Automatically rolls back if the health check fails to return HTTP 200 within 15 seconds.
   * Prunes stale Docker image layers older than 48 hours to preserve disk space.

### 6.2 Frontend Pipeline Lifecycle
The frontend pipeline triggers upon changes to `invoice-frontend/**`:

1. **Dependencies & Build:**
   * Installs clean npm dependencies (`npm ci`).
   * Compiles the React SPA using Vite, embedding the production API URL.
2. **Edge Deployment:**
   * Publishes the compiled `dist/` directory directly to Cloudflare Pages via Cloudflare Wrangler.
   * Changes propagate instantly across all Anycast edge locations without origin downtime.

---

## 7. Database Connection & Lifecycle Guidelines

### 7.1 Connection Pooling Guidelines (`pgxpool`)
To ensure optimal performance and avoid database connection exhaustion:
* **Max Connections (`MaxConns`):** Cap at 25 connections per microservice instance.
* **Min Connections (`MinConns`):** Maintain a baseline of 5 idle connections for instant query execution.
* **Idle Connection Pruning (`MaxConnIdleTime`):** Close idle connections after 15 minutes to release database memory.
* **Connection Lifetime (`MaxConnLifetime`):** Recycle connections every 1 hour to prevent memory leaks and handle DNS changes.
* **Health Checks (`HealthCheckPeriod`):** Verify connection vitality every 1 minute.

### 7.2 Database Schema Migration Guidelines
* All database changes must be written as reversible, versioned SQL migration files (`up.sql` and `down.sql`).
* Migrations are executed automatically as a pre-deployment step before new application containers are started.
* Breaking changes must follow the **Expand and Contract pattern** (add new columns as nullable first; deprecate old columns in a subsequent release).

---

## 8. Operational Runbooks & Disaster Recovery Guidelines

### 8.1 Instant Rollback Procedure (< 60 Seconds)
If an unforeseen bug is identified immediately following a production deployment:
1. SSH into the AWS compute host.
2. Update the `IMAGE_TAG` environment variable to the previous known stable Git commit SHA.
3. Execute a rolling restart with Docker Compose (`docker compose up -d`).
4. Verify application health on `/healthz`.

### 8.2 Point-in-Time Database Restoration (PITR) Procedure
If accidental data corruption occurs:
1. Open the AWS RDS Management Console.
2. Select the PostgreSQL database instance and choose **Restore to Point in Time**.
3. Select the exact timestamp immediately preceding the incident.
4. Launch the temporary restored instance.
5. Update the `MANAGED_DATABASE_URL` configuration to point to the restored database.
6. Restart backend microservices and verify data integrity.

### 8.3 Service Monitoring & Alerting Matrix
Configure external synthetic monitoring with the following threshold guidelines:

| Metric / Trigger | Target Threshold | Action / Severity |
| :--- | :--- | :--- |
| **API Health Probe** | Status != 200 on `/healthz` for > 60s | **P1 Critical:** Page on-call engineer, trigger automated alerts. |
| **High API Latency** | Gateway p95 response time > 500ms for 5m | **P2 Warning:** Inspect database slow queries and connection pool wait times. |
| **HTTP 5xx Error Spike** | Error rate > 2% of total requests over 3m | **P1 Critical:** Check microservice container logs and database connectivity. |
| **Host Disk Capacity** | Disk usage > 80% on NVMe storage | **P2 Warning:** Execute Docker image and container pruning. |
| **Memory Pressure** | RAM usage > 85% for > 5m | **P2 Warning:** Inspect service memory footprints and query caches. |

---

## 9. Summary & Architecture Approval

This deployment architecture provides a robust, highly reliable, and cost-effective production platform (~$38–$45/month). It eliminates unnecessary infrastructure complexity while maintaining strict enterprise standards for uptime, security, data integrity, and regional performance.
