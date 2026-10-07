# TaskBoard — Student Project Guide

**Session 21 DevOps Final Capstone**

This guide is written for students. It explains what TaskBoard is, how the entire system fits together, and how to run every part of it — step by step — on your own machine.

Read this first. Everything else in the repository builds on what is described here.

---

## What is TaskBoard?

TaskBoard is a small but realistic project management application. It is the kind of application that a startup might actually build and ship.

It has:

- A React frontend — the browser UI
- A FastAPI Python backend — the API server
- A PostgreSQL database — persistent storage

The point of the capstone is not the application itself. The point is to take this application and apply every DevOps practice from the course to it: containers, CI/CD, security scanning, infrastructure as code, Kubernetes, Helm, and monitoring.

By the end, you will have done what a real DevOps engineer does in a real company.

---

## How the pieces connect

```text
Your browser
    |
    | HTTP
    v
Nginx (port 3000, or Ingress in Kubernetes)
    |
    | /api/* proxied to backend
    v
FastAPI (port 8000)
    |
    | SQL queries via SQLAlchemy ORM
    v
PostgreSQL (port 5432)
    |
    | metrics scraped by
    v
Prometheus
    |
    | dashboards rendered in
    v
Grafana
```

On your local machine, Docker Compose runs all three tiers together.

In the cloud, Terraform provisions the AWS network and EKS cluster. Helm deploys the application to Kubernetes. GitHub Actions drives the full delivery pipeline from commit to deployment.

---

## Step 1 — Clone the repository

```bash
git clone <YOUR_GITHUB_REPO_URL>
cd session21-python
```

---

## Step 2 — Run the application locally with Docker Compose

This is the fastest way to see a working application. You only need Docker.

### Requirements

- Docker Desktop (Mac/Windows) or Docker Engine + Docker Compose (Linux)

### Run

```bash
docker compose up --build
```

Docker will:

1. Build the Python backend image
2. Build the React frontend image (multi-stage: Node build -> Nginx runtime)
3. Pull the PostgreSQL image
4. Start all three containers on a shared internal network
5. Run database migrations automatically on backend startup

### Open the application

```text
http://localhost:3000
```

You should see the TaskBoard dashboard.

### Open the backend API documentation

```text
http://localhost:8000/docs
```

FastAPI generates an interactive API page automatically. You can test every endpoint from your browser.

### Other backend endpoints

```text
http://localhost:8000/health    — liveness check
http://localhost:8000/ready     — readiness check (verifies database connection)
http://localhost:8000/metrics   — Prometheus-formatted metrics
```

### Stop the application

```bash
docker compose down
```

Remove the database volume too (wipes all task data):

```bash
docker compose down -v
```

---

## Step 3 — Run the backend directly (without Docker)

Use this when you want to iterate quickly on backend code.

### Requirements

- Python 3.12 or newer
- A running PostgreSQL instance (you can start only the database via Docker Compose)

### Start only the database

```bash
docker compose up db -d
```

### Set up Python

```bash
cd backend
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

### Configure the database connection

```bash
export DATABASE_URL='postgresql+psycopg://taskboard:taskboard@localhost:5432/taskboard'
```

### Run migrations

```bash
alembic upgrade head
```

### Start the FastAPI server

```bash
uvicorn app.main:app --reload --port 8000
```

The `--reload` flag restarts the server every time you save a file. Useful during development.

### Verify it is working

```bash
curl http://localhost:8000/health
```

Expected response:

```json
{"status": "healthy"}
```

---

## Step 4 — Run the frontend directly (without Docker)

Use this when you want to iterate quickly on the frontend.

### Requirements

- Node.js 20 or newer
- npm

### Install and start

```bash
cd frontend
npm install
npm run dev
```

Open:

```text
http://localhost:5173
```

The frontend development server proxies `/api` calls to `http://localhost:8000`. Make sure the backend is running at the same time.

---

## Step 5 — Run the tests

Tests must pass before the application can be deployed. This is enforced by the CI/CD pipeline.

```bash
cd backend
pytest -v
```

You should see output like:

```text
tests/test_tasks.py::test_create_task PASSED
tests/test_tasks.py::test_get_tasks PASSED
tests/test_tasks.py::test_update_task PASSED
tests/test_tasks.py::test_delete_task PASSED
tests/test_tasks.py::test_health_endpoint PASSED

5 passed in 1.23s
```

If any test fails, fix the code before proceeding.

---

## Step 6 — Understand the API

The backend exposes these endpoints:

| Method | Path | What it does |
|--------|------|--------------|
| GET | `/` | Root — confirms server is up |
| GET | `/health` | Liveness check — is the process alive? |
| GET | `/ready` | Readiness check — can it serve traffic? |
| GET | `/metrics` | Prometheus metrics |
| GET | `/api/tasks` | List all tasks |
| GET | `/api/tasks/{id}` | Get one task by ID |
| POST | `/api/tasks` | Create a new task |
| PUT | `/api/tasks/{id}` | Update a task |
| DELETE | `/api/tasks/{id}` | Delete a task |
| GET | `/api/tasks/stats` | Summary counts by status |

### Create a task using curl

```bash
curl -X POST http://localhost:8000/api/tasks \
  -H "Content-Type: application/json" \
  -d '{"title": "Set up CI/CD", "status": "in_progress", "priority": "high"}'
```

### List all tasks

```bash
curl http://localhost:8000/api/tasks
```

---

## Step 7 — Build the Docker images manually

If you want to understand the build process:

### Backend image

```bash
docker build -t taskboard-backend:local ./backend
```

### Frontend image

```bash
docker build -t taskboard-frontend:local ./frontend
```

### Inspect the images

```bash
docker images | grep taskboard
```

---

## Step 8 — Push code and watch CI/CD run

After you have configured your GitHub repository and set up the Actions secrets:

```bash
git add .
git commit -m "add: initial application with FastAPI and React"
git push origin main
```

Go to your GitHub repository > Actions tab.

You will see a workflow run start automatically. Watch it:

1. Run pytest
2. Build the React frontend
3. Build both Docker images
4. Scan images with Trivy
5. Push images to GHCR with the commit SHA as the tag

If any step fails, the pipeline stops and no image is promoted.

---

## Step 9 — Provision AWS infrastructure with Terraform

Before deploying to Kubernetes, you need a cluster.

```bash
cd terraform
terraform init
terraform plan
terraform apply
```

Terraform will create:

- A VPC with public and private subnets
- An Internet Gateway and NAT Gateway
- An EKS cluster
- A managed node group (EC2 worker nodes)

After `apply` completes, configure `kubectl`:

```bash
aws eks update-kubeconfig --region ap-south-1 --name <cluster-name>
```

Verify the cluster is reachable:

```bash
kubectl get nodes
```

You should see worker nodes in the `Ready` state.

### When you are done — destroy everything

```bash
terraform destroy
```

Do not skip this step. Leaving AWS infrastructure running costs real money.

---

## Step 10 — Deploy to Kubernetes with Helm

```bash
kubectl apply -f k8s/namespace.yaml

helm upgrade --install taskboard ./helm/taskboard \
  --namespace taskboard \
  --create-namespace
```

Check that everything is running:

```bash
kubectl get pods -n taskboard
```

Expected output:

```text
NAME                                  READY   STATUS    RESTARTS   AGE
taskboard-backend-xxxxxxxxx-xxxxx     1/1     Running   0          2m
taskboard-backend-xxxxxxxxx-yyyyy     1/1     Running   0          2m
taskboard-frontend-xxxxxxxxx-xxxxx    1/1     Running   0          2m
taskboard-postgres-0                  1/1     Running   0          2m
```

Check services:

```bash
kubectl get svc -n taskboard
```

Check the Ingress:

```bash
kubectl get ingress -n taskboard
```

---

## Step 11 — Enable Ingress routing

```bash
helm upgrade --install taskboard ./helm/taskboard \
  --namespace taskboard \
  -f helm/taskboard/values-dev.yaml
```

Add the Ingress hostname to your `/etc/hosts` file:

```text
<INGRESS_IP>    taskboard.local
```

Open the application:

```text
http://taskboard.local
```

---

## Step 12 — View metrics in Prometheus and Grafana

Install the monitoring stack:

```bash
helm upgrade --install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  -f monitoring/prometheus-values.yaml
```

Port-forward Prometheus:

```bash
kubectl port-forward svc/monitoring-kube-prometheus-prometheus -n monitoring 9090:9090
```

Open:

```text
http://localhost:9090
```

Search for `http_requests_total` to see request metrics from TaskBoard.

Port-forward Grafana:

```bash
kubectl port-forward svc/monitoring-grafana -n monitoring 3001:80
```

Open:

```text
http://localhost:3001
```

Default credentials: `admin` / `prom-operator`

---

## Repository structure reference

```text
session21-python/
├── backend/                    # FastAPI Python application
│   ├── app/                    # Main application code
│   │   ├── main.py             # FastAPI app instance, routes registration
│   │   ├── models.py           # SQLAlchemy ORM models
│   │   ├── schemas.py          # Pydantic request/response schemas
│   │   ├── database.py         # Database connection and session factory
│   │   └── routes/             # API route handlers
│   ├── alembic/                # Database migration files
│   ├── tests/                  # Pytest test suite
│   ├── Dockerfile              # Container build instructions
│   └── requirements.txt        # Python package dependencies
├── frontend/                   # React + Vite frontend
│   ├── src/                    # React source files
│   ├── public/                 # Static assets
│   ├── Dockerfile              # Multi-stage Node build + Nginx runtime
│   └── nginx.conf              # Nginx routing config
├── docker-compose.yml          # Local three-tier stack
├── terraform/                  # AWS VPC + EKS infrastructure
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── terraform.tfvars.example
├── helm/taskboard/             # Kubernetes Helm chart
│   ├── Chart.yaml
│   ├── values.yaml
│   ├── values-dev.yaml
│   └── templates/
├── k8s/                        # Namespace and bootstrap manifests
├── monitoring/                 # Prometheus + Grafana Helm values
├── troubleshooting/            # Intentionally broken manifests for lab exercises
├── scripts/                    # Load test and helper scripts
├── .github/workflows/          # GitHub Actions CI/CD pipeline
├── README.md                   # Full instructor and course reference
├── GRADING.md                  # Grading rubric and submission checklist
└── STUDENT-GUIDE.md            # This file
```

---

## Common problems and fixes

### docker compose up fails with "port already in use"

Something else is using port 3000, 8000, or 5432.

Find it:

```bash
lsof -i :8000
```

Kill the process or change the port in `docker-compose.yml`.

### pytest fails with "could not connect to database"

The test database is not running. Start the database container:

```bash
docker compose up db -d
```

Then run pytest again.

### kubectl get pods shows CrashLoopBackOff

Read the logs:

```bash
kubectl logs <pod-name> -n taskboard
```

The most common causes: wrong image tag, missing environment variable, or database not ready.

### Terraform fails with "NoCredentialsError"

You have not configured AWS credentials:

```bash
aws configure
```

Or set environment variables:

```bash
export AWS_ACCESS_KEY_ID=...
export AWS_SECRET_ACCESS_KEY=...
export AWS_DEFAULT_REGION=ap-south-1
```

### Helm upgrade fails with "release not found"

Use `--install` flag:

```bash
helm upgrade --install taskboard ./helm/taskboard -n taskboard
```

---

## What to build for your final project

You will build your own application using this same architecture. The application domain must be your own — not a copy of TaskBoard.

Ideas:

- Clinic appointment booking system
- University library management
- Hostel room allocation system
- Inventory and stock tracker
- Restaurant order management
- Employee leave management
- E-learning quiz platform
- Event registration system

The domain does not matter. The DevOps architecture is the same regardless of what the application does.

See `GRADING.md` for the full list of requirements and what you must submit.

---

## Quick reference commands

```bash
# Local development
docker compose up --build          # Start full stack
docker compose down -v             # Stop and delete data
pytest -v                          # Run tests

# Kubernetes
kubectl get pods -n taskboard
kubectl get svc -n taskboard
kubectl get ingress -n taskboard
kubectl logs <pod> -n taskboard
kubectl describe pod <pod> -n taskboard

# Helm
helm upgrade --install taskboard ./helm/taskboard -n taskboard
helm list -n taskboard
helm rollback taskboard -n taskboard

# Terraform
terraform init
terraform plan
terraform apply
terraform destroy

# GitHub Actions
git add . && git commit -m "feat: ..." && git push origin main
```
