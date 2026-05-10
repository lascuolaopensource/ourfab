GOTRUE_SITE_URL
Suggested: https://app.lascuolaopensource.org (or your real frontend URL)

ADDITIONAL_REDIRECT_URLS
Suggested (comma-separated, no spaces):
https://app.lascuolaopensource.org/reset-password,https://app.lascuolaopensource.org/auth/callback,http://localhost:3000/reset-password,https://localhost:3000/reset-password

SMTP_ADMIN_EMAIL
Suggested: no-reply@lascuolaopensource.org

SMTP_HOST
Suggested: your provider host (e.g. smtp.resend.com, smtp.mailgun.org, smtp.sendgrid.net)

SMTP_USER
Suggested: SMTP username from your provider

SMTP_PASS
Suggested: SMTP password/API token from your provider

SMTP_SENDER_NAME
Suggested: La Scuola Open Source (or your app name)

SECRET_PASSWORD_REALTIME
Suggested: 32-byte random hex, generated with openssl rand -hex 32

SERVICE_URL_SUPABASEKONG (recommended to align with public auth URL)
Suggested: https://supabase.lascuolaopensource.org if TLS is active at that endpoint
(currently it is http://...)

API_EXTERNAL_URL
Suggested: keep as https://supabase.lascuolaopensource.org (already set correctly)
