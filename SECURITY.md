# Security Policy

Nutrivo is a self-hosted nutrition/fitness tracker, designed to be run by a single administrator on their own infrastructure (a home server, a NAS, a small VPS) rather than as a multi-tenant public service.

Its threat model is a bit broader than a typical single-admin app, though: **anyone can self-register as a member** (email + password, no invitation required for the base tier). Member accounts are therefore *not* implicitly trusted — the design assumes a member account could be hostile, and access control is built around that. The administrator account is fully trusted; the goal is to protect the instance from external/unauthenticated attackers and from member accounts overstepping their access, not from a malicious admin.

## Supported Versions

Nutrivo does not currently follow a formal release/version numbering scheme.
Security fixes are applied to the `main` branch; if you are running a fork or an older checkout, please pull the latest `main` before assuming an issue is unpatched.

## Reporting a Vulnerability

If you find a security issue, please **do not open a public GitHub issue** for it.

Instead, use GitHub's private vulnerability reporting:

1. Go to the repository's **Security** tab.
2. Click **Report a vulnerability**.

This opens a private conversation with the maintainer, visible only to the two of you, so the issue can be discussed and fixed before any public disclosure.

Please include:

- What you found and why it's a security issue (not just "this seems wrong").
- Steps to reproduce, or a proof of concept if you have one.
- The affected file(s)/endpoint(s), if you know them.

There's no bug bounty — this is a personal project — but reports are genuinely appreciated, and you'll be credited (if you want to be) once a fix ships.

## What's already in place

For anyone evaluating whether to self-host this, here's a summary of the security-relevant design as it stands today:

**Authentication**

- Passwords hashed with bcrypt (PHP's `password_hash`/`password_verify`), for both member and admin accounts.
- Member passwords must be at least 12 characters, with at least one uppercase letter, one lowercase letter, and one digit, and are rejected if they appear in a list of known-compromised passwords (~46,000 entries of 12+ characters, sourced from [SecLists](https://github.com/danielmiessler/SecLists) and refreshed in the background on container startup — never blocking startup if the download fails).
- Optional TOTP-based two-factor authentication for admin accounts, enforced at login once enabled; disabling it requires re-entering the password, so a hijacked session alone can't downgrade it.
- Failed logins are rate-limited on both the client IP and the target email independently (tighter thresholds for admin than for members), with a time window lockout.
- Login response time doesn't depend on whether the email exists: a dummy password hash is checked even for unknown accounts, to close an easy timing side-channel for account enumeration.
- There is no API route to create an admin account beyond the very first one — additional admins are only created through a CLI script on the server itself, which enforces the same 12-character minimum. This keeps "create an administrator" off the network-facing attack surface entirely.
- The admin login page refuses to render on a mobile browser (checked both client- and server-side) — a deliberate choice to keep admin access tied to a desktop context.

**Tokens**

- API authentication uses short-lived JWTs (HS256, explicit algorithm, secret from an environment variable) — never cookies, so there is no session to forge cross-site in the first place.
- Refresh tokens are high-entropy random values; only their SHA-256 hash is stored server-side, and each one is single-use (rotated on every refresh).
- Third-party secrets a member connects (Garmin/Strava OAuth tokens, their own Gemini API key for photo estimation, the SMTP password in admin settings) are encrypted at rest (libsodium secretbox) — never stored or exposed in plaintext, not even to an admin reading the database.

**Access control**

- Four practical levels: visitor (no account), member, premium member, and administrator (a fully separate role, not just a member flag). Premium status is re-checked against the database on every request that needs it, rather than trusted from the JWT alone — so revoking premium takes effect immediately, not at next token refresh.
- Every mutation of a member-owned resource (a meal entry, a weigh-in, a connected account) verifies ownership against the authenticated member before acting — no ID belonging to another member can be read or altered by guessing/incrementing it.
- Recipes are only readable by premium members and admins, enforced server-side (not just hidden in the UI) — with a dedicated check rather than a blanket "premium-only" middleware, since admins also need access for content management.
- The in-app FAQ has three visibility tiers (visitor/member/premium), also enforced server-side: a visitor calling the endpoint directly gets exactly the visitor-tier entries, nothing more.

**Data handling**

- All database queries are parameterized; searches (`LIKE` included) bind the search term as a value, never interpolate it into the SQL string.
- User-facing text that originates outside our own admin input — food names from OpenFoodFacts/USDA, imported recipe content, anything rendered from a third-party API — is HTML-escaped before being inserted into the page. This was tightened after an internal review found it was inconsistently applied; see *Known trade-offs* below for the honest version of that story.
- Account deletion is two-step: anonymize (email replaced, login disabled, PII stripped) first, then a separate, explicit hard-delete — so "forgetting" a member and "erasing all trace of them" aren't the same click.

**Infrastructure**

- Runs as a single `php:8.2-apache` container behind a reverse proxy (Nginx Proxy Manager) terminating HTTPS.
- The client-IP-for-rate-limiting logic only trusts the `X-Forwarded-For` header when the direct connection is itself coming from a private/reserved IP range — see *Known trade-offs*, the reasoning is the same one as below.
- On startup, the container applies any pending database migrations before serving traffic; the compromised-password list refresh (see above) runs in the background and can never delay or block that startup.

**Photo-based calorie estimation (premium feature)**

- Uses the *member's own* Google Gemini API key, never a platform-wide one — a leaked or exhausted key only affects that one member's quota, and Nutrivo itself never pays for or is rate-limited by this feature's usage.
- Calls are rate-limited server-side (independent of the request retry/fallback logic below), so a member can't use this endpoint to hammer the server or run up their own quota unintentionally fast.
- On a transient failure (the model reporting it's overloaded), the request is retried across a small, configurable chain of models rather than hammering the same one — Google's hosted models get renamed/retired often enough that this list is a config value, not a hardcoded constant.

## Known trade-offs

- **Reverse-proxy IP trust.** The rate-limiter trusts `X-Forwarded-For` for the client IP, but only when the direct connection's own address is a private/reserved one — a reasonable proxy for "this came through my reverse proxy, not straight from the internet." This is a mitigation, not a guarantee: it depends on nothing else on your internal Docker network being able to reach the container directly with a private-range source address. If you ever expose the container's port in a way that lets another internal service (or container) reach it directly, that service could still spoof the header. Don't do that, or lock the header down to your proxy's exact address if you do.
- **JWTs live in the browser's `localStorage`, not an `HttpOnly` cookie.** This was a deliberate trade-off (it keeps the API stateless and avoids CSRF entirely, since nothing is auto-attached cross-site), but it means a successful script-injection bug would be able to read the token directly, unlike a cookie-based session. This is exactly why the escaping gap mentioned above was treated as critical rather than cosmetic once found — with this storage choice, there's no second line of defense once a page can run arbitrary script on it.
- **The escaping fix was reactive, not preventative.** For a stretch of this project's early history, several places rendered third-party text (food names in particular — OpenFoodFacts entries can be edited by anyone) into the page without escaping it. It was found and fixed across every call site found at the time, in the same pass as the two points above, but it was a genuine gap for a real period, not a theoretical one — worth knowing if you're running an older checkout, and worth being a little skeptical that every last call site was caught.
- **No CSP or other hardening response headers yet.** Nutrivo does not currently set a `Content-Security-Policy`, `X-Content-Type-Options`, or similar headers at the application level. Given the previous point, a CSP in particular would have meaningfully limited what a script-injection bug could do even before it was found — it's a real gap, not just a nice-to-have, and it's on the list rather than done.
- **External availability dependencies.** Both the compromised-password list (fetched from GitHub) and the photo-estimation feature (calling Google's API) depend on a third-party service being reachable at the relevant moment. Both are designed to degrade rather than break anything else if unavailable — registration still works without the password-list check, and a photo estimation simply fails with a clear error — but neither is under this project's control.
- **No formal versioning.** Same situation as sibling projects: fixes land on `main`, there's no tagged-release security-support window.
