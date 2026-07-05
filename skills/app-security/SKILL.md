---
name: app-security
description: Application security for everything Claude builds — authentication, secrets, injection prevention, XSS/CSRF, file uploads, rate limiting, and a full security review workflow. Use this skill WHENEVER building or reviewing login/signup flows, password or token handling, payment code, file uploads, user-generated content, admin panels, or any app that will face real users. Also trigger on "אבטחה", "בדיקת אבטחה", "התחברות", "הרשמה", "סיסמאות", "security review", "audit", "is this safe", "pen test", or before any deployment of an app with user data. Do NOT wait for the user to ask about security — apply this proactively to any auth or data-handling code.
---

# Application Security

Two modes: **build mode** (applying secure patterns while writing code) and **review mode** (auditing existing code). Both use the same rules; review mode adds the workflow at the bottom.

Scope note: this skill covers application security. For prompt-injection defense in agents, use the `prompt-injection-defense` skill. For infrastructure/secrets-in-deployment, coordinate with `ship-and-deploy`.

## The Five Vulnerabilities That Actually Ship in AI-Generated Apps

Prioritize these — they account for most real incidents in vibe-coded projects:

1. **Missing object-level authorization (IDOR)** — endpoint checks "is logged in" but not "owns this resource". Every query touching user data includes the ownership filter IN the query.
2. **Secrets in client code** — API keys in React components, service-role keys in frontend, keys committed to git. Anything bundled to the browser is public. `NEXT_PUBLIC_`/`VITE_` prefixed vars are PUBLIC by definition.
3. **Trusting client-side checks** — hiding the admin button is UX, not security. Every privileged action is re-verified server-side.
4. **Unvalidated input reaching interpreters** — SQL, HTML, shell, file paths. Covered per-sink below.
5. **Missing rate limiting on auth and expensive endpoints** — login, signup, password reset, OTP, and anything that sends email/SMS gets rate-limited per-IP AND per-account.

## Authentication

Default recommendation: use a managed provider (Supabase Auth, Clerk, Auth.js, Firebase Auth). Hand-rolling auth is justified only as a learning exercise — say so explicitly.

If implementing manually:
- Passwords: bcrypt (cost ≥ 12) or argon2id. Never SHA/MD5, never reversible encryption, never plaintext — including in logs.
- Sessions: httpOnly + Secure + SameSite=Lax cookies. Tokens in localStorage are readable by any XSS.
- Login errors are uniform: "Invalid email or password" — never reveal which field failed, and signup/reset must not confirm whether an email exists (enumeration).
- Password reset tokens: single-use, ≤1h expiry, hashed at rest, invalidated on password change along with all sessions.
- JWT: verify signature AND algorithm (reject `alg:none`), short expiry with refresh rotation, and a revocation strategy — "stateless" JWTs that can't be revoked are a liability for anything sensitive.

## Injection Prevention, Per Sink

- **SQL**: parameterized queries / ORM only. String-built SQL is forbidden even for "internal" values. If the user's code concatenates SQL, flag it as critical regardless of context.
- **HTML/XSS**: rely on framework escaping (React JSX, Vue templates). `dangerouslySetInnerHTML` / `v-html` / `innerHTML` require sanitization with DOMPurify and a comment justifying use. User-controlled URLs in `href` must be protocol-checked (`javascript:` URIs).
- **Shell**: never interpolate user input into shell commands. Use spawn with argument arrays, or better, a library instead of shelling out.
- **Path traversal**: user-supplied filenames never touch the filesystem path directly. Generate server-side names (UUID), or normalize + verify the resolved path stays inside the allowed directory.
- **NoSQL/query objects**: reject non-string types where strings are expected (`{$gt:""}` login bypass in Mongo).

## CSRF & CORS

- State-changing endpoints: SameSite cookies + framework CSRF protection, or require a custom header (fetch-only APIs). GET requests never mutate state.
- CORS: explicit origin allowlist. `Access-Control-Allow-Origin: *` combined with credentials is never acceptable. Reflecting the request Origin without a whitelist equals `*`.

## File Uploads

- Validate type by magic bytes, not extension or Content-Type header (both attacker-controlled).
- Enforce size limits at the web server layer, not only in app code.
- Store outside the web root or in object storage; serve via signed URLs; set `Content-Disposition` and never execute/include uploaded content.
- Strip EXIF from user images (location privacy).
- SVG uploads are XSS vectors — sanitize or convert.

## Secrets & Configuration

- All secrets in env vars; `.env` in `.gitignore` BEFORE the first commit; provide `.env.example` with placeholders.
- If a key was ever committed: rotate it. Removing from git history is not enough — treat it as leaked.
- Separate keys per environment. Production keys never on dev machines.
- Client/server boundary audit: grep the client bundle for key patterns before shipping.

## Security Headers & Transport (web apps)

Minimum set: `Content-Security-Policy` (start restrictive, allow only what's needed), `X-Content-Type-Options: nosniff`, `Referrer-Policy: strict-origin-when-cross-origin`, `Strict-Transport-Security` in production. Frameworks/hosts often set some — verify, don't assume.

## Rate Limiting & Abuse

- Auth endpoints: aggressive limits (e.g., 5/min/IP + per-account lockout with exponential backoff).
- Expensive endpoints (AI calls, email, search): per-user quotas. An unmetered AI endpoint is an open wallet.
- Public forms: honeypot field + server-side validation minimum; CAPTCHA if abused.

## Logging & Privacy

- Log auth events (login success/fail, password change, permission change) with user + IP — you cannot investigate what you didn't record.
- Never log: passwords, tokens, full card numbers, session IDs, OTPs. Mask PII where practical.
- Errors to users are generic; errors to logs are complete.

## Review Mode Workflow

When asked to audit (or before any deploy of a user-data app), do this in order and produce a findings report:

1. **Map attack surface**: list every route/action/function reachable from outside, its auth requirement, and its inputs. Unauthenticated + accepts input = highest priority.
2. **Trace the five killers** (section above) across that map. For each endpoint ask: validated? authorized at object level? rate-limited if sensitive?
3. **Grep pass** for known smells: string-concatenated queries, `dangerouslySetInnerHTML`, `eval`, `exec(`, hardcoded key patterns (`sk-`, `AKIA`, `eyJ`), `NEXT_PUBLIC_` vars holding secrets, `cors: '*'`, `verify=False`, `rejectUnauthorized: false`.
4. **Secrets audit**: check git history (`git log -p | grep`-style) and client bundles, not just current source.
5. **Report format**: table of findings with Severity (Critical/High/Medium/Low), Location, Issue, Exploit scenario (one line, concrete), Fix. Criticals get the fix implemented immediately, not just reported.

Severity guide: Critical = exploitable now for data access/money (SQLi, IDOR on user data, leaked prod key). High = exploitable with conditions. Medium = defense-in-depth gap. Low = hardening.

## Honest-Communication Rule

Never tell the user their app is "secure." Report what was checked, what was found, what was fixed, and what was NOT covered (e.g., "dependency CVEs and infrastructure were out of scope"). Security is a posture, not a certificate — teach this framing.
