# Phase 08: Docker & Deployment

Containerize the app and configure deployment. By the end of this phase, `docker compose up --build` runs the full stack.

**Prerequisite:** Phases 04 and 06 complete (backend and frontend both working locally).

---

## Multi-Stage Dockerfile

Create `Dockerfile` in the project root with three stages:

```dockerfile
# --- Backend ---
FROM python:3.12-slim AS backend

COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/

WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev
COPY api/ api/

EXPOSE 8020
CMD ["uv", "run", "uvicorn", "api.main:app", "--host", "0.0.0.0", "--port", "8020"]

# --- Frontend Build ---
FROM node:20-slim AS frontend-build

WORKDIR /app
COPY frontend/package*.json ./
RUN npm ci
COPY frontend/ .
RUN npm run build

# --- Frontend Serve ---
FROM nginx:alpine AS frontend

COPY --from=frontend-build /app/dist /usr/share/nginx/html
COPY frontend/nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

Key points:
- `python:3.12-slim` as the base — stable and small
- uv is copied from its official container image
- `uv sync --frozen --no-dev` installs only production dependencies from the lockfile
- Frontend is built with node then served with nginx — no node runtime in production

---

## docker-compose.yml

Create `docker-compose.yml`:

```yaml
services:
  api:
    build:
      context: .
      target: backend
    container_name: myapp-api
    ports:
      - "8020:8020"
    extra_hosts:
      - "host.docker.internal:host-gateway"
    environment:
      - MONGODB_URL=mongodb://host.docker.internal:27017
      - MONGODB_DB_NAME=myapp
      - JWT_SECRET=${JWT_SECRET:-change-me-in-production}
    restart: unless-stopped

  frontend:
    build:
      context: .
      target: frontend
    container_name: myapp-frontend
    ports:
      - "8095:80"
    depends_on:
      - api
    restart: unless-stopped
```

Key points:
- `extra_hosts` maps `host.docker.internal` so the API container can reach MongoDB running on the host
- `restart: unless-stopped` keeps services running after reboots
- Secrets like `JWT_SECRET` come from the shell environment or a `.env` file

---

## Nginx Config (Frontend)

Create `frontend/nginx.conf`:

```nginx
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;

    # SPA: serve index.html for all routes
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Proxy API calls to the backend container
    location /api/ {
        proxy_pass http://api:8020;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Cache static assets
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

The `proxy_pass http://api:8020` works because Docker Compose puts both containers on the same network and `api` resolves to the backend container.

---

## Local Development

### Without Docker (recommended during development)

```bash
# Terminal 1: Backend
uv run uvicorn api.main:app --reload --port 8020

# Terminal 2: Frontend
cd frontend && npm run dev
```

The Vite dev server proxies `/api` requests to the backend automatically.

### With Docker

```bash
docker compose up --build
```

Access the app at `http://localhost:8095`.

---

## Docker Log Rotation

To prevent logs from filling disk, configure Docker's log rotation. Add to `/etc/docker/daemon.json`:

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

This caps each container at 30MB of logs (3 files x 10MB). Restart Docker after changing: `sudo systemctl restart docker`.

---

## Path A: Raspberry Pi Deployment

The Pi runs Docker, MongoDB (on host), nginx (reverse proxy), Cloudflare Tunnel, and an auto-deploy script.

### Deploy a new app

1. Build and start containers on the Pi:
   ```bash
   cd /path/to/appname
   docker compose up -d --build
   ```

2. Add nginx config and Cloudflare DNS record (see `house/adding-new-app.md`)

3. Register the app's ports in `house/applications.md`

4. Add the app to the auto-deploy script's `APPS` array for automatic deployment on push to `main`

### Pi-specific notes

- The Pi's MongoDB instance is shared across apps. Each app uses a separate database (set via `MONGODB_DB_NAME`)
- Apps never access each other's databases directly — cross-app communication goes through REST APIs
- No extra files needed beyond the standard `Dockerfile` and `docker-compose.yml`

---

## Path B: Azure Deployment

Three environments managed by Terraform and GitHub Actions.

### Environments

| Tier | Trigger | URL |
|------|---------|-----|
| **Local** | `docker-compose up --build` | `localhost:<port>` |
| **Dev** | PR opened/updated | `<app>-dev.<region>.azurecontainer.io` |
| **Production** | Push to `main` | Custom domain or `<app>.<region>.azurecontainer.io` |

### Additional Files for Azure

**`Dockerfile.api`** — standalone API image:
```dockerfile
FROM python:3.12-slim
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/
WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev
COPY api/ api/
EXPOSE 8020
CMD ["uv", "run", "uvicorn", "api.main:app", "--host", "0.0.0.0", "--port", "8020"]
```

**`Dockerfile.frontend`** — standalone frontend/nginx image:
```dockerfile
FROM node:20-slim AS build
WORKDIR /app
COPY frontend/package*.json ./
RUN npm ci
COPY frontend/ .
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
COPY frontend/nginx.azure.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

**`frontend/nginx.azure.conf`** — uses `localhost` instead of Compose service name (ACI containers share a network namespace):
```nginx
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location /api/ {
        proxy_pass http://localhost:8020;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

### Terraform Structure

```
infra/
├── providers.tf       # azurerm provider + backend config
├── variables.tf       # All input variable definitions
├── main.tf            # Data sources, locals, ACI container group
├── outputs.tf         # FQDN, IP address
└── terraform.tfvars   # Non-secret prod defaults (committed)
```

Key patterns:
- Use `locals` to derive environment-specific names (e.g., `container_group = is_prod ? "myapp" : "myapp-dev"`)
- Use `data` sources for shared pre-existing resources (resource group, ACR, storage account)
- Pass environment via `-var="environment=dev"` flag
- State stored in Azure Blob Storage

### Secret Management

Secrets flow from GitHub to containers:

```
GitHub Environment Secret (e.g., MONGODB_URL on "production")
  → Workflow job: `environment: production`
  → env: TF_VAR_mongodb_url: ${{ secrets.MONGODB_URL }}
  → Terraform variable "mongodb_url" (sensitive)
  → azurerm_container_group secure_environment_variables
  → Container env var MONGODB_URL
  → Python pydantic-settings: Settings.mongodb_url
```

### GitHub Actions Workflows

**`deploy.yml`** (production):
- Trigger: push to `main`
- Job 1 (`build`): Build + push images to ACR (`:$SHA` + `:latest`)
- Job 2 (`deploy`): `environment: production` → `terraform init` → `workspace select prod` → `plan` → `apply`

**`deploy-dev.yml`** (development):
- Trigger: `pull_request: [opened, synchronize, reopened]`
- Concurrency group with cancel-in-progress
- Job 1 (`build`): Build + push images to ACR (`:dev-$SHA`)
- Job 2 (`deploy`): `environment: development` → `terraform init` → `workspace select dev` → `plan` → `apply`
- Post-deploy: comment on PR with dev URL

### Azure Gotchas

- ACI containers in the same group share a network namespace — they communicate via `localhost`
- ACI memory must be in 0.1 GB increments (0.25 fails, use 0.3)
- MongoDB Atlas free tier needs `0.0.0.0/0` network access for ACI (dynamic IPs)
- `terraform.tfvars` overrides `TF_VAR_` env vars — use `-var` flags for per-environment overrides
- Setting explicit `permissions` on a GitHub Actions job removes all defaults — must include `contents: read` for checkout

### One-Time Bootstrap for Azure

1. Create Terraform state storage account
2. Create a service principal for GitHub Actions
3. Create GitHub Environments (`production`, `development`) with secrets
4. Create `infra/` directory with Terraform files
5. Import existing resources into Terraform state (if any)

---

## Checklist

### All apps
- [ ] `Dockerfile` created (multi-stage: backend, frontend-build, frontend)
- [ ] `docker-compose.yml` created with correct ports and environment
- [ ] `frontend/nginx.conf` created with SPA fallback and API proxy
- [ ] `docker compose up --build` starts both services successfully
- [ ] App is accessible at `http://localhost:<frontend-port>`
- [ ] API proxy works (frontend can call `/api/*` endpoints)

### Path A only (Pi)
- [ ] nginx reverse proxy config added on Pi
- [ ] Cloudflare DNS record created
- [ ] App added to auto-deploy script

### Path B only (Azure)
- [ ] `Dockerfile.api` and `Dockerfile.frontend` created
- [ ] `frontend/nginx.azure.conf` created (uses `localhost`)
- [ ] `infra/` Terraform files created
- [ ] GitHub Environments and secrets configured
- [ ] GitHub Actions workflows created (`deploy.yml`, `deploy-dev.yml`)
- [ ] Test: PR deploys to dev, merge deploys to prod
