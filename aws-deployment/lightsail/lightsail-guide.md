# 🚀 Complete AWS Lightsail, Docker & GitHub Actions CI/CD Production Handbook

> **Project:** Invoice & Receipt Generator Microservices Backend  
> **Repository:** `github.com/SV-TechnologiesPvtLtd/invoice-backend`  
> **Production Host:** AWS Lightsail (Ubuntu Linux)  
> **Container Registry:** GitHub Container Registry (`ghcr.io`)  
> **Bot Authentication:** GitHub App (`sv-tech-ci-bot`)  

---

## 📖 Table of Contents
1. [What is AWS Lightsail?](#1-what-is-aws-lightsail)
2. [Why AWS Lightsail Instead of AWS EC2?](#2-why-aws-lightsail-instead-of-aws-ec2)
3. [Why GitHub Container Registry (GHCR) is the Industry Standard](#3-why-github-container-registry-ghcr-is-the-industry-standard)
4. [GitHub App Bot Setup (`sv-tech-ci-bot`)](#4-github-app-bot-setup-sv-tech-ci-bot)
5. [GitHub Repository Secrets Reference Matrix](#5-github-repository-secrets-reference-matrix)
6. [Lightsail Instance Creation: Choices & Rationale](#6-lightsail-instance-creation-choices--rationale)
7. [Complete Terminal Commands Executed on Lightsail](#7-complete-terminal-commands-executed-on-lightsail)
8. [Connecting via SSH from Local Machine (Windows & Mac)](#8-connecting-via-ssh-from-local-machine-windows--mac)
9. [What is GitHub Actions Cache & Why We Use It?](#9-what-is-github-actions-cache--why-we-use-it)
10. [Swagger Dynamic Host Resolution](#10-swagger-dynamic-host-resolution)
11. [Day-to-Day Lifecycle: What Happens When You Push Code](#11-day-to-day-lifecycle-what-happens-when-you-push-code)
12. [Useful Maintenance Commands & Troubleshooting](#12-useful-maintenance-commands--troubleshooting)
13. [Nginx Reverse Proxy & Subdomain Setup (`invoice-backend.svtechnologies.com.np`)](#13-nginx-reverse-proxy--subdomain-setup-invoice-backendsvtechnologiescomnp)

---

## 1. What is AWS Lightsail?

**AWS Lightsail** is Amazon's simplified, developer-friendly cloud platform designed for launching virtual private servers (VPS), databases, and container workloads with flat, predictable pricing.

* Unlike standard AWS (which requires configuring complex VPCs, Internet Gateways, Subnets, and IAM policies), Lightsail bundles:
  * **Compute (vCPU & RAM)**
  * **SSD Storage (NVMe/EBS)**
  * **Generous Outbound Data Transfer (1 TB – 3 TB free monthly bandwidth)**
  * **Static IP Address & DNS Management**
  * **Built-in Firewall**
into a single flat monthly rate ($3.50 – $10/month).

---

## 2. Why AWS Lightsail Instead of AWS EC2?

| Feature | **AWS Lightsail** (Our Choice) | **AWS EC2** |
| :--- | :--- | :--- |
| **Pricing Predictability** | **Flat, predictable monthly cost** ($5 or $10/mo). Bandwidth is bundled (1 TB – 3 TB free transfer). | Variable and complex. You pay separately for CPU, disk IOPS, and data egress ($0.09/GB adds up fast). |
| **Setup Complexity** | **Zero networking overhead.** Pre-configured firewall, public IP, and network interface out of the box. | Requires manual VPC, Subnet, Route Tables, Internet Gateway, and Security Group configuration. |
| **Microservice Architecture Fit** | Perfect for running Docker Compose with 3–5 microservices + PostgreSQL on a single node. | Ideal when scaling to multi-AZ Kubernetes (EKS) or auto-scaling fleets. Overkill for single-node VPS. |
| **Maintenance** | Simple web dashboard with reboot, firewall, and snapshot features. | Heavy AWS Console with 100+ nested settings. |

---

## 3. Why GitHub Container Registry (GHCR) is the Industry Standard

We publish our pre-compiled Docker images to **`ghcr.io`** (GitHub Container Registry).

* **Zero Credential Sprawl:** GitHub Actions automatically injects a short-lived `${{ secrets.GITHUB_TOKEN }}` to authenticate with GHCR. No third-party passwords or external API tokens are needed.
* **No Pull Rate Limits:** Unlike Docker Hub (which aggressively throttles free accounts to 100–200 pulls / 6 hours and breaks CI/CD), GHCR has no restrictive pull rate limits for authorized workflows.
* **100% Provenance & Auditability:** Every Docker image tag (e.g. `:latest` and `:<commit-sha>`) is directly linked to the exact Git commit and Pull Request in the GitHub UI.
* **Used by Top Projects:** Homebrew, LinuxServer.io, Caddy Web Server, Grafana, and Microsoft distribute their official containers via `ghcr.io`.

---

## 4. GitHub App Bot Setup (`sv-tech-ci-bot`)

Because our backend imports proto definitions from a separate private repository (`SV-TechnologiesPvtLtd/repo-proto`), we created an **organization-level GitHub App** (`sv-tech-ci-bot`).

### Why a GitHub App Instead of Personal Access Tokens (PATs)?
* **Never Expires:** Personal Access Tokens expire every 30–90 days and break CI/CD pipelines. A GitHub App uses cryptographic `.pem` private keys that never expire.
* **Not Tied to an Employee:** If an engineer leaves the company, personal tokens get revoked; a GitHub App belongs to the organization and continues running smoothly.

### How the GitHub App was Configured:
1. **GitHub Organization Settings** ➔ **Developer settings** ➔ **GitHub Apps** ➔ **New GitHub App**.
2. **App Name:** `sv-tech-ci-bot`
3. **Homepage URL:** `https://github.com/SV-TechnologiesPvtLtd`
4. **Webhook:** ❌ **Unchecked** (Not active).
5. **Repository Permissions:**
   * **Contents:** `Access: Read & write` *(Allows pulling `repo-proto` source code)*.
   * **Packages:** `Access: Read & write` *(Allows publishing container images to GHCR)*.
   * **Metadata:** `Access: Read-only` *(Automatically set by GitHub)*.
   * *(All other permissions: No access)*.
6. **Installation:** Installed on `@SV-TechnologiesPvtLtd` with access to **All repositories**.
7. **Private Key:** Generated and downloaded private key `.pem` file.

---

## 5. GitHub Repository Secrets Reference Matrix

Under **`receipt-generator` ➔ Settings ➔ Secrets and variables ➔ Actions**, these secrets must be present:

| Secret Name | Value Description | Example / Source |
| :--- | :--- | :--- |
| `INVOICE_BACKEND_APP_ID` | Numeric App ID of your GitHub App | `1234567` (from GitHub App page header) |
| `INVOICE_BACKEND_APP_PRIVATE_KEY` | Entire content of the `.pem` file | `-----BEGIN RSA PRIVATE KEY...` |
| `LIGHTSAIL_HOST` | Static Public IP of your Lightsail VM | `43.204.88.188` |
| `LIGHTSAIL_USER` | Default Linux SSH user | `ubuntu` |
| `LIGHTSAIL_SSH_KEY` | Private SSH key matching authorized_keys | `-----BEGIN OPENSSH PRIVATE KEY...` |
| `LIGHTSAIL_SSH_PORT` | Standard SSH port | `22` |

---

## 6. Lightsail Instance Creation: Choices & Rationale

### 1. Instance Location (Region)
* **Choice:** Closest geographic region to your primary user base (e.g. `ap-south-1` Mumbai or `ap-southeast-1` Singapore).
* **Reason:** Minimizes network latency (ping time) for REST API requests.

### 2. Platform & Blueprint
* **Platform:** `Linux/Unix`
* **Blueprint:** `OS Only` ➔ **`Ubuntu 24.04 LTS` (or 22.04 LTS)**
* **Reason:** 
  * "App + OS" blueprints (like Bitnami WordPress) install unnecessary web servers and background daemons that consume memory.
  * An **"OS Only" Ubuntu LTS** provides a clean, minimal Linux environment with long-term security updates.

### 3. Instance Plan (Size)
* **Choice:** **2 GB RAM, 1 vCPU, 60 GB SSD** ($10/mo) or **1 GB RAM, 1 vCPU, 40 GB SSD** ($5/mo).
* **Reason:**
  * Go binaries are lightweight and pre-compiled outside the server in GitHub Actions cloud runners.
  * 1 GB – 2 GB RAM comfortably runs PostgreSQL + 3 Go microservices with low memory footprints (<30 MB each).

### 4. Networking & Static IP
* **Choice:** Attached an AWS **Static IP** (`43.204.88.188`).
* **Reason:** Default public IPs change on instance reboot. A **Static IP** guarantees that your DNS records and GitHub Actions deploy scripts remain permanent.

### 5. Firewall Configuration
In the Lightsail **Networking** tab, the following IPv4 firewall rules were configured:

| Protocol | Port Range | Purpose |
| :--- | :--- | :--- |
| **SSH** | `22` | Secure remote management and GitHub Actions automated deployment. |
| **HTTP** | `80` | Standard web traffic and Let's Encrypt SSL certificate issuance. |
| **HTTPS** | `443` | Encrypted SSL API traffic. |
| **Custom TCP** | `8080` | Direct public access to the REST API Gateway. |

---

## 7. Complete Terminal Commands Executed on Lightsail

Below is the complete ledger of setup commands executed on the server:

### 1. Update OS & Install Docker Engine + Compose v2
```bash
# Update Ubuntu package repositories and install Docker + Compose plugin
sudo apt update && sudo apt upgrade -y
sudo apt install -y docker.io docker-compose-v2 curl git

# Add the default 'ubuntu' user to the docker group so sudo is not needed
sudo usermod -aG docker ubuntu
newgrp docker
```

### 2. Configure SSH Authentication for GitHub Actions
```bash
# Ensure strict permissions on SSH folder
mkdir -p ~/.ssh
chmod 700 ~/.ssh

# Add the deployment public key to authorized_keys
echo "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIIC7dVWpkTBBbRxlCH6hbHXHmiNjyqIz2AKrL5Qs4sl9 lightsail-invoice-backend" >> ~/.ssh/authorized_keys

# Lock down permissions (mandatory for OpenSSH security)
chmod 600 ~/.ssh/authorized_keys
```

### 3. Configure 2 GB Swap Memory & Swappiness
To prevent Out-Of-Memory (OOM) kernel freezes or SSH disconnection drops (status 255) during concurrent Docker layer extractions:
```bash
# 1. Allocate and enable 2GB swap file
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# 2. Make swap permanent across reboots
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# 3. Optimize swappiness (prioritizes fast RAM and uses swap only during spikes)
sudo sysctl vm.swappiness=10
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf
```

### 4. Create Backend Workspace & Production Environment Variables
```bash
# Create dedicated directory for the backend
mkdir -p /home/ubuntu/invoice-generator-backend

# Create production .env file
cat << 'EOF' > /home/ubuntu/invoice-generator-backend/.env
ENVIRONMENT=production

# Database Settings
POSTGRES_DB=invoice_generator_db
POSTGRES_USER=invoice_generator_admin
POSTGRES_PASSWORD=!zv0ic3pass@2026
POSTGRES_PORT=5432

# Security & CORS
JWT_SECRET=e9a4f2c83b1d7a0e5f6c8d2a4b9e1f3c7d0a8b2c5e6f4a1b9c8d2e5f0a3b6c9d
ALLOWED_ORIGINS=http://localhost:3000,http://localhost:5173
EOF

# Lock down .env file permissions (readable only by owner)
chmod 600 /home/ubuntu/invoice-generator-backend/.env
```

---

## 8. Connecting via SSH from Local Machine (Windows & Mac)

### On Windows (Command Prompt / PowerShell):

1. **Fix Windows File Permissions on `.pem` key (Avoids `UNPROTECTED PRIVATE KEY FILE` error):**
   * **In Command Prompt (CMD):**
     ```cmd
     icacls "C:\path\to\your-key.pem" /inheritance:r
     icacls "C:\path\to\your-key.pem" /grant:r "%USERNAME%:(R)"
     ```
   * **In PowerShell:**
     ```powershell
     icacls "C:\path\to\your-key.pem" /inheritance:r
     icacls "C:\path\to\your-key.pem" /grant:r "$($env:USERNAME):(R)"
     ```

2. **Connect via SSH:**
   ```bash
   ssh -i "C:\path\to\your-key.pem" ubuntu@43.204.88.188
   ```

### On macOS / Linux:
```bash
chmod 400 ~/path/to/your-key.pem
ssh -i ~/path/to/your-key.pem ubuntu@43.204.88.188
```

---

## 9. What is GitHub Actions Cache & Why We Use It?

In our CI/CD pipeline (`.github/workflows/deploy.yml`), we configured:
```yaml
cache-from: type=gha
cache-to: type=gha,mode=max
```

* **`type=gha`** stands for **GitHub Actions Cache**.
* Without cache, every push triggers a full rebuild from scratch: downloading the Alpine base image, installing `git`, and downloading all Go dependencies over the internet (takes **3–5 minutes per build**).
* With cache enabled:
  * Docker stores pre-built image layers directly in GitHub's high-speed cloud cache.
  * On subsequent pushes, unchanged layers are fetched in **milliseconds**.
  * Docker **only compiles the specific Go files you modified**, dropping build times to **under 45 seconds**.

---

## 10. Swagger Dynamic Host Resolution

### The Issue We Solved:
Initially, `localhost:8080` was hardcoded in `cmd/main.go` (`@host localhost:8080`), `docs.go`, and `swagger.json`. This caused Swagger UI in the browser to send API test requests to `http://localhost:8080` instead of `http://43.204.88.188:8080`.

### The Resolution:
1. Removed `// @host localhost:8080` from `services/gateway/cmd/main.go`.
2. Configured dynamic host resolution in `services/gateway/internal/app/app.go`:
   ```go
   docs.SwaggerInfo.Host = ""
   ```
3. Set `Host: ""` in `services/gateway/docs/docs.go` and removed `host` from `swagger.json`.

**Result:** Swagger UI now uses **relative host resolution** (`window.location.host`), automatically targeting `http://43.204.88.188:8080` in production, `http://localhost:8080` during local dev, and any future custom domain seamlessly.

---

## 11. Day-to-Day Lifecycle: What Happens When You Push Code

Whenever you make changes to a Go file, update a model, or tweak a Dockerfile and run:
```bash
git push origin <branch>
```

The automated pipeline executes in two seamless phases:

```mermaid
sequenceDiagram
    autonumber
    actor Dev as Developer
    participant Git as GitHub (Repo)
    participant CI as GitHub Actions Runner
    participant Bot as GitHub App (sv-tech-ci-bot)
    participant GHCR as GitHub Container Registry (ghcr.io)
    participant LS as AWS Lightsail Server

    Dev->>Git: git push origin <branch>
    Git->>CI: Trigger deploy.yml workflow
    CI->>Bot: Request short-lived installation token
    Bot-->>CI: Issue 1-hour secure token
    
    rect rgb(235, 248, 255)
        Note over CI,GHCR: Continuous Integration (CI)
        CI->>CI: Mount token in memory (BuildKit)
        CI->>CI: Fetch private repo-proto & compile Go binary
        CI->>GHCR: Push images with :latest & :<commit-sha>
    end

    rect rgb(240, 255, 244)
        Note over CI,LS: Continuous Deployment (CD)
        CI->>LS: Connect via SSH (LIGHTSAIL_SSH_KEY)
        CI->>LS: Sync docker-compose.prod.yml & SQL migrations via SCP
        CI->>LS: Authenticate Docker on server with GHCR
        LS->>GHCR: docker compose pull (Only downloads updated layer diffs)
        LS->>LS: docker compose up -d (Zero-downtime rolling restart)
        LS->>LS: docker image prune (Remove stale layers)
    end

    LS-->>Dev: Live API updated at http://43.204.88.188:8080/swagger/index.html
```

### Safety & Resilience Highlights:
1. **Zero Server Building:** Lightsail never builds Go code, never compiles binaries, and never runs out of CPU/RAM during deployment.
2. **Persistent Data:** PostgreSQL data is saved inside Docker volume `postgres_data`. Code deployments **never wipe database tables or records**.
3. **Instant Rollback:** Every deployment is tagged with the exact Git commit SHA (`:<commit-sha>`). If an issue occurs, roll back instantly by running:
   ```bash
   IMAGE_TAG=<previous-sha> docker compose -f docker-compose.prod.yml up -d
   ```

---

## 12. Useful Maintenance Commands & Troubleshooting

### Inspecting Live Containers on Lightsail:
```bash
# Check running containers and ports
docker ps

# Check status of compose services
cd /home/ubuntu/invoice-generator-backend
docker compose -f docker-compose.prod.yml ps

# View live streaming logs from all services
docker compose -f docker-compose.prod.yml logs -f

# View live logs of only the API Gateway
docker compose -f docker-compose.prod.yml logs -f gateway
```

### Accessing PostgreSQL Database CLI on Lightsail:
```bash
# Open interactive PostgreSQL shell
docker exec -it invoice_generator_db psql -U invoice_generator_admin -d invoice_generator_db

# Common psql commands:
#   \dt          -> List all tables
#   \d <table>   -> Inspect table schema & columns
#   SELECT * FROM company_details;
#   SELECT * FROM invoices;
#   \q           -> Exit psql shell
```

### Checking Real-Time CPU & RAM Usage:
```bash
docker stats
```

---

## 13. Nginx Reverse Proxy & Subdomain Setup (`invoice-backend.svtechnologies.com.np`)

To expose the Dockerized API Gateway securely over standard HTTPS (Port 443) using the official company subdomain `invoice-backend.svtechnologies.com.np`, we configured an **Nginx Reverse Proxy** on Lightsail backed by a **Cloudflare Origin SSL Certificate** with **Orange Cloud (Proxied)** protection.

```
[ Client Browser / Frontend ]
             │ (HTTPS: 443)
             ▼
[ Cloudflare Edge Network (DDoS Protection, WAF, Edge SSL) ]
             │ (Encrypted HTTPS: 443 via Cloudflare Origin CA)
             ▼
[ AWS Lightsail: Nginx Reverse Proxy ]
             │ (Local HTTP: 127.0.0.1:8080)
             ▼
[ Docker Container: api_gateway ]
```

---

### 13.1 Cloudflare DNS Subdomain Creation
In the Cloudflare DNS dashboard for `svtechnologies.com.np`:
* **Type:** `A`
* **Name:** `invoice-backend`
* **IPv4 Address:** `43.204.88.188` (Lightsail Static IP)
* **Proxy Status:** 🟠 **Proxied (Orange Cloud)**
* **TTL:** `Auto`

**Live Subdomain:** `https://invoice-backend.svtechnologies.com.np`

---

### 13.2 Cloudflare Origin SSL Certificate (15-Year Validity)
Rather than using Certbot (which requires 90-day renewals, Let's Encrypt rate limits, and temporary bypass of Cloudflare proxying), we use a **Cloudflare Origin CA Certificate**:
* **Validity:** 15 Years (Zero renewal maintenance).
* **Security:** End-to-end encryption between Cloudflare edge servers and Lightsail Nginx.

#### 1. Generate in Cloudflare Dashboard:
1. Navigate to **SSL/TLS** ➔ **Origin Server**.
2. Click **Create Certificate**.
3. Select **RSA (2048)**, validity **15 years**, and click **Create**.
4. Cloudflare outputs two text blocks:
   * **Origin Certificate**
   * **Private Key**

#### 2. Install on Lightsail Server:
Save the certificate and private key to standard SSL paths on the server:

```bash
# Save Origin Certificate
sudo nano /etc/ssl/certs/cloudflare_origin.pem

# Save Private Key
sudo nano /etc/ssl/private/cloudflare_origin.key

# Lock down private key permissions (root read-only)
sudo chmod 600 /etc/ssl/private/cloudflare_origin.key
```

#### ⚠️ Critical PEM Format Gotcha & Automated Fix:
When pasting certificates through terminal emulators (especially from Windows), hidden Windows carriage returns (`\r\n`) or blank lines are often introduced. OpenSSL strictly forbids blank lines inside PEM boundaries (RFC 7468), triggering:
`PEM routines::bad end line` or `Could not read certificate`.

**Run this cleanup command to ensure clean formatting:**
```bash
# 1. Remove all internal blank lines
sudo sed -i '/^$/d' /etc/ssl/certs/cloudflare_origin.pem
sudo sed -i '/^$/d' /etc/ssl/private/cloudflare_origin.key

# 2. Ensure a proper final newline at the end of each file
sudo sed -i -e '$a\' /etc/ssl/certs/cloudflare_origin.pem
sudo sed -i -e '$a\' /etc/ssl/private/cloudflare_origin.key

# 3. Verify certificate validity directly with OpenSSL
openssl x509 -in /etc/ssl/certs/cloudflare_origin.pem -text -noout
```
*(If valid, OpenSSL displays `Issuer: C = US, O = "Cloudflare, Inc."` and certificate details)*.

---

### 13.3 Nginx Installation & Reverse Proxy Configuration

#### 1. Install Nginx:
```bash
sudo apt update && sudo apt install -y nginx
sudo systemctl enable nginx
```

#### 2. Create the Site Configuration:
```bash
sudo nano /etc/nginx/sites-available/invoice-backend.svtechnologies.com.np
```

Paste the production reverse-proxy configuration:

```nginx
# 1. HTTP -> HTTPS Redirect
server {
    listen 80;
    listen [::]:80;
    server_name invoice-backend.svtechnologies.com.np;

    return 301 https://$host$request_uri;
}

# 2. HTTPS Reverse Proxy to Go API Gateway
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name invoice-backend.svtechnologies.com.np;

    # Cloudflare Origin SSL Certificates
    ssl_certificate /etc/ssl/certs/cloudflare_origin.pem;
    ssl_certificate_key /etc/ssl/private/cloudflare_origin.key;

    # SSL Optimization
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;

    # Maximum file upload size (for receipts, invoices, and logos)
    client_max_body_size 25M;

    location / {
        proxy_pass http://127.0.0.1:8080;

        # Upstream proxy headers
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;

        # WebSocket and token-streaming support (e.g. AI invoice assistant)
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        # Timeouts for database seeding or generation operations
        proxy_connect_timeout 90s;
        proxy_send_timeout 90s;
        proxy_read_timeout 90s;
    }
}
```

#### 3. Activate & Restart Nginx:
```bash
# Enable the site via symlink
sudo ln -s /etc/nginx/sites-available/invoice-backend.svtechnologies.com.np /etc/nginx/sites-enabled/

# Remove default boilerplate site
sudo rm -f /etc/nginx/sites-enabled/default

# Test configuration syntax
sudo nginx -t

# Restart Nginx
sudo systemctl restart nginx
```

---

### 13.4 Cloudflare SSL/TLS Mode
In Cloudflare Dashboard for `svtechnologies.com.np`:
1. Go to **SSL/TLS** ➔ **Overview**.
2. Select **Full (strict)**.
   *(Guarantees end-to-end cryptographic verification between Cloudflare edge and Lightsail Nginx)*.

---

### 13.5 Gateway CORS & Swagger Scheme Alignment

1. **CORS Configuration on Lightsail (`.env`)**:
   In `/home/ubuntu/invoice-generator-backend/.env`, ensure `ALLOWED_ORIGINS` includes both the frontend app and the backend Swagger domain:
   ```env
   ALLOWED_ORIGINS=https://invoice-backend.svtechnologies.com.np,https://invoice.svtechnologies.com.np,http://localhost:3000,http://localhost:5173
   ```
   Restart gateway: `docker compose -f docker-compose.prod.yml restart gateway`.

2. **Swagger HTTPS Scheme**:
   In `services/gateway/cmd/main.go`, the Swagger annotations specify:
   ```go
   // @schemes https http
   ```
   Listing `https` first ensures Swagger UI defaults to sending HTTPS requests, eliminating browser Mixed-Content / CORS failure blocks when testing endpoints from `https://invoice-backend.svtechnologies.com.np/swagger/index.html`.

---

### 13.6 Useful Maintenance & Debugging Commands

```bash
# Check Nginx status
sudo systemctl status nginx

# Test Nginx configuration syntax
sudo nginx -t

# View live Nginx error logs (helpful for proxy/timeout debugging)
sudo tail -f /var/log/nginx/error.log

# View live Nginx access logs
sudo tail -f /var/log/nginx/access.log

# Verify certificate validity directly with OpenSSL
openssl x509 -in /etc/ssl/certs/cloudflare_origin.pem -text -noout
```

