# Acceptance — Definition of Done

> ⚠️ THIS IS AN EXAMPLE / SAMPLE — not a real feature.
> It shows what a *filled-in* `acceptance.md` looks like "in the wild."
> Feature used for illustration: **"Password reset via email"**.
> Copy `../acceptance.md` (the blank template) for real work, not this file.

## Feature

Allow a registered user who forgot their password to reset it via a one-time,
time-limited link sent to their verified email. After a successful reset the
user is logged out of all existing sessions and must sign in with the new
password. This closes a top support-ticket category and removes the need for
manual admin password resets.

## Acceptance criteria

- [ ] `POST /auth/forgot-password` with a known email returns `202` and sends a
      reset email containing a single-use token link.
- [ ] The same endpoint with an **unknown** email also returns `202` and sends
      nothing (no user-enumeration leak).
- [ ] The reset token expires after **30 minutes** and is **single-use**
      (second use returns `410 Gone`).
- [ ] `POST /auth/reset-password` with a valid token + new password updates the
      hash (argon2id), returns `200`, and invalidates all of the user's other
      sessions.
- [ ] New password must satisfy the existing password policy (min 12 chars);
      a weak password returns `422` with a clear message.
- [ ] Rate limit: max **5** forgot-password requests per email per hour;
      6th returns `429`.
- [ ] All new endpoints are covered by unit + integration tests; coverage on
      the new module ≥ 90%.

## Non-goals / out of scope

- SMS / phone-based reset (email only for this epic).
- Changing the existing login or signup flows.
- Admin-initiated resets (already exists, untouched).
- Account lockout / brute-force detection beyond the rate limit above.

## Constraints

- **Performance:** forgot-password request returns in < 300 ms p95 (email is
  queued async, not sent inline).
- **Security:** tokens are 256-bit, stored hashed (never plaintext) — see
  ADR-0001. No user enumeration. Secrets via env only.
- **Compatibility:** no breaking changes to existing `/auth/*` endpoints;
  additive migration only.
- **Deadline / budget:** target 1 PR, ~1 day of loop time.

## Verification command

```
npm test && npm run lint && npm run typecheck
```
