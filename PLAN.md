# Implementation Plan: Mobile Chat App

Expo (React Native) + Express/Socket.io + Redis. Project context lives in `CLAUDE.md`.

## Goal

A working chat MVP — **single global room**, **full accounts + auth** (hashed passwords + JWT),
**Redis** as the single data store, **Expo + React Native + TypeScript** frontend, backend
(Express + Socket.io) in **Docker** next to a Redis container.

Confirmed decisions: single global room · Redis storage · full auth · TypeScript frontend.

## Approach

Work proceeds in small, independently-committable steps, in dependency order. Each step is one
reviewable commit with its own files, verification, and suggested commit message. Steps are not
chained — each is reviewed and committed before the next begins.

## Target layout (reached incrementally)

```
/
├── CLAUDE.md · PLAN.md      # context · this plan
├── docker-compose.yaml      # app (server) + redis
├── shared/                  # shared TS types (Message, auth payloads)
├── server/                  # Express + Socket.io backend (TS)
│   ├── Dockerfile · .dockerignore
│   ├── package.json · tsconfig.json
│   └── src/ index.ts · redis.ts · auth.ts · chat.ts
└── client/                  # Expo app (TS)
```

## Steps

### Step 1 — Cleanly finish the staged `server/` move ✅
Commit the already-staged renames into `server/` (no content/Docker changes).
- Verify: `git status` shows only renames; `cd server && npm install && npm run build` compiles the stub.
- Commit: `chore: move backend into server/`

### Step 2 — Project docs: CLAUDE.md + PLAN.md
Create `CLAUDE.md` (project context, no plan) and `PLAN.md` (this plan).
- Commit: `docs: add CLAUDE.md context and PLAN.md`

### Step 3 — Fix the Docker build for the new layout
Move `Dockerfile` + `.dockerignore` into `server/`; repoint build stage to the server context;
add `EXPOSE 3000`. In `docker-compose.yaml` set `build.context: ./server` and `ports: ["3000:3000"]`.
- Verify: `docker compose build` succeeds; `docker compose up app` runs the hello-world.
- Commit: `fix(docker): build server/ context and expose port`

### Step 4 — Express HTTP server + `/health`
Add `express`, `cors` (+ types, `ts-node`, `nodemon`); add `dev` script. Rewrite `src/index.ts` as
an Express app with `express.json()` + `cors()`, `GET /health`, listening on `PORT` (default 3000).
- Verify: `curl localhost:3000/health` → OK.
- Commit: `feat(server): express app with health endpoint`

### Step 5 — Redis service + client
Add a `redis` service (`redis:7-alpine`, `--appendonly yes`, volume `redis-data:/data`) to compose;
add `depends_on` + `REDIS_URL` to `app`. Add `ioredis`; create `src/redis.ts` (one client from
`REDIS_URL`); log connect/error.
- Verify: server logs "redis connected"; `docker compose exec redis redis-cli ping` → PONG.
- Commit: `feat(server): add redis service and client`

### Step 6 — Accounts + auth (signup/login, JWT)
Add `bcryptjs`, `jsonwebtoken`, `dotenv` (+ types); `JWT_SECRET` env. `redis.ts`: `createUser`
(`HSETNX user:<username>`, reject dup), `getUser` (`HGETALL`). `src/auth.ts`: `POST /auth/signup`,
`POST /auth/login` (bcrypt), `signToken`/`verifyToken`; mount router in `index.ts`.
- Verify: signup → token; duplicate → 409; login wrong password → 401.
- Commit: `feat(server): account signup/login with jwt`

### Step 7 — Realtime chat over Socket.io
Add `socket.io`; wrap app in `http.createServer` and attach Socket.io `Server`. `redis.ts`:
`pushMessage` (`RPUSH` + `LTRIM -100 -1`), `getRecentMessages`. `src/chat.ts`: handshake-auth
middleware (`socket.handshake.auth.token` → `verifyToken`); on connect emit `history`; on
`message {text}` build `{id, username, text, ts}`, persist, `io.emit('message', msg)`.
- Verify: two authed clients exchange messages live; `history` loads on reconnect.
- Commit: `feat(server): socket.io global chat with history`

### Step 8 — Shared types
Create `shared/` TS types: `Message`, `User`, auth request/response payloads; reference from server.
- Verify: server type-checks (`npm run build`).
- Commit: `chore: shared message/auth types`

### Step 9 — Scaffold the Expo (TypeScript) app
`npx create-expo-app@latest client --template blank-typescript`; add `socket.io-client`,
`expo-secure-store`, `@react-navigation/native`, `@react-navigation/native-stack`,
`react-native-screens`, `react-native-safe-area-context`.
- Verify: `cd client && npx expo start` boots.
- Commit: `feat(client): scaffold expo typescript app`

### Step 10 — Auth flow (screens + token storage + API client)
`client/src/api.ts` (configurable `BASE_URL`, `signup`/`login`); `client/src/auth.tsx` (auth context,
JWT in `expo-secure-store`); navigation: no token → AuthScreen, token → ChatScreen placeholder.
- Verify: sign up / log in → token persists across reload; bad creds show error.
- Commit: `feat(client): auth screens with secure token storage`

### Step 11 — Chat screen wired to Socket.io
`client/screens/ChatScreen.tsx`: `io(BASE_URL, { auth: { token } })`; `FlatList` of messages (own vs
other styling), handle `history` + `message`, input emits `message`. Reuse `shared/` types.
- Verify: two users on two devices send messages, both receive live; restart `app` container (not
  redis) → history persists (Redis AOF volume).
- Commit: `feat(client): realtime chat screen`

## Notes / risks

- `JWT_SECRET` in compose is a dev placeholder — production needs a real secret.
- Single global room keeps Socket.io simple (no rooms); multi-room is a later `socket.join(room)` extension.
- Steps 1–8 are backend/docs and verifiable via curl/redis-cli; 9–11 need the Expo toolchain/device.

## Codespaces: port forwarding & testing

We develop in a GitHub Codespace (remote container). The server runs in the container; clients
may be local. There are **two ways to reach the server**, and they behave differently:

| URL | Reaches server via | Use for |
| --- | --- | --- |
| `http://localhost:3000` | direct, inside the container | REST Client / curl run inside the Codespace |
| `https://<codespace>-3000.app.github.dev` | GitHub's forwarding proxy | browsers and external devices (e.g. the phone running Expo) |

- **Forwarded ports are private by default.** The proxy then requires a GitHub session cookie.
  Requests without it (REST Client, curl, another device) get a **302 redirect to a GitHub login
  page** — which surfaces in browsers as a misleading **"CORS error" / "failed to fetch."** The
  request never reaches Express, so it is *not* an app CORS problem (our `cors()` returns
  `Access-Control-Allow-Origin: *`).
- **Fix — make the port public** so the proxy passes requests straight through:
  ```
  gh codespace ports visibility 3000:public -c "$CODESPACE_NAME"
  ```
  Verify with an unauthenticated request: `curl https://<codespace>-3000.app.github.dev/health`
  → `200 {"status":"ok"}`. Revert with `...visibility 3000:private...`.
- **Caveats:** public = anyone with the URL can reach it (fine for dev only). Visibility resets on
  Codespace rebuild — to make it stick, add a `.devcontainer/devcontainer.json` `portsAttributes`
  entry for 3000. For backend-only testing, prefer `localhost` (no proxy, never hits this issue).

## Operational notes

- Don't spawn detached dev servers (`(node ... &)`): in this sandbox the resulting process can't
  be signalled later (`kill` → `Operation not permitted`) and squats on port 3000. Use managed
  background tasks, or `npm run dev` in a terminal, so the process stays controllable.
