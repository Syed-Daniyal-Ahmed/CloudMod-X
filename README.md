# CloudModX

> Lightweight AWS-native Internal Developer Platform (IDP) and Module Lifecycle Management System.

[![Backend Tests](https://img.shields.io/badge/pytest-20%20passed-emerald)]()
[![Build Status](https://img.shields.io/badge/build-passing-brightgreen)]()
[![AWS Region](https://img.shields.io/badge/AWS%20Region-ap--south--1-orange)]()
[![EC2](https://img.shields.io/badge/Amazon%20EC2-t3.small-blue)]()
[![Database](https://img.shields.io/badge/Amazon%20RDS-PostgreSQL%2015-blue)]()

---

## 1. Project Purpose

CloudModX is an Internal Developer Platform designed to streamline how cloud-native software modules are registered, versioned, packaged, deployed, and audited across target environments. It provides developers and platform engineers with a unified dashboard to manage module lifecycles, enforce deployment health checks, store release artifacts in Amazon S3, and trigger instantaneous rollbacks when failures occur.

---

## 2. Architecture & AWS Specifications

```
                           [ Web Browser / Client ]
                                      |
                                   TCP 80 (HTTP)
                                      v
                        +---------------------------+
                        |   Amazon EC2 (t3.small)   |
                        |                           |
                        |   +-------------------+   |
                        |   |   React + Nginx   |   |
                        |   |  (Port 80 Ingress)|   |
                        |   +---------+---------+   |
                        |             | (Internal)  |
                        |             v             |
                        |   +-------------------+   |
                        |   |  FastAPI Backend  |   |
                        |   |    (Port 8000)    |   |
                        |   +----+--------+-----+   |
                        +--------|--------|---------+
                                 |        |
              IAM Role (boto3)   |        |  PostgreSQL TCP 5432
        +------------------------+        +--------------------------+
        |                                                            |
        v                                                            v
 +-----------------------+                         +----------------------------------+
 |    Amazon S3 Bucket   |                         |     Amazon RDS PostgreSQL 15     |
 | (Release Artifacts)   |                         |  (Private Subnets, Encrypted)    |
 +-----------------------+                         +----------------------------------+
```

### AWS Infrastructure Services

| Service | Role in CloudModX | Configuration Details |
|:---|:---|:---|
| **Amazon VPC** | Network Isolation | Custom VPC `10.0.0.0/16` across 2 Availability Zones (`ap-south-1a`, `ap-south-1b`). |
| **Amazon EC2** | Application Host | `t3.small` (2 vCPU, 2 GB RAM, 20 GB GP3 encrypted root) running Docker container stack (Nginx + FastAPI). Managed via AWS SSM. |
| **Amazon RDS** | Production Relational DB | `db.t4g.micro` PostgreSQL 15, Single-AZ, 20 GB GP3 storage (KMS encrypted), private subnets only. |
| **Amazon S3** | Package Storage | Bucket for versioned multi-part tarball/zip build artifacts. |
| **AWS IAM** | Identity & Access | Least-privilege EC2 Instance Role for S3 and CloudWatch (no static keys). |
| **CloudWatch** | Observability | Unified logging group (`/cloudmodx/development/application`) + CPU/Storage/Connection alarms. |

---

## 3. Module Lifecycle & Deployment Engine

### Lifecycle Stages
```
  [ Draft ]  ──>  [ Review ]  ──>  [ Active ]  ──>  [ Archived ]
```

### Deployment State Flow
1. **Creation**: Triggered via `POST /api/v1/deployments`. Initial state starts as `pending` / `running`.
2. **Artifact Validation**: System checks that the target `ModuleVersion` has a valid release artifact attached in Amazon S3.
3. **Success State**: If artifact exists, deployment completes with `status: success` and automatically promotes module to `active`.
4. **Failure State & Diagnostics**: If artifact is missing or health checks fail, deployment is marked `status: failed`, capturing actionable diagnostics in `error_message`.
5. **Rollback Engine**: 1-click Rollback (`POST /api/v1/modules/{id}/rollback`) detects the last verified stable release in history and redeploys it immediately.
6. **Audit Trail**: Every action (`CREATE_MODULE`, `UPLOAD_ARTIFACT`, `DEPLOYMENT_SUCCESS`, `DEPLOYMENT_FAILED`, `DEPLOYMENT_ROLLBACK`) generates immutable PostgreSQL audit records.

---

## 4. Local Development Run Instructions

### Prerequisites
- **Docker Desktop** (with Docker Compose v2+)
- **Python 3.11+**
- **Node.js 18+** & npm
- **Terraform v1.5+** & AWS CLI v2 (optional, only for AWS cloud provisioning)

### Network Ports & URLs

| Service | Container Port | Host Port | Local Access URL |
|:---|:---|:---|:---|
| **Frontend UI (Nginx)** | `80` | `80` | `http://localhost` |
| **Backend API (FastAPI)** | `8000` | `8000` | `http://localhost:8000` (Docs: `http://localhost:8000/docs`) |
| **API via Nginx Proxy** | `8000` | `80` | `http://localhost/api/health` |
| **PostgreSQL Database** | `5432` | `5433` | `127.0.0.1:5433` (avoid host port 5432 conflict) |

### Quick Start with Docker Compose (Recommended)

1. **Clone and Configure Environment**:
   ```bash
   cp .env.example .env
   ```
   *Note: In `.env`, default development values are pre-configured for Docker Compose. The backend inside Compose connects to PostgreSQL at `db:5432`.*

2. **Start the Stack**:
   ```bash
   docker compose up --build -d
   ```

3. **Run Migrations & Seed Baseline Data**:
   ```bash
   # Apply database schema migrations
   docker compose exec backend alembic upgrade head

   # Seed development users, modules, versions, and deployments
   docker compose exec backend python -m app.database.seed
   ```

4. **Verify Application Health**:
   ```bash
   curl -f http://localhost:8000/api/health
   # Or via Nginx proxy:
   curl -f http://localhost/api/health
   ```
   Open `http://localhost` in your browser to view the CloudModX dashboard.

5. **Stop Services**:
   ```bash
   docker compose down
   # To stop and remove persistent database volumes:
   docker compose down -v
   ```

---

### Host-Run Development (Without Docker for Backend/Frontend)

If you prefer running Python and React directly on your host machine while using Docker solely for PostgreSQL:

1. **Start only PostgreSQL**:
   ```bash
   docker compose up -d db
   ```

2. **Configure Host Environment**:
   In your `backend/.env` (or root `.env`), set `DATABASE_URL` to connect to the mapped host port `5433`:
   ```env
   DATABASE_URL=postgresql+psycopg2://cloudmodx_user:cloudmodx_password@127.0.0.1:5433/cloudmodx_db
   ```

3. **Backend Setup**:
   ```bash
   cd backend
   python -m venv .venv
   # Windows:
   .venv\Scripts\activate
   # Linux/macOS:
   source .venv/bin/activate

   pip install -r requirements.txt
   alembic upgrade head
   python -m app.database.seed
   uvicorn app.main:app --host 127.0.0.1 --port 8000 --reload
   ```

4. **Frontend Setup**:
   ```bash
   cd frontend
   npm install
   npm run dev
   ```
   Access the frontend at `http://localhost:5173`.

---

## 5. Testing & Verification

### Backend Tests
CloudModX includes a comprehensive pytest suite covering database models, service layers, deployment state transitions, rollback mechanics, and health endpoints:

```bash
# From repository root
python -m pytest backend/tests -v
```
*All 20 test cases execute against an isolated SQLite in-memory database and complete in < 2 seconds.*

### Frontend Production Build
Validate that the React + Tailwind bundle builds cleanly without syntax or bundling errors:

```bash
cd frontend
npm run build
```

---

## 6. Troubleshooting

- **Port 80 Conflict on Host**:
  On Windows or macOS, IIS, Apache, or Skype may hold port 80. You can change the host port in `docker-compose.yml` under `frontend` from `"80:80"` to `"3000:80"`. Access the UI at `http://localhost:3000`.
- **Port 5433 Conflict**:
  PostgreSQL is mapped to host port `5433` specifically to prevent collisions with existing local PostgreSQL installations on `5432`. If port 5433 is in use, modify line 12 of `docker-compose.yml`.
- **Database Status "Degraded"**:
  Check `docker compose logs db` to verify PostgreSQL is running and ready. Ensure the container health check is passing via `docker compose ps`.
- **Backend Connection Refused**:
  Inside Docker Compose, the backend communicates with PostgreSQL via the internal Docker bridge network on service name `db` and port `5432` (`postgresql+psycopg2://...@db:5432/cloudmodx_db`). Only host-run processes use `127.0.0.1:5433`.

---

## 7. Security Model & Production Hardening

- **Restricted Public Surface**: Only TCP Port 80 is open to the public on EC2.
- **Internal Microservices**: FastAPI runs on port 8000 internally inside the Docker network.
- **Private Database**: RDS is located in private subnets with no public IP (`publicly_accessible = false`) and only accepts TCP 5432 from the EC2 Security Group.
- **Strict Production Validation**: In production mode (`ENVIRONMENT=production`), the application runtime strictly validates that `SECRET_KEY` is at least 32 characters long and that `DATABASE_URL` points to an external production endpoint, forbidding placeholder fallbacks or localhost connections.
- **Zero Committed Credentials**: All live database passwords, API keys, and environment variables are excluded via `.gitignore`. Live infrastructure secrets must be provided through secure uncommitted files (`terraform.tfvars`) or environment variables (`TF_VAR_db_password`).
- **Zero Open SSH**: Port 22 is disabled. Management is executed securely via AWS Systems Manager Session Manager (SSM).

---

## 8. AWS Infrastructure Setup & Cost / Teardown

### Provisioning Infrastructure with Terraform

1. **Initialize Terraform**:
   ```bash
   cd infrastructure/terraform
   terraform init
   ```

2. **Supply Required Database Password**:
   Copy the example variables file:
   ```bash
   cp terraform.tfvars.example terraform.tfvars
   ```
   Edit `terraform.tfvars` with a secure master password (at least 8 printable characters; avoid `/`, `@`, `"`, or spaces). `terraform.tfvars` is gitignored and will never be committed. Alternatively, pass via environment:
   ```bash
   export TF_VAR_db_password="YourSecureMasterPassword123!"
   ```

3. **Plan and Deploy**:
   ```bash
   terraform plan
   terraform apply
   ```

### Cost-Conscious Design Highlights

CloudModX is architected to eliminate unnecessary cloud infrastructure spend:
- **Avoided NAT Gateways**: Saves ~$32/month per gateway by positioning the EC2 application host in the public subnet while routing directly to private RDS subnets without managed NAT gateways.
- **Avoided Application Load Balancer**: Nginx reverse proxy running inside the EC2 container stack eliminates ~$16/month ALB baseline costs.
- **Graviton ARM64 RDS Tier**: Uses `db.t4g.micro` providing 20% better price-performance over comparable x86 instances.
- **Encrypted GP3 Storage**: Cost-effective storage baseline for root volumes and database data.

### Teardown Instructions (Destroy All AWS Resources)

To prevent ongoing cloud charges when testing or demonstrations are concluded, run:

```bash
cd infrastructure/terraform
terraform destroy -auto-approve
```
*This command cleanly removes the EC2 instance, RDS PostgreSQL instance, security groups, VPC, CloudWatch alarms, and associated AWS resources.*

---

## 9. Documentation Index

- [Architecture Guide](docs/architecture.md)
- [Deployment & Rollback Workflow](docs/deployment-flow.md)
- [Database Models & Schema](docs/database.md)
- [Security & Isolation Model](docs/security.md)
- [Demo Checklist & Verification Evidence](docs/demo-evidence.md)
