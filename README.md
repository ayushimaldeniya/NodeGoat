# OWASP NodeGoat — DevSecOps Pipeline & Security Hardening

[![DevSecOps CI Pipeline](https://github.com/ayushimaldeniya/NodeGoat/actions/workflows/ci.yml/badge.svg)](https://github.com/ayushimaldeniya/NodeGoat/actions/workflows/ci.yml)

## Project Overview
This repository contains the containerized, security-hardened implementation of **OWASP NodeGoat** (RetireEasy Employee Retirement Savings Management platform) developed for module **IE3142 — DevOps Security** (SLIIT, Year 3 Semester 1, 2026).

The project establishes a comprehensive shift-left DevSecOps architecture featuring multi-container Docker Compose deployment, proactive STRIDE threat modeling, multi-vulnerability secure coding remediations, and an automated GitHub Actions CI/CD pipeline enforcing four security gates and runtime dynamic secrets management.

---

## Group 35 — Team Members & Roles

| Student ID | Member Name | Assigned Project Focus |
| :--- | :--- | :--- |
| **IT24103820** | **Maldeniya A. T.** | DevOps Lead, SAST Automation, NoSQL Injection & Access Control |
| **IT24102560** | **Nirmana A. A. H.** | Infrastructure & Secrets Security, Gate 1 (Gitleaks), Stored XSS |
| **IT24103991** | **Siriwardana A. W. T. B.** | CI/CD Security, Gate 3 (npm audit), Insecure Cookie Attributes |
| **IT24610817** | **Gunasena B. R. S.** | DevSecOps Engineer, Gate 4 (Trivy Container Scan), IDOR |

---

## Architecture & Trust Boundaries

The application is deployed as a multi-tier containerized topology orchestrating three distinct services:
1. **`web` (`nodegoat-web`):** Node.js 18 Alpine runtime running Express.js, exposing port `4000:4000` to the host across **Trust Boundary 1** (Internet/Host to App Tier).
2. **`mongo` (`nodegoat-mongo`):** MongoDB 4.4 database running on isolated Docker container networking (port `27017`), strictly separating the data tier across **Trust Boundary 2** (App Tier to Data Tier).
3. **`vault` (`nodegoat-vault`):** HashiCorp Vault dynamic secret management engine running on port `8200:8200`, dynamically populating application tokens directly into runtime memory.

---

## Prerequisites

Before running the application or security scanners, ensure your host machine has:
* [Docker Desktop](https://www.docker.com/products/docker-desktop/) (v24.0+ recommended with Docker Compose V2 support)
* [Node.js](https://nodejs.org/) (v18+ LTS) & `npm`
* [Git](https://git-scm.com/)

---

## Running the Application Locally

### 1. Clone the Repository
```bash
git clone [https://github.com/ayushimaldeniya/NodeGoat.git](https://github.com/ayushimaldeniya/NodeGoat.git)
cd NodeGoat

2. Configure Environment Variables (Optional Local Fallbacks)
Create a .env file in the root directory (this file is excluded from version control via .gitignore):

PORT=4000
MONGODB_URI=mongodb://mongo:27017/nodegoat
COOKIE_SECRET=LocalDevSuperRandomSecretKey2026!
CRYPTO_KEY=LocalDevEncryptionKeyPassphrase987!

3. Build and Start Multi-Container Services
docker compose up -d --build

4. Verify Application Health
Web Application: Navigate to http://localhost:4000 in your web browser.

Default Seed Accounts:
Administrator: admin / Admin_123
Standard User: user1 / User1_123

Stop Containers:
docker compose down
Running Security Scanners Locally
Gate 1: Secret Detection (Gitleaks)
Inspects the full Git commit log for high-entropy secrets and exposed credentials:
docker run --rm -v "${PWD}:/repo" zricethezav/gitleaks:latest detect --source /repo -v

Gate 2: Static Application Security Testing — SAST (Semgrep)
Standard Auto Baseline Scan:
docker run --rm -v "${PWD}:/src" returntocorp/semgrep semgrep scan --config auto /src/app

Custom Security Regression Rules (NoSQL & Broken Access Control):
docker run --rm -v "${PWD}:/src" returntocorp/semgrep semgrep scan --config /src/nosql-injection-rule.yml --config /src/elevation-of-privilege-rule.yml /src/app --error

Gate 3: Software Composition Analysis — SCA (npm audit)
Audits direct and transitive dependencies against known advisory CVE databases:
npm audit --audit-level=critical

Gate 4: Container Vulnerability Scanning (Trivy)
Scans the container image layers and operating system packages for High and Critical vulnerabilities:
docker build -t nodegoat-app:latest .
trivy image --severity HIGH,CRITICAL nodegoat-app:latest

Optional DAST: Dynamic Application Security Testing (OWASP ZAP)
Scans the live running container for missing security headers and runtime misconfigurations:

docker run --rm -t ghcr.io/zaproxy/zaproxy:stable zap-baseline.py -t [http://host.docker.internal:4000](http://host.docker.internal:4000)

## DevSecOps CI/CD Pipeline Structure
Automated pipeline workflows are configured under .github/workflows/ci.yml and execute on every push and pull_request targeting master:

Checkout & History Hydration: Checks out source code with fetch-depth: 0 to enable full commit history inspection.

Gate 1 (Secrets Scanning): Runs Gitleaks across git history to block credential leaks.

Build & Automated Testing: Installs dependencies (npm install --legacy-peer-deps --ignore-scripts) and executes unit tests (npm test).

Gate 2 (SAST Analysis): Executes broad Semgrep rules alongside blocking custom AST rules (nosql-injection-rule.yml, elevation-of-privilege-rule.yml).

Gate 3 (Dependency SCA): Executes npm audit --audit-level=critical to block supply chain vulnerabilities.

Docker Build & Gate 4 (Container Scan): Builds nodegoat-app:latest and scans using Aquasecurity Trivy.

Build-Breaker Policy: Security gates enforce continue-on-error: false under blocking conditions, halting workflow progression upon detecting unmitigated Critical risks.

Secrets Management Implementation
Application secrets are strictly decoupled from source code in accordance with 12-Factor App standards:

CI/CD Pipeline: Provisioned using GitHub Actions Encrypted Secrets (MONGODB_URI, SESSION_SECRET) utilizing asymmetric Libsodium sealed-box encryption.

Runtime Dynamic Retrieval: Integrated with HashiCorp Vault KV Engine via config/vault-client.js, retrieving dynamic secrets into process memory at boot with a 3000ms timeout graceful fallback to environment variables.
