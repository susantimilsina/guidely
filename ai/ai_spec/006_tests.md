# Spec: Tests — Unit, Integration & Maestro E2E
_Number: 006 | Created: 2026-10-13_

## Summary
Defines the complete testing strategy for v1: unit tests for core business logic, API integration tests against a dedicated Neon test branch, and Maestro E2E flows on the local iOS simulator dev build.

## User Flows
_(Tests are automated; flows describe what each suite exercises.)_

### Unit Tests
- Quota rule: counts correctly, blocks the 4th trip, failed trips don't count, in-flight trips do count.
- Zod output schemas: valid and invalid AI output samples parse or reject correctly.
- Prompt builder: given inputs, produces the expected prompt string and `promptVersion`.
- `generate-trip` steps: each step tested in isolation with Gemini and Unsplash calls mocked.

### Integration Tests (Neon test branch)
- Auth rejection: every endpoint returns 401 without a valid Clerk token.
- Ownership: `GET /api/trips/:id` and `DELETE /api/trips/:id` return 404 for a trip owned by another user.
- Quota enforcement: the 4th trip is rejected with 429; a failed trip does not consume quota.
- Trip CRUD: create → list → get → delete, asserting DB state after each step.
- Webhook signature: `POST /api/webhooks/clerk` rejects invalid signatures (401) and processes valid ones.
- `generate-trip` function: triggered via Inngest Dev Server in the test run; asserts the trip reaches `ready` status and all output fields are populated.

### Maestro E2E (local iOS simulator dev build)
1. Sign in with a test Apple/Google account.
2. Create a trip (Kathmandu, 3 days, 2 travelers, moderate, food + culture).
3. Wait on the loading screen → trip reaches detail within 60 s.
4. Verify hero image, day tabs, hotel cards, and budget breakdown are visible.
5. Navigate to My Trips → delete the trip → confirm it disappears.
6. _(Optional)_ Attempt a 4th trip in the same day → verify the blocked state.

## Acceptance Criteria
- [ ] `npm test` runs all unit and integration tests; all pass on a clean checkout with a `.env` pointing to the Neon test branch.
- [ ] Integration tests use a dedicated Neon test branch (not `dev` or `main`) and clean up after themselves.
- [ ] Each `generate-trip` Inngest step has at least one unit test with mocked external dependencies.
- [ ] Zod schema tests cover at least: a fully valid AI output, a missing required field, and an out-of-range value.
- [ ] Quota unit tests cover: 0 trips (allowed), 2 trips (allowed), 3 active trips (blocked), 3 active + 1 failed (blocked — failed don't free quota beyond 3).
- [ ] Integration tests clean up all created rows after each test (use a transaction rollback or explicit teardown).
- [ ] `maestro test .maestro/` passes on the local iOS simulator dev build with a real Gemini call.
- [ ] Maestro tests do not depend on specific AI-generated content (use existence checks, not exact-match assertions).
- [ ] Webhook integration test exercises: `user.created` insert, `user.updated` upsert, `user.deleted` soft-delete.

## Edge Cases
- **Inngest in integration tests**: use the Inngest Dev Server in test mode or `inngest/testing` utilities to avoid hitting Inngest Cloud.
- **Clerk in integration tests**: use a test Clerk instance; never use the production instance in automated tests.
- **Maestro test Apple sign-in**: Apple sign-in requires a real device or dev build. Set up a dedicated test account in the Clerk dev instance.
- **Flaky E2E due to AI latency**: Maestro flow should wait up to 90 s for the trip to reach `ready` before failing.
- **Test branch isolation**: integration tests must not write to `dev` or `main` Neon branches.

## Out of Scope
- Performance/load testing.
- Android E2E.
- Sentry alert rule configuration (deferred post-v1).
- Visual regression testing.

## Open Questions
- **Test runner**: Jest for unit and integration tests? _(Recommend Jest + `ts-jest`, consistent with Expo's ecosystem.)_
- **Integration test DB teardown**: transaction rollback per test (fastest) or explicit `DELETE` (more realistic)? _(Recommend transaction rollback for speed.)_
- **CI**: no CI pipeline in v1 (local only). Tests are run manually. _(Document this assumption to avoid confusion later.)_

## Affected Areas
- `__tests__/unit/quotaService.test.ts`
- `__tests__/unit/tripOutputSchema.test.ts`
- `__tests__/unit/promptBuilder.test.ts`
- `__tests__/unit/generateTrip.steps.test.ts`
- `__tests__/integration/trips.test.ts`
- `__tests__/integration/webhooks.test.ts`
- `__tests__/integration/generateTrip.function.test.ts`
- `.maestro/happy-path.yaml`
- `.maestro/delete-trip.yaml`
- `jest.config.ts`, `jest.setup.ts`
- `.env.test` (or test-specific env vars for Neon test branch and Clerk test instance)
