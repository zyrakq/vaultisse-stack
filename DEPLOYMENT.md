# DEPLOYMENT — vaultisse

Upstream: https://github.com/AlbertAmat/vaultisse (analysis commit `cbcad08`, tag `v1.3.8`).

## Product parts

| part | purpose | image |
|------|---------|-------|
| `vaultisse` | Single Node deployable: Express API + built Vue client served on one port (`API_PORT`, image EXPOSE 3000). Personal book library with vaults, ISBN/metadata lookups (bundled YAZ Z39.50 client for SBN/Italian library sources), ebook uploads stored in the database, auto-migration of the schema at startup (`/app/db` upgrade SQL). Health endpoint `/api/rest/app/version`. | published: `ghcr.io/albertamat/vaultisse` (tag `1.3.8`) |

No other deployable part exists upstream. The upstream compose adds `db` (PostgreSQL) and an optional `cloudflared` profile (see catalog).

## Interaction map

- browser → `vaultisse:<API_PORT>` (HTTP; UI + `/api/rest/*`). Only entrypoint.
- `vaultisse` → `vaultisse-db:5432` (PostgreSQL). Mandatory at startup — the container faults without it (migrations and every repository).
- `vaultisse` → outbound HTTPS (Google Books / LibraryThing APIs, Z39.50 sources): optional, only exercised when metadata lookup with provider keys is used. Not required for the core flow (manual book entry works offline).
- Absent DB → app cannot start; absent outbound access → metadata lookup degrades, core CRUD unaffected.

One group only (`vaultisse` + its DB); no inter-group links → no `step-ca-trust` requirement anywhere.

## Functional base

Single part: `vaultisse` (tier `core`), plus mandatory infra:

| item | verdict | evidence |
|------|---------|----------|
| PostgreSQL | mandatory-infra stage | upstream compose runs `postgres:18-alpine` with `databaseSchema.sql` init and schema auto-migration in the image (`server/src/migrate`); app errors without it. Collected as the `database/{postgresql,pgrust}` extension pair (§8) — never a plain base service. |
| OIDC | mandatory-infra stage (standard oidc extension, staged after base gate) | fully env-configurable in the code (AppService.ts ~397: `OIDC_ISSUER/CLIENT_ID/CLIENT_SECRET/REDIRECT_URI/SCOPES` default `openid profile email`/`BUTTON_LABEL`/`GROUPS_CLAIM` default `groups`/`PERSONAL_VAULT`, plus `SSO_ONLY`), openid-client authorization-code+PKCE. Not product-mandated for a first run but always-attempted per staged protocol; app also runs on password auth. |
| S3 | REFUSED | No object storage support in the code: zero S3/AWS/MinIO references in `server/src`; uploads go through multer `memoryStorage` into the database; the only volume writes are logs (`/app/logs`). No s3fs fallback either — the only volume holds logs, which s3fs would not benefit. |

Core app env (base): `API_PORT`, `FRONT_END_URL`, `JWT_SECRET`, `SESSION_TIME`, `MAX_SESSION_AGE_DAYS`; plain-HTTP gate flip `ALLOW_HTTP=true` (secure cookies otherwise); `TRUST_PROXY=true` behind a proxy; `REGISTRATION_ENABLED` defaults on. First user registers at `/register`.

## Base stage plan

| stage | content | gate |
|-------|---------|------|
| A (core) | `vaultisse` + `vaultisse-db` via the `database/postgresql` fragment; plain-HTTP environment first | Functional gate: open the app in a browser → register the first user via `/register` → log in → create a vault → add a book manually → verify it in the vault → log out → log back in. Confirm health `/api/rest/app/version`. |
| A2 | swap `database/postgresql` → `database/pgrust`, rebuild, re-run the A gate | clean → keep pgrust; failure → revert to postgresql and note why |
| B (mandatory infra: oidc) | `oidc` extension against the kanidm provider; re-deploy a no-oidc variant kept as pre-OIDC proof then the oidc variant | SSO button login creates/links a user; existing password account still works (unless `SSO_ONLY=true`) |

No outer groups → no cross-group link gate.

## Extension catalog

| extension | type | status | dependencies |
|-----------|------|--------|--------------|
| `oidc` | infra | `available` (unimplemented; staged as base stage B above) | kanidm provider; `OIDC_*` env block |
| `database/pgrust` | infra | `available` (unimplemented; A2 swap of the mandatory pair) | none |
| `s3-storage` | infra | `refused` — no S3 support in the app (uploads in Postgres); no suitable volume for s3fs-fuse | — |
| `cloudflared` | product (optional companion) | `available` upstream compose profile (`cloudflare`), `CLOUDFLARE_TUNNEL_TOKEN` | none |
| metadata provider keys (`GOOGLE_BOOKS_API_KEY`, `LIBRARYTHING_API_KEY`) | product feature (env-only) | `available` — enables online metadata lookup; optional, manual entry works without | outbound HTTPS; third-party API keys |

## Open hypotheses

| hypothesis | confidence | what the gate must observe |
|------------|-----------|---------------------------|
| `FRONT_END_URL` semantics: exact-URL storage/comparison cookie+JWT issuer scope behind a reverse proxy (TLS termination on another host) — correct default per environment (plain local URL in base/forwarding, `https://<VIRTUAL_HOST>` in TLS environments) is not yet confirmed. | medium | Stage A plain run + a TLS-environment re-run: login/logout + book add over the proxied domain without CSRF/origin errors. |
| `REGISTRATION_ENABLED` default-on + `/register` first-user flow: whether `/register` becomes unavailable after the first user (admin-level docs) — affects how the gate creates accounts. | medium | Register first user; check whether additional `/register` attempts are refused or a public registration page remains. |
| `ALLOW_HTTP` / `TRUST_PROXY` interaction behind nginx-proxy: whether setting both is required or the app misbehaves without `TRUST_PROXY`. | medium | Stage A with published port works; TLS stage re-checks cookie flags via `TRUST_PROXY=true`. |
| `SSO_ONLY` + group claim mapping (kanidm groups via `OIDC_GROUPS_CLAIM`): Stage B scope only. | low–medium | Stage B gate shows SSO user with expected groups/personal vault. |
