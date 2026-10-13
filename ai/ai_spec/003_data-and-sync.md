# Spec: Data & Sync — Prisma Schema, Webhooks & Account Deletion
_Number: 003 | Created: 2026-10-13_

## Summary
Establishes the Postgres data model via Prisma, wires Clerk `user.*` webhooks through Inngest to keep the `users` table in sync, and implements in-app account deletion as required by the App Store.

## User Flows
### Happy Path — User Created
1. New user signs in via Clerk (Phase 2).
2. Clerk fires the `user.created` webhook to `POST /api/webhooks/clerk`.
3. The server verifies the webhook signature with `verifyWebhook`.
4. Inngest event `clerk/user.created` is sent → `sync-clerk-user` function runs → `users` row inserted.

### Happy Path — Account Deletion
1. Signed-in user goes to Profile → taps "Delete Account".
2. App shows a confirmation alert ("This permanently deletes your account and all trips.").
3. User confirms → client calls Clerk's user delete → `user.deleted` webhook fires.
4. Inngest `sync-clerk-user` handles `clerk/user.deleted` → soft-deletes the user (`deletedAt` set) → cascade-deletes all their trips.
5. App clears the local session and navigates to Sign-In.

### Alternative Flows
- **Webhook arrives before `users` row exists** (race on first request): `POST /api/trips` falls back to `clerk.users.getUser(clerkUserId)` to create the row inline.
- **User updated** (`user.updated` webhook): `sync-clerk-user` upserts `email`, `name`, `imageUrl`.

## Acceptance Criteria
- [ ] Prisma schema defines `User` and `Trip` models exactly as specified (types, enums, indexes, cascade delete).
- [ ] `prisma migrate dev` runs cleanly against the Neon `dev` branch.
- [ ] `POST /api/webhooks/clerk` rejects requests with an invalid signature (401).
- [ ] `user.created` webhook inserts a `users` row (verified via DB query).
- [ ] `user.updated` webhook upserts `email`, `name`, `imageUrl`.
- [ ] `user.deleted` webhook soft-deletes the user and cascade-deletes their trips.
- [ ] In-app "Delete Account" flow completes end-to-end: Clerk delete → webhook → DB soft-delete → session cleared → Sign-In screen.
- [ ] Integration tests (Neon test branch) cover webhook signature rejection, user insert, update, and deletion.
- [ ] `.env.example` includes `CLERK_WEBHOOK_SECRET`.

## Edge Cases
- **Webhook delivery failure**: Inngest retries handle transient failures (3 retries). After all retries fail, the error is sent to Sentry.
- **Duplicate `user.created`**: upsert on `clerkUserId` so replays are idempotent.
- **Tunnel down locally**: if ngrok is not running, new users rely on the inline fallback (assumption 1 in the spec).
- **Delete confirmation dismissed**: no action taken; user remains signed in.

## Out of Scope
- Trip generation or quota logic (Phase 3).
- User profile editing (out of v1 entirely).
- Android / web.

## Open Questions
- Soft-delete strategy: `deletedAt` on `User` is set, but queries must always filter `WHERE deletedAt IS NULL`. Should we add a global Prisma middleware for this, or filter explicitly in each service call? _(Recommend explicit filtering for clarity in v1.)_

## Affected Areas
- `prisma/schema.prisma` — `User` and `Trip` models, enums (`TripStatus`, `GroupType`, `BudgetTier`, `Interest`)
- `prisma/migrations/` — initial migration
- `server/db.ts` — Prisma client singleton with Neon adapter
- `server/services/userService.ts` — upsert, soft-delete
- `server/inngest/functions/syncClerkUser.ts` — `sync-clerk-user` function
- `server/inngest/client.ts` — event types including `clerk/user.*`
- `app/api/webhooks/clerk+api.ts` — signature verification + Inngest send
- `app/(app)/profile.tsx` — delete account UI and flow
