# Guidely: AI Trip Planner, v1 Spec

## Context
This is a greenfield project; `/Users/susankeepme/Documents/projects/react/guidely` is empty.

Guidely is an iOS app. A user signs in with Google or Apple, describes a trip (destination, dates, travelers, budget, interests), and gets an AI-generated plan: a day-by-day itinerary, hotel suggestions and a budget breakdown. Generation runs in the background, and the app shows a loading screen until the trip is ready.

This spec came out of a structured interview. It records every decision, the assumptions made, and the open risks, so the implementation plan has no guesswork.

---

## 1. Spec summary

### Product & users
- **Problem:** Planning a trip is slow, scattered research. Guidely produces a coherent, budget-aware itinerary in under a minute.
- **User:** Leisure travelers on iPhone.
- **Must do well:** Fast, reliable generation of a believable, budget-appropriate, day-by-day plan.
- **Aha moment:** The first generated trip appears on the detail screen, with a hero photo, budget breakdown and a full day-by-day plan.

### Scope
**In v1**
- iOS app (Expo)
- Google and Apple sign-in
- Create-trip form
- Background AI generation with a polling loading screen
- Trip detail screen
- My Trips list, with delete
- 3 trips/day quota
- In-app account deletion (App Store requirement)
- Sentry
- Unit, API integration and Maestro E2E tests

**Out of v1**
- Android and web
- Maps and pins
- Editing or swapping activities
- Regenerating a trip
- Share links
- Payments and subscriptions
- Push notifications
- Offline mode
- i18n (English only)
- Bookings
- Per-place photos
- Free-text notes, deliberately left out to avoid prompt injection

### Core flows
1. **Sign-in:** Clerk native sign-in with Apple (`useSignInWithApple`) and Google (`useSignInWithGoogle`). The `user.created` webhook goes to Inngest, which inserts the `users` row.
2. **First run:** Home screen with an empty state, a "Plan your first trip" CTA, 2–3 example destination chips, and "N trips left today".
3. **Create trip:** A form wizard:
   - Destination: free text, max 100 characters
   - Start date: today or later
   - Days: 1–14
   - Travelers: 1–10
   - Group type: solo / couple / family / friends
   - Budget tier: budget / moderate / luxury
   - Interests: multi-select chips (food, culture, adventure, nature, nightlife, shopping, relaxation)
   - Review screen, then **Generate**
4. **Generate:**
   - `POST /api/trips` checks the input with Zod, verifies the Clerk JWT, and checks the quota inside a transaction.
   - It inserts the trip with status `pending`, sends the Inngest event `trip/generate.requested`, and returns `{ tripId }`.
5. **Loading screen:**
   - Polls `GET /api/trips/:id` every 2.5 s.
   - Shows rotating status copy while waiting.
   - When the trip is `ready`, it opens Trip Detail; when it's `failed`, it shows the error and a "Try again" button that pre-fills the form.
   - After 3 minutes it shows "Still working; we'll keep it in My Trips."
6. **Trip detail:**
   - Hero image from Unsplash, with photographer and Unsplash credit
   - Overview: destination, dates, travelers, tier
   - Budget breakdown (stay, food, transport, activities, total) against the selected tier
   - Hotel suggestions: 2–4, with name, area, nightly price range and why each fits
   - Day tabs, each with morning, afternoon and evening activities: title, description, rough time, estimated cost
7. **My Trips:** A list on the home screen with status badges and swipe or long-press to delete.
8. **Account:** Profile, sign out, and delete account.
   - Deleting calls Clerk's user delete.
   - The `user.deleted` webhook goes to Inngest, which soft-deletes the user and cascade-deletes their trips.

### Data model (Neon Postgres via Prisma)
- **User**
  - `id` (uuid PK), `clerkUserId` (unique)
  - `email`, `name`, `imageUrl`
  - `createdAt`, `updatedAt`, `deletedAt?`
- **Trip**
  - `id` (uuid PK), `userId` (FK, cascade delete, indexed with `createdAt`)
  - `status`: enum `pending | generating | ready | failed`
  - Inputs:
    - `destinationInput`, `startDate`
    - `days` (1–14), `travelers` (1–10)
    - `groupType` (enum), `budgetTier` (enum), `interests` (enum[])
  - Outputs:
    - `destinationName`, `country`, `currency` (normalized by the AI)
    - `coverImage` (jsonb: Unsplash URLs, photographer name and link, `download_location`)
    - `itinerary` (jsonb), `hotels` (jsonb), `budgetBreakdown` (jsonb), all validated with Zod before saving
  - Metadata: `failureReason?`, `model`, `promptVersion`, `inputTokens`, `outputTokens`, `createdAt`, `completedAt?`
- **Why jsonb for the outputs:** in v1 they're read-only and always loaded as a whole, so jsonb keeps the schema simple. Edit and swap in v2 can normalize days and activities into tables later.
- **Quota rule (no separate table):** a user may create a trip only while their count of trips is below 3, counting trips created since 00:00 UTC whose status is `pending`, `generating` or `ready`.
  - Failed trips drop out of that count automatically, so a failure costs no quota.
  - In-flight trips do count, which blocks spam-clicking.

### Architecture
- **One Expo project.** SDK 57, Expo Router, TypeScript, NativeWind, TanStack Query.
- **Backend:** Expo Router API routes (`+api.ts`), run **locally only** for now through the Expo dev server.
  - **No EAS Hosting or deployment** in this phase; the deploy target will be decided later.
  - The server code avoids host-specific APIs (standard `Request`/`Response`, env vars only), so it can move to EAS Hosting, Vercel or another host later.
- **API routes:**
  - `POST /api/trips`
  - `GET /api/trips`
  - `GET /api/trips/:id`
  - `DELETE /api/trips/:id`
  - `GET /api/me/quota`
  - `POST /api/webhooks/clerk`
  - `GET|POST|PUT /api/inngest`
- **Business logic** lives in a `server/` folder (services, Prisma client, Zod schemas, prompts) that only the API routes import. The client never holds secrets.
- **Auth:** the client sends the Clerk session token as a Bearer header. The server checks it with `@clerk/backend` (`verifyToken` or `authenticateRequest`).

### Integrations: who does what
| Service | Responsibility |
|---|---|
| **Clerk** (`@clerk/expo` 4.x, `@clerk/backend`) | Google and Apple sign-in, sessions, account deletion, and `user.*` webhooks (checked with `verifyWebhook`). |
| **Neon** | Postgres. A `main` branch for prod and a `dev` branch for local work. Integration tests run on a separate test branch. |
| **Prisma** (pin **7.10.0** stable; *not* the v8 RC) | ORM. Uses the `prisma-client` generator with `@prisma/adapter-neon` (Neon serverless driver), so the code stays edge-portable for a later deploy. |
| **Inngest** (`inngest` 4.x) | Function `generate-trip` (described below). Function `sync-clerk-user` (created, updated, deleted). |
| **Gemini** (`@google/genai`) | Validates and normalizes the destination, and generates the structured trip JSON. The model comes from the `GEMINI_MODEL` env var, defaulting to `gemini-3.8-flash`. |
| **Unsplash** | Searches for one cover photo per trip. The image is **hotlinked** (not re-hosted) using imgix params like `w`, `q` and `fm`, with photographer credit shown. `download_location` is triggered when the photo is first displayed. |
| **ImageKit** | Serves our own static assets (onboarding and empty-state illustrations, example-destination images). Wired up and ready for user uploads and avatars in v2. It is **not** used for Unsplash photos, because Unsplash's rules require hotlinking. |
| **Sentry** | Client: `@sentry/react-native` with the Expo config plugin. Server: errors from the API routes and Inngest functions are captured behind a small `reportError()` wrapper. Locally it uses a Node Sentry SDK; it can be swapped for the host-specific SDK once a deploy target is chosen. |

### Background work: `generate-trip` (Inngest)
- **Trigger:** `trip/generate.requested { tripId }`.
- **Settings:** idempotency key `tripId`, `retries: 3`, concurrency of 1 per user.
- **Steps** (each step runs as its own request, which keeps steps short for whatever host we use later):
  1. `mark-generating`: set the trip's status to `generating`.
  2. `normalize-destination`: a short Gemini call that returns `{ valid, name, country, currency }`. If the destination isn't valid, throw `NonRetriableError("We couldn't find that destination")`.
  3. `generate-plan`: a Gemini structured-output call that returns the itinerary, hotels and budget breakdown. A Zod parse failure throws, so the step retries.
  4. `fetch-cover`: Unsplash search. This step is **non-fatal**: if it fails, the trip falls back to a bundled placeholder.
  5. `save`: write the outputs, token counts and `completedAt`, and set status to `ready`.
- **Failure handling:** `onFailure` sets status to `failed` and stores a user-safe `failureReason`, and the error goes to Sentry.

### Non-functional
- **Latency:** generation should finish in under 60 s at p95. API reads should respond in under 300 ms.
- **Scale:** small. Hundreds to low thousands of users.
- **Cost:** about $0.025 per trip at today's Flash price, roughly doubling from 2027-01-01. The quota caps spend at 3 trips per user per day.
- **Unsplash quota:** the demo tier allows 50 requests/hour, which is fine for local development. **Production approval (1,000/hour) is needed before any public launch.**
- **Security:**
  - All secrets stay server-side.
  - Inputs are Zod-validated and length-capped.
  - Interests are an enum, so there's no free text in the prompt besides the destination.
  - Every trip query is scoped to its owner.
  - Webhook signatures are verified.
- **Privacy:** We store only name, email and avatar from Clerk, plus trips. Deleting an account removes everything.
- **Accessibility:** Dynamic Type, VoiceOver labels, and contrast-safe NativeWind tokens.

### Environments & delivery
- **Local dev only for this phase.** There is no EAS Hosting, EAS Build/Submit or TestFlight yet.
  - **App:** an iOS dev build run with `npx expo run:ios` on the simulator or a device. Apple sign-in doesn't work in Expo Go.
  - **API:** Expo API routes served by the Expo dev server.
  - **Services:** Neon `dev` branch, Inngest Dev Server (`npx inngest-cli dev`), Clerk dev instance.
  - **Clerk webhooks:** a tunnel (e.g. ngrok) exposes `/api/webhooks/clerk` to Clerk.
  - **Env vars:** a `.env` file, with server keys unprefixed and only safe values in `EXPO_PUBLIC_*`. `.env.example` is committed.
- **Deferred:** the production host, Neon `main`, Inngest Cloud, the Clerk prod instance, EAS Build/Submit, TestFlight and EAS Update.

### Quality bar (definition of done)
- **Unit tests:**
  - Quota rule
  - Zod schemas for AI output
  - Prompt builder
  - Each `generate-trip` step, with Gemini and Unsplash mocked
- **API integration tests:** run on a Neon test branch. Cover auth rejection, ownership, quota, create/list/get/delete, and webhook signature checks.
- **Maestro E2E:** on the local iOS simulator dev build. Happy path is sign-in (with a test account), create trip, loading screen, then trip detail. Also covers delete.

---

## 2. Assumptions (you didn't give a firm answer on these)
1. **First-request race:** if `POST /api/trips` finds no `users` row because the webhook hasn't been processed yet, it creates the row on the spot from Clerk's `getUser`. You chose webhook sync, so this is only a fallback.
2. **Limits:** days are 1–14, travelers are 1–10, and the destination is at most 100 characters.
3. **Currency:** budget amounts are in the destination's local currency, with a USD equivalent shown alongside.
4. **Quota reset:** midnight UTC, not the user's local time.
5. **Polling:** every 2.5 s, giving up after 3 minutes with a "still working" message.
6. **Client state:** TanStack Query for server state. No offline persistence in v1.
7. **Server errors:** reported through the `reportError()` wrapper. Sentry alert rules (e.g. a spike in generation failures) are set up but not part of the v1 gate. You didn't select alerts as a requirement.
8. **Language:** English only. Dates and currency are formatted with the device locale.
9. **Interests list:** food, culture, adventure, nature, nightlife, shopping, relaxation.
10. **Repo layout:** a single Expo repo with a `server/` folder, not a monorepo.
11. **Deployment deferred:** the deploy target (EAS Hosting, Vercel, etc.) will be chosen later. Until then, nothing runs outside your machine except Neon, Clerk, Gemini and Unsplash.

## 3. Open risks / unknowns
1. **Prisma inside Expo API routes (highest risk).** Metro bundles the API routes, and Prisma has no official Expo guide. *Mitigation:* a day-1 local spike. If it fails, I come back to you with options (Drizzle or the raw Neon driver). I won't swap anything without asking.
2. **Inngest has no Expo adapter.** We'll use the generic fetch handler (`inngest/edge` or `InngestCommHandler`); this is part of the spike.
3. **Deploying later may force rework.** Expo API routes are still beta. Whichever host we choose later may have CPU, duration or bundling limits; EAS Hosting, for example, has no server-side Sentry and its free plan allows only 10 CPU-ms per request. *Mitigation:* keep the server code host-agnostic and keep each step short.
4. **Clerk webhooks need a tunnel locally.** If the tunnel is down, new users rely on the fallback that creates their row on first request (assumption 1).
5. **Unsplash production approval.** This is needed before launch. Also, cover photos may be generic or wrong for small or obscure destinations.
6. **Gemini API is changing.** Google's docs are moving to the Interactions API. We'll pin the `@google/genai` version and wrap calls behind one `llm.generateTrip()` function.
7. **Gemini price doubles on 2027-01-01.** The model is configurable by env var, so we can drop to Flash-Lite if needed.
8. **AI accuracy.** Hotels and prices may be made up or out of date. We'll show "AI-generated estimates; verify before booking."
9. **App Store review:** requires in-app account deletion and Sign in with Apple (both in scope), plus privacy nutrition labels for the data sent to Gemini.

---

## 4. Proposed implementation phases (detailed plan to follow on approval)
0. **Local spike (go/no-go):** Expo SDK 57 + an API route on the local dev server, covering Prisma 7.10 + Neon adapter, Inngest serve with one test function on the Dev Server, Clerk token checks, and the server-side Sentry reporter.
1. **Foundation:** Expo app, NativeWind, Clerk Apple/Google sign-in, Sentry RN, auth-gated navigation, ImageKit setup.
2. **Data & sync:** Prisma schema and migrations, Clerk webhook to Inngest `sync-clerk-user`, account deletion.
3. **Trip API & generation:** Zod schemas, quota service, trip endpoints, `generate-trip` Inngest function, Gemini wrapper, Unsplash service.
4. **UI:** Home and My Trips, create-trip wizard, loading screen, trip detail, profile, plus empty, error and failed states.
5. **Tests:** unit, integration (Neon test branch), and Maestro flows on the local simulator.

## Verification
- **Spike:** the local API route successfully runs a DB read and write, an Inngest function (visible in the Dev Server UI), a verified Clerk token, and sends a Sentry event.
- **End to end on the local iOS dev build:**
  - Sign in with Apple and with Google.
  - Generate a Kathmandu trip (3 days, 2 travelers, moderate) and see the detail screen within 60 s.
  - Confirm the 4th generation in a day is blocked.
  - Confirm an invalid destination ("asdfgh") fails cleanly without using quota.
  - Delete the account and check that the DB rows are gone.
- **Monitoring:** check the Inngest Dev Server UI for run history, Sentry for errors in both the app and the API, and the Neon console for data.
- **Tests:** `npm test` (unit and integration) and `maestro test .maestro/` pass locally.
