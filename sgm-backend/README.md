# SGM Backend API

`sgm-backend` is a NestJS API for an internal operations platform that manages vehicle paperwork workflows for dealerships.

It handles two business flows in one backend:

1. **Trámites** (`matricula` / registration cases): creation requires an invoice PDF, then tracks full lifecycle state changes, checklist documents, payments, shipping, and closure / reopen / cancel actions.
2. **Servicios** (non-matricula services): transfer, lien registration/removal/modification, duplicate documents/plates, color/motor/service changes, account transfer, and custom services with template-driven `serviceData`.

Core behavior:
- Assigns a unique business ID per case using `year + dealer code + consecutive`.
- Stores uploaded files in structured local storage and keeps versioned metadata in DB.
- Maintains full audit history for status transitions.
- Detects overdue cases from configurable alert rules.
- Generates and exports `cuenta de cobro` (account statement) data and filled PDF output from a template.
- Exposes operational reporting endpoints (summary, paginated list, CSV export).
- Includes catalog, client, user, auth, payment, shipment, and file-download modules.
- Designed to be consumed by the SGM Desktop (Electron) client.

## Tech Stack

| Layer | Technology |
|---|---|
| Framework | `NestJS 11`, `TypeScript`, `Node.js 20` |
| ORM / DB | `Prisma 7` (`@prisma/adapter-pg`) + `PostgreSQL` |
| Auth | `jsonwebtoken` + `bcrypt` |
| Hardening | `Helmet`, CORS allowlist, `Nest Throttler`, global validation + normalized error filter |
| File handling | `Multer` (multipart disk uploads), local filesystem storage, versioned metadata |
| PDF | `pdf-lib` — PDF validation + template-coordinate generation |
| API docs | `Swagger` (gated by `SWAGGER_ENABLED` env flag, exposed at `/docs`) |
| DevOps | Docker multi-stage build, `docker-compose` for Postgres, `systemd` timer integration for ops scripts |
| Testing | `Jest` + `Supertest` |

## Security and Hardening

- Global `ValidationPipe` with `whitelist: true` and `forbidNonWhitelisted: true`.
- JWT guard on all protected routes.
- Global API throttling and stricter login throttling (`LOGIN_RATE_TTL_MS` / `LOGIN_RATE_LIMIT`).
- CORS controlled via `CORS_ORIGINS` allowlist.
- Swagger docs gated by `SWAGGER_ENABLED` / environment.
- PDF uploads constrained by MIME type, max file size, and max page count.
- CSV export sanitized against formula-injection attacks.
- File download errors do not leak internal storage paths.

## API Modules

| Module | Routes |
|---|---|
| `auth` | `/auth/login`, `/auth/me` |
| `tramites` | `/tramites` — full lifecycle CRUD |
| `servicios` | `/servicios` — service lifecycle |
| `files` | `/files/:id/download` |
| `payments` | payment records + attachments |
| `shipments` | shipment guide linking |
| `reports` | `/reports/summary`, `/reports/tramites`, `/reports/export.csv` |
| `catalogs` | `/catalogs/*` — lookup data |

## Environment Variables

Copy `.env.example` to `.env` and set real values:

```bash
cp .env.example .env
```

Main variables:

| Variable | Purpose |
|---|---|
| `PORT` | HTTP port |
| `NODE_ENV` | `development` \| `production` |
| `DATABASE_URL` | Postgres connection string |
| `JWT_SECRET` | Token signing secret |
| `JWT_EXPIRES_IN` | Token lifetime (e.g. `8h`) |
| `STORAGE_ROOT` | Absolute path for uploaded file storage |
| `MAX_PDF_PAGES` | Max pages allowed per PDF upload |
| `MAX_UPLOAD_MB` | Max file size per upload |
| `LOGIN_RATE_TTL_MS` | Throttle window for login endpoint |
| `LOGIN_RATE_LIMIT` | Max login attempts per window |
| `API_RATE_TTL_MS` | Global throttle window |
| `API_RATE_LIMIT` | Max requests per global window |
| `CORS_ORIGINS` | Comma-separated allowed origins |
| `SWAGGER_ENABLED` | `true` to expose `/docs` |
| `REPORTS_SUMMARY_MAX_ROWS` | Row cap for summary report |

## Install

```bash
npm install
```

## Run

```bash
# development
npm run start:dev

# production
npm run build
npm run start:prod
```

## Database

```bash
# generate Prisma client
npx prisma generate

# apply migrations
npx prisma migrate deploy

# seed base data
npx prisma db seed
```

## Key Scripts

| Script | Purpose |
|---|---|
| `npm run start:dev` | Start with hot-reload |
| `npm run build` | Compile to `dist/` |
| `npm run start:prod` | Run compiled build |
| `npm run test` | Unit tests (Jest) |
| `npm run test:e2e` | End-to-end tests (Supertest) |

## Notes

- This backend is designed to be consumed by the SGM Desktop Electron app.
- Keep `.env` out of source control and rotate all secrets before any production deployment.
- Ops scripts in `ops/` handle yearly data cleanup and client deduplication; integrate with `systemd` timers for scheduled execution.
