# 📚 Vaultisse Stack

Self-hosted personal book library: organize books into vaults, enrich them
with ISBN/metadata lookups (bundled YAZ Z39.50 client for SBN/Italian library
sources plus Google Books / LibraryThing), and upload ebooks that are stored
directly in PostgreSQL.

## 🧩 Components

### 📖 [Vaultisse](app/)

Express API with a built-in Vue client served on a single port (3000).
Personal book library with vaults, ISBN/metadata lookups, and ebook uploads
persisted in the database. This is the compose entry point — see
[`app/README.md`](app/README.md) for detailed setup.

### 🌐 [Let's Encrypt Manager](modules/ssl-automation/letsencrypt-manager/)

Automatic SSL certificate generation and renewal using Let's Encrypt for
production deployments with internet access.

### 🔒 [Step CA Manager](modules/ssl-automation/step-ca-manager/)

Self-signed trusted certificate authority for virtual Docker networks
without internet access. Automatically manages and distributes CA
certificates within isolated environments.

## 🚀 Quick Start

Each component has its own README with detailed setup instructions. Choose
the certificate management solution that fits your deployment scenario.

```bash
cd app/
sb build
cd build/forwarding/<combo>/
cp .env.example .env
docker compose up --build -d
```

Access: `http://localhost:3000` — register the first user at `/register`.

## 🧱 Base Parts

- **vaultisse** (port 3000): the product — API + UI in one container.
- **vaultisse-db**: PostgreSQL, provided via the
  [`database/postgresql`](app/components/extensions/) extension pair
  (mandatory infra, never a plain base service).

Extension notes per [`DEPLOYMENT.md`](DEPLOYMENT.md):

- `s3-storage`: **refused** — the app stores uploads in Postgres, no S3 support.
- `oidc`: staged as base stage B (mandatory infra, wired against kanidm after the base gate).
- `cloudflared`: available as an optional product extension (upstream `cloudflare` compose profile).

## 📋 Requirements

- Docker & Docker Compose
- Domain name (for production deployments)
- Email address (for Let's Encrypt)

## 📄 License

This project is dual-licensed under:

- [Apache License 2.0](LICENSE-APACHE)
- [MIT License](LICENSE-MIT)
