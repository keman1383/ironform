# IRONFORM — Handoff v13

## App Overview
Single-file PWA. React (UMD/Babel, no build step), Firebase Auth + Firestore. Mobile-first (iOS home screen PWA). Deployed by replacing the hosted `.html` file.

**Current version:** 2.9.1  
**Firebase project:** `trainerdata-c5e64` — credentials hardcoded. Keep private.

---

## Working Style
- Targeted `str_replace` for narrow changes; confirm scope before non-trivial work
- User deploys by replacing the hosted file; comfortable with short manual edits

---

## Screens
`home` → `preset_preview` → `workout` → `summary`  
Also: `custom_builder`, `goals`, `history`, `profile`, `exercise_db`

---

## Architecture

### Key State
| State | Purpose |
|---|---|
| `activeWorkout` | `{ date, exercises, setRepsMap, intensityMode, templateLabel, templateId }` |
| `completedSets` | `{ "${exId}_${setIdx}": true }` |
| `perSetData` | `{ "${exId}_${setIdx}": { weight, reps } }` |
| `skippedExIds` | `Set<exId>` — reset on launch/cancel |
| `deferredExIds` | "come back later" queue |
| `midWorkoutAddedExIds` | Snapshotted in `finishWorkoutWithState` before state clears |
| `restTimerDuration` | Default 60s; adjustable in 15s increments on preset preview |
| `builtinOverrides` | `{ [exId]: {...} }` from `profile.builtinExOverrides` |
| `isFinishingRef` | `useRef(false)` — guards double-call of `finishWorkoutWithState` |
| `dumbbellModal` | `bool` — controls Push/Pull bottom sheet on home screen |

### Key Functions
- **`computeWeightForEx(exId, def, iModeKey)`** — 1RM×pct → history progression → default. Single path used everywhere.
- **`computeWeightForExDB(exId, def)`** — calls `computeWeightForEx`, divides by 3, rounds to nearest 5, floors at 5. Used only for dumbbell workouts.
- **`getEffective1RM`** — fresh stored (≤15d) → Epley → expired stored
- **`calcEpley1RM`** — best `weight × (1 + reps/30)` across history; skips `ex.skipped === true`
- **`safeProgressWeight(current, goal, weeksSince, unit)`** — per-week cap; units: `lbs:10, reps:2, secs:15`
- **`finishWorkoutWithState`** — builds record, stamps `skipped:true`, detects PRs and matched goals; guarded by `isFinishingRef`
- **`toggleSet`** — mark/unmark; auto-advance, rest timer, or auto-finish; deferred auto-clear in `setTimeout(0)`
- **`adjustWeightGlobal`** — snapshots exercise + `completedSets` before updater (see Known Issues)
- **`addExToActiveWorkout`** — two chained functional updaters; second navigates to new exercise
- **`removeSet(exId, si)`** — re-indexes `completedSets` and `perSetData`; hidden when 1 set remains
- **`swapExercise`** — migrates `completedSets`/`perSetData` key prefixes `oldExId_N` → `newExId_N`
- **`launchDumbbellWorkout(tmplId)`** — fixed 3×10, 60s rest, no core injection, no preset preview. Uses `computeWeightForExDB` for all weights. Sets `intensityMode:"grind"` for history label consistency.

### Firebase Collections
`users/{uid}` — profile, templateExtras, builtinExOverrides  
`users/{uid}/workouts/{id}` — date, exercises w/ setData, intensityMode, templateLabel  
`users/{uid}/customExercises/{fid}`  
`users/{uid}/liftGoals/{id}` — exId, targetWeight  
`users/{uid}/oneRepMaxes/{exId}` — weight, updatedAt  
`fbDel1RM` is inline via `db.collection(...).doc(exId).delete()`.

### Components
`App` (monolithic), `MidWorkoutAddModal` (extracted — fixes iOS keyboard bug), `ModalStrict`, `Modal`, `MiniChart`, `CalendarView`, `Confetti`, `IntensityModePicker`, `Stepper`, `NavBar`

---

## Exercise System
- 51 built-ins across 7 categories + custom exercises in Firestore
- `allExercises` = `BUILTIN_EXERCISES` + `builtinOverrides` + `customExs` (merged at render)
- Equipment types: `barbell`, `dumbbell`, `cable`, `assisted`, `machine`, `bodyweight`
- `bigIncrement`: barbell/assisted → 2.5; dumbbell/cable → 5; machine/bodyweight → `ex.increment`
- **Assisted logic is inverted** — lower weight = PR/progress. Applied in `computeWeightForEx`, `finishWorkoutWithState`, summary, and goal achievement checks.
- **`unit:"secs"`** — used by Plank. Hides weight adjuster in set row, labels counter "secs", stores duration in the `reps` field. Excluded from volume calc. `safeProgressWeight` caps at 15s/week.
- **`CORE_EX_IDS`** — 7 core exercise IDs; one is randomly injected into every built-in workout at launch (not shown on preset preview — known limitation). Not injected into dumbbell workouts.

### Built-in Exercises of Note
- **Plank** — Core, bodyweight, `unit:"secs"`, default 30s
- **Pull-Up** — Back, bodyweight, reps
- **Pull-Up (Assisted)** — Back, assisted, lbs (inverted logic)
- **Incline Press** — Chest, barbell (formerly "Incline Dumbbell Press")

### Dumbbell-Only Exercises (added v2.9.1)
These are flagged `equipmentType:"dumbbell"` and only appear in dumbbell templates. They are visible in the exercise DB.

| ID | Name |
|---|---|
| `db_chest_press` | DB Chest Press |
| `db_incline_press` | DB Incline Press |
| `db_fly` | DB Fly |
| `db_tricep_kick` | DB Tricep Kickback |
| `db_pullover` | DB Pullover |
| `db_bent_row` | DB Bent-Over Row |
| `db_single_arm_row` | DB Single Arm Row |
| `db_concentration` | DB Concentration Curl |

Reused existing dumbbell exercises: `lateral_raise`, `oh_db_press`, `hammer_curl`, `assisted_dip`.

---

## Workout Templates
Regular templates (appear as home tiles, get core injection):
- `push` — Push Day
- `pull` — Pull Day
- `legs` — Leg Day

Dumbbell templates (flagged `dumbbell:true` — excluded from home tiles, excluded from core injection, launched via bottom sheet only):
- `db_push` — Dumbbell Push: `db_chest_press`, `db_incline_press`, `db_fly`, `lateral_raise`, `oh_db_press`, `db_tricep_kick`, `assisted_dip`
- `db_pull` — Dumbbell Pull: `db_pullover`, `db_bent_row`, `db_single_arm_row`, `db_concentration`, `hammer_curl`

---

## Intensity Modes
| Mode | % 1RM | Sets | Reps |
|---|---|---|---|
| Recovery | 65% | 4 | 5 |
| Building | 75% | 3 | 10 |
| Pushing | 90% | 2 | 2 |
| Custom | — | user | user |

Dumbbell workouts bypass intensity mode entirely (fixed 3×10, 60s rest). Stored as `intensityMode:"grind"` in workout record.

---

## Quirks & Gotchas
- **No build step** — single `<script type="text/babel">`, no imports.
- **`circleBackToDeferred` must be declared before `skipToNext`** — const arrows don't hoist; breaking this = blank workout screen.
- **`defaultWeight` is a seed, not a floor** — floor in `computeWeightForEx` is `Math.max(scaled, 5)`.
- **Profile card headers** — use `card-header` / `card-header-navy` CSS classes with `margin:-16px -16px 12px`. Cards must NOT have `padding:0` or headers bleed outside the boundary.
- **Global weight display box** — `width:148, overflow:"hidden"` inline. Do not revert to `minWidth`.
- **Per-set weight display box** — `width:64` inline, `fontSize:13`.
- **Pre-workout extras → post-workout prompt** — `launchWorkout` seeds `midWorkoutAddedExIds` with extras not already in `templateExtras`, so the "Keep these?" prompt fires for preview-screen additions too.
- **1RM UI** on Profile covers Bench, Squat, Lat Pull Down only. All others via inline 🎯 + Epley.
- **Goal progress for assisted exercises** — uses inverted pct: `goal.targetWeight / current` instead of `current / goal.targetWeight`.
- **Dumbbell template `dumbbell:true` flag** — used to exclude from home tile loop and core injection. Don't remove it.

---

## Known Issues
1. **Global weight adjuster overwrites completed sets** — snapshot timing issue; needs fresh approach.
2. **Barbell input won't clear to empty** — fix: store `barbellWeight` as string in draft, parse on save.
3. **Inline 🎯 goal input dismisses iOS keyboard on each keystroke** — goal input is inside workout re-render tree; fix: extract to stable named component. Discuss before touching.

---

## Dead Ends
- **Rest timer audio/haptics** — iOS Safari requires AudioContext in synchronous user-gesture call stack; `setInterval` ticks don't qualify. `navigator.vibrate` not on iOS. Only fix: native wrapper (incompatible with single-file format).

---

## Future Features
- Show random ab exercise on preset preview
- Create new exercise mid-workout ("Create & Add" path in `MidWorkoutAddModal`)
- Goal achievement → prompt for new goal
- Cardio exercise type (duration logging)
- Plateau detection
- Media Session API for rest timer controls

---

## How to Resume
> "I'm continuing work on a fitness app called IRONFORM. The handoff doc and current source are attached. Please read both before we proceed."
