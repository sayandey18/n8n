# n8n production stack

Postgres + n8n + isolated Code-node task runners (JavaScript and Python) +
Nginx Proxy Manager (TLS/reverse proxy) + Watchtower (weekly auto-updates).

## Files

| File | Purpose |
|---|---|
| `docker-compose.yml` | The stack |
| `.env.example` | Copy to `.env` and fill in |
| `n8n-task-runners.json` | Allow-list of modules the Code node can import (JS + Python) |

## 1. Prerequisites

- A server with Docker Engine and the Compose plugin installed.
- A domain name with an A/AAAA record pointing at the server's public IP.
- Ports 80 and 443 open to the internet.

## 2. First-time setup

```bash
cp .env.example .env
openssl rand -hex 32   # run twice, paste into N8N_ENCRYPTION_KEY and N8N_RUNNERS_AUTH_TOKEN
```

Edit `.env`: set `N8N_HOST` to your domain, `GENERIC_TIMEZONE` to your IANA
timezone, a strong `POSTGRES_PASSWORD`, and the two secrets above.

```bash
docker compose up -d
```

## 3. Configure Nginx Proxy Manager

1. Open `http://<server-ip>:81`. Default login is `admin@example.com` /
   `changeme` — **change it immediately**.
2. **Hosts → Proxy Hosts → Add Proxy Host**:
   - Domain Names: your `N8N_HOST` value
   - Scheme: `http`, Forward Hostname/IP: `n8n`, Forward Port: `5678`
   - Toggle **Websockets Support** on (n8n's editor needs it for live updates)
3. **SSL tab**: request a new Let's Encrypt certificate, enable **Force SSL**.
4. Save. Visit `https://<your-domain>` — you should land on n8n's first-run
   "Set up owner account" screen.

## 4. How Code node JavaScript + Python support works

n8n never runs user Code-node scripts inside the main process in this stack.
The `n8n-runners` sidecar container executes them in isolation — this is the
setup n8n's own docs require for any instance holding real credentials.

- **JavaScript** runs natively.
- **Python** runs on n8n's native Python runner (`N8N_NATIVE_PYTHON_RUNNER=true`),
  a real CPython interpreter — not the older browser-based Pyodide/WASM subset.

Both `n8nio/n8n` and `n8nio/runners` **must be the same version**. This stack
pins both to `:latest` so Watchtower updates them together weekly; if you
ever switch to pinned version tags, update both at once.

### Adding more Code node modules/packages

For security, the Code node can only import what's explicitly allow-listed
in `n8n-task-runners.json` (setting these as plain container environment
variables does *not* work — the launcher's config file takes precedence).

- JS builtins → `NODE_FUNCTION_ALLOW_BUILTIN`
- JS npm packages already in the runner image → `NODE_FUNCTION_ALLOW_EXTERNAL`
- Python standard library → `N8N_RUNNERS_STDLIB_ALLOW`
- Python packages already in the runner image → `N8N_RUNNERS_EXTERNAL_ALLOW`

To add a third-party package (e.g. `pandas`, `lodash`) that isn't already
bundled in the `n8nio/runners` image, you need to extend that image and
rebuild — see [n8n's task runner docs](https://docs.n8n.io/deploy/host-n8n/configure-n8n/set-up-task-runners/#adding-extra-dependencies)
for the `Dockerfile` pattern (`pnpm add <pkg>` / `uv pip install <pkg>`),
then reference `image: your-registry/n8n-runners-custom:tag` in place of
`n8nio/runners:latest` for the `n8n-runners` service.

After editing `n8n-task-runners.json`, apply changes with:

```bash
docker compose restart n8n-runners
```

## 5. Weekly auto-updates

Watchtower checks every Sunday at 04:00 (server timezone) for new images and
updates any container labeled `com.centurylinklabs.watchtower.enable=true` —
that's `n8n`, `n8n-runners`, and `nginx-proxy-manager`. It restarts one
container at a time and prunes the old image afterward.

`postgres` is intentionally excluded and pinned to `postgres:16-alpine` — an
automatic major-version bump could break the data directory. Upgrade it
yourself, deliberately, after a backup:

```bash
# edit the image tag in docker-compose.yml, then:
docker compose pull postgres
docker compose up -d postgres
```

To trigger an update check manually instead of waiting for the schedule:

```bash
docker compose restart watchtower
docker logs watchtower --tail 50
```

## 6. Backups

Back up these named volumes regularly (they hold everything that matters):

- `postgres_data` — all workflows, credentials (encrypted), execution history
- `n8n_data` — n8n's local config directory
- `npm_data`, `npm_letsencrypt` — Nginx Proxy Manager config and certificates

```bash
docker run --rm -v n8n-stack_postgres_data:/data -v $(pwd):/backup alpine \
  tar czf /backup/postgres_data_$(date +%F).tar.gz -C /data .
```

(Volume names are prefixed with the compose project directory name by
default — check `docker volume ls` if the above doesn't match.)

Also keep `.env` and `n8n-task-runners.json` backed up somewhere safe
(outside of version control, since `.env` holds secrets).

## 7. Security notes

- n8n's port (5678) is **not** published to the host — only reachable
  through the `proxy` Docker network by Nginx Proxy Manager. Only 80, 443,
  and NPM's 81 admin port are exposed.
- Restrict access to port 81 (NPM admin UI) with a firewall rule to your
  own IP, or put it behind NPM's own access-list feature once TLS is live.
- `N8N_ENCRYPTION_KEY` must never be lost or changed after credentials have
  been saved, or those credentials become unreadable.
- Watchtower has full access to the Docker socket (needed to manage
  containers) — keep the host itself patched and access-controlled.
