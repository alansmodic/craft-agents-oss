# Deploying Craft Agents to Railway

This guide walks through deploying the headless Craft Agents server to [Railway](https://railway.com) so you can:

- Open the **WebUI** in a browser at `https://<your-app>.up.railway.app`
- Connect the **desktop app** or **CLI** as a thin client over `wss://<your-app>.up.railway.app`

The repo includes `railway.json` and `Dockerfile.server`, so most of the setup is just clicking through Railway's dashboard.

## What gets deployed

A single service running the Craft Agents headless server. It serves three things on the same public port:

- `wss://…/` — WebSocket RPC for desktop/CLI clients
- `https://…/` — WebUI (login → SPA)
- `https://…/health` — unauthenticated health probe used by Railway

A persistent **Volume** holds your config, sessions, encrypted credentials, OAuth tokens, workspaces, and skills.

## One-time setup

### 1. Create the project

1. In the Railway dashboard, **New Project → Deploy from GitHub repo**, pick your fork of this repo.
2. Railway detects `railway.json` and uses `Dockerfile.server`. The first build takes 5–10 minutes (Bun install + Vite build of the WebUI).

### 2. Attach a volume

The server stores everything under `CRAFT_CONFIG_DIR` (defaults to `~/.craft-agent`). Without a volume, every redeploy wipes your sessions and credentials.

1. In the service, **Variables → New Volume**.
2. **Mount path:** `/data`
3. **Size:** start with `5 GB` (see [Volume sizing](#volume-sizing) below — you can resize later without recreating).

### 3. Set environment variables

In **Variables**, add the following:

| Variable | Value | Notes |
|---|---|---|
| `CRAFT_SERVER_TOKEN` | `<random>` | **Required.** Bearer token for clients. Generate with `openssl rand -hex 32`. ≥16 chars, validated for entropy. |
| `CRAFT_CONFIG_DIR` | `/data` | Points all server state at the mounted volume. |
| `CRAFT_WEBUI_PASSWORD` | `<password>` | Optional. A short password for the browser login form. Falls back to `CRAFT_SERVER_TOKEN` if unset (you'd have to type the full token). |
| `CRAFT_WEBUI_WS_URL` | `wss://${{RAILWAY_PUBLIC_DOMAIN}}` | Tells the WebUI which `wss://` URL the browser should connect to. Use Railway's variable reference syntax. |

The Dockerfile already sets `CRAFT_RPC_HOST=0.0.0.0`, `CRAFT_WEBUI_DIR`, and the WhatsApp worker paths — don't override those.

`railway.json` injects `CRAFT_RPC_PORT=$PORT` and passes `--allow-insecure-bind` (Railway terminates TLS at the edge, so the container speaks plain `ws://` internally — the bind guard would otherwise refuse to start).

### 4. Generate a public domain

**Settings → Networking → Generate Domain**. Railway exposes the service's `$PORT` over HTTPS/WSS automatically.

### 5. Deploy

Push to your default branch (or click **Deploy** in the dashboard). Watch the deploy logs — on success you'll see:

```
CRAFT_SERVER_URL=ws://0.0.0.0:<port>
CRAFT_SERVER_TOKEN=<your-token>
CRAFT_WEBUI_URL=http://0.0.0.0:<port>
```

The internal `ws://` is correct — Railway wraps it as `wss://` at the edge.

## Connecting

### Browser (WebUI)

Open `https://<your-app>.up.railway.app`, log in with `CRAFT_WEBUI_PASSWORD` (or the full `CRAFT_SERVER_TOKEN` if you didn't set a separate password). The first time you log in, you'll set up your workspace and add an LLM connection (Anthropic API key, etc.) through the UI.

### Desktop app (thin client)

```bash
CRAFT_SERVER_URL=wss://<your-app>.up.railway.app \
CRAFT_SERVER_TOKEN=<your-token> \
bun run electron:start
```

The desktop app renders the UI locally; sessions, tools, and LLM calls run on the Railway server.

### CLI

```bash
export CRAFT_SERVER_URL=wss://<your-app>.up.railway.app
export CRAFT_SERVER_TOKEN=<your-token>

bun run apps/cli/src/index.ts ping
bun run apps/cli/src/index.ts sessions
```

## Volume sizing

Railway volumes can be **resized later** from the dashboard, so you don't need to get this exactly right up front:

- **5 GB** is plenty for a single user — sessions are stored as JSONL, credentials are tiny, OAuth state is tiny. Months of use typically fits in a few hundred MB.
- Bump to **20–50 GB** if you plan to use the server-side filesystem source on large repos, or if you'll attach lots of PDFs/images to sessions (those get persisted alongside session data).

To resize: **Service → Volume → Edit Size**. Railway grows the volume in place; no data loss.

## Operational notes

- **Single instance only.** The server writes a startup lock file in `CRAFT_CONFIG_DIR` and is single-tenant by design. Keep `numReplicas: 1`. Don't enable horizontal scaling.
- **Cold starts** take 10–30s while the session manager initialises. The healthcheck timeout in `railway.json` is 180s to be safe.
- **Resource sizing.** Idle memory is ~300–500 MB; under load (long Claude sessions with many tool calls) plan for 1–2 GB. Build needs ~4 GB RAM (Vite bumps Node heap to 4 GB).
- **Updates.** Push to your default branch and Railway redeploys. Your volume persists across deploys.

## Troubleshooting

**Healthcheck fails / deploy stuck.** Check the deploy logs for the `CRAFT_SERVER_URL=...` line. If you see "Refusing to bind to a network address without TLS", the start command didn't pick up `--allow-insecure-bind` — verify `railway.json` was committed. If you see permission errors on `/data`, see below.

**`EACCES` writing to `/data`.** Railway volumes are owned by `root` by default; the container runs as the non-root `craftagents` user. Open a Railway shell on the service and run `chmod 777 /data` once, then redeploy. (You can also run `chown -R craftagents:craftagents /data` if you prefer tighter perms.)

**WebUI says "Cannot connect to server" after login.** `CRAFT_WEBUI_WS_URL` is wrong or unset — make sure it's `wss://${{RAILWAY_PUBLIC_DOMAIN}}` with the Railway variable reference syntax (double curly braces).

**Token rejected.** `CRAFT_SERVER_TOKEN` must be ≥16 characters with reasonable entropy (rejects single-char repeats, warns on <8 unique chars). Use `openssl rand -hex 32`.

**Lock file conflict after a crash.** The server takes a lock at `$CRAFT_CONFIG_DIR/.server.lock`. It auto-clears on PID-reuse detection across container restarts, but if you see `Another server instance is already running`, delete `/data/.server.lock` via a Railway shell and redeploy.
