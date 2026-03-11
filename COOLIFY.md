# Deploy PostHog to Coolify

This repo is wired for [Coolify](https://coolify.io) (or any Docker Compose host) with the hobby stack.

## 1. Link repo and push

```powershell
cd "c:\Users\youssef.george\Downloads\posthog-master"
git add .
git commit -m "chore: add Coolify compose and compose scripts"
git branch -M main
git push -u origin main
```

(Use a [GitHub token](https://github.com/settings/tokens) if prompted.)

## 2. In Coolify

1. **New resource** → **Docker Compose**.
2. **Repository**: `https://github.com/youssef-george/Posthog-openserver.git`, branch `main`.
3. **Compose file**: set to `docker-compose.coolify.yml`. It extends `docker-compose.base.yml` in the same repo; both must be present (they are). If your Coolify supports multiple files, you can use `docker-compose.base.yml` and `docker-compose.coolify.yml` together.
4. **Root directory**: leave default (repo root).
5. **Build**: Coolify will build services that have `build: context: ./rust` (capture, replay-capture, etc.). The `web` and `worker` use pre-built images from `REGISTRY_URL`/`POSTHOG_APP_TAG`.

### If Coolify supports only one compose file

Merge base + coolify into one file, or use the **full** compose in one file. For “one file” setups, you can:

- Set **Compose file** to `docker-compose.coolify.yml` and ensure `docker-compose.base.yml` is in the same repo root (extends will resolve).
- Or duplicate the base content into `docker-compose.coolify.yml` so it’s self-contained (not ideal for upkeep).

## 3. Environment variables (Coolify UI)

Set these in the Coolify environment variables for the Compose resource (no quotes in UI unless the value has spaces):

| Variable | Required | Example / note |
|----------|----------|----------------|
| `DOMAIN` | Yes | Your public domain, e.g. `posthog.example.com` |
| `POSTHOG_SECRET` | Yes | Long random string, e.g. `openssl rand -hex 32` |
| `ENCRYPTION_SALT_KEYS` | Yes | e.g. `openssl rand -hex 16` |
| `REGISTRY_URL` | Yes | `ghcr.io/posthog/posthog` (or your registry) |
| `POSTHOG_APP_TAG` | Yes | `latest` or a tag |
| `POSTHOG_NODE_TAG` | No | `latest` (for plugins image) |
| `TLS_BLOCK` | No | Leave empty for production Let’s Encrypt |
| `OPT_OUT_CAPTURE` | No | `false` (default) |

Generate secrets (run locally):

```bash
openssl rand -hex 32   # POSTHOG_SECRET
openssl rand -hex 16   # ENCRYPTION_SALT_KEYS
```

In Coolify, for values with `$` or special chars, use “Is Literal?” so they aren’t interpreted.

## 4. GeoIP and `share` volume

The stack expects `./share/GeoLite2-City.mmdb`. Options:

- **Before first deploy**: one-time download on the server (in repo root after clone):
  ```bash
  mkdir -p share
  curl -L 'https://mmdbcdn.posthog.net/' --http1.1 | brotli --decompress -o share/GeoLite2-City.mmdb
  ```
- Or add a small init container / entrypoint that downloads it if missing (not included here).

Coolify may mount the repo root; then `./share` in the compose is that directory.

## 5. Ports and proxy

- **proxy** (Caddy) exposes `80` and `443`; point your Coolify proxy/domain to this service.
- Set **DOMAIN** to the same hostname you use in Coolify (e.g. `posthog.yourdomain.com`).

## 6. First deploy

- Deploy from Coolify. First boot can take several minutes (migrations, Kafka topics, etc.).
- Health: `https://<DOMAIN>/_health` should return 200.

## Edits made in this repo (summary)

| Change | Purpose |
|--------|--------|
| `git init` + `remote origin` → `https://github.com/youssef-george/Posthog-openserver.git` | Link project to your GitHub repo |
| `compose/start`, `compose/wait`, `compose/temporal-django-worker` | Entrypoint scripts used by hobby web and temporal worker (same as `bin/deploy-hobby`) |
| `docker-compose.coolify.yml` | Hobby stack with paths fixed for this repo: `./docker`, `./rust`, `./posthog` at root; clickhouse `/idl` → named volume `idl-data` |
| `.env.example` | Env vars to set in Coolify |
| `COOLIFY.md` | This deploy guide |

Reference: [PostHog self-host hobby](https://posthog.com/docs/self-host/deploy/hobby), [Coolify Docker Compose](https://coolify.io/docs/knowledge-base/docker/compose).

---

## Security checklist (web app)

PostHog’s web app is safe for production when configured correctly. Use this checklist for your Coolify deploy.

### 1. Env vars (required for safety)

| Variable | Why |
|----------|-----|
| `SECRET_KEY` | **Required.** Django sessions, CSRF, signing. Server exits in prod if you use the default. Use a long random value (e.g. `openssl rand -hex 32`). |
| `ENCRYPTION_SALT_KEYS` | **Required.** Encrypted fields (e.g. data warehouse credentials). Use a unique value (e.g. `openssl rand -hex 16`). Don’t leave the default `00beef...` in production. |
| `DEBUG` | **Must be unset or `false`** in production. Default is `false`. If `true`, secure cookies and SSL redirect are off and dev-only behavior is enabled. |
| `ALLOWED_HOSTS` | Default is `*`. For production, set to your domain(s), e.g. `posthog.yourdomain.com` or comma-separated list. Restricts Host header acceptance. |

### 2. HTTPS and cookies

- With `DEBUG=false` and `TEST` unset, the app sets `SECURE_SSL_REDIRECT=True`, `SESSION_COOKIE_SECURE=True`, `CSRF_COOKIE_SECURE=True` (see `posthog/settings/access.py`).
- Put the app behind Coolify’s proxy (or Caddy) with TLS. Set `SITE_URL` to your public HTTPS URL (e.g. `https://posthog.yourdomain.com`). If the app is behind a reverse proxy, set `IS_BEHIND_PROXY=true` so `X-Forwarded-Proto` is trusted.

### 3. CORS

- Default is `CORS_ORIGIN_ALLOW_ALL=True` so any origin can call the API (needed for capture/ingestion from customer sites). To restrict the UI only, set `CORS_ALLOWED_ORIGINS` to your frontend origin(s) (e.g. `https://posthog.yourdomain.com`). Capture endpoints still need to accept requests from customer domains.

### 4. Access control

- API views use `TeamAndOrgViewSetMixin` with `TeamMemberAccessPermission` / `OrganizationMemberPermissions` and team-scoped querysets. Data is filtered by team/organization; don’t disable or bypass these.
- AGENTS.md: “Always filter querysets by `team_id`”.

### 5. SQL / HogQL

- PostHog uses parameterized queries and HogQL parsing. The repo’s `.agents/security.md` and semgrep rules (e.g. `.semgrep/rules/hogql-no-fstring.yaml`) guard against SQL/HogQL injection. Don’t interpolate user input into raw SQL or HogQL strings; use placeholders/`ast.Constant` where needed.

### 6. Rate limiting

- Many endpoints are throttled (signup, auth, HogQL, API bursts, etc.) via `posthog/rate_limit.py`. Rate limiting can be toggled with instance setting `RATE_LIMIT_ENABLED`.

### 7. Quick audit

- Ensure **DEBUG** is not set (or is false).
- Ensure **SECRET_KEY** and **ENCRYPTION_SALT_KEYS** are set to unique, random values (not defaults).
- Set **ALLOWED_HOSTS** to your domain(s).
- Use HTTPS and **SITE_URL**; set **IS_BEHIND_PROXY=true** if behind Coolify/Caddy.
