# Axonyx self-host — install guide

One `docker compose` stack: **self-hosted Supabase** + the **Axonyx dashboard** + the
**single-tenant governance proxy**. Prebuilt images are pulled from GHCR — no source
build required. Deploy per company.

## 1. Prerequisites
- Linux host with **Docker** and **Docker Compose v2.20+** (`docker compose version`).
- Open ports (defaults): **3000** dashboard, **9999** proxy, **8000** Supabase (Kong/Studio).
- ~4 GB RAM free.

## 2. Get the files
```bash
git clone https://github.com/axonyxai/axonyx-selfhost-deploy.git
cd axonyx-selfhost-deploy
cp .env.example .env
```

## 3. Configure `.env`
Must-change values for a real deployment:

| Var | Set to |
|-----|--------|
| `POSTGRES_PASSWORD` | long random string |
| `JWT_SECRET` | random ≥ 40 chars |
| `ANON_KEY` / `SERVICE_ROLE_KEY` | JWTs signed with `JWT_SECRET` — generate at <https://supabase.com/docs/guides/self-hosting/docker#generate-api-keys> |
| `DASHBOARD_PASSWORD`, `SECRET_KEY_BASE`, `VAULT_ENC_KEY` | random strings |
| `SUPABASE_PUBLIC_URL`, `API_EXTERNAL_URL` | `http://YOUR_HOST:8000` (must be reachable from users' browsers) |
| `SITE_URL` | `http://YOUR_HOST:3000` |
| `PROXY_API_KEY` *(optional)* | company-wide proxy bearer key |
| `ANTHROPIC_API_KEY` *(optional)* | enables Haiku-judged detection |

> **Quick local trial:** the pre-filled **demo** keys work as-is on `localhost`. They are
> public/insecure — **regenerate everything above before exposing the host.**

## 4. Start
```bash
docker compose up -d       # pulls images + brings up Supabase (~1–2 min first run)
docker compose ps          # wait for supabase-* healthy
```

## 5. Apply the product schema (once)
```bash
docker compose run --rm migrate
```
> Stops on the first error (`ON_ERROR_STOP`). If a migration fails, report it — the
> schema was authored for managed Supabase and a few files may need small fixups.

## 6. Sign in & configure
- **Supabase Studio** `http://YOUR_HOST:8000` (`supabase` / `DASHBOARD_PASSWORD`) — create an auth user, or use the dashboard sign-up.
- **Dashboard** `http://YOUR_HOST:3000` — sign in; configure applications (proxy keys + rule profiles), approved AI models, DLP/detection.

> **Email sign-up works out of the box.** OAuth providers are **off by default** — the
> dashboard's "Sign in with Microsoft" returns *"provider is not enabled"* until you turn it on.

### Enable "Sign in with Microsoft" (Entra ID)
1. In **Entra ID → App registrations → New registration**. Add a **Web** redirect URI:
   `http://YOUR_HOST:8000/auth/v1/callback` (use `https://…` if you front it with TLS).
2. Create a **client secret**; note the **Application (client) ID** + secret **value**.
3. In `.env` set:
   ```
   AZURE_ENABLED=true
   AZURE_CLIENT_ID=<application-client-id>
   AZURE_SECRET=<client-secret-value>
   # single-tenant only (restrict to your org):
   # AZURE_URL=https://login.microsoftonline.com/<tenant-id>/v2.0
   ```
4. `docker compose up -d auth` (recreates the auth container with the new env).

## 7. Send AI traffic through the proxy
```
POST http://YOUR_HOST:9999/v1/chat/completions
Authorization: Bearer <an application key, or PROXY_API_KEY>
```
Governed (DLP, prompt-injection, model routing) and logged to `ai_runs`, shown in the dashboard.

## Operations
```bash
docker compose pull && docker compose up -d   # update to latest images
docker compose logs -f frontend proxy
docker compose down          # stop (keeps DB in ./supabase/volumes/db/data)
# Full reset — the database is a BIND MOUNT, so `down -v` does NOT wipe it:
docker compose down && sudo rm -rf ./supabase/volumes/db/data && mkdir -p ./supabase/volumes/db/data
```
Back up `./supabase/volumes/db/data`.
