# Spec: Local Spike — Go / No-Go
_Number: 001 | Created: 2026-10-13_

## Summary
A one-day technical proof-of-concept that validates every high-risk integration before any product code is written. If anything fails, alternatives are surfaced before committing to the architecture.

## User Flows
### Happy Path
1. Developer runs `npx expo run:ios` on the simulator.
2. An API route (`/api/health`) executes a Prisma read and write against the Neon `dev` branch.
3. A test Inngest function is triggered via `npx inngest-cli dev` and appears in the Dev Server UI.
4. The API route verifies a Clerk session token and returns the user's Clerk ID.
5. The API route triggers a Sentry event visible in the Sentry dashboard.
6. All four checks pass → spike is green.

### Alternative Flows
- Any check fails → document the failure, evaluate alternatives (Drizzle vs raw Neon driver; Inngest edge handler variant), and surface options before proceeding.

## Acceptance Criteria
- [ ] `GET /api/health` returns 200 with a DB round-trip confirmed (Prisma + Neon adapter).
- [ ] Inngest Dev Server UI shows a completed run for the test function.
- [ ] A valid Clerk Bearer token is verified server-side via `@clerk/backend`; an invalid token returns 401.
- [ ] A Sentry test event appears in the project dashboard (both client-side `Sentry.captureException` and server-side `reportError()`).
- [ ] No host-specific APIs used (standard `Request`/`Response`, env vars only).
- [ ] `.env.example` is committed with all required keys, no values.

## Edge Cases
- **Prisma + Metro bundler conflict**: Metro may fail to bundle Prisma's native code. Mitigation: try `@prisma/adapter-neon` with the serverless driver first; if blocked, evaluate Drizzle.
- **Inngest adapter**: `inngest/edge` or `InngestCommHandler` must work within Expo Router API routes. Test both if the first fails.
- **Apple sign-in in simulator**: not available in Expo Go; requires a dev build (`npx expo run:ios`). Google sign-in should work in both.

## Out of Scope
- Any product UI beyond a bare `/api/health` route.
- EAS Build, TestFlight, or any deployment target.
- Integration or E2E tests (those belong to Phase 5).

## Open Questions
- If Prisma + Metro is unresolvable, do we switch to Drizzle or the raw Neon HTTP driver? _(Decision deferred to spike outcome.)_
- Which Inngest handler variant (`inngest/edge` vs `InngestCommHandler`) works best in Expo Router?

## Affected Areas
- `app/api/health+api.ts` — spike route
- `server/db.ts` — Prisma client setup with Neon adapter
- `server/inngest/client.ts` and `server/inngest/functions/test.ts` — test function
- `server/sentry.ts` — `reportError()` wrapper
- `.env`, `.env.example`
- `package.json` — dependency versions pinned (Prisma **7.10.0**, Inngest 4.x, `@clerk/expo` 4.x, `@clerk/backend`)
