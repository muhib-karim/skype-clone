# Skype Clone

A Skype-style communication app in a TypeScript monorepo: real-time messaging, WebRTC audio/video calls, contacts, group chat, file sharing and Stripe payments. The backend is Node.js/Express with Socket.io and MongoDB; the web client is React with Vite. Mobile (React Native) and desktop (Electron) clients are in progress.

## Why it's interesting

- **Real-time product, end to end.** One codebase covers the REST API, WebSocket events, WebRTC call signalling and the React client that consumes them.
- **Shared contract.** Types and a client-side crypto module live in `packages/shared` and are used by every client.
- **Built with a multi-agent workflow.** The work was split across three roles (backend, frontend, DevOps), each with its own scope and coordination rules. See [AGENTS.md](AGENTS.md) and [COORDINATION.md](COORDINATION.md).

## How it works

```
React web (Vite, :3000) ──REST /api/*──▶ Express API (:5000) ──▶ MongoDB (Mongoose)
        │                                      │                    Redis (optional)
        └──── Socket.io ─────────────────────▶ socket handlers ──▶ Stripe (payments)
              messages, presence, calls,
              WebRTC offer/answer/ICE relay
```

- **REST API** (`packages/backend/src/routes`): auth, users, messages, calls, contacts, groups, payments and encryption public keys. All `/api/*` routes are behind Helmet, CORS and rate limiting.
- **Socket.io** (`packages/backend/src/socket/socketHandlers.ts`) handles `message:send`, `message:mark-read`, `call:initiate|accept|decline|end`, `group:create|join|leave`, `contact:add|accept|block`, and relays `webrtc:offer`, `webrtc:answer` and `webrtc:ice-candidate` between peers.
- **Calls**: the web client (`packages/web/src/components/VideoCall.tsx`) uses `simple-peer` for WebRTC. The server only does signalling.
- **Auth**: JWT access and refresh tokens, passwords hashed with bcrypt.
- **Encryption**: `packages/shared/src/crypto.ts` implements RSA-OAEP key exchange and AES-GCM message encryption with the Web Crypto API. The backend stores and serves public keys (`/api/encryption/*`). End-to-end encryption is still listed as remaining work in [AGENTS.md](AGENTS.md).

## Quickstart

Prerequisites: Node.js 18+, and MongoDB running locally (or Docker).

### Local

```bash
npm install                               # installs all workspaces
cp .env.example packages/backend/.env     # then edit secrets (JWT, Stripe)
npm run dev:backend                       # API on http://localhost:5000
npm run dev:web                           # web app on http://localhost:3000
```

### Docker

```bash
npm run docker:dev    # docker-compose up --build: MongoDB, Redis, backend, web
npm run docker:down   # stop and remove volumes
```

### Tests and checks

```bash
cd packages/backend && npm test   # Jest + Supertest + mongodb-memory-server
npm run validate:env              # checks required environment variables
```

## Status

| Area | State |
|------|-------|
| Backend API, models, Socket.io handlers | Implemented |
| Web client (auth, chat, calls, file upload, payments, theming) | Implemented |
| Encryption module and public-key endpoints | Code present; end-to-end flow listed as remaining work |
| Mobile app (React Native) | In progress: screens and services exist, no native `android/` or `ios/` projects yet |
| Desktop app (Electron) | In progress |
| Push notifications | In progress (`packages/mobile/src/services/pushNotifications.ts`) |
| Tests | Backend only: `tests/models/User.test.ts` and `tests/routes/auth.test.ts` |
| CI/CD | Workflows in `.github/workflows` are defined but trigger on `main`/`develop`, while the default branch is `master`, so they have not run here |

There are no measured results in the repo yet. Coverage (>95%), API latency (<100 ms) and uptime figures in the docs are targets, not measurements. Artillery load-test scenarios are in [`performance/`](performance/) (`npm run perf:auth`, `npm run perf:messaging`), but no results have been committed.

## API overview

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/api/auth/register`, `/api/auth/login`, `/api/auth/refresh` | Account and tokens |
| GET | `/api/auth/me` | Current user |
| POST / GET | `/api/messages`, `/api/messages/:userId` | Send / fetch direct messages |
| GET | `/api/messages/group/:groupId` | Group messages |
| POST / PUT | `/api/calls`, `/api/calls/:callId/{accept,decline,end}` | Call lifecycle |
| GET | `/api/calls/history` | Call history |
| POST | `/api/payments/create-intent`, `/api/payments/webhook` | Stripe payment intent and webhook |
| PUT / GET | `/api/encryption/public-key`, `/api/encryption/public-key/:userId` | Public-key exchange |

Contacts, groups and users have their own routers under `/api/contacts`, `/api/groups` and `/api/users`.

## Project layout

```
packages/
  backend/    Express API, Mongoose models, Socket.io handlers, Jest tests
  web/        React + Vite client
  shared/     Shared types and crypto utilities
  mobile/     React Native client (in progress)
  desktop/    Electron client (in progress)
docs/         ADRs, backlog, testing strategy, CI/CD and observability plans
performance/  Artillery load-test scenarios
scripts/      Env validation, test orchestration, release checklist
nginx/        Reverse-proxy config for the production compose profile
```

## Documentation

- [AGENTS.md](AGENTS.md): roles, commands and conventions
- [COORDINATION.md](COORDINATION.md): how work was split between roles
- [PROJECT_MANAGEMENT.md](PROJECT_MANAGEMENT.md): epics, sprint status and risks
- [DEVOPS.md](DEVOPS.md): Docker and deployment notes
- [docs/architecture/](docs/architecture/): ADRs (monorepo structure, real-time architecture)
- [docs/quality/TESTING_STRATEGY.md](docs/quality/TESTING_STRATEGY.md): testing approach and coverage targets
- [docs/deployment/CI_CD_PIPELINE.md](docs/deployment/CI_CD_PIPELINE.md): planned pipeline
- [docs/monitoring/OBSERVABILITY_STACK.md](docs/monitoring/OBSERVABILITY_STACK.md): planned monitoring

## License

MIT (declared in [package.json](package.json)).
