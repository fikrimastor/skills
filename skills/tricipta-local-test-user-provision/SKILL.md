---
name: tricipta-local-test-user-provision
description: "Create or restore a login-capable test user on the local Tricipta dev stack (Keycloak edafy realm + local asset-api DB), and diagnose the 'not registered for this system' / not_registered audit-log denial."
---

# Provision a login-capable test user (Tricipta local stack)

SSO login needs the identity in **two** places. Creating it in Keycloak alone always fails.

| Half | Where | Why |
|---|---|---|
| IdP identity | Keycloak `edafy` realm :8180 | password grant + `sub` |
| Local row | `users` in `tricipta-edafy-asset-api-db` | `ProvisionOIDCUser` denies unregistered identities |

## Diagnostics first

Symptom: HTTP 403 `Your account is not registered for this system`.

```bash
# does Keycloak have them?
curl -s "http://localhost:8180/admin/realms/edafy/users?username=<name>"

# does the local DB?
docker exec edafy-db psql -U user -d tricipta-edafy-asset-api-db -c "SELECT id,email,username,is_active,keycloak_id,role_id FROM users;"

# evidence of denials (reason is the key field)
docker exec edafy-db psql -U user -d tricipta-edafy-asset-api-db \
  -c "SELECT action,actor_email,details,created_at FROM ast_audit_logs ORDER BY created_at DESC LIMIT 10;"
```

`reason:"not_registered"` = Keycloak fine, local row missing. `reason:"inactive"` = row exists, `is_active=false`.

## Fix: insert the local row

Insert only what you actually know; leave unknown columns NULL (production often shows `''` — equivalent).

```bash
printf '%s\n' "$SQL" | docker exec -i edafy-db psql -U user -d tricipta-edafy-asset-api-db -v ON_ERROR_STOP=1 -f /dev/stdin
```

Required minimum: `id`, `email`, `username`, `password` (NOT NULL — any bcrypt string; unused on the OIDC path), `is_active=true`, `email_verified=true`. Leave `keycloak_id` NULL so the email-fallback branch links the `sub` itself.

## Verify (proves the path, not just the row)

```bash
curl -s -X POST http://localhost:8085/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"email":"<email>","password":"<pw>"}' -w '\nHTTP %{http_code}\n'
```

Expect 200. Then confirm the three side effects that only a full run produces:

```sql
SELECT keycloak_id,last_login_at FROM users WHERE email='<email>';  -- keycloak_id NULL->sub, last_login_at moved
SELECT * FROM tenant_users;   -- row with realm-role (e.g. viewer)
SELECT * FROM tenants;        -- subdomain='default' auto-created if absent
```

## Traps

- **Keycloak username is unique.** `testuser` already exists in the seeded realm — creating it again is a 409. If a request names both a username and a new email, resolve which is authoritative (the pasted DB row beats loose prose) before writing anything.
- **Do not mutate the seeded Keycloak user** to satisfy an email mismatch; insert/align the local row instead.
- **`role_id` is legitimately NULL** when authorization rides on Keycloak realm roles. Local `ast_roles` is empty until `docker exec edafy-api /root/seeder --iam`. Such an identity authenticates but has **no** `add_user`/admin permissions — don't use it for user-management paths.
- `/root/seeder` exists only after a `docker compose up -d --build api`.
- Admin token for realm queries comes from the **master** realm: `client_id=admin-cli` + `KEYCLOAK_ADMIN`/`KEYCLOAK_ADMIN_PASSWORD` from `tricipta-edafy-asset-api/.env`.
- Never echo secret values from `.env` into transcripts or docs; describe shape only.
