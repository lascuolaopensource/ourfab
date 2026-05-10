# Self-Hosted Supabase Checklist (Cloud Guide Adaptation)

This is the **second guide**, rewritten in a compact execution format.

Goal:

- mirror the Shelf cloud setup flow using the same numbered sections (`1` to `9`)
- adapt everything to your **self-hosted Docker Supabase**
- include **recommended `docker-compose.yml` edits** (documented only, not applied)

References:

- [Shelf Supabase Setup Guide](https://docs.shelf.nu/supabase-setup)
- [Supabase Self-Hosting with Docker](https://supabase.com/docs/guides/self-hosting/docker)

---

## 1) Create Your Supabase Project

In self-hosted mode, this means: secure and validate your local stack configuration before boot.

### `.env` checklist

- set a public base URL for Kong/API gateway:
  - `SERVICE_URL_SUPABASEKONG=https://<your-host>`
- ensure all secrets are non-placeholder values.

### Commands

```bash
cd /Users/giovanniabbatepaolo/Desktop/x
docker compose config >/tmp/supabase-compose-resolved.yaml
docker compose ps
```

---

## 2) Get Your Connection Details

Use your self-hosted endpoints and keys (from `.env`), not cloud project credentials.

### App mapping

- `SUPABASE_URL` -> `SERVICE_URL_SUPABASEKONG`
- `SUPABASE_ANON_PUBLIC` -> `SERVICE_SUPABASEANON_KEY`
- `SUPABASE_SERVICE_ROLE` -> `SERVICE_SUPABASESERVICE_KEY`

### DB mapping

- Postgres password -> `SERVICE_PASSWORD_POSTGRES`
- DB host/port/db -> `POSTGRES_HOSTNAME`, `POSTGRES_PORT`, `POSTGRES_DB`

### Commands

```bash
docker compose up -d supabase-kong supabase-rest supabase-db
docker compose ps supabase-kong supabase-rest supabase-db
```

---

## 3) Configure Database Connection Mode

Your stack already runs Supavisor in transaction mode.

### `.env` values to verify/tune

```dotenv
POOLER_DEFAULT_POOL_SIZE=20
POOLER_MAX_CLIENT_CONN=100
POOLER_DB_POOL_SIZE=5
```

### Commands

```bash
docker compose up -d supabase-supavisor
docker compose logs --since=10m supabase-supavisor
```

---

## 4) Setup Authentication

Configure URL, redirects, email auth, and JWT settings for GoTrue (`supabase-auth`).

### `.env` values

```dotenv
API_EXTERNAL_URL=https://<your-host>
GOTRUE_SITE_URL=https://<your-app-domain>
ADDITIONAL_REDIRECT_URLS=https://<your-app-domain>/reset-password,http://localhost:3000/reset-password,https://localhost:3000/reset-password

ENABLE_EMAIL_SIGNUP=true
ENABLE_EMAIL_AUTOCONFIRM=false
DISABLE_SIGNUP=false
JWT_EXPIRY=3600
```

### Commands

```bash
docker compose up -d supabase-auth
docker compose logs --since=10m supabase-auth
```

---

## 5) Create Storage Buckets

Your setup uses `supabase-storage` + MinIO.

### `.env` values

```dotenv
SERVICE_USER_MINIO=<strong-user>
SERVICE_PASSWORD_MINIO=<strong-password>
```

### Commands (start storage path)

```bash
docker compose up -d supabase-minio minio-createbucket supabase-storage
docker compose ps supabase-minio minio-createbucket supabase-storage
```

### Commands (ensure required buckets)

```bash
docker compose exec -T supabase-db psql -U postgres -d postgres <<'SQL'
insert into storage.buckets (id, name, public)
values
  ('profile-pictures','profile-pictures', true),
  ('assets','assets', false),
  ('kits','kits', false),
  ('files','files', true)
on conflict (id) do nothing;

select id, name, public from storage.buckets order by name;
SQL
```

### Commands (restrict direct object writes by anon/authenticated)

```bash
docker compose exec -T supabase-db psql -U postgres -d postgres <<'SQL'
alter table storage.objects enable row level security;

do $$
begin
  if not exists (
    select 1
    from pg_policies
    where schemaname = 'storage'
      and tablename = 'objects'
      and policyname = 'deny_all_object_writes_for_anon_authenticated'
  ) then
    create policy deny_all_object_writes_for_anon_authenticated
    on storage.objects
    for all
    to anon, authenticated
    using (false)
    with check (false);
  end if;
end $$;
SQL
```

---

## 6) Complete Your `.env` File

Minimum required secure values:

```dotenv
SERVICE_URL_SUPABASEKONG=https://<your-host>
POSTGRES_HOSTNAME=supabase-db
POSTGRES_PORT=5432
POSTGRES_DB=postgres

SERVICE_SUPABASEANON_KEY=<jwt anon key>
SERVICE_SUPABASESERVICE_KEY=<jwt service_role key>
SERVICE_PASSWORD_JWT=<jwt secret>
SERVICE_PASSWORD_POSTGRES=<postgres password>

SMTP_ADMIN_EMAIL=no-reply@<domain>
SMTP_HOST=<smtp host>
SMTP_PORT=587
SMTP_USER=<smtp user>
SMTP_PASS=<smtp password>
SMTP_SENDER_NAME=<sender name>
```

### Commands

```bash
docker compose up -d
docker compose ps
```

---

## 7) Generate Session Secrets

Generate each secret with:

```bash
openssl rand -hex 32
```

Recommended vars to rotate/generate:

- `SERVICE_PASSWORD_JWT`
- `SERVICE_PASSWORD_POSTGRES`
- `SERVICE_PASSWORD_ADMIN`
- `SERVICE_PASSWORD_PGMETACRYPTO`
- `SERVICE_PASSWORD_LOGFLARE`
- `SERVICE_PASSWORD_LOGFLAREPRIVATE`
- `SERVICE_PASSWORD_SUPAVISORSECRET`
- `SERVICE_PASSWORD_VAULTENC`
- `SECRET_PASSWORD_REALTIME`
- `SERVICE_PASSWORD_MINIO`

Then:

```bash
docker compose up -d
```

---

## 8) Setup Email (Required)

Your auth flow depends on SMTP in self-hosted mode.

### Commands

```bash
docker compose up -d supabase-auth
docker compose logs --since=15m supabase-auth
```

Manual test:

- signup OTP email
- login OTP/magic email
- reset-password email

---

## 9) Optional Integrations

Map/geocoding is usually app-layer only; no mandatory Supabase infra edit required.

Example app envs:

- `MAPTILER_TOKEN`
- `GEOCODING_USER_AGENT`

---

## Recommended `docker-compose.yml` Edits (Do Not Apply Yet)

These are recommended improvements for your next compose revision.

### A) Force OTP length to 6 (if supported by your GoTrue version)

In `supabase-auth.environment`:

```yaml
- GOTRUE_MAILER_OTP_LENGTH=6
```

Why: aligns with Shelf’s 6-digit OTP expectation.

### B) Disable unused phone auth path

In `supabase-auth.environment`:

```yaml
- GOTRUE_EXTERNAL_PHONE_ENABLED=false
- GOTRUE_SMS_AUTOCONFIRM=false
```

Why: smaller attack surface.

### C) Improve restart resilience

For critical services (`supabase-kong`, `supabase-auth`, `supabase-rest`, `supabase-storage`, `supabase-supavisor`):

```yaml
restart: unless-stopped
```

### D) Restrict Studio exposure

- Keep Studio private (VPN/internal network) whenever possible.
- Keep Kong as primary public entrypoint.

### E) Make bucket bootstrap explicitly idempotent

- Ensure your bootstrap script always verifies/creates:
  - `profile-pictures` (public)
  - `assets` (private)
  - `kits` (private)
  - `files` (public)

### F) Production hardening

- pin image digests
- move secrets out of plaintext `.env` where possible
- apply firewall restrictions for Postgres, MinIO console, Studio

---

## Final Verification

```bash
docker compose ps
docker compose logs --since=10m supabase-kong supabase-auth supabase-rest supabase-storage supabase-supavisor
```

Acceptance checks:

- Kong URL reachable at `SERVICE_URL_SUPABASEKONG`
- Studio works
- OTP emails work
- bucket list correct
- unauthorized browser writes are blocked by policy
