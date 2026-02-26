# Gestion-De-Tramites-vehiculares-Rtramite

Monorepo for the SGM (Sistema de Gestión de Trámites Vehiculares) project.

## Structure

- [`sgm-backend/`](./sgm-backend) — NestJS REST API backend (TypeScript, Prisma, PostgreSQL)
- [`sgm-frontend/`](./sgm-frontend) — React + Electron desktop frontend (TypeScript, Vite)

## Getting Started

### Backend

```bash
cd sgm-backend
cp .env.example .env   # fill in your environment variables
npm install
npm run start:dev
```

### Frontend

```bash
cd sgm-frontend
cp .env.example .env   # fill in your environment variables
npm install
npm run dev
```