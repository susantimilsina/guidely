# Spec: UI Screens — All Client-Facing Views
_Number: 005 | Created: 2026-10-13_

## Summary
Implements every user-facing screen: Home (empty and populated My Trips list), the create-trip form wizard, the polling loading screen, the trip detail screen, and the profile screen. Uses NativeWind, TanStack Query, and Expo Router.

## User Flows

### Happy Path — First Run
1. Signed-in user with no trips lands on Home → empty state with "Plan your first trip" CTA, 2–3 example destination chips, and "N trips left today" badge.
2. User taps a destination chip → chip pre-fills the destination field in the Create Trip form.

### Happy Path — Create Trip
1. User taps "Plan your first trip" or the FAB → Create Trip wizard opens.
2. **Step 1 — Destination**: free-text input, max 100 chars. Tapping "Next" validates non-empty.
3. **Step 2 — Dates**: date picker for start date (today or later); stepper for number of days (1–14).
4. **Step 3 — Travelers**: stepper for traveler count (1–10); radio for group type (solo / couple / family / friends).
5. **Step 4 — Budget & Interests**: budget tier radio (budget / moderate / luxury); multi-select interest chips (food, culture, adventure, nature, nightlife, shopping, relaxation). At least one interest required.
6. **Step 5 — Review**: read-only summary of all inputs. "Generate" button.
7. User taps "Generate" → `POST /api/trips` fires → navigates to Loading screen with `tripId`.

### Happy Path — Loading Screen
1. Screen polls `GET /api/trips/:id` every 2.5 s.
2. Shows rotating status copy ("Finding the best spots…", "Building your itinerary…", etc.).
3. When status becomes `ready` → navigates to Trip Detail.
4. When status becomes `failed` → shows error message + "Try again" button (pre-fills the create-trip form with the same inputs).
5. After 3 minutes of polling → shows "Still working — we'll keep it in My Trips." + "View My Trips" button.

### Happy Path — Trip Detail
1. Hero image (Unsplash, hotlinked) fills the top; photographer credit and Unsplash logo shown.
2. Overview card: destination, country, dates, traveler count and group type, budget tier.
3. Budget breakdown: stay, food, transport, activities, total — in local currency with USD equivalent. Disclaimer: "AI-generated estimates; verify before booking."
4. Hotel suggestions: 2–4 cards, each with name, area, nightly price range, and why it fits.
5. Day tabs: one tab per day. Each tab shows morning, afternoon, evening activity cards (title, description, rough time, estimated cost).
6. `download_location` is triggered on the Unsplash API when the hero image first renders.

### Happy Path — My Trips
1. Home screen shows a list of trips with destination name, dates, status badge (`pending`, `generating`, `ready`, `failed`).
2. User swipes left on a trip (or long-presses) → "Delete" action appears.
3. User confirms delete → `DELETE /api/trips/:id` → trip removed from list.
4. Tapping a `ready` trip → opens Trip Detail.
5. Tapping a `pending`/`generating` trip → opens Loading screen for that trip.

### Happy Path — Profile
1. User opens Profile: avatar, name, email.
2. Taps "Sign out" → session cleared → Sign-In.
3. Taps "Delete Account" → confirmation alert → Clerk delete → Sign-In. _(Detailed in spec 003.)_

## Acceptance Criteria
- [ ] Empty state renders on Home with CTA and example destination chips.
- [ ] "N trips left today" reflects the live quota from `GET /api/me/quota` (TanStack Query, auto-refresh on focus).
- [ ] Create-trip wizard validates each step before allowing "Next"; the user cannot skip required fields.
- [ ] Start date picker disallows past dates.
- [ ] Destination input enforces a 100-character max.
- [ ] Days stepper is bounded 1–14; travelers stepper is bounded 1–10.
- [ ] At least one interest must be selected before proceeding from Step 4.
- [ ] Review screen shows all inputs accurately; "Edit" links return to the correct step.
- [ ] "Generate" button is disabled (and shows a spinner) while the `POST /api/trips` request is in flight.
- [ ] Loading screen shows rotating status copy that cycles every ~4 s.
- [ ] Loading screen stops polling when the screen is backgrounded; resumes on foreground.
- [ ] "Try again" pre-fills the create-trip wizard with the failed trip's exact inputs.
- [ ] Trip detail hero image is hotlinked (not re-hosted); photographer name and Unsplash attribution are visible.
- [ ] `download_location` ping fires once when the hero image mounts.
- [ ] Day tabs render all activities for each day; tab count matches the `days` input.
- [ ] Hotel cards show all required fields (name, area, price range, reason).
- [ ] My Trips list updates optimistically on delete (TanStack Query mutation).
- [ ] Status badges display correct copy and colour for all four statuses.
- [ ] All interactive elements have VoiceOver labels.
- [ ] Text scales correctly with Dynamic Type.
- [ ] Colour tokens meet WCAG AA contrast ratios.

## Edge Cases
- **No internet during polling**: show a "No connection" banner; resume polling when connectivity returns.
- **Trip deleted while on Loading screen**: `GET /api/trips/:id` returns 404 → show "Trip not found" + "Back to My Trips".
- **Zero trips remaining**: "N trips left today" shows "0 trips left"; "Plan your first trip" CTA is replaced with "Come back tomorrow".
- **Very long destination name**: truncate with ellipsis in list cards; full name shown on detail screen.
- **Image fails to load**: show a styled placeholder with the destination initials or a bundled fallback.
- **"Try again" for a failed trip where quota is now at 3**: server returns 429; show "You've reached your daily limit" error on the loading screen instead of navigating to the wizard.

## Out of Scope
- Editing or swapping activities.
- Sharing a trip.
- Maps, pins, or navigation integration.
- Per-place photos.
- Free-text notes.
- Android / web.
- Dark mode (not in scope unless NativeWind tokens are set up for it).

## Open Questions
- **Day tab scroll behaviour**: should tabs be a horizontal swipe (ScrollView pager) or a segmented control? _(Recommend horizontal swipe for comfort with 14 days.)_
- **Hero image aspect ratio**: fixed 16:9 crop or full-bleed with variable height? _(Recommend fixed 16:9 with Unsplash `w=800&h=450&fit=crop` params.)_
- **Rotating copy**: hardcoded strings or server-driven? _(Recommend hardcoded in v1 — simple and no network dependency.)_

## Affected Areas
- `app/(app)/index.tsx` — Home + My Trips list
- `app/(app)/create-trip/` — wizard steps (one file per step or a single stepper component)
- `app/(app)/trips/[id]/loading.tsx` — polling screen
- `app/(app)/trips/[id]/index.tsx` — trip detail
- `app/(app)/profile.tsx` — profile + sign out + delete account
- `components/TripCard.tsx`, `DayTab.tsx`, `HotelCard.tsx`, `BudgetBreakdown.tsx`, `ActivityCard.tsx`
- `hooks/useTrip.ts`, `useMyTrips.ts`, `useQuota.ts`, `useCreateTrip.ts` — TanStack Query hooks
- `constants/statusCopy.ts` — rotating loading messages
