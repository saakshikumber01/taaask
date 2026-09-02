# Tiny Wins — App Knowledge Base

## Document purpose and source of truth

This document captures the current, end-to-end behavior of the Tiny Wins app: its product intent, runtime architecture, data model, user flows, calculations, persistence, reminders, offline behavior, UI system, and known limitations.

The analysis is based on the exact code currently present in this repository:

- `index.html` — the complete shipped React application, already compiled and minified into one HTML file.
- `public/manifest.json` — Progressive Web App metadata.
- `public/sw.js` — offline service worker.
- `package.json` and `package-lock.json` — the local Vite development/build wrapper.

Important constraint: the repository does **not** contain the original component source files, TypeScript types, tests, or source maps. The React code and CSS are readable only as a production bundle embedded in `index.html`. The behavior described below is confirmed from that bundle. Names such as “Home module” or “Store provider” are logical names used for maintainability; they are not separate source files today.

## 1. Product overview

Tiny Wins is a private, mobile-first personal routine and wellbeing tracker. Its central product idea is to reduce pressure: help the user do and record the next manageable action rather than optimize every part of life.

The app combines several related tools:

- A recurring weekly timetable.
- Daily routine completion and skipping.
- A general task list.
- Movement, sleep, eye-break, and mood tracking.
- A social-posting tracker.
- A ByWitness creative-content pipeline and focus timer.
- Day-level planning and retrospective records.
- Configurable reminders.
- Weekly progress and streak calculations.
- Local backup, restore, and reset.
- Installable/offline PWA behavior.

The current defaults are personalized for a user named **Saakshi** and for a creative project called **ByWitness**.

### Product tone

The copy is intentionally warm, informal, forgiving, and mildly playful. It repeatedly reinforces that partial effort counts, skipped days are not moral failures, rest is useful, and imperfect publishing is preferable to paralysis. Success actions show short randomized messages and sometimes confetti.

### Privacy model

The app has no user accounts, backend, cloud database, analytics integration, or application-level API calls. Normal app data remains in browser storage on the current origin/device. Exported backups are downloaded as plain JSON.

## 2. Repository and build structure

```text
taaask/
├── .gitignore
├── README.md
├── app_knowledge_base.md
├── index.html
├── package.json
├── package-lock.json
└── public/
    ├── icon-192.png
    ├── icon-512.png
    ├── icon-maskable.png
    ├── manifest.json
    └── sw.js
```

Generated files appear in `dist/` after a production build and are intentionally ignored by Git.

### Local tooling

- Package manager: npm.
- Development/build tool: Vite.
- Installed Vite version at the time of analysis: `7.3.6`.
- `npm run dev`: starts the Vite development server.
- `npm run build`: builds production files into `dist/`.
- `npm run preview`: serves the built `dist/` output locally.

The original app bundle includes React and React DOM `19.2.8`. React is embedded directly in the shipped bundle rather than declared as a repository dependency.

### Production output

Vite extracts the inline module script from the source HTML into a hashed JavaScript asset while preserving the app behavior:

```text
dist/
├── assets/index-[hash].js
├── icon-192.png
├── icon-512.png
├── icon-maskable.png
├── index.html
├── manifest.json
└── sw.js
```

## 3. Runtime architecture

The app is a client-only React single-page application.

At runtime, the architecture is effectively:

```text
Browser document
└── React root
    └── Store provider
        └── App shell
            ├── Current screen
            ├── Global toast
            ├── Global quick-add button
            ├── Bottom navigation
            └── Quick-add bottom sheets
```

### Store provider

One React context owns the entire persisted application state. It exposes:

- `state`: the complete app state object.
- `ready`: whether initial storage loading has finished.
- `actions`: every state mutation function.
- `say(message, tone)`: displays a toast for 2.6 seconds.
- `toast`: the current toast payload.

All state writes follow the same pattern:

1. Clone the current state with `structuredClone` when available.
2. Fall back to JSON serialize/parse cloning if necessary.
3. Mutate the clone.
4. Set it as the new React state.
5. Debounce persistence for 120 milliseconds.

This makes each update immutable from React's point of view even though action implementations mutate the fresh clone internally.

### App shell and navigation

Navigation is local React state, not URL routing. Reloading the page always returns to Home.

Primary bottom tabs:

1. Home
2. Timetable
3. Tracker
4. Plan

Secondary screens are reached through the profile/settings screen:

- Edit timetable
- The List
- Weekly progress
- Me
- Eyes
- Reminders

The fixed quick-add button is available across all screens. Opening a primary tab scrolls the main content area back to the top. Secondary screens do not appear selected in the bottom navigation.

## 4. First launch and initialization flow

```text
Open app
  → show “Getting your day…” loading screen
  → choose storage adapter
  → read key `tinywins:state:v1`
  → parse saved JSON if present
  → merge/migrate state
  → otherwise create version-2 default state
  → render Home
  → start future reminder timers for the current day
  → persist subsequent changes after a 120 ms debounce
```

If saved JSON cannot be parsed, it is treated as missing and a new default state is created. The invalid stored value is not explicitly deleted, but the freshly initialized state will be persisted after a subsequent state change.

## 5. Persistence and storage fallback

The persisted key is:

```text
tinywins:state:v1
```

Storage selection happens once when the bundle loads:

1. Try writing and deleting a localStorage probe key.
2. If successful, use `window.localStorage` and report the adapter name as `local`.
3. If localStorage fails and a host-provided `window.storage` API exists, use that and report `host`.
4. Otherwise use an in-memory `Map` and report `memory`.

Consequences:

- `local`: data survives refreshes for the same browser origin.
- `host`: persistence depends on the embedding host's storage API.
- `memory`: data is lost when the page is reloaded or closed.

The Settings screen displays the selected adapter name to the user.

### Backup export

Export serializes the entire state as pretty-printed JSON and downloads it as:

```text
tiny-wins-YYYY-MM-DD.json
```

The backup contains everything, including journal text, mood history, task notes, content data, and settings. It is not encrypted.

### Backup restore

Restore accepts `.json` or `application/json` files. Validation only checks that:

- the JSON parses;
- the root is an object; and
- `routine` is an array.

The imported object is then passed through the migration/default-merging function and immediately persisted. There is no detailed schema validation for nested records, field types, malformed dates, unknown enum values, or unreasonable numeric values.

### Reset

“Start over” opens a destructive confirmation sheet. Confirming:

1. Removes the persisted storage key.
2. Creates a completely new default state.
3. Generates new IDs for default routine blocks and reminders.
4. Later persists the fresh state through the normal debounced save effect.

All tasks, content, daily logs, journals, settings, and customizations are discarded.

## 6. State schema

The state version is currently `2`.

Conceptual schema:

```js
{
  version: 2,
  profile: {
    name: string,
    createdAt: ISODateTimeString
  },
  routine: RoutineItem[],
  reminders: Reminder[],
  tasks: Task[],
  content: ContentItem[],
  log: Record<DateKey, DailyRoutineLog>,
  body: Record<DateKey, BodyLog>,
  eyes: Record<DateKey, EyeLog>,
  me: Record<DateKey, CheckIn>,
  focus: Record<DateKey, number>,
  social: Record<DateKey, SocialPostLog>,
  plans: Record<DateKey, DayPlan>,
  settings: Settings
}
```

`DateKey` is a local-calendar string in `YYYY-MM-DD` form.

### RoutineItem

```js
{
  id: string,            // generated with `rt_` prefix
  time: "HH:MM",        // 24-hour local time
  title: string,
  category: CategoryId,
  duration: number,      // minutes
  days: number[],        // Monday=0 through Sunday=6
  note: string,
  enabled: boolean
}
```

### Reminder

```js
{
  id: string,            // generated with `rm_` prefix
  time: "HH:MM",
  text: string,
  enabled: boolean,
  days: number[]         // Monday=0 through Sunday=6
}
```

### Task

```js
{
  id: string,            // generated with `tk_` prefix
  title: string,
  category: CategoryId | null,
  due: "YYYY-MM-DD" | null | "",
  priority: "high" | "normal" | "low",
  notes: string,
  done: boolean,
  doneAt: number | null, // epoch milliseconds
  createdAt: number      // epoch milliseconds
}
```

### ContentItem

```js
{
  id: string,            // generated with `bw_` prefix
  title: string,
  type: ContentType,
  platform: Platform,
  status: "ideas" | "creating" | "editing" | "ready" | "published",
  createdAt: number,
  publishOn: "YYYY-MM-DD" | null | "",
  publishedAt: number | null,
  notes: string,
  link: string,
  stats: {
    views?: number,
    likes?: number,
    comments?: number,
    shares?: number,
    saves?: number,
    followers?: number
  }
}
```

Content types:

- Instagram Post
- Instagram Reel
- Carousel
- Story
- LinkedIn
- Behance
- YouTube
- Other

Platforms:

- Instagram
- LinkedIn
- Behance
- YouTube
- Other

### DailyRoutineLog

```js
{
  done: Record<RoutineItemId, number>,    // completion timestamp
  skipped: Record<RoutineItemId, true>,
  notes: Record<RoutineItemId, string>
}
```

Completion and skip are mutually exclusive for a routine item on a given date. Per-day routine notes are supported in state, although the current visible UI does not expose an editor for them.

### BodyLog

```js
{
  strength?: boolean,
  walk?: boolean,
  stretch?: boolean,
  sleepAt?: "HH:MM"
}
```

### EyeLog

```js
{
  breaks: number, // short breaks
  long: number    // longer breaks
}
```

### CheckIn

```js
{
  mood?: "good" | "okay" | "meh" | "sad" | "heavy" | "angry" | "overwhelmed" | null,
  journal?: string,
  savedAt: number
}
```

### SocialPostLog

```js
{
  posted: true,
  platform: Platform,
  note: string
}
```

This record is separate from the ByWitness content pipeline. Marking a social post does not create or publish a content item, and publishing a content item does not mark the daily social tracker.

### DayPlan

```js
{
  items: Array<{
    id: string,          // generated with `pl_` prefix
    text: string,
    tag: "bywitness" | "recharge" | "other",
    done: boolean
  }>
}
```

### Settings

```js
{
  theme: "paper" | "blush" | "sage" | "ink",
  postsPerWeek: number,
  focusHoursPerWeek: number,
  strengthPerWeek: number,
  socialPostsPerWeek: number,
  sleepTarget: "HH:MM",
  notifications: boolean
}
```

Numeric goals are clamped to zero or greater in the settings UI.

## 7. IDs and ordering

IDs are generated client-side from:

```text
prefix + current timestamp in base 36 + five random base-36 characters
```

This is sufficiently collision-resistant for one local user but is not a formal UUID system.

Routine and reminder displays are time-sorted. Times before 04:00 are treated as belonging to the end of the waking day for sorting, so `00:15` and `00:30` appear after late-evening entries rather than at the top.

The dedicated Edit Timetable screen preserves the stored array order and provides manual move-up/move-down controls. The normal Timetable screen time-sorts the selected day's enabled items.

New tasks and content items are inserted at the beginning of their arrays.

## 8. Categories, priorities, stages, and themes

### Routine/task categories

| ID | Label | Emoji | Accent |
|---|---|---:|---|
| `exercise` | Exercise | 💪 | Peach |
| `bywitness` | ByWitness | 🎨 | Lavender |
| `work` | Office work | 💻 | Periwinkle |
| `recharge` | Recharge | 🎧 | Green |
| `break` | Break | ☕ | Sand |
| `medicine` | Medicine | 💊 | Pink |
| `eyes` | Eyes | 👁️ | Teal |
| `me` | Me | 🫶 | Mauve |
| `sleep` | Sleep | 😴 | Grey-blue |

Unknown category IDs fall back visually to the Break category.

### Task priorities

| ID | Label | Emoji | Sort rank |
|---|---|---:|---:|
| `high` | Important | 🔥 | 0 |
| `normal` | Normal | 🌱 | 1 |
| `low` | Low pressure | 🫧 | 2 |

### Content stages

```text
Ideas → Creating → Editing → Ready → Published
```

The pipeline's “Move →” action advances exactly one stage and stops at Published.

### Themes

- Paper: warm cream background, dark brown ink.
- Blush: pale rose background.
- Sage: pale green background.
- Ink: dark mode with light foreground.

Changing a theme sets `data-theme` on the root HTML element and updates the browser/PWA theme-color meta tag from the active paper/background color.

## 9. Default state

### Default profile and goals

- Name: Saakshi
- Theme: Paper
- ByWitness published posts per week: 3
- ByWitness focus hours per week: 7
- Strength sessions per week: 3
- Social account posts per week: 3
- Sleep target: 00:30
- `notifications` setting: false

### Default recurring routine

| Time | Title | Category | Duration | Days | Note |
|---|---|---|---:|---|---|
| 08:45 | Wake up | Sleep | 15 min | Every day | Feet on floor. That’s it. |
| 09:00 | Home exercise | Exercise | 30 min | Mon–Fri | Get stronger, not smaller. |
| 09:30 | Walk | Exercise | 15 min | Every day | — |
| 09:45 | Eye break | Eyes | 5 min | Mon–Fri | — |
| 10:30 | Office work | Work | 210 min | Mon–Fri | Morning block |
| 14:00 | Lunch | Break | 45 min | Every day | Please leave the laptop. |
| 14:45 | Medicine | Medicine | 5 min | Every day | After lunch |
| 15:00 | Office work | Work | 180 min | Mon–Fri | Afternoon block |
| 18:00 | Coffee break | Break | 20 min | Mon–Fri | — |
| 18:20 | Office work | Work | 70 min | Mon–Fri | Last stretch |
| 19:30 | Work OFF | Work | 15 min | Mon–Fri | Close the laptop. Actually close it. |
| 20:00 | ByWitness Hour | ByWitness | 60 min | Every day | One tiny step for your future self. |
| 21:00 | Dinner | Break | 45 min | Every day | — |
| 21:45 | Evening walk | Exercise | 20 min | Every day | — |
| 22:15 | Recharge | Recharge | 45 min | Every day | Music, a book, or paint. No screens that ask for anything. |
| 23:00 | Medicine | Medicine | 5 min | Every day | Night dose |
| 00:15 | Wind down | Sleep | 15 min | Every day | — |
| 00:30 | Sleep | Sleep | 15 min | Every day | Tomorrow-you needs you. |

All default blocks begin enabled.

### Default reminders

All default reminders repeat every day and begin enabled:

| Time | Message |
|---|---|
| 08:45 | Good morning. Feet on floor. ☀️ |
| 09:00 | 20 minutes. Let’s get stronger. 💪 |
| 09:45 | Give those eyes a break 👁️ |
| 14:00 | Lunch time. Please leave the laptop. 🍱 |
| 14:45 | Medicine after lunch 💊 |
| 18:00 | Coffee o’clock ☕ |
| 19:30 | WORK IS DONE. Close the laptop. 🔴 |
| 20:00 | ByWitness time. Future-you is watching 👀 |
| 21:00 | Dinner. You are not a productivity machine. 🍝 |
| 22:15 | Recharge: music, a book, or paint 🎧 |
| 23:00 | Night medicine 💊 |
| 23:45 | Tiny reset before bed 🌙 |
| 00:15 | Start winding down. |
| 00:30 | Bedtime. Tomorrow-you needs you. |

## 10. State migration

Version-1 or unversioned state is migrated to version 2 during load and restore.

Migration behavior:

- Legacy routine category `body` becomes `exercise`.
- Legacy routine/task category `life` becomes `break`.
- Default medicine and recharge routine entries are added if no existing routine title matches them case-insensitively.
- Default medicine and recharge reminders are added if a reminder with the same time does not already exist.
- The state version becomes 2.
- Default top-level state is shallow-merged with saved state.
- Settings receive an additional nested merge, so missing settings inherit current defaults.

There is no migration path beyond version 2 yet.

## 11. Global quick-add flow

The floating plus button opens “What do you want to add?” with five choices:

1. Task → task editor.
2. ByWitness idea → content editor.
3. Reminder → reminder editor.
4. Timetable block → routine editor.
5. Note → journal note sheet.

After choosing a type, the chooser closes and the relevant bottom sheet opens. Completing or closing that sheet exits the full quick-add flow.

### Quick note behavior

A quick note is saved into today's journal:

- If there is no journal, the note becomes the journal.
- If a journal already exists, two newline characters and the new note are appended.
- Existing mood and other check-in fields are preserved.
- `savedAt` is updated.

## 12. Home screen

Home is the default screen and the primary “what should I do now?” experience.

### Header

- Time-sensitive greeting:
  - Before 05:00: “Still up” 🌙
  - 05:00–11:59: “Good morning” ☀️
  - 12:00–16:59: “Good afternoon” 🌤️
  - 17:00–21:59: “Good evening” 🌆
  - 22:00 onward: “Winding down” 🌙
- Displays the profile name.
- Displays a deterministic daily greeting selected from a small phrase bank.
- Profile avatar shows the first character of the current name and opens Settings.
- The clock refreshes every 20 seconds.

### Streak cards

- Exercise streak.
- Personal work streak, which corresponds to ByWitness activity.

### Current/next routine card

The app identifies the most recently started routine block for the selected day, using the before-04:00 end-of-day normalization.

For today:

- If that block is neither done nor skipped, it is shown as “Now.”
- “Let’s go” immediately marks it complete; it is not a timer/start state.
- “Skip” toggles the skipped state and shows a compassionate randomized message.
- Completing a routine block shows confetti for 1.6 seconds and a randomized success toast.
- Completing an Exercise block from Home also marks a body metric:
  - If the title contains “walk” case-insensitively, it marks `walk`.
  - Otherwise it marks `strength`.

The current-block algorithm only compares start times. It does not check the calculated end time, so an unfinished block remains “Now” until another scheduled block starts.

If there is no actionable current block, Home shows the next uncompleted/unskipped item, or an all-clear state.

### Today/yesterday switch

Home can display Today or Yesterday.

- Today shows current/upcoming logic.
- Yesterday shows all scheduled blocks for catch-up.
- Yesterday's checkboxes can retroactively change the routine log.
- The schedule correctly uses yesterday's weekday.

### Progress card

Shows completed routine blocks versus the day's effective total and a circular percentage.

Skipped items are normally removed from the denominator. If every item is skipped, the denominator falls back to the original scheduled-item count rather than becoming zero.

### Coming-up list

For today, up to three later incomplete/unskipped blocks are shown. For yesterday, all blocks are shown. Each row can be marked complete.

### Food for thought

Shows:

- A longer thought from a fixed 25-item library.
- A tiny goal from a fixed 16-item library.

Initial choices are deterministic from the date. Tapping “Another” advances a local counter and then chooses random alternatives. This counter is not persisted.

## 13. Timetable screen

The Timetable screen is a weekly recurring schedule view.

### Day selection

- Monday through Sunday pills.
- Defaults to the current weekday.
- A dot marks the current weekday.
- Selecting a day shows enabled routine blocks assigned to that weekday.

### Schedule list

- Rows are sorted by normalized time.
- Shows start and calculated end time.
- Shows title, category, and category color.
- Highlights the most recently started item as `NOW` when viewing today.
- Only today's rows have completion checkboxes.
- Tapping a row opens the routine editor and changes the global recurring block, not a one-day exception.

### Where the day goes

Durations are totaled by category for the selected weekday and displayed in hours rounded to one decimal place.

### Add/edit routine item

Fields:

- Title, required.
- Start time.
- Duration in minutes, with UI minimum 5 and step 5.
- Category.
- Repeat weekdays.
- Optional note.

New routine items default to 09:00, 30 minutes, weekdays, enabled, and the legacy `life` category. Because `life` is no longer a defined category, it displays with the Break fallback unless the user selects another category. This is a confirmed current inconsistency.

Existing items can be deleted. There is no separate undo.

## 14. Edit Timetable screen

Reached through Settings → Edit timetable.

This screen is the management view for the stored routine array:

- Lists every routine item, including disabled items.
- Shows time, category, recurrence summary, and duration.
- Enables/disables items with a switch.
- Moves an item one position up or down in stored order.
- Opens the same editor used by Timetable and quick add.
- Adds and deletes items.

Stored manual order affects this management list, but regular schedule screens independently sort by time.

## 15. Tracker screen

Tracker has three internal tabs:

1. Exercise
2. Social
3. ByWitness

Its internal selected tab is not persisted; reopening Tracker begins on Exercise.

### 15.1 Exercise tracker

Tracks Strength, Walk, and Stretch independently.

#### Weekly grid

- Shows Monday through Sunday.
- Future dates are disabled.
- Current and past days can be toggled.
- Weekly summary reports Strength against the configured strength goal and Walks against a fixed goal of 7.

#### Today cards

Each activity has a large checkbox and forgiving explanatory text. Marking a previously incomplete activity shows a success toast.

#### Sleep

- Displays the configured sleep target.
- Accepts a manual bedtime.
- “Now” records the current local `HH:MM` and shows a toast.
- The saved value belongs to today's local date.

There is deliberately no weight tracking.

### 15.2 Social tracker

This is a lightweight daily record of posting on a design account.

#### Today flow

If not posted:

1. Choose a platform.
2. Optionally describe the post.
3. Tap “Yes, I posted.”

If posted, the card shows the platform/note and offers Undo.

#### Weekly grid

- Shows Monday through Sunday.
- Future days are disabled.
- Past/current days can be toggled quickly.
- Quick-toggling an empty day records Instagram with an empty note.
- Displays weekly count against `socialPostsPerWeek`.
- Displays a daily posting streak.
- Shows a success panel when the weekly count reaches the goal.

#### History

Shows up to the ten most recent posted-day records, newest first.

### 15.3 ByWitness tracker

This combines published-content goals, focus time, and the content pipeline.

#### Weekly summary

- Published posts this week versus `postsPerWeek`.
- Logged focus hours this week versus `focusHoursPerWeek`.
- Publishing-week streak.

#### Weekly goal list

Renders one slot for each configured post goal. Published content items fill slots in array order. Reaching the goal displays a congratulatory panel and triggers confetti when the count crosses the goal during the mounted session.

#### Focus timer

- Starts an in-memory stopwatch.
- Can pause and resume.
- “Log N min” rounds elapsed seconds to the nearest whole minute, adds it to today's focus total, and resets the stopwatch.
- Elapsed time below roughly 30 seconds rounds to zero and is discarded when logged.
- Manual `+15`, `+30`, and `+60` buttons add focus minutes immediately.
- Reset subtracts today's entire current focus total; the state action clamps the result at zero.

The running/paused timer itself is not persisted and is lost when the ByWitness component unmounts, the page reloads, or the browser discards it. Background-tab timer throttling may also reduce accuracy.

#### Content pipeline

Filters:

- All
- Ideas
- Creating
- Editing
- Ready
- Published

Each card shows stage, type, title, optional planned date, and any entered performance numbers.

Actions:

- “Move →” advances one stage.
- The overflow button opens full editing.
- Content can be added, edited, or deleted.

When “Move →” changes a content item to Published, `publishedAt` is set to the current timestamp. Moving it away from Published through the state action clears `publishedAt`.

Important current inconsistency: changing the Stage field to Published inside the full editor uses a generic update action and does **not** automatically set `publishedAt`. Likewise, creating a new item already set to Published leaves `publishedAt` null. Such an item looks Published in the pipeline but is absent from weekly publication metrics until it is moved through the pipeline action or its data is otherwise corrected.

The saved `link` field is stored but is not rendered as a clickable link in the pipeline.

## 16. Plan screen

Plan is a combined future-planning and historical-record view with Week and Month modes.

### Day plans

Every date can contain small plan items tagged:

- ByWitness
- Recharge
- Other

Plan items can be added with Enter or the Add button, marked done, and deleted. There is no text editing action; changing text requires delete and recreate.

### What happened

For today and past dates, the expanded day shows chips for:

- Routine blocks completed/total.
- Movement.
- ByWitness focus minutes.
- Social posting and platform.
- Published content count.
- Bedtime.
- Eye-break count.
- Journal text, when present.

Future dates hide the retrospective section but still allow planning.

### Week mode

- Defaults to the current week, Monday through Sunday.
- Today begins expanded.
- Previous/next week navigation has no restriction.
- Each collapsed day summarizes activity with emojis, plan completion, and routine percentage.
- Future, empty, and past quiet days receive contextual labels.

The selected expanded date is local component state. Navigating to another week can leave the old date selected but invisible until a day in the new week is clicked.

### Month mode

- Displays a Monday-first calendar grid.
- Previous/next month navigation has no restriction.
- Movement, ByWitness activity/publication, and social posting are represented by colored dots.
- Future dates can be selected for planning.
- Changing month selects the first day of the new month.

Monthly summary cards show:

- Average routine percentage for elapsed days in that month.
- Number of movement days.
- Number of social-posting days.
- ByWitness focus hours.

The calculation also derives a published-content total internally, but the current month summary UI does not display it.

## 17. The List

Reached through Settings → The List.

### Add flow

- The inline field creates a minimal normal-priority task with no category or due date.
- Enter and the Add button submit it.
- The details button opens the full task sheet.

### Full task editor

Fields:

- Title, required.
- Priority.
- Optional category.
- Optional due date.
- Optional notes.

Existing tasks can be edited or deleted.

### Filters

- All: all incomplete tasks.
- Today: incomplete tasks with due date less than or equal to today; this includes overdue tasks.
- Upcoming: incomplete tasks due after today, sorted by due date.
- ByWitness: incomplete tasks with `bywitness` category.
- Personal: incomplete tasks categorized as Me, Break, Exercise, Recharge, Medicine, Eyes, or Sleep.
- Work: incomplete tasks with `work` category.
- Completed: completed tasks, newest completion first.

No-due-date tasks appear in All and category filters but not Today or Upcoming.

All non-completed result sets are finally sorted by priority: Important, Normal, then Low pressure. JavaScript's stable sort preserves prior ordering among equal-priority items.

Completing a task sets `doneAt` and shows a success toast. Reopening it clears `doneAt`.

## 18. Weekly Progress

Reached through Settings → Weekly progress.

The screen uses the current Monday–Sunday week and excludes future dates where appropriate.

### Eight metrics

1. **Routine** — total completed routine blocks versus effective scheduled total through today.
2. **Exercise** — days with the explicit `strength` body flag versus `strengthPerWeek`.
3. **Walking** — days with the explicit `walk` body flag versus a fixed goal of 7.
4. **ByWitness** — content items with status Published and a `publishedAt` timestamp in the week versus `postsPerWeek`.
5. **ByWitness hours** — sum of focus minutes divided by 60, rounded to one decimal, versus `focusHoursPerWeek`.
6. **Sleep** — days whose bedtime is no more than 15 minutes later than the configured target versus 7.
7. **Posted on account** — daily social-post logs versus `socialPostsPerWeek`.
8. **Eye care** — days with at least one short or long eye break versus 7.

The Exercise metric does not count Walk or Stretch unless Strength is also marked. Movement streak logic is broader and counts any of those three.

### Sleep target comparison

For comparison, times before noon are shifted into the post-midnight end of the day. A bedtime counts when it is at or before the target plus a 15-minute grace period.

### Overall percentage

For each metric:

```text
metric ratio = min(1, value / goal)
```

The overall score is the rounded arithmetic mean of all eight ratios multiplied by 100.

Metrics with a goal of zero contribute a ratio of zero rather than being omitted. Therefore setting one or more goals to zero can lower the overall score.

### Strongest and “close” messaging

- Strongest area is the non-zero-goal metric with the highest completion ratio.
- The code sorts metrics from highest to lowest, filters incomplete metrics, and selects the **last** one as “You’re close on.” This is actually the lowest incomplete ratio, not the closest to completion. The label and implementation currently disagree.

### Daily bar chart

- Displays routine completion for the current week.
- Future bars have zero height.
- Non-future bars have a visual minimum height of 6% even when actual completion is 0%, provided they are rendered.

## 19. Streak logic

### Routine-day streak

A day qualifies when at least 60% of its effective routine total is complete.

- Searches backward for at most 400 days.
- Today counts if qualified.
- If today has some progress but is below 60%, today is ignored and the search continues from yesterday.
- If today has zero completed blocks, the streak stops immediately.
- Once a prior day fails, the streak stops.

### Movement-day streak

A date qualifies when any of these is true:

- Strength marked.
- Walk marked.
- Stretch marked.
- Any Exercise-category routine block completed.

Today is treated as a grace day: if today does not qualify, the calculation still checks yesterday. After that, the first non-qualifying day stops the streak. Search limit is 400 days.

### Personal-work/ByWitness day streak

A date qualifies when any of these is true:

- Focus minutes greater than zero.
- A content item has a `publishedAt` timestamp on that date.
- A ByWitness-category routine block is complete.

It uses the same today-grace behavior and 400-day cap as movement.

### Social-post day streak

A date qualifies when `social[date].posted` is true. It uses the same today-grace behavior and 400-day cap.

### Publishing-week streak

A week qualifies when at least one content item has `publishedAt` within that Monday–Sunday week.

- Searches backward for at most 200 weeks.
- The current week is a grace period: if nothing has been published yet, the previous week can still start the streak.
- After leaving the current week, the first empty week stops the streak.

## 20. Eyes screen

Reached through Settings → Eyes.

### 20–20 timer

- Default countdown: 20 minutes (`1200` seconds).
- Start/Pause toggles the running interval.
- When it reaches zero, the app tries to send a browser notification: “Look away 👁️ — Twenty seconds, something far away.”
- A 20-second rest countdown appears.
- The 20-minute countdown resets and continues cycling if it was running.
- Tapping Done ends the visible rest countdown and logs one short break.
- “Break now” resets the 20-minute countdown and opens the 20-second rest state immediately.

The 20-second rest ending by itself does not automatically log a break. Only Done or the explicit logging cards change the eye-break totals.

### Manual logs

- “Eye breaks today” increments short breaks.
- “Longer screen breaks” increments long breaks.
- There is no decrement or reset control in the UI.

### Weekly strip

Shows the combined short + long total for each day of the current week.

The screen explicitly states that it is a break tracker, not treatment or a replacement for eye care.

## 21. Me screen

Reached through Settings → Me.

### Daily check-in

The user can select one mood:

- Good
- Okay
- Meh
- Sad
- Heavy
- Angry
- Overwhelmed

A journal entry is optional. Save is enabled if either a mood is selected or journal text is non-empty.

Saving overwrites today's mood/journal fields and updates `savedAt`. It does not create multiple check-ins per day.

### Recent entries

Shows up to seven previous dates, newest first, excluding today. Entries show mood emoji and up to two visual lines of journal text.

The screen offers supportive copy but no clinical assessment, escalation logic, or external mental-health integration.

## 22. Reminders and notifications

Reached through Settings → Reminders.

### Reminder management

- Add, edit, and delete reminders.
- Set time and message.
- Select repeat weekdays.
- Enable/disable each reminder.
- List is time-sorted with before-04:00 times at the end of the waking day.

### Scheduling model

After state initialization, the app:

1. Gets the current weekday and minutes since local midnight.
2. Selects enabled reminders assigned to today.
3. Schedules `setTimeout` only for times strictly later than now.
4. At the scheduled time, attempts a Web Notification.
5. If that attempt fails or permission is unavailable, shows an in-app toast.
6. Schedules a rescan for about five seconds after local midnight.

Changing any reminder ID, time, enabled flag, or weekday list cancels all current reminder timers and reschedules them.

### Permission behavior

The Reminders screen can request browser notification permission.

- Granted: system notifications can be created while the page context is active.
- Denied: scheduled reminders fall back to in-app toasts.
- Unsupported: same fallback while the app is open.

Notifications use title `Tiny Wins`, the reminder text as body, the 192px icon as icon/badge, and the title as the notification tag.

### Important limitations

- This is `setTimeout` scheduling in the page, not server push, background sync, alarm APIs, or service-worker notification scheduling.
- Timers generally do not survive a closed app, discarded tab, device restart, or browser suspension.
- If the app opens after a reminder's time, that reminder is not delivered late.
- Browser background throttling can delay timers.
- The persisted `settings.notifications` flag is changed by the permission flow and by the Settings toggle, but the scheduling code does **not** check it. Consequently, the “In-app nudges” toggle currently does not disable or enable scheduling; reminders are scheduled regardless. Actual system-notification delivery is governed by browser permission, and fallback toasts occur while the app is open.

## 23. Settings screen

Opened from the Home avatar.

### Navigation hub

- Edit timetable
- The List
- Weekly progress
- Me
- Eyes
- Reminders

### Profile and goals

- Name updates immediately on every input change.
- Goal inputs coerce values to numbers and clamp to zero or higher.
- Sleep target is a time input.

### Theme

Four visual themes update immediately and persist.

### Install state

- Listens for `beforeinstallprompt` and stores the deferred prompt.
- Detects installed/standalone display mode.
- Detects iPhone/iPad/iPod user agents and gives Safari “Add to Home Screen” instructions.
- If the browser exposes an install prompt, displays an Install button and calls it.
- Listens for `appinstalled` to update the UI.

### Data controls

- Export backup.
- Restore backup.
- Start over with confirmation.

## 24. PWA and offline behavior

### Manifest

- Name and short name: Tiny Wins.
- Description: “Don't fix your whole life today. Just do the next thing.”
- Start URL: `./index.html`.
- Scope: `./`.
- Display: standalone.
- Orientation: portrait.
- Background/theme color: `#FBF7F2`.
- Icons: 192px, 512px, and 512px maskable.

### Service-worker registration

On window load, the app registers `./sw.js` when:

- service workers exist; and
- the URL protocol begins with `http` (therefore both `http:` and `https:` pass).

Service workers still require a browser secure context; localhost is treated as secure by modern browsers, while arbitrary plain-HTTP hosts normally are not.

### Cache strategy

Cache name: `tiny-wins-v1`.

Install pre-caches:

- `./`
- `./index.html`
- `./manifest.json`
- `./icon-192.png`
- `./icon-512.png`

The maskable icon is not explicitly pre-cached.

On activation, all caches with other names are deleted and the worker immediately claims clients.

For every GET request:

1. Return a cached response if one exists.
2. Otherwise fetch from the network.
3. Clone and cache the network response asynchronously.
4. If network fetch fails, return cached `./index.html` as a general fallback.

Non-GET requests are ignored by the service worker.

### Offline/deployment caveats

- The cache is cache-first with a fixed version string. Updating the app without changing `tiny-wins-v1` may leave old HTML cached indefinitely.
- In a Vite production build, `index.html` points to a hashed JS asset. Old cached HTML may reference a removed old hash after deployment.
- The JS asset is not in the explicit install list; it becomes cached after the first successful page load.
- The fallback returns HTML for any failed GET, including requests that expected another content type.
- Fetched responses are cached without checking `response.ok`.
- Registering this production-style service worker during Vite development can cache development resources and make changes appear stale. Clearing site data or unregistering the service worker may be required while developing.

## 25. UI and interaction system

### Layout

- Designed as a portrait phone app.
- Main app column has a maximum width of 430px and is centered on larger screens.
- Full viewport height.
- Main content scrolls independently.
- Bottom navigation and quick-add button are fixed.
- Safe-area environment variables support notched mobile devices.

### Components

Logical reusable primitives include:

- Primary/soft/ghost/outline/danger pill buttons.
- Icon buttons.
- Filter chips.
- Labeled input wrappers.
- Text inputs, textareas, and selects.
- Circular and horizontal progress indicators.
- Checkbox circles.
- Switches.
- Empty states.
- Confetti overlay.
- Bottom-sheet modal.
- Header.
- Toast.
- Weekday picker.
- Category picker.

### Motion

- Pop, rise, sheet-up, fade, check-draw, confetti-fall, and breathing animations.
- Pressed controls scale slightly.
- Progress indicators animate changes.
- `prefers-reduced-motion: reduce` reduces animations and transitions to near-zero duration and one iteration.

### Typography and visual language

- System UI sans-serif for body text.
- Rounded system-display stack for headings.
- Warm paper/card surfaces, large rounded corners, low-contrast shadows, and category pastels.
- Tabular numerals for times, durations, counts, and percentages.

## 26. Accessibility behavior

Confirmed positive patterns:

- Icon-only buttons generally have `aria-label` and title text.
- Checkboxes use `role="checkbox"` and `aria-checked`.
- Switches use `role="switch"` and `aria-checked`.
- Tracker tabs use tab roles and `aria-selected`.
- Current bottom navigation uses `aria-current="page"`.
- Expandable days expose `aria-expanded`.
- Bottom sheets use `role="dialog"`, `aria-modal="true"`, and a label.
- Escape closes an open sheet.
- Clicking the backdrop closes a sheet.
- Keyboard Enter submits several quick-entry fields.
- Focus-visible outlines are globally styled.
- Reduced-motion preferences are respected.

Known accessibility gaps or risks:

- Dialogs do not implement a focus trap or explicit focus restoration.
- The viewport sets `maximum-scale=1`, which can hinder user zoom.
- Some color combinations and small 9.5–12px labels may require formal contrast/readability testing.
- Some emoji convey state visually; most, but not necessarily all, have nearby text.
- Dynamic toast messages are not marked as an ARIA live region.
- Confetti is decorative but not explicitly hidden from assistive technology.
- The app has not been tested here with keyboard-only navigation or screen readers.

## 27. Privacy and security characteristics

### What does not leave the device in normal use

- Routine and completion history.
- Tasks and notes.
- Content plans and metrics.
- Mood and journal entries.
- Sleep, movement, focus, social, and eye data.
- Settings.

There is no application backend or telemetry code in the bundle.

### Risks and boundaries

- localStorage is plain text and accessible to scripts running on the same origin.
- Backup JSON is plain text and may contain sensitive personal information.
- There is no passcode, encryption, multi-user isolation, or authentication.
- Clearing browser/site data removes the local record unless a backup exists.
- Different origins, ports, browsers, profiles, and devices have separate storage.
- Restore performs minimal validation and can import internally inconsistent data.
- The content link field is stored as inert text in current views, reducing direct link-based risk but also limiting functionality.
- User strings are rendered through React text interpolation, not inserted as raw HTML, which limits ordinary stored-XSS exposure.

## 28. Important cross-feature relationships

### Routine completion → movement

Completing an Exercise routine from **Home** additionally marks Strength or Walk in the body tracker. Completing the same routine from **Timetable** does not perform this extra body-tracker update.

Movement streaks still recognize any completed Exercise routine, so the streak can advance even if explicit body flags do not.

### Content publication → metrics

Only content with both:

- `status === "published"`; and
- a valid `publishedAt`

counts toward weekly posts and publishing streaks.

### Social tracker versus content pipeline

These are independent:

- Social tracker measures whether the design account posted on a date.
- Content pipeline measures creative pieces and their stages/publication timestamps.

Both can represent the same real-world post, but the user must record each separately if both metrics should change.

### Plan retrospective

The Plan screen is the main read-only aggregation surface. It combines routine, body, focus, social, content, sleep, eyes, mood/journal, and planned items by date.

## 29. Known functional limitations and edge cases

1. **Compiled-only codebase** — future changes must currently edit a large minified bundle or reconstruct proper source modules. This is the biggest maintainability issue.
2. **No URL routing** — refresh always returns Home; screens cannot be deep-linked or bookmarked.
3. **Notifications are foreground timers** — reminders are not reliable background alarms.
4. **Nudges setting is not wired** — `settings.notifications` is displayed and mutated but not checked by the scheduler.
5. **Service-worker cache can become stale** — fixed cache version plus cache-first HTML is unsafe for repeated deployments.
6. **Development service-worker interference** — the worker may cache Vite development resources.
7. **Published editor inconsistency** — selecting Published in the editor does not create `publishedAt`.
8. **New routine default uses legacy category** — new items default to `life`, which visually falls back to Break.
9. **“You’re close on” chooses lowest incomplete ratio** — likely opposite of intended behavior.
10. **Zero goals reduce overall percentage** — zero-goal metrics contribute 0 rather than being excluded.
11. **Current block has no end-time check** — the latest started unfinished block remains current until the next block starts.
12. **Focus timer is ephemeral** — navigation/reload loses elapsed time.
13. **Reminder times already passed are skipped** — there is no catch-up delivery.
14. **Quick social backfill assumes Instagram** — toggling a past day from the weekly grid cannot choose another platform or note.
15. **Routine edit is global** — there are no one-off daily overrides.
16. **Plan items cannot be edited** — only toggle/delete/recreate.
17. **Eye logs cannot be decremented** through the UI.
18. **Routine day notes are unused by UI** even though supported in state.
19. **Content links are not actionable** in displayed cards.
20. **Minimal import validation** can produce runtime errors or misleading metrics from malformed backups.
21. **No tests or lint script** are present for product logic.
22. **No error boundary** is visible around the React app.
23. **No cross-tab synchronization** — storage events are not observed, so two open tabs can overwrite each other's state.
24. **No timezone migration policy** — data uses local calendar keys and epoch timestamps; travel/timezone changes can alter how publication timestamps group into days/weeks.
25. **Week mode expanded date can become stale** after navigating weeks.
26. **All-skipped routine denominator fallback** displays 0 against the original total rather than an empty/no-applicable day.
27. **Focus stopwatch rounding** can discard short sessions.
28. **Browser timer throttling** can affect focus, eye, and reminder timing.
29. **Reset is irreversible inside the app** unless a backup was exported.
30. **No data version validation on import** beyond the shallow migration path.

## 30. Logical module inventory for future source reconstruction

If the bundle is reconstructed into maintainable source, the current behavior naturally separates into:

### Core

- `state/defaultState`
- `state/schema`
- `state/migrations`
- `state/storageAdapters`
- `state/actions`
- `state/StoreProvider`
- `dates/calendar`
- `dates/timeFormatting`
- `metrics/weeklyProgress`
- `metrics/streaks`
- `notifications/scheduler`
- `pwa/installPrompt`

### Screens

- `HomeScreen`
- `TimetableScreen`
- `TrackerScreen`
  - `ExerciseTracker`
  - `SocialTracker`
  - `ByWitnessTracker`
  - `FocusTimer`
  - `ContentPipeline`
- `PlanScreen`
  - `WeekView`
  - `MonthView`
  - `DayPlan`
  - `DayRecord`
- `SettingsScreen`
- `RoutineManagerScreen`
- `TaskListScreen`
- `WeeklyProgressScreen`
- `EyeBreakScreen`
- `CheckInScreen`
- `RemindersScreen`

### Editors and shared UI

- `RoutineEditorSheet`
- `TaskEditorSheet`
- `ContentEditorSheet`
- `ReminderEditorSheet`
- `QuickAddSheet`
- `QuickNoteSheet`
- `BottomSheet`
- `BottomNavigation`
- `Toast`
- `ProgressRing`
- `ProgressBar`
- `Checkbox`
- `Switch`
- `CategoryPicker`
- `WeekdayPicker`
- `EmptyState`
- `Confetti`

### Static configuration

- categories
- priorities
- content types/platforms/stages
- plan tags
- mood options
- themes
- default routine
- default reminders
- motivational copy libraries

## 31. Recommended test inventory

No automated product tests currently exist. A faithful test suite should cover:

### State and persistence

- Default-state creation.
- Every action's add/update/delete/toggle behavior.
- Done/skip mutual exclusion.
- Storage adapter selection and failure fallback.
- Debounced saving.
- Export/restore round trip.
- Invalid backup handling.
- Version-1 migration.
- Reset behavior.

### Date/time

- Monday-first weekday conversion.
- Leap years and month boundaries.
- Local `YYYY-MM-DD` keys.
- Before-04:00 sorting.
- Sleep targets across midnight.
- Current-block selection.
- Midnight reminder rescheduling.
- DST changes and timezone changes.

### Metrics

- Each weekly metric.
- Skipped routine denominator.
- All-skipped days.
- Routine 60% threshold.
- Today grace behavior for streaks.
- Publishing-week streak.
- Zero goals.
- Content with Published status but missing `publishedAt`.

### User flows

- All five quick-add paths.
- Routine/task/content/reminder CRUD.
- Home completion side effects.
- Timetable completion difference.
- Focus timer pause/resume/log/reset.
- Eye timer and manual logging.
- Week/month planning.
- Mood and note appending.
- Social posting and backfill.
- PWA installation UI states.

### PWA

- Install pre-cache completeness.
- Offline first load after warm cache.
- Upgrade from one cache version to another.
- Hashed-asset deployment updates.
- Navigation fallback without returning HTML for unrelated assets.

### Accessibility

- Keyboard traversal.
- Dialog focus trap/restore.
- Screen-reader announcements.
- Color contrast in all themes.
- 200% zoom/reflow.
- Reduced-motion behavior.

## 32. Development workflow

Install and run:

```bash
npm install
npm run dev
```

Build and preview production output:

```bash
npm run build
npm run preview
```

Before debugging unexpected stale UI during development, inspect the browser's Application/Storage panel and unregister the existing service worker or clear site data.

### Current safe-change guidance

Because `index.html` contains generated/minified React and CSS:

- Small metadata, manifest, service-worker, or build changes are straightforward.
- Product-logic edits directly in the bundle are fragile.
- A major next engineering step should be recovering or recreating the original component source, then making the build reproducible from that source.
- Preserve compatibility with the current state schema and storage key when reconstructing, or provide a deliberate migration.

## 33. Summary of the app's operating model

Tiny Wins is one local state object viewed through several focused screens. The recurring routine defines what is expected; daily logs record what happened; specialized trackers record movement, focus, posting, sleep, eyes, and mood; plans attach intentions to dates; weekly/monthly calculations transform those records into gentle feedback. All of it is stored locally, and the PWA layer makes the already-loaded app available offline.

The app is functionally rich despite having no backend. Its most important technical risks are not missing functionality but maintainability and reliability: only a compiled bundle is available, notification scheduling is foreground-only, some settings/actions are inconsistently connected, and the service-worker update strategy can serve stale builds. Any continued development should preserve the privacy-first, low-pressure product voice while reconstructing a modular source tree and adding tests around the date, metric, persistence, and notification rules documented here.
