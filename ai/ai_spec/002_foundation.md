# Spec: Foundation — Expo App, Auth & Navigation
_Number: 002 | Created: 2026-10-13_

## Summary
Bootstraps the Expo (SDK 57) project with NativeWind styling, Clerk Google/Apple sign-in, Sentry for the client, ImageKit for static assets, and auth-gated navigation via Expo Router. The result is a signed-in shell with no trip functionality yet.

## User Flows
### Happy Path
1. User opens the app for the first time → sees the Sign-In screen.
2. User taps "Continue with Apple" → native Apple auth sheet → session created → lands on Home.
3. User taps "Continue with Google" → Google OAuth → session created → lands on Home.
4. Already signed-in user reopens the app → Clerk restores the session → lands directly on Home.
5. User signs out from the Profile screen → session cleared → returns to Sign-In.

### Alternative Flows
- Network error during sign-in → show a toast with "Sign-in failed. Try again."
- Apple sign-in not available (non-Apple device) → Apple button is hidden.

## Acceptance Criteria
- [ ] `npx expo run:ios` builds and runs on the simulator without errors.
- [ ] NativeWind 4.x is configured; at least one styled component renders correctly.
- [ ] "Continue with Apple" completes the full Clerk sign-in on a dev build (not Expo Go).
- [ ] "Continue with Google" completes the full Clerk sign-in.
- [ ] Authenticated users cannot reach the Sign-In screen; unauthenticated users cannot reach Home.
- [ ] Sentry `@sentry/react-native` is initialised with the Expo config plugin; a test `captureException` event appears in Sentry.
- [ ] ImageKit base URL is configured; at least one static asset (e.g. empty-state illustration) loads from ImageKit.
- [ ] Sign out clears the session and redirects to Sign-In.
- [ ] `.env.example` includes `EXPO_PUBLIC_CLERK_PUBLISHABLE_KEY`, `EXPO_PUBLIC_IMAGEKIT_URL`, and `SENTRY_DSN`.

## Edge Cases
- **Apple sign-in unavailable in Expo Go**: all E2E testing of Apple auth must happen on a `npx expo run:ios` dev build.
- **Session restoration race**: on cold start, show a loading/splash state until Clerk resolves the session to avoid a flash of the Sign-In screen.
- **Google sign-in redirect**: configure the Clerk OAuth redirect URI correctly for the local dev build bundle ID.

## Out of Scope
- Trip creation, list, or detail screens (Phase 4).
- Push notifications.
- Account deletion (Phase 2).
- Android / web targets.

## Open Questions
- None — stack is decided (Expo SDK 57, NativeWind 4.x, Clerk `@clerk/expo` 4.x, Expo Router).

## Affected Areas
- `app/_layout.tsx` — root layout with ClerkProvider and Sentry init
- `app/(auth)/sign-in.tsx` — sign-in screen
- `app/(app)/_layout.tsx` — auth-gated layout
- `app/(app)/index.tsx` — home shell (empty state placeholder)
- `app/(app)/profile.tsx` — profile + sign-out
- `tailwind.config.js` / `global.css` — NativeWind setup
- `plugins/sentry.js` — Sentry Expo config plugin
- `constants/imagekit.ts` — ImageKit base URL
- `app.json` / `app.config.ts` — bundle ID, Sentry plugin, scheme for OAuth redirect
- `.env`, `.env.example`
