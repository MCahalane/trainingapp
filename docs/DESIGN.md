# TrainingApp — Design (v0.1 draft)

> Status: **draft for discussion** · 14 Oct 2026 · Working name only.
>
> A coaching app for one trainer and his clients. **Logging feels like Strong**
> (fast, works with no signal, PRs, "previous" values). **Coaching feels like
> Trainerize** (programs, calendar, messaging, check-ins, health data flowing in).

Contents
1. [Goals](#1-goals)
2. [Users & roles](#2-users--roles)
3. [Platform decision](#3-platform-decision)
4. [Features](#4-features)
5. [Navigation & key screens](#5-navigation--key-screens)
6. [Architecture](#6-architecture)
7. [Data model](#7-data-model)
8. [Security & privacy](#8-security--privacy)
9. [Health integrations](#9-health-integrations)
10. [Tech stack & repo layout](#10-tech-stack--repo-layout)
11. [Roadmap](#11-roadmap)
12. [Running costs](#12-running-costs)
13. [Decisions & open questions](#13-decisions--open-questions)

---

## 1. Goals

**Goals**
- Logging a set mid-workout takes **≤ 2 taps** and works with **no signal**, as in Strong.
- The trainer can **build a program on his phone** and assign it to a client in minutes.
- Trainer and client **talk in the app**, including about a specific workout ("great deadlifts today").
- A client's **weight, steps, sleep and nutrition (MyFitnessPal)** reach the trainer without anyone typing them in.

**Not in v1**
- A marketplace or multi-trainer business (the schema allows it, but we don't build for it).
- Payments, video calls, our own food database, AI coaching.
- Android-first. Both of you use iPhones, so Android comes later.

## 2. Users & roles

| Role | Who | Main jobs |
|---|---|---|
| **Trainer** | Your trainer (phone-first) | Build and assign programs, review clients' workouts, message clients, review check-ins, track adherence |
| **Client** | You + his other clients | Do today's workout, log sets, message the coach, submit check-ins, track progress |
| **Admin** | You, the developer | Supabase dashboard, global exercise library |

One app, two roles. The trainer also gets the full client experience for his own training.

## 3. Platform decision

**The constraint: Apple Health has no web API.** HealthKit data stays on the iPhone, and only a native iOS app can read it. A browser app or PWA can't read Apple Health. It also can't get MyFitnessPal data, because the realistic route to MFP data goes through Apple Health (see §9).

| Option | Apple Health | Effort | Notes |
|---|---|---|---|
| Web app / PWA | ❌ | Lowest | Rules out the health features you want |
| Native Swift (SwiftUI) | ✅ | High | iOS only. Android or web would mean a rewrite |
| **Expo (React Native + TypeScript)** | ✅ | Medium | One codebase for iOS, Android later, and web |

**Recommendation: Expo.**
- We ship an iOS app now, and the same code runs as a **web app** so the trainer can build programs on a laptop when he wants to.
- Android stays possible later.
- **EAS Update** pushes JavaScript fixes straight to phones without waiting for App Store review.

Distribution:
- **TestFlight** while we build. Builds expire after 90 days, which is fine for development.
- Then the **App Store**, either public or *unlisted* (only people with the link can find it).

## 4. Features

Legend: **C** = client, **T** = trainer, **P#** = roadmap phase (§11).

| Area | Inspired by | C | T | Phase |
|---|---|---|---|---|
| Active workout logging (sets, rest timer, supersets, notes) | Strong | ✅ | ✅ | P1 |
| Exercise library + custom exercises + demo videos | Strong | ✅ | ✅ | P1 |
| Templates & folders | Strong | ✅ | ✅ | P1 |
| History, calendar, PRs, exercise charts | Strong | ✅ | view | P1 |
| Measurements | Strong | ✅ | view | P1 |
| Import Strong CSV history | — | ✅ | — | P1 |
| Programs with phases, prescriptions, scheduling | Trainerize | view | ✅ | P2 |
| Client roster, adherence, trainer notes | Trainerize | — | ✅ | P2 |
| 1:1 chat with photos/video, push notifications | Trainerize | ✅ | ✅ | P3 |
| Workout comments & reactions, activity feed | Trainerize | ✅ | ✅ | P3 |
| Weekly check-ins, progress photos, nudges | Trainerize | ✅ | ✅ | P3 |
| Apple Health read/write, MFP nutrition, dashboard widgets | Both | ✅ | view | P4 |
| Live Activity / Dynamic Island, Watch, habits, Android, payments | — | | | P5 |

### 4.1 Workout logging (matching Strong)
- Start an **empty workout**, a **template**, or **today's program workout**.
- The active workout opens as a **sheet over the app** that you can minimise to a bar and keep browsing, as in Strong.
- **Header**: name (auto-named "Afternoon Workout" and so on), date, elapsed timer, **Finish** / **Cancel**.
- **Set table** columns: `Set | Previous | kg | Reps | ✓`
  - **Previous** shows the same set from the last time you did that exercise. Tap it to copy the values.
  - When the workout comes from the trainer's program, the **target** shows as placeholder text in the kg/reps fields (e.g. `8–10`), plus a line such as *"Target: 4 × 8–10 @ RPE 8"*.
  - **Set types**: warm-up (W), normal, drop (D), failure (F). Tap the set number to change it.
  - Ticking ✓ **starts the rest timer**. Each exercise has its own rest length. When rest ends you get a local notification and a vibration, even with the phone locked.
- **Exercise ⋯ menu** (same as Strong): Add Note, Add Sticky Note (it shows again next time), Add Warm-up Sets (calculator), Update Rest Timers, Replace Exercise, Create Superset, Preferences (weight unit, bar type), Remove.
- **Tracking types** per exercise: weight × reps, reps only, weighted or assisted bodyweight, duration (plank), and distance + time (cardio).
- **Plate calculator** that uses the bar type: Olympic 20 kg, Short 15 kg, EZ 7.5 kg, Hex 34 kg, None.
- **Finish** opens a summary with duration, volume and PRs 🏆, and an optional *"Update template with these values?"* prompt. The coach is notified (P3).
- **Offline-safe**: every tap is saved on the device straight away. The workout survives the app being killed and uploads when there's signal.

### 4.2 Exercise library
- A **global library** seeded from an open dataset. The candidate is *free-exercise-db*: public domain, about 800 exercises with images. **We can't copy Strong's images or data.**
- **Trainer custom exercises** carry his own demo video (recorded on his phone) and coaching cues. All his clients see them.
- **Client custom exercises** are private to that client.
- Search, filter by **body part** and **category/equipment**, and jump with an A–Z index.
- **Exercise detail** tabs:
  - About: video and cues.
  - History.
  - Charts: estimated 1RM, best set and volume over time.
  - Records: PRs.

### 4.3 Templates & programs
- A **template** is a reusable workout, as in Strong ("Push 2025 Weekend"). Templates can go in folders. Clients keep their own templates alongside the ones the trainer assigns.
- A **program** is the trainer's plan for one client, as in Trainerize ("Michael's program"). It is split into **phases** such as "Week 1–4" (28 Aug – 24 Sep), and each phase holds workouts (Lower body, Pull, Push) with an estimated duration, exercise count and cover image.
- **Per-set prescription**:
  - reps or a rep range;
  - weight, %1RM or RPE/RIR;
  - rest, tempo and notes;
  - supersets and circuits.
- **Library → assign**: the trainer keeps programs in his library. "Assign to client" **copies** the program with a start date, so tweaking one client's plan never changes another client's.
- **Phase rollover**: when a phase ends the next one starts automatically. The trainer gets a nudge a week before: *"Michael's phase ends Sunday. Build the next one?"*
- **Progression hint** (later): suggest the next weight from the last session. This builds the *"If able, increase weight by 5 kg"* note into the app.

### 4.4 Calendar & scheduling
- The trainer can pin program workouts to days (Mon/Wed/Fri) or leave them as *"3 this week, any order"*.
- The client calendar shows each workout as planned, done ✓, skipped or missed, plus check-in due dates.
- **Workouts per week goal** with a progress widget, as in Strong.

### 4.5 History, PRs & progress
- **History** is grouped by month. Each card shows name, date, duration, volume, PR count, best set per exercise and notes. A calendar view is available.
- **PR types**: heaviest weight, best estimated 1RM (Epley: `w × (1 + reps/30)`), most reps at a given weight, best set volume. Calculated on the server (§6).
- **Dashboard widgets**: workouts per week vs goal, weekly sets per muscle group, body-weight trend, daily macros vs target.
- **Import from Strong**: Strong can export to CSV, so your history back to 2025 comes with you.

### 4.6 Measurements & progress photos
- The same list as Strong:
  - **Core**: Weight, Body Fat, Caloric Intake.
  - **Body part**: Neck, Shoulders, Chest, L/R Bicep, L/R Forearm, Upper Abs, Waist, Hips, L/R Thigh, L/R Calf.
- **Source**: manual entry or Apple Health. Smart scales fill weight and body fat automatically.
- **Progress photos** (front/side/back) are visible only to the client and their trainer, with side-by-side comparison.

### 4.7 Communication
- **1:1 chat** between trainer and client: text, photos, and **videos for form checks**. Voice notes come later.
- **Workout-linked messages**: commenting on a finished workout posts a **workout card** into the chat, and tapping it opens the full workout.
- **Reactions** (👍 🔥 💪) on workouts and messages.
- **Broadcast**: the trainer sends one message to all or selected clients. It arrives as a normal 1:1 message, so replies stay private.
- **Push notifications** for new messages, workout comments, program updates and check-ins due. The rest timer uses local notifications.
- **Unread badges and read receipts**. A typing indicator comes later.
- **Automated nudges** that the trainer controls, e.g. *"No workout in 4 days"* or *"Check-in due Sunday"*.
- **Private trainer notes** on each client (injuries, preferences). The client never sees them.
- **Out of scope**: video calls. Use FaceTime or WhatsApp.
- **Later**: group chat / community.

### 4.8 Check-ins (and habits later)
- **Weekly check-in form** built by the trainer:
  - body weight and progress photos;
  - 1–10 sliders for energy, sleep, stress, hunger and adherence;
  - free text.
  
  Each client has a due day.
- The trainer reviews the check-in and replies, and the reply lands in the chat.
- **Habits** (P5): daily ticks such as "10k steps" or "3 L water", completed automatically from Apple Health where possible.

### 4.9 Trainer home
- **Client roster** with each client's status at a glance:
  - last workout;
  - this week's adherence (●●○ 2/3);
  - unread messages and check-ins waiting;
  - phase ending soon;
  - flags such as no workout for 4+ days.
- **Activity feed**: completed workouts, PRs, check-ins and new measurements across all clients. The trainer can react or comment inline.
- **Client profile** tabs:
  - Overview: adherence, weight trend, steps, sleep, macros.
  - Program, Calendar, History, Progress.
  - Chat and Notes.

## 5. Navigation & key screens

**Client tabs**: `Home · History · (+) Workout · Progress · Chat`

| Tab | Contents |
|---|---|
| Home | Today's workout and Start button, check-in due, latest coach message, dashboard widgets. Avatar opens profile and settings |
| History | List plus calendar (from Strong) |
| Workout (+) | Quick start, **current program** (phase and its workouts), my templates, example templates |
| Progress | Measurements, photos, exercises (library and per-exercise stats), health data |
| Chat | Thread with the coach |

**Trainer tabs**: `Clients · Feed · Library · Chat · Me`

| Tab | Contents |
|---|---|
| Clients | Roster, then client profile |
| Feed | Activity across all clients |
| Library | Programs, templates, exercises, check-in forms |
| Chat | Inbox of client threads and broadcast |
| Me | His own training (full client experience) and settings |

Rough wireframes (layout only, not visual design):

```
CLIENT · HOME                        ACTIVE WORKOUT (from program)
+-------------------------------+    +-------------------------------+
| (MC)  Today              (!)  |    | [timer]  Push Workout [Finish]|
|                               |    | 14 Oct  ·  0:12:31            |
| +- Phase 2 · Week 5 --------+ |    |                               |
| | PUSH WORKOUT              | |    | Incline Bench Press (BB)  ... |
| | 6 exercises · ~58 min     | |    | Pin: elbows tucked, 2s down   |
| | [      Start workout    ] | |    | Target: 4 x 8-10 @ RPE 8      |
| +---------------------------+ |    | Set  Previous   kg    Reps  v |
| Coach: "Go heavier on         |    |  W   40 x 10   [40 ] [10 ] [v]|
|  incline this week"           |    |  1   60 x 10   [62.5][ 9 ] [v]|
| [ ] Weekly check-in due Sun   |    |  ------- rest 1:47 ---------  |
|                               |    |  2   60 x 9    [    ][8-10][ ]|
| This week  ##-  2/3           |    |  3   60 x 8    [    ][8-10][ ]|
| Steps 8,412 · Sleep 7h10      |    | [ + Add set ]                 |
| Kcal 2140/2400 · P 150/180g   |    |                               |
+-------------------------------+    | [ Add Exercises ]             |
| Home  Hist  (+)  Prog   Chat  |    | [ Cancel Workout ]            |
+-------------------------------+    +-------------------------------+

TRAINER · CLIENTS                    CHAT (client <-> coach)
+-------------------------------+    +-------------------------------+
| Clients (12)        [+Invite] |    | <  Coach                      |
| [Search]  [Needs attention]   |    |                               |
|                               |    | +- Push Workout · 58m ------+ |
| Michael C.         (2) msgs   |    | |  7,900 kg · 3 PRs         | |
| Push · today · 3 PRs          |    | +---------------------------+ |
| Week ##-  Wt 82.1 (down)      |    |       Great session! Bump     |
| ----------------------------- |    |       incline to 65 next  <   |
| Sarah K.      Check-in ready  |    | > Shoulder felt off on OHP,   |
| Week ###  Phase ends in 5d    |    |   video attached [video]      |
| ----------------------------- |    |                               |
| Tom R.        ! 6 days idle   |    |                               |
| No workout since Wed          |    | [+] [ Message...       ] [>]  |
+-------------------------------+    +-------------------------------+
| Clients Feed Library Chat Me  |
+-------------------------------+
```

## 6. Architecture

```
+------------------------- iPhone (Expo app) --------------------------+
|  Screens (Expo Router)                                               |
|   - Active workout store (Zustand + MMKV)   <- fully offline         |
|   - Server data cache (TanStack Query, persisted)                    |
|   - HealthKit module  <->  Apple Health  <- MFP, Garmin, Oura, scales|
|   - expo-notifications (push + local rest-timer alerts)              |
+-----------+-----------------------------------------^----------------+
            | HTTPS (supabase-js)                     | Realtime (websocket)
+-----------v-----------------------------------------+----------------+
|  SUPABASE                                                            |
|   Auth ........ Sign in with Apple, email magic link                 |
|   Postgres .... tables + Row Level Security + RPCs (finish_workout)  |
|   Realtime .... chat messages, feed, unread counts                   |
|   Storage ..... private buckets: photos, videos, avatars             |
|   Edge Fns .... send-push, accept-invite, (later) OAuth callbacks    |
|   pg_cron ..... nudges, phase rollover, missed-workout marking       |
+-----------+----------------------------------------------------------+
            v
     Expo Push Service -> APNs -> phones
```

### Key flows

**Finishing a workout**
1. The app creates IDs (UUIDs) on the phone, so a workout exists before it ever reaches the server.
2. Every set edit is saved on the device immediately.
3. On **Finish**, the app calls one database function, `finish_workout(payload)`. In a single transaction it:
   - inserts the workout, its exercises and its sets;
   - **calculates PRs on the server**;
   - marks the scheduled workout done;
   - creates a feed event for the trainer.
4. The call is **idempotent**: the same ID never creates a duplicate, so retrying after bad gym signal is safe.
5. The app then writes an *HKWorkout* to Apple Health, which counts toward the Activity rings.

**Sending a message**
1. Any attachment uploads to Storage first.
2. The app inserts a row into `messages`. Row Level Security checks that the sender is a member of the conversation.
3. Realtime delivers the message to the other person's open app instantly.
4. A database webhook runs the `send-push` Edge Function, which sends it through Expo Push to APNs for phones that aren't open.

**Health sync**
1. Runs on app open, plus HealthKit background delivery (best-effort).
2. The app reads the last N days and rolls them up into **daily summaries**.
3. It upserts them into `health_daily`, only for metrics the client agreed to share.
4. The trainer sees the data plus a *"last synced"* time.

**Why the live workout isn't synced to the server:** gym signal is unreliable and Strong works offline for the same reason. The trainer doesn't need to watch sets in real time for v1.

## 7. Data model

Conventions:
- **UUID** primary keys, created on the phone for anything that can be made offline.
- `created_at` / `updated_at` on every table.
- **Weights stored in kg** and converted for display.
- Soft delete (`deleted_at`) where history matters.

**People & coaching**
- `profiles`: id (= auth user), role (`trainer|client`), full_name, avatar_path, unit_pref (`kg|lb`), timezone
- `coaching_links`: trainer_id, client_id, status (`invited|active|paused|ended`), started_at. **Every trainer permission depends on this table.**
- `invites`: code, trainer_id, email, expires_at, accepted_by
- `trainer_notes`: trainer_id, client_id, body. Private to the trainer.

**Exercise library**
- `exercises`: id, owner_id (null = global), name, equipment (`barbell|dumbbell|machine|cable|bodyweight|kettlebell|band|cardio|other`), body_part, secondary_muscles[], tracking_type (`weight_reps|reps|duration|distance_time|assisted`), instructions, image_path, video_path, default_bar_type, archived
- `exercise_prefs`: user_id, exercise_id, bar_type, rest_seconds, sticky_note, unit

**Plans** (what *should* happen)
- `template_folders`: id, owner_id, name
- `workout_templates`: id, owner_id, folder_id?, program_phase_id?, name, notes, est_minutes, cover_path, position
- `template_exercises`: id, template_id, exercise_id, position, superset_group, rest_seconds, notes
- `template_sets`: id, template_exercise_id, position, set_type, reps_min, reps_max, weight_kg, pct_1rm, rpe, duration_s, distance_m, tempo
- `programs`: id, trainer_id, client_id (null = library program), name, description, status (`draft|active|completed`), copied_from_id
- `program_phases`: id, program_id, name, position, start_date, end_date
- `scheduled_workouts`: id, client_id, template_id, date, status (`planned|done|skipped|missed`), workout_id?

**Logs** (what *actually* happened)
- `workouts`: id, user_id, template_id?, scheduled_workout_id?, name, started_at, ended_at, notes, volume_kg, set_count, healthkit_uuid
- `workout_exercises`: id, workout_id, exercise_id, position, superset_group, notes
- `workout_sets`: id, workout_exercise_id, position, set_type (`warmup|normal|drop|failure`), weight_kg, reps, duration_s, distance_m, rpe, completed
- `personal_records`: id, user_id, exercise_id, kind (`max_weight|est_1rm|max_reps|max_volume`), value, workout_set_id, achieved_at

Plans and logs are kept separate on purpose. The client can deviate from the plan (swap an exercise, add a set) without changing the program, and the trainer can compare *prescribed vs done*.

**Body & health**
- `measurements`: id, user_id, kind (`weight|body_fat|calories|neck|chest|waist|…`), value, unit, measured_at, source (`manual|apple_health`), external_id
- `progress_photos`: id, user_id, storage_path, pose (`front|side|back`), taken_at
- `health_daily`: user_id, date, metric (`steps|active_kcal|resting_hr|hrv|sleep_min|diet_kcal|protein_g|carbs_g|fat_g`), value, source. Primary key: (user_id, date, metric).
- `health_sharing`: user_id, metric, shared_with_trainer. **The client decides what the coach sees.**
- `nutrition_targets`: client_id, kcal, protein_g, carbs_g, fat_g, set_by, effective_from

**Communication**
- `conversations`: id, kind (`direct|group`), created_by
- `conversation_members`: conversation_id, user_id, last_read_at, muted
- `messages`: id, conversation_id, sender_id, body, attachment_path, attachment_kind (`image|video|audio`), workout_id?, check_in_id?, reply_to_id?, created_at, edited_at, deleted_at
- `reactions`: user_id, target_kind (`message|workout`), target_id, emoji
- `feed_events`: id, client_id, kind (`workout_done|pr|check_in|measurement|phase_ending`), ref_id, created_at
- `check_in_forms`: id, trainer_id, name, questions (jsonb), due_weekday
- `check_ins`: id, client_id, form_id, due_date, submitted_at, answers (jsonb), reviewed_at
- `push_tokens`: user_id, token, platform
- `notification_prefs`: user_id, kind, enabled

**Integrations** (P5, only if needed)
- `connected_accounts`: user_id, provider, status, external_user_id, token (kept in Supabase Vault), last_synced_at

## 8. Security & privacy

**Row Level Security on every table.** Most rules hang off one helper function, `is_my_client(uid)`. It is true when an **active** `coaching_link` exists between the logged-in trainer and that client.

| Data | Client | Trainer |
|---|---|---|
| Own workouts, measurements, photos | read/write | read (own clients only) |
| Programs, templates, schedule assigned to them | read | read/write |
| Logged workouts | write | read + comment/react, **no editing** |
| Health metrics | write | read, **only the metrics the client shares** |
| Messages | members of the conversation only | members of the conversation only |
| Trainer notes | ❌ | read/write |

- **Storage**: private buckets with paths like `{user_id}/…`, served through short-lived signed URLs. Trainers get access through the same `is_my_client` rule.
- **Ending coaching** removes the trainer's access immediately. The client keeps their data.
- **Server-side numbers**: PRs and totals are calculated in the database, not trusted from the app.
- **RLS tests**: automated tests (pgTAP) check *"client A can't see client B"*. This is the most important test suite in the project.
- **Health data**:
  - **Apple rules**: HealthKit data can't be used for advertising or sold. The app needs a privacy policy and clear permission prompts.
  - **UK/EU (GDPR)**: if you're in the UK or EU, health data is *special category* data. That means:
    - explicit consent at onboarding;
    - a sharing toggle per metric;
    - data export and **in-app account deletion** (the App Store requires the deletion part anyway);
    - a Supabase project hosted in a **UK/EU region**.

## 9. Health integrations

**Strategy: Apple Health is the hub.** Most of the apps and wearables your clients already use **write into Apple Health**. One native integration therefore covers most of them at no extra cost.

| Source | How we get the data | What we get | When |
|---|---|---|---|
| **Apple Health** | Native HealthKit, on the phone | weight, body fat, steps, active kcal, resting HR, HRV, sleep, workouts, **nutrition** | P4 |
| **MyFitnessPal** | **No public API** for new developers. The client turns on MFP → Apple Health sharing and **we read it from Apple Health** | daily kcal, protein, carbs, fat | P4 (via Apple Health) |
| Garmin, Oura, Whoop, Strava, Withings scales | Their apps write to Apple Health | steps, sleep, HR, weight, cardio | P4 (via Apple Health) |
| **Fitbit** | Fitbit doesn't sync to Apple Health. Its cloud API (moving to Google's Health API) needs OAuth | steps, sleep, HR | Only if a client needs it |
| **Google Fit / Android** | Google Fit APIs are being shut down. Android uses **Health Connect** (on the phone, like HealthKit) | same as Apple Health | When an Android client appears |
| Aggregators (Terra, etc.) | One paid API for everything | everything | Not worth it: priced at hundreds of dollars a month |

**We also write to Apple Health:**
- completed strength workouts, which close Activity rings;
- body weight entered in the app.

**Caveats**
- HealthKit only works in a **native development build**. It doesn't work in Expo Go or on the web.
- iOS **never tells an app that read permission was denied**, for privacy reasons. "No data" and "denied" look the same, so the UI has to handle that kindly.
- Background delivery is best-effort, so we also sync whenever the app opens.
- MFP → Apple Health sharing is switched on inside MFP by the client. We should check whether their MFP plan affects which data it shares.

## 10. Tech stack & repo layout

| Layer | Choice |
|---|---|
| App | **Expo** (React Native) + **TypeScript**, Expo Router |
| Styling | NativeWind (Tailwind classes) + a small in-house component kit |
| Server data | TanStack Query (cache kept on the phone between launches) |
| Offline / active workout | Zustand + react-native-mmkv |
| Charts | victory-native |
| Health | `@kingstinct/react-native-healthkit` (includes Expo config plugin) |
| Notifications | expo-notifications + Expo Push Service |
| Media | expo-image-picker, expo-image, expo-video |
| Backend | **Supabase**: Postgres, Auth, RLS, Realtime, Storage, Edge Functions, pg_cron |
| Schema | SQL migrations in the repo (Supabase CLI) with generated TypeScript types |
| Builds & releases | EAS Build, Submit and Update |
| Tests | Jest + React Native Testing Library, **pgTAP for RLS**, Maestro for end-to-end flows |
| Errors | Sentry |

```
trainingapp/
  app/                  Expo Router screens: (client)/, (trainer)/, workout/, chat/
  src/
    features/           workout, exercises, programs, chat, checkins, health, progress
    components/         shared UI kit
    lib/                supabase client, units, 1RM / plate math
  supabase/
    migrations/         schema + RLS policies
    functions/          edge functions (send-push, accept-invite)
    seed/               exercise library
    tests/              pgTAP RLS tests
  docs/DESIGN.md        this file
```

## 11. Roadmap

Each phase ends with something **you use for real**.

| Phase | Scope | Done when… |
|---|---|---|
| **P0 Foundations** | Expo app, Supabase project (UK/EU region), sign-in (Apple + email), profiles/roles, exercise seed, CI, TestFlight pipeline | You and your trainer can sign in on a TestFlight build |
| **P1 "Strong for us"** | Exercise library, templates, offline active workout, rest timer, history, PRs, charts, measurements, Strong CSV import | You log real gym sessions in it **instead of Strong for 2 weeks** |
| **P2 Coaching** | Invites, client roster, program builder (phases, prescriptions), assign + schedule, client program and calendar, trainer views client history, trainer notes | Your trainer builds **your next phase** in the app |
| **P3 Communication** | Chat with photos/video, push, workout comments and reactions, feed, check-ins, progress photos, nudges | Your trainer **stops using WhatsApp/Trainerize** to message you |
| **P4 Health** | Apple Health read/write, MFP nutrition via Apple Health, macro targets, dashboard widgets, trainer health overview | Your trainer sees your weight, steps, sleep and macros without asking |
| **P5 Nice-to-haves** | Live Activity / Dynamic Island rest timer, Apple Watch, habits, group chat, Android + Health Connect, Fitbit, Stripe payments, AI weekly summaries | — |

## 12. Running costs

| Item | Cost |
|---|---|
| Apple Developer Program (needed for TestFlight, App Store, HealthKit) | $99 / year |
| Supabase | Free while building. **Pro (~$25/mo) for production**: free projects pause when idle and have no backups |
| Expo EAS (builds/updates) | Free tier is enough to start |
| Expo Push, Sentry | Free tiers |
| Google Play (only if Android) | $25 one-time |

**Roughly $99/year plus about $25/month once real clients are on it.** Check current prices before committing.

## 13. Decisions & open questions

**Proposed decisions** (push back on any of them):
1. **Native iOS app built with Expo.** Apple Health requires native code; the web build is a bonus for the trainer.
2. **Supabase** for auth, database, realtime chat, file storage and server functions.
3. **Apple Health as the integration hub**, with MyFitnessPal data arriving through Apple Health.
4. **Offline-first logging**: the active workout lives on the phone until Finish.
5. **One app with two roles.** The trainer can also train himself.
6. **Plans and logs are separate.** Clients may deviate from the plan, and the trainer sees what was prescribed vs what was done.
7. **kg is stored in the database.** lb is a display preference.

**Questions for you and your trainer**
1. Is this only for your trainer, or might **other trainers** use it one day? This affects branding and billing; the schema already allows it.
2. Roughly **how many clients**? Are any on **Android**?
3. Does he want to **charge clients through the app** (Stripe subscriptions), or keep billing outside it?
4. Should he **set macro targets**, or just see what clients log in MFP?
5. What does he **ask clients in check-ins** today? We'll copy his current questions.
6. Will he **record his own exercise demo videos**, or are stock images fine to start?
7. Should existing clients and programs **move over from Trainerize**? Exporting from it may have to be manual.
8. **Name and branding**: his business brand, or something new?
9. **What does he dislike about Trainerize?** This is the most useful design input we could get.
10. Can clients **edit assigned workouts** (swap an exercise, add sets), or are they locked? *Suggestion: allow it, and highlight the differences to the trainer.*
