---
name: repro-token-revocation-mismatch
description: Prove or disprove that a token revocation/logout path is a silent no-op because the writer and reader key the same store differently — when the existing test hand-seeds the key and therefore passes on buggy and fixed code alike.
---

# Reproduce an asymmetric token-revocation mismatch

Use when a logout / blacklist / deny-list / cache-invalidation path may be a no-op:
the code looks right on both sides, tests pass, but revoked tokens seem to stay valid.
The signature failure is **writer and reader keying the same store with different values**,
hidden by a test that seeds the key by hand instead of exercising the writer.

## 1. Read both sides and name the key on each

Do not read only the middleware/guard. Find the **writer** (the revoke/blacklist call site)
and the **reader** (the lookup), and write down the exact expression each one uses
as the key. Typical asymmetry:

- writer: `sha256(raw_token)` because structured parsing of the token failed for this token type;
- reader: `claims.jti` / `claims.sub` whenever that field is non-empty, hashing only as a fallback.

If the two expressions can differ for the token type actually in play, you have the bug.

Trace *every* path that reaches the writer (logout, refresh, session-expiry) and confirm they
all funnel to the same helper — otherwise you are fixing one of N call sites.

## 2. Prove the key is really non-empty in production shape

A mismatch only bites if the reader's preferred branch wins. Confirm the field is populated
by the real issuer — do not assume it from the struct definition.

Mint a real token from the running issuer and decode the payload:

```bash
curl -s -X POST "$TOKEN_ENDPOINT" \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -d "grant_type=password&client_id=$CID&client_secret=$SECRET&username=$U&password=$P&scope=openid"
# then base64url-decode the access token's middle segment and print the claim of interest
```

Check the claim is present **and non-empty**. Empty means the reader's fallback applies and
there is no bug. Non-empty means the reader's preferred branch always wins.

## 3. Reproduce with the real writer logic, not a hand-written key

This is the step that distinguishes proof from speculation. Write a throwaway test that:

1. computes the key exactly as the **writer** does (call the writer helper if you can);
2. seeds the store with that value;
3. drives the real **reader** path with a token/claims fixture whose preferred key is the
   *other* value;
4. asserts the guard rejects.

A failing assertion with `next called = true` is the proof. Log both keys explicitly — the
log line naming `writer stored key = <hash>` vs `reader key checked = <jti>` is the artifact
worth pasting into the report.

Delete the probe file afterwards and re-verify the tree is clean.

## 4. Bound the severity before reporting it

Revocation bypass severity depends entirely on token lifetime. Read the issuer's
`accessTokenLifespan` (and any per-client override, which silently beats the realm default)
so the impact statement is `<=15 minutes` rather than `indefinitely`. A defect with a correct
bounded window reads as credible; an unbounded claim invites dismissal.

State clearly whether the defect is pre-existing (reader logic identical in the base revision)
or introduced, and say which revision you compared against.

## 5. Report the test that hid it

Any existing test that seeds the store with the same literal the fixture's preferred key
contains is a **false-confidence test**: it passes on buggy and fixed code alike. Name it,
quote the seeded value, and state that closing the defect requires a test whose seed comes
from the writer path. This is usually the most actionable line in the whole report.

## Pitfalls

- Fixing the reader to "also check the hash" silently changes behaviour for the other token
  type; prefer making both sides derive the key from one shared function.
- A `jti`-keyed deny-list also breaks when the issuer reuses or omits jti across refresh —
  check refresh-token behaviour, not just login.
- If the store is a cache with TTLs set by the writer, a mismatched key can look "fixed"
  after the TTL expires; re-run the read before concluding anything.
