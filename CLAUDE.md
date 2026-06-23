# CLAUDE.md

Project context for Claude Code. (The implementation plan lives in `PLAN.md`.)

## What this is

A mobile chat application. A React Native (Expo) app talks to an Express + Socket.io
backend over WebSockets. Users have real accounts and chat in a single global room.

## Tech stack

- **Frontend:** Expo + React Native, TypeScript (`client/`).
- **Backend:** Node.js + Express + Socket.io, TypeScript (`server/`).
- **Data store:** Redis — backs both user accounts and recent message history.
- **Auth:** account signup/login with bcrypt-hashed passwords and JWTs.
- **Infra:** backend + Redis run via Docker Compose.
- **Shared:** common TypeScript types in `shared/`.

## Product decisions

- **Single global chat room** — everyone shares one stream (no rooms/DMs yet).
- **Full accounts + auth** — signup/login, hashed passwords, JWT-authenticated sockets.
- **Redis** as the single data store: `user:<username>` hashes for accounts, a capped
  `chat:messages` list for history (last ~100 messages).
- **TypeScript** on both frontend and backend.

## Repo layout

```
/
├── docker-compose.yaml   # app (server) + redis services
├── shared/               # shared TS types (Message, User, auth payloads)
├── server/               # Express + Socket.io backend (TS)
│   ├── Dockerfile · .dockerignore
│   ├── package.json · tsconfig.json
│   └── src/ index.ts · redis.ts · auth.ts · chat.ts
└── client/               # Expo app (TS)
```

## Running

- **Backend (Docker):** `docker compose up --build` — starts `app` (port 3000) and `redis`.
  Health check: `curl localhost:3000/health`.
- **Backend (local dev):** `cd server && npm run dev` (needs a Redis reachable at `REDIS_URL`).
- **Frontend:** `cd client && npx expo start`. On a physical device set the API base URL to the
  host's LAN IP (not `localhost`).

## Environment variables (server)

- `PORT` — HTTP/socket port (default `3000`).
- `REDIS_URL` — e.g. `redis://redis:6379` in Docker, `redis://localhost:6379` locally.
- `JWT_SECRET` — token signing secret. The value in compose is a dev placeholder; use a real
  secret in production.

## Conventions

- Work lands in small, independently-committable steps; each step is one reviewable commit.
- Keep frontend and backend agreeing on wire shapes via `shared/` types.
