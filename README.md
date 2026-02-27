# SGM — Sistema de Gestión de Trámites Vehiculares

> Internal operations platform for managing vehicle registration workflows and transit services end-to-end.

---

## About

**SGM** is a private, internal system built for operations teams at dealerships.  
It is composed of two tightly coupled projects:

| Project | Description |
|---|---|
| [`sgm-backend/`](./sgm-backend) | NestJS REST API — business logic, persistence, file handling, PDF generation, reporting |
| [`sgm-frontend/`](./sgm-frontend) | Electron + React desktop app — UI shell, IPC auth bridge, workflow management |

The platform covers two business flows:

1. **Trámites** — `matricula` / vehicle registration cases: invoice intake, full lifecycle state machine, checklist documents + versioned PDF uploads, payments with attachments, shipment guide linking, account statement (`cuenta de cobro`) totals and PDF generation, overdue monitoring, CSV reporting.
2. **Servicios** — non-registration transit services: transfers, lien registration/removal/modification, duplicate documents/plates, color/motor/service changes, account transfer, and custom services driven by service-data templates.

---

## Architecture

```
┌─────────────────────────────────┐
│         SGM Desktop             │  Windows desktop app (Electron 30)
│  React 18 + TypeScript + Vite   │  ← renderer process
│  TanStack Query + Ant Design    │
│  Axios (Bearer token + 401)     │
│  electron-store / safeStorage   │
└────────────┬────────────────────┘
             │ HTTP (SGM API)
┌────────────▼────────────────────┐
│         SGM Backend             │  NestJS 11 REST API
│  Prisma 7 + PostgreSQL          │
│  JWT + bcrypt + Helmet          │
│  Multer + pdf-lib               │
│  Docker multi-stage build       │
└─────────────────────────────────┘
```

---

## SGM Desktop — Frontend

`SGM Desktop` is an internal Windows desktop app that gives operations teams a single workspace to manage trámites and servicios from intake to closure.

### What it does

- Authenticates users and persists the session token securely (Electron `safeStorage`).
- Trámite workspace: create, filter, view detail, change state, manage checklist/PDFs, record payments, link shipments, download account-statement PDFs.
- Servicios workspace: create from templates, edit service data, advance states, record payments.
- Overdue queue, business reports dashboard, CSV export, and user administration.
- Configurable backend fallback modes (`off | auto | force`) for local/mock continuity during development or API instability.

### Stack

| Layer | Technology |
|---|---|
| Desktop shell | `Electron 30` — main / preload / IPC |
| UI framework | `React 18` + `TypeScript 5` |
| Build tool | `Vite 7` |
| Routing | `React Router` (`createHashRouter`) — desktop-safe hash routing |
| Server state | `TanStack Query v5` — caching, invalidation, background refetch |
| HTTP client | `Axios` — Bearer token injection + 401 interceptor |
| UI system | `Ant Design 5` |
| Token storage | `electron-store` + Electron `safeStorage` (platform encryption) |
| Packaging | `electron-builder` — Windows NSIS installer + portable builds |
| Tooling | `ESLint` + strict TypeScript config |

### Security

- Renderer runs with `contextIsolation`, `sandbox`, and `webSecurity` enabled.
- Auth token lives exclusively in the main process; the renderer accesses it only through typed IPC calls.
- New windows and external navigation are blocked by default.
- Mock mode defaults to `off` to prevent silent fallback to fake data in production.

---

## SGM Backend — API

`sgm-backend` is a NestJS REST API that drives the full business logic of the SGM platform.

### What it does

- **Auth** — JWT login (`/auth/login`, `/auth/me`), bcrypt password hashing.
- **Trámites** (`/tramites`) — creation with required invoice PDF; state machine with cancel / finalize / reopen; checklist file uploads; payment records; shipment links.
- **Servicios** (`/servicios`) — service templates, state progression, service payments.
- **Files** (`/files/:id/download`) — controlled downloads that do not leak internal storage paths.
- **Cuenta de cobro** — computes totals and fills a PDF template using `pdf-lib` coordinate mapping.
- **Reports** (`/reports/summary`, `/reports/tramites`, `/reports/export.csv`) — paginated listings and injection-safe CSV exports.
- **Catalogs** (`/catalogs/*`) — lookup data consumed by the desktop app.
- **Business ID** — auto-assigned unique ID per case: `year + dealer code + consecutive`.
- **Audit trail** — full status-transition history stored for every case.
- **Overdue detection** — configurable alert rules flag cases past their SLA.
- **Ops scripts** — yearly data cleanup and client-deduplication utilities with `systemd` timer integration.

### Stack

| Layer | Technology |
|---|---|
| Framework | `NestJS 11`, `TypeScript`, `Node.js 20` |
| ORM / DB | `Prisma 7` (`@prisma/adapter-pg`) + `PostgreSQL` |
| Auth | `jsonwebtoken` + `bcrypt` |
| Hardening | `Helmet`, CORS allowlist, `Nest Throttler`, global validation + error filter |
| File handling | `Multer` (multipart), local filesystem storage with versioned metadata |
| PDF | `pdf-lib` — validation + template-based generation |
| API docs | `Swagger` (gated by `SWAGGER_ENABLED`) |
| DevOps | Docker multi-stage build, `docker-compose` for Postgres |
| Testing | `Jest` + `Supertest` |

### Security & Hardening

- Global `ValidationPipe` with `whitelist: true` and `forbidNonWhitelisted: true`.
- JWT guard on all protected routes.
- Global + stricter per-route throttling for `/auth/login`.
- CORS controlled via `CORS_ORIGINS` allowlist.
- PDF uploads validated by MIME type, file size cap, and max page count.
- CSV export sanitized against formula-injection attacks.

---

## Monorepo Structure

```
.
├── sgm-backend/      # NestJS API
│   ├── src/
│   ├── prisma/
│   ├── ops/          # maintenance scripts
│   ├── templates/    # PDF templates
│   ├── Dockerfile
│   └── docker-compose.yml
└── sgm-frontend/     # Electron + React desktop app
    ├── src/          # React application (routes, pages, API clients)
    ├── electron/     # Main process, preload, IPC auth bridge
    └── public/       # Static assets
```

---

## Getting Started

### Prerequisites

- Node.js 20+
- PostgreSQL (or use the provided `docker-compose`)
- Windows (for full Electron packaging; renderer dev works cross-platform)

### Backend

```bash
cd sgm-backend
cp .env.example .env        # fill in DATABASE_URL, JWT_SECRET, etc.
npm install
npx prisma generate
npx prisma migrate deploy
npx prisma db seed
npm run start:dev
```

Key environment variables (see `.env.example` for the full list):

| Variable | Purpose |
|---|---|
| `DATABASE_URL` | Postgres connection string |
| `JWT_SECRET` | Token signing secret |
| `STORAGE_ROOT` | Absolute path for uploaded file storage |
| `CORS_ORIGINS` | Comma-separated list of allowed origins |
| `SWAGGER_ENABLED` | `true` to expose `/docs` |

### Frontend

```bash
cd sgm-frontend
cp .env.example .env        # set VITE_API_BASE_URL
npm install

# renderer only (browser dev)
npm run dev

# full Electron dev
npm run electron:dev

# package Windows installer
npm run dist:win

# package portable exe
npm run dist:portable
```

Key environment variables:

| Variable | Purpose |
|---|---|
| `VITE_API_BASE_URL` | SGM backend base URL |
| `VITE_MOCK_MODE` | `off` \| `auto` \| `force` — backend fallback mode |

---

## Resume Highlights

**SGM Desktop**
- Built and maintained a production desktop operations platform using Electron + React + TypeScript for end-to-end vehicle registration and service workflows.
- Implemented secure desktop auth integration with preload IPC, isolated renderer context, and protected token handling.
- Designed a resilient API layer with TanStack Query + Axios, including response normalization and configurable backend/mock fallback modes.
- Delivered operational modules for workflow states, document/PDF lifecycle, payments, shipments, account-statement PDFs, overdue queueing, CSV exports, and user management.
- Packaged and shipped Windows installers and portable executables with `electron-builder`.

**SGM Backend**
- Built a modular NestJS + Prisma + PostgreSQL backend for vehicle-workflow operations (registrations and non-registration services) used by dealership-facing desktop tooling.
- Implemented end-to-end case lifecycle management: status state machine, audit history, payment/shipment linking, cancellation/reopen/finalization flows.
- Designed secure document pipeline with multipart uploads, file versioning, checklist synchronization, and controlled file downloads.
- Implemented account-statement (`cuenta de cobro`) computation and PDF generation from template coordinates using `pdf-lib`.
- Added reporting APIs (summary, filtered listings, CSV export with injection-safe sanitization) and operational data maintenance scripts.

---

## Notes

- Keep `.env` files out of source control; rotate all secrets before any production deployment.
- The desktop app expects the backend to be reachable at `VITE_API_BASE_URL`; set `VITE_MOCK_MODE=auto` to allow graceful degradation during development.
- If Electron exits instantly in your shell, verify `ELECTRON_RUN_AS_NODE` is not set in your environment.