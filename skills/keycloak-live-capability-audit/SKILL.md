---
name: keycloak-live-capability-audit
description: "Audit what authentication/recovery capabilities a Keycloak-backed stack ACTUALLY has — probe the live realm admin API and the rendered login HTML instead of trusting realm JSON, code comments, or docs (they disagreed in every case). Use when asked 'do we have forgot password / SSO / MFA / account recovery' or to inventory which Keycloak endpoints an app calls."
---

# Keycloak Live Capability Audit

**Premise (confirmed in every case so far):** realm JSON, code comments, and docs all disagree with the live realm. The realm-import fixture is frequently stale; in one audit it pinned Keycloak 25.0 with no IdP while the running container was 26.7.0 with an enabled Microsoft Entra IdP. **Never answer a capability question from a file.**

Answers come from three live surfaces, in this order:

1. **Live realm admin API** — authoritative flags.
2. **Rendered login HTML** — what the user can actually click.
3. **Non-destructive mutation probes** — does the path *function*, not just exist.

---

## 1. Read the live realm flags (never the fixture)

Use the app's own admin client when one exists; otherwise a master `admin-cli` password grant.

```bash
# KC26 kcadm.sh has NO get-access-token subcommand — use a direct grant instead:
TOK=$(curl -s -X POST "http://localhost:8180/realms/master/protocol/openid-connect/token" \
  -d "client_id=admin-cli" -d "username=admin" -d "password=admin" \
  -d "grant_type=password" | jq -r '.access_token')

docker exec <kc-container> /opt/keycloak/bin/kcadm.sh config credentials \
  --server http://localhost:8080 --realm master --user admin --password admin

docker exec <kc-container> /opt/keycloak/bin/kcadm.sh get realms/<realm> \
  | jq '{resetPasswordAllowed, smtpServer, registrationAllowed, verifyEmail, rememberMe,
         loginWithEmailAllowed, duplicateEmailsAllowed, editUsernameAllowed,
         passwordPolicy, requiredActions: [.requiredActions[]?.alias]}'
```

**Decisive flags:**
- `resetPasswordAllowed: false` → self-service forgot-password OFF.
- `smtpServer: {}` → **no email can ever send**; the reset-email path is structurally broken.
- `requiredActions` containing `UPDATE_PASSWORD` → admin-driven reset is possible even when self-service is off.

Also enumerate the IdP cohort — it changes the answer:

```bash
docker exec <kc-container> /opt/keycloak/bin/kcadm.sh get identity-provider/instances -r <realm> \
  | jq '[.[] | {alias, providerId, enabled, trustEmail, syncMode: .config.syncMode}]'
```

An enabled `microsoft` IdP with `syncMode: FORCE` means those users hold **no Keycloak password at all** — their recovery lives in Entra ID, not Keycloak. Local-password and federated users need *different* recovery answers.

## 2. Confirm on the rendered login page (the only truth for "can the user click it")

Fetch the real auth URL and grep the HTML — do not trust the theme directory:

```bash
curl -s "http://localhost:8180/realms/<realm>/protocol/openid-connect/auth?client_id=<client>&redirect_uri=<uri>&response_type=code&scope=openid" \
  > /tmp/kc_login.html
grep -o -E 'reset-credentials[^"]*|loginResetCredentialsUrl|forgot' /tmp/kc_login.html   # empty = no link
grep -o -E 'id="social-[^"]*"|/realms/[^"]*broker/[^"]*' /tmp/kc_login.html            # broker IdP entry
```

- Absent `reset-credentials` href ⇒ no "Forgot password?" link **regardless of the theme source**.
- A custom theme may ship **only `template.ftl`** (no `login.ftl`, no `login-reset-password.ftl`) — the native link can't even render. Check `ls themes/<theme>/login/*.ftl`.
- The `broker/<alias>/login` href is the browser→Keycloak leg for the federated cohort; pair it with wherever that IdP's own password reset lives.

## 3. Non-destructive probes — existence vs function

**Existence, without mutating anyone's password:** issue the admin call against a syntactically valid but **nonexistent UUID**.

```
PUT /admin/realms/{realm}/users/00000000-0000-0000-0000-0000000000ff/reset-password
  {"type":"password","value":"Xx-Probe-9!","temporary":true}
→ 404 "User not found"  ⇒ route is LIVE and reaches Keycloak
→ 401/405               ⇒ route is not wired
```

**Function:** the same call against a *real* user, then read the error signature.

| Endpoint | Result | Meaning |
|---|---|---|
| `PUT .../users/{id}/execute-actions-email?lifespan=86400` `["UPDATE_PASSWORD"]` (real user) | **500** `Invalid sender address 'null'` | SMTP unconfigured — the email path is broken |
| same (bogus uuid) | 404 `User not found` | the route itself works |
| `GET /realms/{realm}/login-actions/reset-credentials` | **400** | self-service reset page unreachable (flag off) |

**Never probe this with a real user's password** when the user said read-only. Verify afterwards that the probe left no trace: `requiredActions` should still be `[]` and credentials unchanged.

## 4. Verify the *application's* credentials, not just an equivalent token

Probing with a master `admin-cli` token proves the *endpoint* works — not that the *app* can reach it. The app's admin client is usually a separate confidential client with `serviceAccountsEnabled: true`.

```bash
CID=$(grep -E '^KEYCLOAK_ADMIN_CLIENT_ID='      .env | cut -d= -f2- | tr -d '"')
CSEC=$(grep -E '^KEYCLOAK_ADMIN_CLIENT_SECRET=' .env | cut -d= -f2- | tr -d '"')
curl -s -X POST "http://localhost:8180/realms/<realm>/protocol/openid-connect/token" \
  -d "client_id=$CID" -d "client_secret=$CSEC" -d "grant_type=client_credentials" \
  | jq -r 'if .access_token then "OK" else .error end'
```

Then confirm the **conditional wiring** is actually taken at runtime:

```bash
docker exec <api-container> printenv | grep -i KEYCLOAK          # is ADMIN_CLIENT_ID set?
docker logs <api-container> 2>&1 | grep -iE "Admin client|NOT configured"
```

Look for `if cfg.KeyCloakAdmin.AdminClientID != "" { ...WithIdP(adapter)... } else { "endpoints will return errors" }` — a set-but-stale secret or an absent client silently disables every password/session endpoint.

## 5. Classify every hit by reachability

A `password`/`reset` grep across repos returns dead code, comments, and dormant config as often as live features. For each hit ask: is it in a registered route, called by any consumer, and reachable in the rendered UI?

**Trap — dead config:** grep the config *field*, the env *parse site*, the `.env` **value**, and every **consumer** separately. A field parsed from a populated env var with **zero consumers** is "configured and abandoned" — a much stronger finding than "unused field". Grepping only `internal/` hides the populated `.env` line.

```bash
grep -rn "FieldName\|ENV_VAR_NAME" --include=*.go . | grep -v _test.go   # then check .env for the value
```

**Trap — swallowed side-effect errors:** grep for `_ =` on capability-bearing calls:

```go
_ = s.idp.SendPasswordResetEmail(ctx, kcUserID) // non-fatal; user can use forgot-password
```

A discarded error plus a confident comment is a reliable marker of a silently broken user journey — here the email 500s *and* self-service recovery was disabled at realm level, so the comment was false in both clauses.

**Trap — UI calling a nonexistent store method:** a page can import and call a method the store never exports; it compiles, so it reads as "the feature exists". Verify the export list, not just the call site.

## 6. Report shape

Lead with the binary answer (yes / no / partial), then the evidence table, then the drift. Distinguish three separate claims that are easy to conflate:

- the endpoint **exists** (registered route),
- the endpoint **is wired** (client configured, branch taken),
- the endpoint **functions** (no 500, SMTP present, email delivered).

"Exists and works" without the third is a false headline.
