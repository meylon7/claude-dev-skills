---
name: testing-strategy
description: >-
  Decide what to test, write high-value tests (Vitest/Jest, Playwright, pytest), and run a verification loop so features are proven working — not assumed working. Use this skill WHENEVER implementing any non-trivial feature or bugfix, when the user asks "בדיקות", "טסטים", "tests", "לבדוק שזה עובד", "QA", "it's broken again", or when a bug the user already fixed reappears. Also apply proactively: after building any backend endpoint, payment/auth flow, or data transformation, tests are part of "done" — don't wait to be asked.
---

# Testing Strategy

The goal is not coverage percentage. The goal is: when Claude says "done", it's demonstrated, and when code changes later, regressions announce themselves. AI-assisted development makes this MORE important — code gets rewritten wholesale, and tests are the contract that survives rewrites.

## What to Test, In Priority Order

Test value = (probability of breaking × cost of breaking) ÷ cost of writing. Ranked:

1. **Money and irreversible actions** — payments, credits, deletions, emails sent. One test here outvalues fifty UI tests.
2. **Auth and permissions** — for each protected resource: correct user succeeds, wrong user gets 403/404, anonymous gets 401. These are three cheap tests per endpoint and they catch IDOR, the most common real vulnerability.
3. **Business logic with branches** — pricing rules, eligibility, state transitions, date math. Pure functions: test exhaustively including edges (zero, negative, empty, max, timezone boundaries).
4. **Data transformations** — parsers, importers, format converters. Feed them real-world ugly samples, not clean fixtures.
5. **Critical-path E2E** — the 2-3 flows that ARE the product (signup→core action→result). One Playwright test each.

Explicitly deprioritize: trivial getters, framework behavior, styling, third-party libraries, and exhaustive UI-state testing. Snapshot tests of whole components are noise — they fail on every intentional change and get blindly regenerated.

## Test Shape Rules

- Structure: Arrange–Act–Assert, one behavior per test. Name describes behavior, not method: `rejects order when inventory is zero`, not `test handleOrder 2`.
- Test through the public interface (the API route, the exported function). Tests coupled to private internals break on refactor without catching bugs — the exact opposite of their job.
- Each test independent: fresh state, no reliance on execution order, parallel-safe.
- Determinism: inject clocks (`vi.useFakeTimers` / freezegun) and seed randomness. A flaky test is worse than no test because it trains everyone to ignore failures.
- Mock at system boundaries only (network, payment provider, email, LLM APIs). Do NOT mock your own modules to make a test pass — that tests the mock. For the database, prefer a real ephemeral DB (SQLite in-memory, or Postgres via testcontainers/Supabase local) over mocking the ORM.

```ts
// Vitest example of the auth trio for one endpoint
describe("GET /api/orders/:id", () => {
  it("returns the order for its owner", async () => { /* 200 + body */ });
  it("returns 404 for another user's order", async () => { /* not 403 — don't confirm existence */ });
  it("returns 401 when unauthenticated", async () => { /* ... */ });
});
```

## The Verification Loop (how Claude must work)

For every feature/bugfix, in this order:

1. **Before coding**: state the acceptance checks — "this is done when X, Y, Z pass." For a bugfix: FIRST write a failing test that reproduces the bug. A bug without a reproducing test will return.
2. **Implement.**
3. **Run the tests and show real output.** Never claim tests pass without executing them. If the environment can't run them, say so explicitly instead of implying success.
4. **Run the full suite**, not just the new tests — the point is catching regressions elsewhere.
5. A feature with failing or skipped tests is NOT done. Don't delete or weaken a failing assertion to get green; the test is the spec — if the spec changed, say so and change it deliberately.

TDD (test-first) is the default for: pure logic, parsers, pricing, anything with a crisp spec. Test-after is acceptable for exploratory UI work — but the auth trio and money paths still get tests before merge.

## Stack Defaults

- **TypeScript/JS**: Vitest (unit/integration), Playwright (E2E), MSW for mocking HTTP at the network layer.
- **Python**: pytest + fixtures, httpx test client for APIs.
- Scripts: `test`, `test:watch`, `test:e2e` in package.json; tests colocated (`foo.test.ts` next to `foo.ts`) or in `tests/` mirroring src — pick one, stay consistent.
- CI: tests run on every push (see ship-and-deploy skill). A test that only runs on a laptop protects nothing.

## E2E With Playwright — Keep It Small and Solid

- 2–5 journeys max for a small product. E2E is expensive and brittle; it's a smoke layer, not the pyramid's body.
- Select by role/label/test-id (`getByRole('button', { name: 'שלם' })`), never by CSS classes that styling changes will break. For Hebrew RTL apps, role/label selectors also verify accessibility naming for free.
- Use Playwright's auto-waiting; explicit `waitForTimeout` is a flakiness confession.
- Seed test data via API/DB setup, not by clicking through the UI to arrange state.

## Coverage Philosophy

Track coverage to find UNtested critical code, never as a target. 60% covering money+auth+logic beats 95% padded with trivialities. If asked to "raise coverage," respond by finding the highest-risk uncovered paths and testing those.

## Definition of Done (report this to the user)

1. Acceptance checks stated up front and now demonstrably passing (output shown)
2. Bugfixes include the regression test that failed before the fix
3. Auth trio present on every protected endpoint touched
4. Money/irreversible paths tested including failure branches
5. Full suite green, zero flaky/skipped tests introduced
6. Tests run in CI, not just locally
