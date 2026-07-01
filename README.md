# Axonyx self-host (deploy)

Deploy Axonyx — dashboard + governance proxy — for a single company on your own Docker
host, backed by a bundled self-hosted Supabase. Prebuilt images, one `docker compose`.

**→ [INSTALL.md](INSTALL.md)**

```bash
git clone https://github.com/axonyxai/axonyx-selfhost-deploy.git
cd axonyx-selfhost-deploy
cp .env.example .env        # set keys + URLs (demo keys work for a local trial)
docker compose up -d
docker compose run --rm migrate
```

Images: `ghcr.io/axonyxai/axonyx-frontend`, `axonyx-proxy`, `axonyx-db-migrate` (public).
Single-tenant by design — no multi-company routing, no external Axonyx services.
