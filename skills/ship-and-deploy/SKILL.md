---
name: ship-and-deploy
description: Take a project from "works on my machine" to deployed and monitored — environments, env vars, builds, CI, Vercel/Docker/Supabase deployment, domains, and a pre-launch checklist. Use this skill WHENEVER the user wants to deploy, publish, go live, ship to users, connect a domain, set up CI, or fix "works locally but broken in production". Trigger on "לעלות לאוויר", "להעלות את האתר", "דיפלוי", "פרודקשן", "deploy", "production", "publish", "hosting", "Vercel", "domain", "CI/CD" — and proactively at the end of any project when the user asks "מה עכשיו" or the app is feature-complete.
---

# Ship & Deploy

Deployment is not the last step — projects that treat it that way discover env, build, and data problems all at once on launch day. Apply the environment discipline from project start; run the launch checklist at the end.

## Environments: The Contract

Three environments minimum in concept, even for solo projects: **local** (dev machine), **preview** (per-branch/PR deploys), **production**. Rules:

- Same codebase everywhere; behavior differences come ONLY from env vars. No `if (isDev)` feature forks that make prod untested code.
- Separate resources per environment: separate DB (or at least separate Supabase project), separate API keys, separate payment mode (Stripe test vs live). Dev pointed at the prod database is how data gets destroyed.
- Config precedence documented: `.env.local` > `.env` > platform dashboard. One source of truth per environment.

## Env Vars Discipline

- `.env` in `.gitignore` before first commit; committed `.env.example` lists every var with placeholder + one-line comment. This file IS the deployment documentation.
- Validate env at boot, fail fast with a clear message — a missing var should crash startup, not produce `undefined` bugs at 2am:

```ts
import { z } from "zod";
export const env = z.object({
  DATABASE_URL: z.string().url(),
  STRIPE_SECRET_KEY: z.string().startsWith("sk_"),
  NEXT_PUBLIC_APP_URL: z.string().url(),
}).parse(process.env);
```

- Client-exposed vars (`NEXT_PUBLIC_`/`VITE_`) are public — secrets never get these prefixes. Audit before launch.
- Rotating a leaked key beats scrubbing git history. Assume committed = leaked (see app-security skill).

## Build Must Be Boring

- `npm run build` locally must pass before any deploy attempt. Type errors, lint errors, and "it builds on Vercel differently" all get resolved locally first.
- Pin the Node version (`engines` in package.json / `.nvmrc`) — version drift between laptop and platform is a classic "works locally" cause.
- Lockfile committed, always. `npm ci` (not `install`) in CI/builds for reproducibility.
- Never `ignoreBuildErrors: true` / `eslint: { ignoreDuringBuilds: true }` to "just ship" — that converts compile-time errors into runtime user-facing errors.

## Platform Playbooks

**Vercel (default for Next.js/static/frontend-heavy):**
1. Push repo to GitHub → import in Vercel → framework auto-detected.
2. Set env vars in dashboard per-environment (Production / Preview / Development) BEFORE first deploy; redeploy after changes (env is baked at build for client vars).
3. Preview deploys per PR are free QA — share the preview URL, test there, then merge to main = production.
4. Serverless constraints to design around: function timeout (default 10s hobby), no persistent filesystem, cold starts. Long jobs → background functions/queue (see backend skill).

**Supabase (DB/auth/storage):**
- Local dev with `supabase start`; schema changes via migration files; `supabase db push` / CI migration step to prod — never hand-edit prod tables (see database-design skill).
- Confirm RLS enabled on every table before launch; service-role key server-side only.

**Docker (when the app isn't platform-shaped — long-running processes, custom binaries, on-prem clients):**
- Multi-stage build (deps → build → slim runtime), non-root user, `.dockerignore` (node_modules, .env, .git).
- One process per container; compose for app+db locally.
- Healthcheck endpoint (`/healthz` returning 200 + DB ping) — orchestrators and uptime monitors need it.

**Static/game builds (itch.io, S3, GitHub Pages):** relative asset paths, test the production bundle locally (`npx serve dist`) — path and CORS issues appear only in built output.

## CI: Minimum Viable Pipeline

Every push: install → lint → typecheck → test → build. GitHub Actions skeleton:

```yaml
name: ci
on: [push, pull_request]
jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - uses: actions/setup-node@v5
        with: { node-version: 22, cache: npm }
      - run: npm ci
      - run: npm run lint && npm run typecheck
      - run: npm test
      - run: npm run build
```

Main branch deploys automatically only if CI is green. This is the whole point: unverified code cannot reach users.

## Domains & HTTPS

- Domain via registrar → platform's DNS instructions (CNAME/A records). Propagation up to 48h — do this days before a launch date, not launch morning.
- HTTPS is automatic on Vercel/Netlify/Cloudflare — verify the redirect http→https and www→apex (or reverse, pick one canonical).
- Set the canonical URL env var (`NEXT_PUBLIC_APP_URL`) — OAuth callbacks, emails, and OG tags break with mismatched URLs.

## Observability: Minimum for Real Users

- **Error tracking**: Sentry (or platform equivalent) on both client and server, with source maps, from day one of real users. Console.log in production is not observability.
- **Uptime**: external ping on `/healthz` (UptimeRobot/BetterStack free tier) with alert to email/WhatsApp.
- **Logs**: structured (JSON), request-id correlated (matches the error contract in backend-api-design skill). Know where platform logs live BEFORE the incident.
- **Analytics**: at minimum page views + the one core conversion event — you can't improve what you can't see.

## Backups & Rollback

- DB: verify the platform's automatic backups are ON and know the restore procedure — a backup never test-restored is a hope, not a backup. Before risky migrations: manual snapshot.
- Rollback plan: Vercel = promote previous deployment (instant); Docker = keep previous image tag. Practice the rollback once before you need it.
- User-uploaded files: object storage with versioning, not container disk.

## Pre-Launch Checklist (run and report item-by-item)

1. `npm run build` clean locally; CI green on main
2. All env vars set in platform for production; boot-time validation passes; no secrets in client bundle (grep the build output)
3. Separate prod resources: DB, keys, payment live-mode consciously enabled
4. RLS / authorization verified on prod data paths (app-security review done)
5. Migrations applied to prod DB via files, schema matches code
6. Custom domain + HTTPS + canonical redirect working; OAuth callback URLs updated to prod domain
7. Error tracking receiving a test event; uptime monitor active
8. Backups verified ON + restore procedure known; rollback tested once
9. 404/500 pages exist and are branded; favicon + OG image + title/description set
10. Real-device pass: mobile + slow network (throttle) + RTL rendering if Hebrew
11. Legal minimum if collecting user data: privacy policy + contact
12. Post-launch watch: check error tracker and logs at +1h and +24h — schedule it
