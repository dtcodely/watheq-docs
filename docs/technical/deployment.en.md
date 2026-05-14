---
title: Deployment Guide
---

# Deployment Guide

This guide walks you through configuring the system locally for development purposes, as well as outlining production deployment instructions.

![Image suggestion: An illustration incorporating logos of operational tools such as Docker, Node.js, and PM2]

## Prerequisites

To ensure an error-free deployment and run cycle, confirm the presence of the following software on your server/machine:
- **Node.js** version 18.x or above.
- **Docker Desktop** (or Docker Engine) for running infrastructural services like the Database, Cache, and pgAdmin.
- **npm** version 9.x or above.

---

## 1. Init Infrastructure Services via Docker

The project embeds a cohesive `docker-compose.yml` containing definitions for necessary operational dependencies.

```bash
# Launch background services (PostgreSQL Database, Redis queue cache, pgAdmin UI)
docker-compose up -d

# Verify bounding container state
docker-compose ps
```

### Default Service Configurations:
| Service | Mapped Port | Default Access Credentials |
|---|---|---|
| **PostgreSQL** | 5454 | user: postgres / pass: postgres |
| **Redis** | 6379 | — |
| **pgAdmin** | 5051 | admin@watheq.com / admin |

---

## 2. Running The Backend Application

After establishing infrastructure containers, initialize the core backend server:

```bash
cd backend
npm install
npx prisma generate
npx prisma migrate deploy
npm run dev
```

> **Note:** The backend operates natively at `http://localhost:3000`.

---

## 3. Running The Frontend (Web UI)

To launch the compiled interactive user interface served by Vite and React:

```bash
cd web
npm install
npm run dev
```

> **Note:** The local server will boot and open the browser dynamically on `http://localhost:5173`.

---

## 4. Fundamental Environment Variables (`backend/.env`)

Guarantee the presence of a localized `.env` configuration mapping the database and tokens properly:
```env
DATABASE_URL="postgresql://postgres:postgres@localhost:5454/watheq"
JWT_SECRET="your-secret-key"
REDIS_URL="redis://localhost:6379"
NODE_ENV="development"
PORT=3000
```

---

## 5. Critical Step: Injecting Advanced Search GIN Indexes (Phase 5)

Because Watheq invokes raw PostgreSQL indicators unmapped by Prisma sequentially, it is mandatory to inject the GIN index SQL commands manually either on fresh bootstrap or subsequent to any database purge.

```bash
# 1. Export the specialized migration SQL script from host to Docker container
docker cp "prisma\migrations\20260509225603_add_gin_index_text_content\migration.sql" watheq-db:/tmp/gin_index.sql

# 2. Invoke PSQL to interpret the index schema
docker exec watheq-db psql -U postgres -d watheq -f /tmp/gin_index.sql
```

![Image suggestion: Standard terminal or pgAdmin snippet visualizing active indexes loaded successfully]

---

## 6. Approaching Database Manager (pgAdmin)

Visually inspecting tables via the graphical platform uses:
- **Link:** `http://localhost:5051`
- **Email:** `admin@watheq.com`
- **Password:** `admin`

When wiring the Server setup inside the UI, provision these details: `Host: watheq-db`, `Port: 5432`, `DB: watheq`.

---

## 7. Production Builds

For elevated performance constraints typical for live operation:

```bash
# 1. Bundle lightweight frontend statics (Target routing e.g. through Nginx)
cd web && npm run build

# 2. Compile TypeScript backend sources to native Javascript
cd backend && npm run build

# 3. Supervise continuous uptime using PM2
pm2 start dist/server.js --name watheq-backend
```

---

## 8. Current Internal Documentation Portal (MkDocs)

This very interactive documentation site serves as extensive technical alignment resources, leveraging `Material for MkDocs`.

```bash
# From the project root parameter
cd docs-site
.\venv\Scripts\mkdocs serve
```
Subsequently, the bilingual site runs over `http://localhost:8000`.
