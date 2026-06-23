# Chat App

A mobile chat application: a React Native (Expo) client talking to an Express + Socket.io
backend over WebSockets. Users sign up / log in and chat in a single global room. Redis stores
accounts and recent message history; the backend and Redis run in Docker.

> **Status:** under active development. See [`PLAN.md`](./PLAN.md) for the implementation
> roadmap and [`CLAUDE.md`](./CLAUDE.md) for deeper project context.

## Tech stack

| Layer    | Tech                                          |
| -------- | --------------------------------------------- |
| Frontend | Expo, React Native, TypeScript (`client/`)    |
| Backend  | Node.js, Express, Socket.io, TypeScript (`server/`) |
| Storage  | Redis (accounts + message history)            |
| Auth     | bcrypt-hashed passwords + JWT                 |
| Infra    | Docker Compose                                |

## Layout

```
/
├── docker-compose.yaml   # app (server) + redis services
├── shared/               # shared TS types
├── server/               # Express + Socket.io backend
└── client/               # Expo app
```

## Getting started

### Backend (Docker)

```bash
docker compose up --build      # starts the server (port 3000) and Redis
curl localhost:3000/health     # health check
```

### Backend (local dev)

```bash
cd server
npm install
npm run dev                    # needs Redis reachable at $REDIS_URL
```

### Frontend

```bash
cd client
npx expo start
```

On a physical device, point the app's API base URL at your host machine's LAN IP (not
`localhost`).

## Environment variables (server)

| Variable     | Description                          | Default                  |
| ------------ | ------------------------------------ | ------------------------ |
| `PORT`       | HTTP / WebSocket port                | `3000`                   |
| `REDIS_URL`  | Redis connection URL                 | `redis://localhost:6379` |
| `JWT_SECRET` | Token signing secret (set in prod!)  | dev placeholder          |
