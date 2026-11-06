# 📚 Vaultisse

A modular Docker Compose configuration system for [Vaultisse](https://github.com/AlbertAmat/vaultisse) (tag `v1.3.8`) — a self-hosted personal book library with vaults, ISBN/metadata lookups (bundled YAZ Z39.50 client for SBN/Italian library sources), and ebook uploads stored in PostgreSQL.

## 🚀 Quick Start

### 1. Configure Environment

```bash
cp components/base/.env.example .env
```

Then append the database pair variables and set the two secrets:

```bash
cat components/extensions/database/postgresql/.env.example >> .env
# Edit .env: set JWT_SECRET (openssl rand -hex 32) and POSTGRES_PASSWORD
```

### 2. Build Configurations

```bash
cd app/ && sb build
```

This creates all environment × combo combinations under `build/` based on [`stackbuilder.toml`](stackbuilder.toml).

### 3. Choose Your Configuration

```bash
# For development with port forwarding
cd build/forwarding/entire/

# For production with Let's Encrypt SSL
cd build/letsencrypt/entire/

# For production with Step CA SSL
cd build/step-ca/entire/
```

### 4. Deploy

```bash
docker compose up -d
```

Access: `http://localhost:3000` (forwarding mode). The first user registers at `/register`; afterwards log in and create your first vault.

> ℹ️ `sb` merges no env files for environments today: the TLS environments
> (`letsencrypt`, `step-ca`) need the overlay-only variables (`VIRTUAL_HOST`,
> `LETSENCRYPT_HOST`, `LETSENCRYPT_EMAIL`, …) added to `.env` manually —
> see the Environment Variables section below.

## 🧩 Product Parts

The product's functional base per [`DEPLOYMENT.md`](../DEPLOYMENT.md) (stack-conventions §20) — one compose service per part:

- **vaultisse**: Single deployable — Express API + built Vue client served on one port (`API_PORT`, image EXPOSE 3000). Personal book library with vaults, ISBN/metadata lookups, ebook uploads stored in the database, and schema auto-migration at startup. Health endpoint: `/api/rest/app/version`. Logs go to the `vaultisse-logs` volume.
- **vaultisse-db** (mandatory infra): PostgreSQL, provided by the `database/postgresql` extension fragment (`database/pgrust` is the Stage A2 swap variant). The app faults without it — migrations and every repository need it at startup.

## 📁 Project Structure

- [`components/`](components/) — Source compose components
  - [`base/`](components/base/) — Base vaultisse configuration (environment-agnostic)
  - [`environments/`](components/environments/) — Environment-specific overlays (forwarding, letsencrypt, step-ca)
  - [`extensions/`](components/extensions/) — Extension components (`database/postgresql`, `database/pgrust`)
- [`build/`](build/) — Generated configurations (auto-generated, ready to deploy)
- [`stackbuilder.toml`](stackbuilder.toml) — Build configuration for stackbuilder

## 🔧 Available Configurations

### Environments

- **forwarding**: publishes `API_PORT` on the host; plain-HTTP gate on (`ALLOW_HTTP=true`), no proxy in front. Local/development default.
- **letsencrypt**: joins the external `letsencrypt-network` + `letsencrypt-vhost` volume for nginx-proxy with automatic Let's Encrypt certificates; TLS-derived defaults owned here (`TRUST_PROXY=true`, `ALLOW_HTTP=false`, `FRONT_END_URL=https://$VIRTUAL_HOST`).
- **step-ca**: same TLS wiring against the external `step-ca-network` + `step-ca-vhost` volume for self-signed trusted CA deployments without internet access.

### Extensions

Status per the extension catalog in [`DEPLOYMENT.md`](../DEPLOYMENT.md) (stack-conventions §20):

| extension | status | notes |
|-----------|--------|-------|
| `database/postgresql` | **implemented** (default combo `entire`) | Mandatory infra — `vaultisse-db` with healthcheck; the app depends on it (`service_healthy`). |
| `database/pgrust` | `available` | Stage A2 swap of the mandatory database pair (`malisper/pgrust:v0.2`); rebuild and re-run the Stage A gate, keep if clean. |
| `oidc` | `available` | Base stage B — SSO against the kanidm provider via the `OIDC_*` env block; not scaffolded yet. |
| `s3-storage` | `refused` | No S3 support in the app (uploads go into Postgres); no suitable volume for s3fs-fuse. |
| `cloudflared` | `available` | Optional upstream compose profile (`cloudflare`) — Cloudflare tunnel companion via `CLOUDFLARE_TUNNEL_TOKEN`. |
| metadata provider keys | `available` | Env-only product feature: `GOOGLE_BOOKS_API_KEY`, `LIBRARYTHING_API_KEY` enable online metadata lookup; manual entry works without. |

## 🔧 Environment Variables

### Base Configuration

- `COMPOSE_PROJECT_NAME`: Project name for Docker Compose (default: `vaultisse`)
- `IMAGE`: Docker image (default: `ghcr.io/albertamat/vaultisse:1.3.8`)
- `LOG_MAX_SIZE`: json-file log max size (default: `10m`)
- `LOG_MAX_FILE`: json-file log file count (default: `3`)
- `LOG_COMPRESS`: compress rotated logs (default: `true`)
- `API_PORT`: In-container listen port; the forwarding overlay publishes the same port on the host (default: `3000`)
- `JWT_SECRET`: **Required** — session signing secret; generate with `openssl rand -hex 32`
- `SESSION_TIME`: Session-cookie lifetime in milliseconds (default: `3600000`)
- `MAX_SESSION_AGE_DAYS`: Absolute session lifetime in days, forces re-login (default: `30`)
- `REGISTRATION_ENABLED`: Public `/register` page (default: `true`)
- `TZ`: Timezone (default: `UTC`)

### Database Configuration (both variants)

- `POSTGRES_DB`: Database name (default: `vaultisse`)
- `POSTGRES_USER`: Database user (default: `vaultisse`)
- `POSTGRES_PASSWORD`: **Required** — set a strong password in `.env`
- `DB_IMAGE`: Database image override (default: `postgres:alpine`, or `malisper/pgrust:v0.2` for the pgrust variant)
- `COMPOSE_EXTENDS` / `COMPOSE_FILE`: sb dual-file idiom wiring (`:docker-compose.db.yml` / `docker-compose.yml${COMPOSE_EXTENDS:-}`) — keep as generated

### Forwarding Overlay (overlay-only — add to `.env` manually)

- `FRONT_END_URL`: Public origin used for CORS/CSP and cookie/JWT issuer scope (default: `http://localhost:3000`)
- `ALLOW_HTTP`: Plain-HTTP gate flip — secure cookies otherwise (default: `true` in forwarding)
- `TRUST_PROXY`: Trust `X-Forwarded-*` headers behind a reverse proxy (default: `false` in forwarding)

### Let's Encrypt / Step CA Overlays (overlay-only — add to `.env` manually)

- `VIRTUAL_HOST`: Domain for nginx-proxy (required)
- `VIRTUAL_PORT`: Port nginx-proxy targets (default: `API_PORT`, i.e. `3000`)
- `LETSENCRYPT_HOST`: Domain for the SSL certificate (required)
- `LETSENCRYPT_EMAIL`: Email for certificate registration (required)
- `FRONT_END_URL`: Public origin (default: `https://$VIRTUAL_HOST`)
- `TRUST_PROXY`: `true` behind nginx-proxy
- `ALLOW_HTTP`: `false` — TLS only
- `VHOST_PATH`: Target path of the injected vhost config (default: `/vhost.d/$VIRTUAL_HOST`)

## 🌐 Networks

- **forwarding**: `vaultisse-network` (internal bridge)
- **letsencrypt**: `vaultisse-network` + `letsencrypt-network` (external)
- **step-ca**: `vaultisse-network` + `step-ca-network` (external)

## 🔒 Security

⚠️ **Production Checklist:**

- Set `JWT_SECRET` to a fresh random value (`openssl rand -hex 32`) — never ship the default.
- Set a strong `POSTGRES_PASSWORD`.
- In TLS environments keep `ALLOW_HTTP=false` and `TRUST_PROXY=true`; only the forwarding environment runs plain HTTP locally.
- `FRONT_END_URL` must match the public origin exactly (cookie + JWT issuer scope).
- Consider `REGISTRATION_ENABLED=false` after creating your account if `/register` remains publicly available.
- The letsencrypt/step-ca overlays inject `client_max_body_size 0;` into the proxy vhost so large ebook uploads pass through.

## 🆘 Troubleshooting

**Startup Issues:**
- `vaultisse` fault-looping at startup almost always means the database is missing or unhealthy — check `vaultisse-db` health (`pg_isready`) first; the app auto-migrates its schema at startup and needs Postgres before anything else.
- `JWT_SECRET` / `POSTGRES_PASSWORD` unset → compose fails fast (`${VAR:?set in .env}`).

**Metadata Lookup Issues:**
- Online lookups (Google Books / LibraryThing / Z39.50 sources) need outbound HTTPS and, for the hosted APIs, the provider keys; manual book entry works fully offline.

**SSL Issues:**
- **Let's Encrypt**: Verify domain DNS and the letsencrypt-manager; the app must be reachable on `VIRTUAL_HOST`.
- **Step CA**: Check the step-ca-manager and the virtual network config (`step-ca-network` external).

## ✅ Stage A Functional Gate

Stage A = product parts + database, **forwarding** environment. The 7-step gate (per [`DEPLOYMENT.md`](../DEPLOYMENT.md)):

1. `vaultisse-db` becomes healthy (`pg_isready`); `vaultisse` starts after it (`depends_on: service_healthy`), logs show startup + auto-migration, no fault loop.
2. `curl http://localhost:3000/api/rest/app/version` returns 200 with a version payload.
3. Browser opens `http://localhost:3000` → login page renders.
4. Register the first user via `/register` (observe whether `/register` stays available afterwards — open hypothesis).
5. Log in → create a vault → add a book manually → verify it appears in the vault.
6. Log out → log back in with the same account.
7. `docker compose restart` → login still works, the book is still there (state persisted in `vaultisse-db`).

## 📝 Notes

- The `build/` directory is automatically generated and should not be edited manually.
- Environment variables in generated files use `$VARIABLE_NAME` format for proper interpolation.
- Each generated configuration includes a complete `docker-compose.yml` and `.env.example`.
- Use `sb build` to regenerate all configurations after modifying components.
- `sb` merges no env files for environments today: assemble `.env` from the base + database pair examples and add the overlay-only variables for your chosen environment manually.
