# Guidely: Implementation Plan

**Source of truth:** [SPEC.md](./SPEC.md). Every phase ends with a working, testable app, and Phase 0 is a go/no-go gate.

**Scope:** local dev only. There's no deployment, EAS Hosting, EAS Build or TestFlight in this plan.

## Pinned versions
Confirm each with `npm view <pkg> version` at install time. If one has moved, keep the major version.

| Package | Version |
|---|---|
| `expo` | SDK 57 (React Native 0.86) |
| `prisma`, `@prisma/client` | 7.10.0 (**not** the v8 RC) |
| `@prisma/adapter-neon`, `@neondatabase/serverless` | latest compatible with Prisma 7.10 |
| `@clerk/expo` | 4.x |
| `@clerk/backend` | latest |
| `@clerk/expo-google-signin` | latest |
| `inngest` | 4.x |
| `@google/genai` | 2.x (exact version pinned) |
| `@sentry/react-native` | 8.x |
| Server-side Sentry SDK | Node SDK, chosen in the Phase 0 spike |
| `nativewind` + `tailwindcss` | versions per the NativeWind docs for SDK 57 |
| `@tanstack/react-query` | 5.x |
| `zod` | latest |
| `vitest` | unit and integration tests |
| Maestro CLI | E2E tests |

## Target structure
```
guidely/
  app/                         # Expo Router
    _layout.tsx                # Providers: Clerk, TanStack Query, Sentry
    (auth)/sign-in.tsx
    (app)/_layout.tsx          # auth gate
    (app)/index.tsx            # Home + My Trips
    (app)/trips/new.tsx        # create-trip wizard
    (app)/trips/[id]/generating.tsx
    (app)/trips/[id]/index.tsx # trip detail
    (app)/profile.tsx
    api/
      trips+api.ts             # POST, GET (list)
      trips/[id]+api.ts        # GET, DELETE
      me/quota+api.ts
      webhooks/clerk+api.ts
      inngest+api.ts           # GET/POST/PUT
  server/
    db.ts                      # Prisma + Neon adapter singleton
    auth.ts                    # requireUser(request) → verifies JWT, upserts the user if missing
    errors.ts                  # reportError(), HttpError, json helpers
    quota.ts
    schemas/                   # Zod: trip input, AI output, API responses
    llm/                       # gemini client, prompts, normalizeDestination, generateTrip
    unsplash.ts
    inngest/                   # client, functions/generate-trip.ts, functions/sync-clerk-user.ts
  src/                         # client-only code
    api/                       # fetch wrapper with Clerk token, query hooks
    components/                # UI primitives (NativeWind)
    features/                  # trip-form, trip-detail, trips-list
    lib/imagekit.ts            # IK URL builder for static assets
  prisma/schema.prisma
  tests/unit/  tests/integration/
  .maestro/
  .env.example
```

---

## Phase 0: Local spike (go/no-go)
**Goal:** prove the risky integrations work inside Expo API routes before building any features.

**Tasks**
1. `npx create-expo-app@latest guidely` (SDK 57, TypeScript). Turn on server output in `app.json` so API routes work: `web.output: "server"`.
2. Add Prisma 7.10 with the `prisma-client` generator, `@prisma/adapter-neon` and a one-model schema (`SpikeRow`). Run `migrate dev` against the Neon `dev` branch.
3. Add `app/api/spike+api.ts` that writes a row, reads it back, and returns JSON.
4. Add Inngest: create the client, serve it through a fetch-style handler in `app/api/inngest+api.ts`, and add a `spike/hello` function. Run `npx inngest-cli dev` and trigger it.
5. Clerk: verify a hardcoded test session token with `@clerk/backend` inside the spike route.
6. Sentry: in `reportError()`, send a test exception from an API route.

**Exit criteria (all must pass):**
- The DB round-trip works.
- The Inngest run shows as completed in the Dev Server UI.
- Clerk token verification succeeds.
- The test event appears in Sentry.

**If Prisma fails to bundle:** stop and bring options to you (Drizzle or the raw Neon driver). Nothing gets swapped without asking.

## Phase 1: Foundation
- NativeWind setup, design tokens (colors, spacing, type scale) and base components: Button, Input, Chip, Card, Screen, EmptyState, ErrorState.
- Sentry React Native with the Expo config plugin, and `Sentry.wrap(RootLayout)`.
- Clerk:
  - `ClerkProvider` with the `expo-secure-store` token cache.
  - Sign-in screen with Apple (`@clerk/expo/apple`) and Google (`@clerk/expo/google` plus the `@clerk/expo-google-signin` plugin).
  - Run `npx expo prebuild` and `npx expo run:ios`.
- Route groups: `(auth)` and `(app)`, with redirects based on `useAuth().isSignedIn`.
- `src/api/client.ts`: a fetch wrapper that attaches `Authorization: Bearer <getToken()>` and parses errors as typed `ApiError`s.
- TanStack Query provider.
- `src/lib/imagekit.ts`: URL builder (`tr:w-…,q-…,f-auto`) for static illustrations hosted on ImageKit.

**Done when:** sign-in works with both Apple and Google on the simulator, and signing out returns to the sign-in screen.

## Phase 2: Data & sync
- **Full Prisma schema:** `User`, `Trip`, and the enums `TripStatus`, `GroupType`, `BudgetTier`, `Interest`, with an index on `(userId, createdAt)`. Run the migration.
- `server/auth.ts` `requireUser(req)`:
  - Verifies the JWT and loads the user by `clerkUserId`.
  - If the user row is missing, fetches them with `clerkClient.users.getUser` and upserts (assumption 1).
  - Returns 401 on any auth failure.
- **`POST /api/webhooks/clerk`:** checks the signature with `verifyWebhook`, then calls `inngest.send("clerk/user.{created|updated|deleted}")`.
- **Inngest `sync-clerk-user`:** upserts on created/updated. On deleted, it hard-deletes the user, and the trips go with them through the cascade (the spec said "soft-delete + cascade"; a hard delete is simpler and meets the privacy goal).
- **Profile screen:** sign out, plus **Delete account**. Delete asks for confirmation, calls `user.delete()` (Clerk), then signs out.
- **Local tunnel:** document the ngrok steps in the README.

**Done when:** a new sign-in creates a `users` row through the webhook, and deleting the account removes it.

## Phase 3: Trip API & generation
- **Zod schemas** in `server/schemas/`:
  - `TripInput`: destination (2–100 characters, trimmed), `startDate` (today or later), days (1–14), travelers (1–10), `groupType`, `budgetTier`, interests (1–7 of the enum).
  - `DestinationResult`: `{ valid, name, country, currencyCode }`.
  - `TripPlan`:
    - `days[]`, each with `{ day, title, morning/afternoon/evening: Activity }`; an `Activity` is `{ title, description, timeHint, estCost }`.
    - `hotels[2..4]`, each with `{ name, area, priceRangePerNight, whyItFits }`.
    - `budget`: `{ currency, stay, food, transport, activities, total, totalUsd }`.
- **`server/quota.ts`:** `getRemaining(userId)` and `assertCanCreate(userId)`. Counts the user's non-failed trips since midnight UTC, inside a serializable transaction together with the trip insert.
- **Endpoints:**
  - `POST /api/trips`: validate, check quota, insert a `pending` trip, `inngest.send`, return 201 `{ tripId }`. Return 429 `QUOTA_EXCEEDED` when the quota is used up.
  - `GET /api/trips`: list the user's trips (summary fields only).
  - `GET /api/trips/:id`: return the trip if the user owns it, otherwise 404.
  - `DELETE /api/trips/:id`: delete it if the user owns it.
  - `GET /api/me/quota`: return `{ used, limit: 3, remaining, resetsAt }`.
- **`server/llm/`:**
  - A single Gemini client.
  - `normalizeDestination()` and `generateTripPlan()` request structured JSON from the model, then always re-check it with Zod.
  - Prompts are versioned with a `PROMPT_VERSION` constant.
  - The model name comes from `GEMINI_MODEL` (default `gemini-3.8-flash`).
- **`server/unsplash.ts`:** `findCover(destination)` returns `{ urls, photographer, profileUrl, downloadLocation }`, or null on any error.
- **Inngest `generate-trip`:** steps exactly as in the spec, with `idempotency: event.data.tripId`, `retries: 3`, `concurrency: { key: event.data.userId, limit: 1 }`, and `onFailure` setting status to `failed` with `failureReason`, plus `reportError`.
- **Unsplash `download_location`:** a small `POST /api/trips/:id/cover-viewed` endpoint triggers it once. *(This adds an endpoint not listed in the spec; it is needed for Unsplash compliance.)*

**Done when:** a curl request with a valid token creates a trip, which reaches `ready` in the Inngest Dev Server and has a valid plan. An input of "asdfgh" ends as `failed`, and the quota is unchanged.

## Phase 4: UI
- **Home:**
  - Greeting and "N trips left today".
  - With no trips: a "Plan your first trip" CTA plus 3 example chips that pre-fill the form.
  - Otherwise, the My Trips list: cover thumbnail, destination, dates, status badge. Long-press or swipe to delete, with confirmation.
- **Create wizard** (`trips/new`):
  - Steps: destination, then dates and days, then travelers and group, then budget tier, then interests, then review.
  - Fields are validated with the shared Zod `TripInput`.
  - On submit, open `generating`. A 429 shows the "come back tomorrow" state.
- **Generating screen:**
  - Rotating copy and a progress animation.
  - TanStack Query `refetchInterval: 2500` while the trip is `pending` or `generating`.
  - When `ready`, replace the screen with the detail screen.
  - When `failed`, show the reason and a "Try again" button that pre-fills the wizard.
  - After 3 minutes, show "still working" with a link to Home.
- **Trip detail:**
  - Hero with an Unsplash URL (`&w=1200&q=80&fm=webp`) and credit links (UTM parameters).
  - Overview card and budget breakdown card, with the total vs USD.
  - Hotels list and day tabs with activity cards.
  - Disclaimer: "AI-generated estimates; verify before booking."
- **Accessibility:** labels on every touchable, Dynamic Type supported, minimum 44pt hit targets.

**Done when:** the verification walkthrough in the spec passes manually.

## Phase 5: Tests
- **Unit tests (Vitest):**
  - Quota math, including the UTC boundary.
  - `TripInput` and `TripPlan` schemas (valid and invalid fixtures).
  - The prompt builder.
  - `generate-trip` steps, using `@inngest/test` with Gemini and Unsplash mocked: success, invalid destination, bad JSON then retry, Unsplash failure then placeholder.
- **Integration tests (Vitest, Neon `test` branch, reset before each run):**
  - Call the API route handlers directly with a `Request`.
  - Clerk verification is mocked at the `requireUser` boundary.
  - Cover: 401 without a token, ownership returns 404, the 4th create returns 429, the create/list/get/delete flow, and webhook signature rejection.
- **Maestro** (`.maestro/`), on the iOS simulator dev build:
  - `create-trip.yaml`: a signed-in test user (Clerk test email or OTP), the wizard, the generating screen, the detail screen. Gemini is real or stubbed through a `MOCK_LLM=1` env var.
  - `delete-trip.yaml`.
- **Scripts:** `npm test`, `npm run test:integration`, `npm run e2e`.

**Done when:** all three test suites pass locally.

---

## Accounts and keys you'll need (before Phase 0)
- **Neon:** a project with a `dev` branch (and a `test` branch later). You'll need `DATABASE_URL`.
- **Clerk:**
  - A dev instance with Google and Apple enabled.
  - Keys: `EXPO_PUBLIC_CLERK_PUBLISHABLE_KEY`, `CLERK_SECRET_KEY`, and later `CLERK_WEBHOOK_SIGNING_SECRET`.
  - Apple sign-in needs an Apple Developer account and the bundle ID set up.
- **Google AI Studio:** `GEMINI_API_KEY`.
- **Inngest:** nothing needed locally; the Dev Server runs without keys.
- **Unsplash:** a demo app. You'll need `UNSPLASH_ACCESS_KEY`.
- **ImageKit:** `EXPO_PUBLIC_IMAGEKIT_URL_ENDPOINT`.
- **Sentry:** an RN project and a Node project. You'll need `EXPO_PUBLIC_SENTRY_DSN`, `SENTRY_DSN_SERVER` and `SENTRY_AUTH_TOKEN`.
