# Probo - Setup & Deployment Guide

## Application Overview

**Probo** is an open-source SOC-2 compliance platform built for startups.

### Architecture

| Component | Technology | Location |
|-----------|------------|----------|
| **Backend** | Go 1.25.5 + GraphQL | `cmd/probod/`, `pkg/` |
| **Frontend Console** | React 19 + TypeScript + Vite | `apps/console/` |
| **Frontend Trust Center** | React 19 + TypeScript + Vite | `apps/trust/` |
| **Database** | PostgreSQL 17.4 | Docker/Cloud |
| **File Storage** | MinIO/S3 | Docker/AWS S3 |

---

## Local Development Setup

### Prerequisites

- **Go** 1.21+ (Download from https://go.dev/dl/)
- **Node.js** 20+ (22+ recommended)
- **Docker** & Docker Compose

### Step 1: Clone Repository

```bash
git clone https://github.com/legaciestechno2016/probo.git
cd probo
git submodule update --init --recursive
```

### Step 2: Start Docker Services

```bash
docker compose up -d
```

**Services started:**
- PostgreSQL (port 5433)
- MinIO S3 (ports 9000, 9001)
- Grafana (port 3001)
- Prometheus (port 9191)
- Mailpit (ports 1025, 8025)
- Chrome Headless (port 9222)

### Step 3: Install Dependencies

```bash
# Go dependencies
go mod download

# Node.js dependencies
npm ci
```

### Step 4: Build the Application

```bash
# Build email templates
npm --workspace @probo/emails run build

# Build frontend console
npm --workspace @probo/console run build

# Build frontend trust center
npm --workspace @probo/trust run build

# Build backend
go build -ldflags "-X 'main.version=0.108.0' -X 'main.env=prod'" -o bin/probod cmd/probod/main.go
```

### Step 5: Configure Environment

Create `apps/console/.env`:
```
VITE_API_URL=http://localhost:8088
```

### Step 6: Start the Application

```bash
# Start backend (in one terminal)
./bin/probod -cfg-file cfg/dev.yaml

# Start frontend (in another terminal)
npm --workspace @probo/console run dev
```

### Local URLs

| Service | URL |
|---------|-----|
| Frontend Console | http://localhost:5173 |
| Backend API | http://localhost:8088 |
| GraphQL Playground | http://localhost:8088/api/console/v1 |
| MinIO Console | http://localhost:9001 |
| Grafana | http://localhost:3001 |
| Mailpit (Email Testing) | http://localhost:8025 |

---

## Production Deployment Guide

### Deployment Architecture Options

Since Probo has both **Backend (Go)** and **Frontend (React)**, you need to deploy them separately:

| Component | Recommended Platform | Alternative |
|-----------|---------------------|-------------|
| Backend API | **Render** (Web Service) | Railway, Fly.io |
| Frontend | **Vercel** | Netlify, Cloudflare Pages |
| Database | **Render PostgreSQL** | Railway, Neon, Supabase |
| File Storage | **AWS S3** | Cloudflare R2, MinIO |

---

## Option 1: Deploy Backend on Render

### Step 1: Create Render Account
- Go to https://render.com and sign up

### Step 2: Create PostgreSQL Database
1. Dashboard → New → PostgreSQL
2. Name: `probo-db`
3. Plan: Free (for testing) or Starter ($7/month)
4. Note the **Internal Database URL**

### Step 3: Create Web Service for Backend
1. Dashboard → New → Web Service
2. Connect your GitHub repo
3. Configure:
   - **Name**: `probo-api`
   - **Region**: Choose closest
   - **Branch**: `main`
   - **Runtime**: Docker
   - **Plan**: Free or Starter

4. Add Environment Variables:
```
PROBOD_PG_ADDR=<internal-db-host>:5432
PROBOD_PG_USERNAME=<db-user>
PROBOD_PG_PASSWORD=<db-password>
PROBOD_PG_DATABASE=<db-name>
PROBOD_API_ADDR=0.0.0.0:10000
PROBOD_BASE_URL=https://your-frontend-domain.vercel.app
PROBOD_ENCRYPTION_KEY=<generate-base64-key>
```

5. Create a `render.yaml` in repo root:
```yaml
services:
  - type: web
    name: probo-api
    runtime: docker
    plan: free
    healthCheckPath: /health
    envVars:
      - key: PORT
        value: 10000
```

### Step 4: Create Dockerfile.render
```dockerfile
FROM golang:1.25-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o probod cmd/probod/main.go

FROM alpine:latest
RUN apk --no-cache add ca-certificates
WORKDIR /app
COPY --from=builder /app/probod .
COPY --from=builder /app/cfg ./cfg
EXPOSE 10000
CMD ["./probod", "-cfg-file", "cfg/prod.yaml"]
```

---

## Option 2: Deploy Frontend on Vercel

### Step 1: Create Vercel Account
- Go to https://vercel.com and sign up with GitHub

### Step 2: Import Project
1. Dashboard → Add New → Project
2. Import your GitHub repo
3. Configure:
   - **Framework Preset**: Vite
   - **Root Directory**: `apps/console`
   - **Build Command**: `npm run build`
   - **Output Directory**: `dist`

### Step 3: Add Environment Variables
```
VITE_API_URL=https://probo-api.onrender.com
```

### Step 4: Deploy
- Click "Deploy"
- Wait for build to complete
- Your app will be at `https://your-project.vercel.app`

---

## Option 3: Full Stack on Render (Alternative)

You can deploy both backend and frontend on Render:

### Backend: Same as above

### Frontend: Static Site
1. Dashboard → New → Static Site
2. Configure:
   - **Name**: `probo-console`
   - **Build Command**: `cd apps/console && npm run build`
   - **Publish Directory**: `apps/console/dist`

---

## Configuration Files to Create

### cfg/prod.yaml (Production Config)
```yaml
unit:
  metrics:
    addr: "0.0.0.0:8081"

probod:
  base-url: "https://your-frontend-domain.vercel.app"
  encryption-key: "${PROBOD_ENCRYPTION_KEY}"

  api:
    addr: "0.0.0.0:${PORT}"
    cors:
      allowed-origins:
        - "https://your-frontend-domain.vercel.app"

  pg:
    addr: "${DATABASE_URL}"
    pool-size: 20

  auth:
    disable-signup: false
    cookie:
      name: "SSID"
      domain: "your-frontend-domain.vercel.app"
      secret: "${COOKIE_SECRET}"
      duration: 24
      secure: true

  aws:
    region: "${AWS_REGION}"
    bucket: "${AWS_S3_BUCKET}"
    access-key-id: "${AWS_ACCESS_KEY_ID}"
    secret-access-key: "${AWS_SECRET_ACCESS_KEY}"
    endpoint: "${AWS_S3_ENDPOINT}"
```

---

## Environment Variables Reference

### Backend (Required)
| Variable | Description |
|----------|-------------|
| `PROBOD_PG_ADDR` | PostgreSQL host:port |
| `PROBOD_PG_USERNAME` | Database username |
| `PROBOD_PG_PASSWORD` | Database password |
| `PROBOD_PG_DATABASE` | Database name |
| `PROBOD_ENCRYPTION_KEY` | 32+ byte base64 key |
| `PROBOD_BASE_URL` | Frontend URL |

### Backend (Optional)
| Variable | Description |
|----------|-------------|
| `PROBOD_OPENAI_API_KEY` | OpenAI API key (for AI features) |
| `AWS_ACCESS_KEY_ID` | S3 access key |
| `AWS_SECRET_ACCESS_KEY` | S3 secret key |
| `AWS_S3_BUCKET` | S3 bucket name |

### Frontend
| Variable | Description |
|----------|-------------|
| `VITE_API_URL` | Backend API URL |

---

## Quick Reference Commands

### Local Development
```bash
# Start all services
docker compose up -d

# Build and run backend
go build -o bin/probod cmd/probod/main.go
./bin/probod -cfg-file cfg/dev.yaml

# Start frontend dev server
npm --workspace @probo/console run dev

# Stop all services
docker compose down
```

### Production Build
```bash
# Build frontend for production
npm --workspace @probo/console run build

# Build backend binary
CGO_ENABLED=0 GOOS=linux go build -o bin/probod cmd/probod/main.go
```

---

## Troubleshooting

### Port Conflicts
If ports are already in use, modify `compose.yaml`:
- PostgreSQL: Change `5432:5432` to `5433:5432`
- API: Update `cfg/dev.yaml` to use different port

### Database Connection Issues
1. Ensure PostgreSQL is running: `docker compose ps postgres`
2. Check logs: `docker compose logs postgres`

### Build Errors
1. Initialize git submodules: `git submodule update --init --recursive`
2. Clear cache: `npm cache clean --force && npm ci`

---

## Support & Resources

- **Documentation**: https://www.getprobo.com/docs
- **Discord**: https://discord.gg/8qfdJYfvpY
- **GitHub Issues**: https://github.com/getprobo/probo/issues
