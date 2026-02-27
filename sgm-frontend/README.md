# SGM Desktop (Electron + React)

Internal Windows desktop app for operations teams to manage `tramites` (vehicle registration workflows) and `servicios` (non-registration workflows) end-to-end.

It covers intake, creation, lifecycle/state tracking, document checklist + versioned PDF uploads, payments with attachments, shipment guide linking, account statement (`cuenta de cobro`) totals + PDF generation, overdue monitoring, CSV reporting, and user administration.

The app is backend-first (SGM API) and includes controlled fallback modes (`off | auto | force`) for local/mock continuity during development or API instability.

## What This App Does

- Authenticates users and persists the session token securely (Electron `safeStorage`).
- Shows the main tray of tramites with filters and detail view.
- Creates new tramites with required invoice PDF.
- Manages tramite lifecycle:
  - state changes
  - checklist and file uploads
  - payments with attachments
  - shipment guide linking
  - account statement (`cuenta de cobro`) PDF download
- Manages non-matricula servicios:
  - create from templates
  - edit service data
  - track service states
  - record service payments
- Displays overdue workflows, business reports dashboard, and CSV exports.
- User administration.

## Stack

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

## Security Notes

- Electron renderer runs with `contextIsolation`, `sandbox`, and `webSecurity`.
- Auth token lives exclusively in the main process; the renderer accesses it only through typed IPC calls.
- New windows and external navigation are blocked by default from main process.
- Mock mode defaults to `off` to avoid silent fallback to fake data in production use.

## Project Structure

- `src/` — React application (routes, pages, API clients)
- `electron/` — Electron main process, preload, IPC auth bridge
- `public/` — static assets

## Environment

Create `.env` from `.env.example`:

```bash
cp .env.example .env
```

Variables:

| Variable | Purpose |
|---|---|
| `VITE_API_BASE_URL` | SGM backend base URL |
| `VITE_USE_MOCKS` | Legacy toggle (kept for compatibility) |
| `VITE_MOCK_MODE` | `off` \| `auto` \| `force` — backend fallback mode |

## Install

```bash
npm install
```

## Run

```bash
# web renderer only
npm run dev

# electron + renderer
npm run electron:dev
```

## Build

```bash
# web build
npm run build

# electron main build
npm run electron:build
```

## Package Windows App

```bash
# nsis installer
npm run dist:win

# portable exe
npm run dist:portable
```

Artifacts are generated in `release/`.

## Notes

- If Electron does not open and exits instantly in your shell, verify `ELECTRON_RUN_AS_NODE` is not set.
- The app expects the backend API documented in the `sgm-backend` directory.
- Set `VITE_MOCK_MODE=auto` to allow graceful degradation when the API is temporarily unreachable during development.
