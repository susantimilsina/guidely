# Spec: Trip API & AI Generation
_Number: 004 | Created: 2026-10-13_

## Summary
Implements all trip API endpoints, the 3-trips/day quota service, the `generate-trip` Inngest function (Gemini + Unsplash), and the Zod schemas that validate AI output before it is persisted. This is the core backend of Guidely.

## User Flows
### Happy Path — Create & Generate
1. Client sends `POST /api/trips` with a Clerk Bearer token and validated trip inputs.
2. Server verifies the JWT, checks quota (< 3 active trips today for this user), inserts a `pending` trip, sends `trip/generate.requested`, returns `{ tripId }`.
3. Inngest `generate-trip` runs:
   - **Step 1 `mark-generating`**: sets status → `generating`.
   - **Step 2 `normalize-destination`**: Gemini call returns `{ valid, name, country, currency }`. Invalid destination throws `NonRetriableError`.
   - **Step 3 `generate-plan`**: Gemini structured-output call. Zod parse failure retries (up to 3 total).
   - **Step 4 `fetch-cover`**: Unsplash search. Non-fatal: on failure, falls back to a bundled placeholder.
   - **Step 5 `save`**: writes outputs, token counts, `completedAt`; sets status → `ready`.
4. Client polls `GET /api/trips/:id` every 2.5 s and sees `ready` → renders trip detail.

### Alternative Flows
- **Quota exceeded**: `POST /api/trips` returns 429 with `{ error: "quota_exceeded" }`.
- **Invalid destination**: step 2 throws `NonRetriableError`; `onFailure` sets `failureReason`, status → `failed`.
- **Gemini Zod parse failure**: step 3 retries up to 3 times; if all fail, `onFailure` handles it.
- **Unsplash failure**: step 4 catches the error, logs to Sentry, and the trip saves with the placeholder image.
- **Client polls > 3 minutes**: trip remains in DB; client shows "Still working, we'll keep it in My Trips."
- **Delete trip**: `DELETE /api/trips/:id` — only the trip owner can delete; returns 404 for non-owners.

## Acceptance Criteria
- [ ] `POST /api/trips` rejects unauthenticated requests (401).
- [ ] `POST /api/trips` validates inputs with Zod; invalid inputs return 400 with field errors.
- [ ] `POST /api/trips` enforces the quota: the 4th trip in a single UTC day returns 429.
- [ ] Failed trips do not count toward the quota; the user can immediately retry.
- [ ] In-flight (`pending` + `generating`) trips count toward quota, blocking spam-clicking.
- [ ] Quota check and trip insert happen inside a single Postgres transaction to prevent races.
- [ ] `GET /api/trips` returns only the authenticated user's trips, ordered by `createdAt DESC`.
- [ ] `GET /api/trips/:id` returns 404 if the trip belongs to another user.
- [ ] `DELETE /api/trips/:id` returns 404 for non-owners; deletes and returns 204 for the owner.
- [ ] `GET /api/me/quota` returns `{ used, limit, resetsAt }` for the current user.
- [ ] `generate-trip` Inngest function completes within 60 s at p95.
- [ ] An invalid destination produces a user-safe `failureReason` and `failed` status; no quota consumed.
- [ ] Gemini output is validated with Zod before saving; a failed parse triggers a retry.
- [ ] Unsplash cover photo is hotlinked (not re-hosted); photographer credit and `download_location` are saved.
- [ ] `download_location` is triggered the first time a cover photo is displayed (client-side call).
- [ ] AI output JSON includes the disclaimer "AI-generated estimates; verify before booking."
- [ ] `model`, `promptVersion`, `inputTokens`, `outputTokens` are stored on every trip.
- [ ] Unit tests cover: quota rule, Zod output schemas, prompt builder, each Inngest step (Gemini + Unsplash mocked).
- [ ] Integration tests (Neon test branch) cover: auth rejection, ownership, quota, create/list/get/delete, `generate-trip` function (Inngest Dev Server).

## Edge Cases
- **Concurrent `POST /api/trips` from the same user**: the transaction's quota check prevents both succeeding if the 3rd slot is taken simultaneously.
- **Inngest retry mid-step**: each step is idempotent. `mark-generating` is safe to re-run (upsert). `generate-plan` and `fetch-cover` re-run the external call; `save` is an upsert on `tripId`.
- **Gemini model change**: the model is read from `GEMINI_MODEL` env var (default `gemini-3.8-flash`), so it can be swapped without a code change.
- **Unsplash demo quota (50 req/hour)**: acceptable for local dev; production approval (1,000/hour) is needed before public launch.
- **`generate-trip` hangs**: Inngest has a 3-minute function timeout by default. If exceeded, `onFailure` fires and sets status to `failed`.

## Out of Scope
- Editing or swapping activities (v2).
- Regenerating a trip (v2).
- Per-place photos (v2).
- Share links, payments, push notifications (all v2+).

## Open Questions
- **Prompt versioning**: should `promptVersion` be a string (e.g. `"v1.0"`) incremented manually, or a git SHA? _(Recommend a short manual semver string for readability.)_
- **Budget currency display**: amounts are in destination's local currency with a USD equivalent. Does the AI return both, or does the client convert? _(Recommend AI returns both; eliminates a client-side FX dependency.)_

## Affected Areas
- `server/services/quotaService.ts` — quota check + transaction
- `server/services/tripService.ts` — CRUD operations
- `server/services/llm.ts` — `generateTrip()`, `normalizeDestination()` Gemini wrappers
- `server/services/unsplash.ts` — cover photo fetch
- `server/schemas/tripOutput.ts` — Zod schemas for AI output (itinerary, hotels, budgetBreakdown)
- `server/prompts/generateTrip.ts` — prompt builder + `promptVersion` constant
- `server/inngest/functions/generateTrip.ts` — 5-step Inngest function
- `server/inngest/client.ts` — `trip/generate.requested` event type
- `app/api/trips+api.ts` — `POST /api/trips`, `GET /api/trips`
- `app/api/trips/[id]+api.ts` — `GET /api/trips/:id`, `DELETE /api/trips/:id`
- `app/api/me/quota+api.ts` — `GET /api/me/quota`
- `app/api/inngest+api.ts` — Inngest serve handler
- `.env.example` — `GEMINI_API_KEY`, `GEMINI_MODEL`, `UNSPLASH_ACCESS_KEY`, `INNGEST_SIGNING_KEY`, `INNGEST_EVENT_KEY`
