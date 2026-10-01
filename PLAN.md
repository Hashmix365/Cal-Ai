# NutriSnap — AI Calorie Tracking App Implementation Plan

## Context

Building a mobile calorie-tracking app (similar to Cal AI) as a portfolio/learning project. Users photograph meals or type descriptions, AI estimates itemized macros, and the app tracks daily progress toward personalized nutrition goals. The app is free with no monetization in v1.

---

## Tech Stack

| Layer | Tool |
|---|---|
| Frontend | Expo + React Native, Expo Router (file-based routing) |
| Styling | NativeWind v4 (Tailwind CSS for RN) |
| State | TanStack Query (server state) |
| Auth | Clerk (Google + Apple sign-in) |
| Database | Neon Postgres + Drizzle ORM |
| Validation | Zod |
| AI / Background Jobs | Trigger.dev + OpenAI (GPT-4o-mini) |
| Image Storage | ImageKit |
| Push Notifications | Expo Notifications |
| Error Tracking | Sentry |

---

## Phase 1: Project Setup & Configuration

1. **Initialize Expo project** with Expo Router template (`npx create-expo-app@latest --template tabs`)
2. **Install and configure core dependencies**:
   - NativeWind v4 (Tailwind CSS setup, `tailwind.config.js`, babel/metro config)
   - Clerk Expo SDK (`@clerk/clerk-expo`) — configure Google + Apple OAuth providers
   - Drizzle ORM (`drizzle-orm`, `drizzle-kit`) + Neon serverless driver (`@neondatabase/serverless`)
   - TanStack Query (`@tanstack/react-query`)
   - Zod
   - Sentry Expo SDK (`@sentry/react-native`)
   - `expo-camera`, `expo-image-picker`, `expo-image-manipulator` (for compression)
   - `expo-notifications`
   - ImageKit SDK or REST API client
3. **Set up environment variables**: Clerk keys, Neon connection string, OpenAI API key, ImageKit keys, Sentry DSN, Trigger.dev API key
4. **Set up Drizzle schema and migrations** (see Data Model below)
5. **Set up Trigger.dev project** with the Expo API Routes integration

---

## Phase 2: Database Schema (Drizzle + Neon)

### Core Tables

**users**
- `id` (uuid, PK)
- `clerk_id` (text, unique, not null) — links to Clerk user
- `email` (text)
- `name` (text)
- `age` (integer)
- `height_cm` (numeric) — always store metric internally
- `weight_kg` (numeric)
- `activity_level` (enum: sedentary, lightly_active, moderately_active, very_active)
- `goal` (enum: lose_fast, lose_steady, maintain, gain_steady, gain_fast)
- `unit_preference` (enum: metric, imperial)
- `daily_calories_target` (integer) — AI-calculated
- `daily_protein_target` (integer)
- `daily_carbs_target` (integer)
- `daily_fat_target` (integer)
- `theme_preference` (enum: light, dark, system) — default system
- `notification_reminders_enabled` (boolean, default true)
- `reminder_times` (jsonb) — array of times for meal reminders
- `onboarding_completed` (boolean, default false)
- `expo_push_token` (text, nullable)
- `created_at` (timestamp)
- `updated_at` (timestamp)

**meals**
- `id` (uuid, PK)
- `user_id` (uuid, FK → users)
- `image_url` (text, nullable) — ImageKit URL
- `input_type` (enum: photo, text)
- `input_text` (text, nullable) — for manual text entries
- `total_calories` (integer)
- `total_protein` (numeric)
- `total_carbs` (numeric)
- `total_fat` (numeric)
- `ai_confidence` (numeric) — overall confidence score 0-1
- `logged_at` (timestamp) — when the meal was eaten
- `created_at` (timestamp)

**meal_items**
- `id` (uuid, PK)
- `meal_id` (uuid, FK → meals)
- `name` (text) — e.g., "grilled chicken breast"
- `portion_description` (text) — e.g., "1 medium piece (~150g)"
- `calories` (integer)
- `protein` (numeric)
- `carbs` (numeric)
- `fat` (numeric)
- `confidence` (numeric) — per-item confidence 0-1
- `sort_order` (integer)

### Indexes
- `meals.user_id` + `meals.logged_at` (composite) — for daily timeline queries
- `users.clerk_id` (unique)

---

## Phase 3: Authentication (Clerk)

1. **Configure Clerk provider** in the app root layout (`app/_layout.tsx`)
2. **Build auth screens**:
   - Sign-in screen with Google and Apple buttons (using Clerk's `useOAuth` hook)
   - Loading/splash screen while checking auth state
3. **Auth flow routing** (in Expo Router layout):
   - Unauthenticated → Auth screen
   - Authenticated + `onboarding_completed === false` → Onboarding flow
   - Authenticated + onboarded → Home (tabs)
4. **Create user record in Neon** on first sign-in via Clerk webhook or on first API call (check if `clerk_id` exists, if not create)

---

## Phase 4: Onboarding Flow

A multi-step screen flow (not skippable). Each step is a screen in a stack navigator under `app/(onboarding)/`.

### Screens (in order):
1. **Welcome** — brief app intro, "Let's personalize your experience"
2. **Unit Preference** — metric or imperial toggle
3. **Basic Info** — age, height, weight (inputs adapt to unit preference)
4. **Activity Level** — 4 options with descriptions (sedentary, lightly active, moderately active, very active)
5. **Goal** — 5 options (lose fast, lose steady, maintain, gain steady, gain fast) with descriptions of pace
6. **Calculating...** — loading animation while Trigger.dev job sends profile to OpenAI and gets personalized daily targets
7. **Your Plan** — show the AI-calculated daily targets (calories, protein, carbs, fat). "Let's get started" button → mark `onboarding_completed = true`, navigate to home

### Trigger.dev Job: `calculate-goals`
- Input: user profile (age, height, weight, activity level, goal)
- Sends structured prompt to OpenAI asking for daily calorie, protein, carbs, fat targets
- Saves targets to users table
- Returns targets to client

---

## Phase 5: Home Screen (Daily Timeline + Progress)

**Route**: `app/(tabs)/index.tsx`

### Layout:
1. **Top section — Daily Progress**
   - Circular progress ring or bar for calories (consumed / target)
   - Three smaller bars for protein, carbs, fat
   - Date selector (today by default, swipe or tap to browse past days)

2. **Bottom section — Meal Timeline**
   - Scrollable list of meals for the selected day
   - Each meal card shows: thumbnail image (or text icon), meal item names, total calories, time
   - Tap a meal card → meal detail screen (view/edit)
   - Empty state: illustration + "Log your first meal" prompt

3. **Floating Action Button** — "+" or camera icon → opens meal logging flow

### API Route: `GET /api/meals?date=YYYY-MM-DD`
- Returns meals + meal_items for the given date for the authenticated user
- Also returns daily totals and targets

---

## Phase 6: Meal Logging Flow

### Photo Path:
1. User taps FAB → option to "Take Photo" or "Enter Manually"
2. **Camera screen** — uses `expo-camera` or `expo-image-picker` (camera mode)
3. After capture:
   - Compress image on-device using `expo-image-manipulator` (resize to ~1024px max dimension, JPEG quality ~0.7)
   - Upload compressed image to ImageKit via their upload API
   - Get back the ImageKit URL
   - Call API route → triggers Trigger.dev job
4. **Loading screen** — "Analyzing your meal..." with animation
5. **Results screen** — show itemized food breakdown (from AI)

### Text Path:
1. User taps "Enter Manually" → text input screen
2. Type description (e.g., "2 scrambled eggs, toast with butter, orange juice")
3. Call API route → triggers Trigger.dev job (no image, just text)
4. Same loading → results screen

### Results/Edit Screen:
- List of detected food items, each showing: name, portion, calories, protein, carbs, fat
- Confidence indicator per item (green/yellow/red dot based on score)
- Each item is editable (tap to modify values)
- Can delete items or add new ones manually
- "Confirm & Save" button → saves meal + meal_items to database
- "Retake / Re-enter" button → go back

### Trigger.dev Job: `analyze-meal`
- Input: `{ imageUrl?: string, text?: string, userId: string }`
- Builds OpenAI prompt asking for itemized food breakdown with macros and confidence scores
- Uses GPT-4o-mini with vision (for photos) or text (for manual entry)
- Parses AI response with Zod schema validation
- Returns structured result: `{ items: [{ name, portion, calories, protein, carbs, fat, confidence }], totalConfidence }`
- On save: writes to `meals` + `meal_items` tables

### API Routes:
- `POST /api/meals/analyze` — triggers the Trigger.dev job, returns job ID
- `GET /api/meals/analyze/:jobId` — polls for job result (or use Trigger.dev's realtime)
- `POST /api/meals` — saves confirmed meal + items to database

---

## Phase 7: Profile Screen

**Route**: `app/(tabs)/profile.tsx`

### Layout:
- User avatar + name (from Clerk)
- Editable fields: age, height, weight, activity level, goal, unit preference
- Current daily targets display (calories, protein, carbs, fat)
- "Recalculate Goals" button — re-triggers the `calculate-goals` Trigger.dev job with updated profile
- Theme toggle (light / dark / system)
- Notification settings (reminders on/off, reminder times)
- Sign out button

### API Routes:
- `GET /api/user` — get current user profile
- `PUT /api/user` — update profile fields
- `POST /api/user/recalculate-goals` — triggers goal recalculation

---

## Phase 8: Push Notifications

### Setup:
1. Configure `expo-notifications` in app root
2. Request notification permissions on first launch (after onboarding)
3. Save Expo Push Token to user record in database

### Notification Types:

1. **Meal reminders** — scheduled local notifications at user-configured times (e.g., 8am, 12pm, 7pm). Use `expo-notifications` scheduling API (runs on-device, no server needed).

2. **Analysis complete** — when a Trigger.dev job finishes analyzing a meal, send a push notification via Expo Push API if the user navigated away. Trigger.dev job calls Expo Push API as the final step.

3. **Daily summary** — Trigger.dev scheduled cron job runs at end of day (e.g., 9pm in user's timezone). Calculates daily totals, sends a summary push: "You hit 85% of your calorie goal today!"

### API/Jobs:
- Trigger.dev cron job: `daily-summary` (runs nightly)
- Expo Push API integration in Trigger.dev jobs

---

## Phase 9: Dark Mode

1. **Theme context/provider** — wraps the app, reads from user's `theme_preference` (light/dark/system)
2. **NativeWind dark mode** — NativeWind v4 supports `dark:` variant. Configure `darkMode: 'class'` in Tailwind config
3. **Toggle in profile** — updates user preference in DB + local state
4. **System detection** — use `useColorScheme()` from React Native when preference is "system"

---

## Phase 10: Error Tracking (Sentry)

1. Initialize Sentry in app root with `@sentry/react-native`
2. Wrap navigation with Sentry routing instrumentation
3. Add Sentry to API routes for backend error tracking
4. Capture exceptions in Trigger.dev jobs

---

## Phase 11: Meal Detail Screen

**Route**: `app/meal/[id].tsx`

- View a previously logged meal
- Shows image (if photo), item breakdown, macros, time logged
- Edit button → same edit UI as the results screen (modify items, portions, macros)
- Delete meal button (with confirmation)
- Save changes updates the database and recalculates daily totals

---

## Data Flow Diagram

```
[User takes photo]
    → compress on-device (expo-image-manipulator)
    → upload to ImageKit
    → POST /api/meals/analyze { imageUrl }
        → Trigger.dev job: analyze-meal
            → OpenAI GPT-4o-mini (vision)
            → parse response with Zod
            → return itemized breakdown
    → show results to user (editable)
    → user confirms
    → POST /api/meals { items, totals }
        → save to Neon Postgres (meals + meal_items)
    → refresh home screen (TanStack Query invalidation)
```

---

## Assumptions

1. **AI Model**: GPT-4o-mini for development. Production model decision deferred.
2. **Deployment**: TBD — will recommend based on stack needs (likely Vercel or Railway).
3. **Meal reminders**: Implemented as local scheduled notifications on-device (no server cron needed for reminders).
4. **Daily summary timing**: 9pm in user's timezone. Timezone detected from device and stored in user profile.
5. **No rate limiting for v1**: Free and unlimited since this is a portfolio project.
6. **ImageKit URLs are accessible to OpenAI**: We'll use public URLs (or signed URLs with expiry) so OpenAI can fetch the image.
7. **Expo API Routes** are sufficient for all backend needs in v1.

## Open Risks

1. **Expo API Routes maturity** — relatively new feature. If we hit limitations (long-running requests, cold starts), may need to extract to a standalone server.
2. **AI accuracy** — food photo analysis accuracy varies by meal complexity and photo quality. Editable results mitigate this, but user trust depends on reasonable first guesses.
3. **Push notification complexity** — full notifications (reminders + analysis complete + daily summaries) is significant work. Consider building meal reminders (local) first, then analysis-complete, then daily summaries.
4. **ImageKit → OpenAI pipeline** — need to verify that ImageKit-hosted URLs work with OpenAI vision API without auth issues.
5. **Trigger.dev + Expo API Routes integration** — need to confirm Trigger.dev can be initialized within Expo's server context.

---

## Verification Plan

1. **Auth flow**: Sign in with Google/Apple → lands on onboarding → complete onboarding → lands on home. Sign out and back in → lands on home (skips onboarding).
2. **Onboarding**: Fill all fields → loading screen → AI returns daily targets → displayed on "Your Plan" screen → saved to DB.
3. **Photo logging**: Take photo → compressed → uploaded to ImageKit → AI returns itemized breakdown → edit an item → save → appears on home screen with correct totals.
4. **Text logging**: Type meal description → AI returns breakdown → save → appears on home.
5. **Daily progress**: Log multiple meals → progress bars update correctly against daily targets.
6. **Profile edit**: Change weight and goal → recalculate → new targets reflected on home screen.
7. **Notifications**: Verify meal reminders fire at configured times. Verify analysis-complete push arrives. Verify daily summary push.
8. **Dark mode**: Toggle between light/dark/system → all screens render correctly.
9. **Error cases**: Submit blurry/non-food photo → AI still returns best guess with low confidence indicator.

---

## Implementation Order (Recommended)

Build in this order to always have a working app to test:

1. Phase 1 — Project setup & config
2. Phase 2 — Database schema
3. Phase 3 — Auth (Clerk)
4. Phase 4 — Onboarding
5. Phase 5 — Home screen (with mock data initially)
6. Phase 6 — Meal logging (core feature)
7. Phase 11 — Meal detail/edit screen
8. Phase 7 — Profile screen
9. Phase 9 — Dark mode
10. Phase 8 — Push notifications
11. Phase 10 — Sentry error tracking
