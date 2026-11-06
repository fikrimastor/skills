---
name: auditing-oidc-jwt-validation-gaps
description: "Statically determine exactly which checks a backend's OIDC/JWT validator actually performs (signature, alg pinning, exp/nbf, issuer, audience, introspection, revocation) and report the gaps — instead of trusting docs or the presence of a ValidateToken function. Use when asked how a service validates Keycloak/OIDC tokens, or before assuming an access token is fully verified."
---

# Auditing what an OIDC/JWT validator actually checks

Goal: produce a checked-vs-unchecked table grounded in source, not a description of intent.
A validator can exist, be named `ValidateToken`, and still be missing audience, alg pinning, or revocation.

## Procedure

1. **Find every call site, not just the validator.** `grep -rn 'ValidateToken(' <repo>/internal`.
   Separate two distinct things:
   - the **exchange** path (authorization_code / password / refresh grant) — often delegates all
     verification to the IdP and does no local check at all;
   - the **re-validation** path where the returned access token is parsed locally.
   Report both. "The backend verifies the token" is ambiguous until you say which layer.

2. **Read the `jwt.ParseWithClaims` options list literally.** Each option is one check:
   - `WithValidMethods([...])` → alg pinning. Absent = alg confusion risk (also check the keyfunc
     for a signing-method type assertion).
   - `WithIssuedAt`, `WithExpirationRequired`, `WithLeeway` → time claims.
   - `WithIssuer`, `WithAudience`, `WithSubject` → registry/claim checks.
   - jwt/v5 validates `exp`/`nbf` by default; `iat` only with `WithIssuedAt`.

3. **Prove absence by grep, not by reading.** `WithAudience` / `WithIssuer` / `WithExpirationRequired`
   / `token/introspect` / `introspect` over the whole repo. Zero hits is the evidence.

4. **Hunt dead validation fields.** A struct field named e.g. `expectedIssuer` that is assigned in the
   constructor but never read in the validate method means issuer checking happens through some *other*
   mechanism (a map, a switch) or not at all. Grep the field name; if it only appears at its declaration
   and assignment, it is dead.

5. **Check what the issuer check actually covers.** Dual-URL deployments (internal service URL vs
   browser-facing public URL) often widen validation to a *set* of acceptable issuers. Note whether the
   map is built from discovery output — that can make the accepted set broader than intended.

6. **Key material handling.** Where do JWKS keys come from, is there a refresh TTL, and is the refresh
   de-duped (singleflight/mutex) so concurrent requests don't stampede the IdP? Is there a
   use-stale-on-refresh-failure path? Stale-key fallback is a deliberate availability tradeoff — name it.

7. **Revocation timing.** A blacklist/introspection check in per-request middleware is NOT the same as
   one at exchange. Check whether a token revoked after mint still gets a fresh session at exchange time.

8. **Separate authentication from authorization.** Signature validity is not permission to log in.
   Find the provisioning/registration gate and report it as its own stage.

## Output shape

Two tables: the positive path (what is checked, with file:line), and a gaps table with the grep evidence
for each absence. Then one sentence naming the load-bearing gap and why an attacker cares — e.g. no
`aud` check means a token minted by the same realm for a *different* client validates fine.

## Traps

- Do not conclude "validated" from an option list that omits audience. Signature + issuer is a real,
  common, and materially weaker posture.
- Do not treat a passed unit test named `ValidateToken_WhenTokenValid` as coverage — read what claims
  that test asserts.
- `expectedIssuer`-style fields and `validIssuers` maps can coexist; the field being dead does not mean
  issuer checking is absent, and vice versa.
