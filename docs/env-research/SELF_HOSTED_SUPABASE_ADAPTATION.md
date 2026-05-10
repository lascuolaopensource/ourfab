# Self-Hosted Supabase Setup (Shelf Guide Adaptation)

This document mirrors the numbered sections in the Shelf cloud guide, adapted for your **self-hosted Supabase Docker stack**.  
Scope: **no `docker-compose.yml` edits** right now, only `.env` changes and server operations.

---

## 1) Create Your Supabase Project

In self-hosted mode, this step becomes: initialize and secure your existing stack configuration.

### Changes
- Ensure every `SERVICE_PASSWORD_*` and key variable in `.env` is unique and not a placeholder.
- Set the public API URL used by Studio/Auth:
  - `SERVICE_URL_SUPABASEKONG` (the externally reachable URL of Kong/API gateway)

### Operations to run on server

```bash
# from repo root
docker compose config >/tmp/supabase-compose-resolved.yaml
```

Review the resolved file for unresolved `${...}` placeholders.

---

## 2) Get Your Connection Details

In self-hosted mode, connection details come from your `.env` + your gateway domain.

### Changes
- For your application:
  - API URL = `SERVICE_URL_SUPABASEKONG`
  - anon key = `SERVICE_SUPABASEANON_KEY`
  - service role key = `SERVICE_SUPABASESERVICE_KEY`
- For DB runtime/migrations, use your self-hosted Postgres/Supavisor endpoint and `SERVICE_PASSWORD_POSTGRES`.

### Operations to run on server

```bash
# validate key vars are present (prints names only)
rg "^(SERVICE_URL_SUPABASEKONG|SERVICE_SUPABASEANON_KEY|SERVICE_SUPABASESERVICE_KEY|SERVICE_PASSWORD_POSTGRES|POSTGRES_DB|POSTGRES_PORT|POSTGRES_HOSTNAME)=" .env
```

---

## 3) Configure Database Connection Mode

Your compose already sets Supavisor to transaction pooling:
- `POOLER_POOL_MODE=transaction` on `supabase-supavisor`.

### Changes
- Keep transaction mode (recommended for app runtime).
- Tune pool settings in `.env` if needed:
  - `POOLER_DEFAULT_POOL_SIZE`
  - `POOLER_MAX_CLIENT_CONN`
  - `POOLER_DB_POOL_SIZE`

### Operations to run on server

```bash
docker compose up -d supabase-supavisor
docker compose ps supabase-supavisor
docker compose logs --since=5m supabase-supavisor
```

---

## 4) Setup Authentication

Map Shelf auth expectations to `supabase-auth` (`gotrue`) envs.

### Changes
- Set site/external URL:
  - `GOTRUE_SITE_URL`
  - `API_EXTERNAL_URL`
- Set redirect allow-list:
  - `ADDITIONAL_REDIRECT_URLS` (comma-separated)
- Enable email auth:
  - `ENABLE_EMAIL_SIGNUP=true`
  - `ENABLE_EMAIL_AUTOCONFIRM=false` (recommended unless intentional)
- Keep JWT secret strong:
  - `SERVICE_PASSWORD_JWT`

### Important note about OTP length
- Shelf expects 6-digit OTP.
- Your current compose env list does **not** expose an explicit OTP-length variable for GoTrue.
- Validate actual OTP behavior through signup/login; if not 6 digits, that will require a later compose/env surface change.

### Operations to run on server

```bash
docker compose up -d supabase-auth
docker compose logs --since=10m supabase-auth
```

---

## 5) Create Storage Buckets

Your stack uses `supabase-storage` + MinIO (`supabase-minio`), with bucket bootstrap via `minio-createbucket`.

### Changes
- Ensure MinIO credentials in `.env` are set:
  - `SERVICE_USER_MINIO`
  - `SERVICE_PASSWORD_MINIO`
- Create required buckets (`profile-pictures`, `assets`, `kits`, `files`) if your bootstrap script does not already do so.
- Apply restrictive storage RLS/policies matching Shelf behavior.

### Operations to run on server

```bash
# start storage path
docker compose up -d supabase-minio minio-createbucket supabase-storage
docker compose ps supabase-minio minio-createbucket supabase-storage
```

Then apply SQL policies from Postgres:

```bash
docker compose exec -T supabase-db psql -U postgres -d postgres <<'SQL'
-- Example: verify buckets table exists
select id, name, public from storage.buckets order by name;
SQL
```

If buckets are missing, insert them (adapt as needed):

```bash
docker compose exec -T supabase-db psql -U postgres -d postgres <<'SQL'
insert into storage.buckets (id, name, public)
values
  ('profile-pictures','profile-pictures', true),
  ('assets','assets', false),
  ('kits','kits', false),
  ('files','files', true)
on conflict (id) do nothing;
SQL
```

---

## 6) Complete Your `.env` File

This is the key adaptation point: map Shelf cloud variables to your self-hosted names.

### Changes
- Required core self-hosted vars (from your compose):
  - `SERVICE_URL_SUPABASEKONG`
  - `SERVICE_PASSWORD_POSTGRES`
  - `SERVICE_PASSWORD_JWT`
  - `SERVICE_SUPABASEANON_KEY`
  - `SERVICE_SUPABASESERVICE_KEY`
  - `POSTGRES_DB`
  - `POSTGRES_PORT`
  - `POSTGRES_HOSTNAME`
- For email/auth flows:
  - `SMTP_ADMIN_EMAIL`
  - `SMTP_HOST`
  - `SMTP_PORT`
  - `SMTP_USER`
  - `SMTP_PASS`
  - `SMTP_SENDER_NAME`

### Operations to run on server

```bash
# restart affected services after .env update
docker compose up -d supabase-kong supabase-auth supabase-rest supabase-storage supabase-studio
docker compose ps
```

---

## 7) Generate Session Secrets

Generate high-entropy values and paste into `.env`.

### Changes
- Regenerate and set at minimum:
  - `SERVICE_PASSWORD_JWT`
  - `SECRET_PASSWORD_REALTIME`
  - `SERVICE_PASSWORD_SUPAVISORSECRET`
  - `SERVICE_PASSWORD_VAULTENC`
  - `SERVICE_PASSWORD_PGMETACRYPTO`
  - `SERVICE_PASSWORD_LOGFLARE`
  - `SERVICE_PASSWORD_LOGFLAREPRIVATE`
  - `SERVICE_PASSWORD_POSTGRES`
  - `SERVICE_PASSWORD_ADMIN`
  - `SERVICE_PASSWORD_MINIO`

### Operations to run on server

```bash
openssl rand -hex 32
```

Run it once per secret, update `.env`, then:

```bash
docker compose up -d
```

---

## 8) Setup Email (Required)

For self-hosted GoTrue, email delivery is fully driven by SMTP env vars.

### Changes
- Fill SMTP vars in `.env`:
  - `SMTP_ADMIN_EMAIL`
  - `SMTP_HOST`
  - `SMTP_PORT`
  - `SMTP_USER`
  - `SMTP_PASS`
  - `SMTP_SENDER_NAME`
- Keep `ENABLE_EMAIL_SIGNUP=true` if email OTP flow is required.

### Operations to run on server

```bash
docker compose up -d supabase-auth
docker compose logs --since=10m supabase-auth
```

Then perform a real signup/reset flow and confirm emails are sent.

---

## 9) Optional Integrations

Cloud-only extras from Shelf (MapTiler/geocoding) remain app-layer concerns; they do not require changes to this Supabase self-hosted compose.

### Changes
- No mandatory Supabase infra change.
- Configure optional app envs in your app deployment only (not in Supabase stack) unless your app is co-located in this same project.

### Operations to run on server

```bash
# no required Supabase Docker operation for this step
docker compose ps
```

---

## Quick Verification Checklist

- Kong/API reachable at `SERVICE_URL_SUPABASEKONG`
- Studio healthy (`supabase-studio`)
- Auth healthy (`supabase-auth`)
- Storage healthy (`supabase-storage`) + buckets present
- Supavisor healthy (`supabase-supavisor`) in transaction mode
- End-to-end signup/login/password-reset tested with your SMTP provider
