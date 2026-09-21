---
name: suunto-gym
description: >
  Personalized strength-training coach. Generates specific progressive-overload
  programs and adapts them on two axes: BEFORE training, using Suunto recovery
  data (HRV, sleep) to gate intensity; AFTER training, using what actually
  happened (weights/reps hit, RPE, PRs, lap-inferred completion from the
  watch) to decide progress/hold/deload per lift. Cross-checks health-skill
  context (injuries, conditions, meds) so programming never contradicts what's
  already known. Data lives in the user's health-skill person folder so
  training history stays unified with their health record. Use when the user
  wants to plan gym sessions, log a workout, check what to train today, or
  review progress. Not for medical advice — persistent pain routes to
  health-skill triage.
---

# Suunto Gym

Specialized strength coach layered on top of the `health-skill` person workspace.
It does NOT duplicate health-skill's generic workout-plan generator — it reuses
health-skill's storage (workouts, PRs) and adds real progressive-overload
programming health-skill doesn't do, plus Suunto recovery gating.

## Adaptation loop — before training vs. after training

Two separate axes, don't conflate them:

- **Before training** (`/suunto-gym today`): Suunto recovery data (HRV, sleep) gates
  *intensity for today's session only*. It never changes `PROGRAM.md` itself
  — a bad-recovery day means "do 70% of today's plan," not "the program is
  wrong."
- **After training** (`/suunto-gym log` → `/suunto-gym review`): actual performance —
  weight/reps hit, RPE if given, PRs, and lap-inferred completion from the
  watch — is what drives real programming changes: progress, hold, or
  deload per lift, written back into `PROGRAM.md`. This is the loop that
  actually adapts the program long-term; recovery gating only ever adapts a
  single day.

## Split

The split lives in `PROGRAM.md`'s frontmatter — read it there, don't assume
one (Upper/Mid/Legs, PPL, whatever the user and Claude agreed on during
setup). Watch Guides are titled by session name from `PROGRAM.md` (e.g.
"PUSH A" or "LEGS"), never by day-of-week — which day maps to which session
varies week to week.

## Watch display

`push_strength_guide` lays out every step to use the watch's small screen
optimally — 3 fields on the steps you glance at mid-lift, 4 on the calm
prep step (all inside the 4-5 the schema supports), ordered by priority (the
schema gives the first field the best placement/biggest size). Numbers
(timers, HR) get the big slots; text is cropped first if the screen is
tight. Live heart rate is on every step.

Session flow: prep → set → rest → set → … → last set → prep → set → … → done.

- **Prep** (before every exercise, including the very first): count-up
  stopwatch → HR → weight/sets (`"60kg 3x10"`) → exercise name; title is
  the exercise counter (`4/7`). This is the time to walk to the station and
  set up the weight, and HR tells you how long to take — so it's always a
  self-paced stopwatch, never a countdown. A lap press starts the exercise.
- **Set**: exercise name → target → HR; title is the set counter (`2/3`).
  What to do comes first; HR is a glance, not the thing being acted on.
- **Rest between sets**: countdown (see below) → HR → `Next: set 3/3`;
  title is the same set counter (`2/3` = rest after set 2), so you always
  know where you are in the exercise.

### Text budget

The tool hard-truncates name/detail text at 54 characters, but that's a
last-resort safety cap, not a target — a name that gets cut there is cut
mid-word, and two long fields shown together push each other off screen
before either field limit is even reached. Write names and details
watch-safe at the source, in `PROGRAM.md` and `plan-week.md`, so nothing
downstream ever relies on the truncation:

- Exercise name: aim for **≤24 characters**. Abbreviate consistently — `BB`
  (barbell), `DB` (dumbbell), `15°` not "15 degrees", drop redundant words
  ("Bench Press 15°" not "Incline Barbell Bench Press at 15 Degrees").
- Detail string: aim for **≤16 characters** — `"60kg 3x10"`, `"bw 3x12/leg"`,
  not a sentence.
- Same budget applies wherever those strings reappear: the prep screen
  before each exercise and the `notification` text `push_strength_guide`
  sends on each set.
- If a name can't be shortened without losing what it means (e.g.
  distinguishing two grip variants), shorten it anyway and keep the full
  name only in `PROGRAM.md`'s prose notes — the watch never needs the long
  form, only Claude reading `PROGRAM.md` does.

### What's configurable

`push_strength_guide` takes two optional params, passed through
`/suunto-gym plan` step 3. Defaults are the recommended choice for most
users — only override when the user explicitly asks:

- **`restMode`** (rest *between sets* only — the prep before and between
  exercises is always a stopwatch) — `"countdown"` (default): counts down
  from `restSec` and auto-advances into the next set with a vibration,
  hands-free. `"stopwatch"`: counts up and advances on a lap press — offer
  this if the user wants to pace set-to-set rest by feel/HR too.
- **`lapGranularity`** — `"perSet"` (default, **recommended for AI
  analysis**): one step per set plus one per rest, so laps bound every
  individual set and rest — this is what makes per-set HR/duration readable
  from the synced workout at all, and what `/suunto-gym log`'s lap
  cross-check (below) depends on. `"perExercise"`: prep plus one step per
  whole exercise instead (all sets folded into the `detail` string, e.g.
  `"60kg 3x10"`), no between-set rests (so `restMode` doesn't apply), like
  `push_workout_guide` — a shorter Guide list to scroll through, but only
  one lap per exercise, so per-set analysis is lost. Offer this only if the
  user explicitly prioritizes a shorter list over per-set data.

## Data model

Everything lives inside the user's health-skill person folder:

```
<health-root>/<person-id>/
├── HEALTH_PROFILE.json        # health-skill owns — injuries, conditions, meds
├── workouts[], personal_records[], workout_plans[]   # inside HEALTH_PROFILE.json,
│                                                        written via care_workspace.py
└── gym/
    ├── PROGRAM.md              # current mesocycle: split per frontmatter, exercises/sets/reps/kg — source of truth
    └── plan-week.md            # this week's sessions: display text plus the sets/restSec
                                 # breakdown per exercise pushed to the watch
```

`care_workspace.py log-workout` and `run-summary` already parse freeform text,
detect PRs, and store into `HEALTH_PROFILE.json`. Use them for logging — don't
reimplement. `PROGRAM.md` is where the actual exercise-by-exercise programming
lives, because health-skill's built-in generator (`workout-plan`) round-robins
a fixed ~40-exercise bank and isn't specific enough for real strength work.

## First run — `/suunto-gym setup`

1. Find the health-skill root. Look for an existing person folder under
   `~/Documents/Projects/Health/` (or wherever the user already keeps it —
   check `~/.claude/skills/health-skill/care-workspace/people/` for prior use).
   If none exists, ask the user where to put it, then run:
   ```
   python3 ~/.claude/skills/health-skill/scripts/care_workspace.py init-project --root <root> --name "<name>"
   ```
2. Read `HEALTH_PROFILE.json` for existing injuries/conditions/meds — do not
   ask the user to repeat what health-skill already knows.
3. Interview only what's missing:
   - Goal: strength / hypertrophy / mix
   - Days per week and session length
   - Equipment: full gym / home / specific machines
   - Experience level, known working weights or 1RMs for main lifts
   - Injuries not already in HEALTH_PROFILE.json
4. Write `gym/PROGRAM.md` — a real mesocycle (4-8 weeks), sessions per the
   split agreed in the interview, with starting weights, sets,
   reps, and a progression rule per lift (e.g. "+2.5kg when all sets hit top
   of rep range for 2 sessions running"). Name exercises watch-safe from the
   start — see "Watch display" → "Text budget" above. Before finalizing any exercise,
   cross-check it against the injuries/conditions in `HEALTH_PROFILE.json`
   using the same broad categories health-skill's own exercise bank uses
   (shoulder, lower back, knee, etc. — see `contra` fields in
   `~/.claude/skills/health-skill/scripts/training.py`'s `EXERCISE_BANK` for
   the taxonomy). If a chosen lift plausibly loads an injured area, swap it
   for an equivalent that doesn't rather than programming around the
   contradiction silently.
5. Register the plan in health-skill's schema too, so it shows up in
   health-skill's own dashboard/reports:
   ```
   python3 ~/.claude/skills/health-skill/scripts/care_workspace.py workout-plan \
     --root <root> --person-id <id> --goal "<goal>" --available "<n>x<min>min per week" \
     --equipment "<list>" --injuries "<list>"
   ```
   `PROGRAM.md` stays the actual source of truth Claude follows session to session.
6. Push all sessions in the split to the watch (see `/suunto-gym plan` below) so they're
   ready immediately.

## `/suunto-gym plan` — weekly refresh, pushes to watch

Run whenever the mesocycle's next week changes (new working weights after
progression, a deload, an exercise swap).

1. Update `PROGRAM.md` with the new numbers/exercises for all sessions in the split.
2. Write `gym/plan-week.md` with, per exercise per session: the display text
   (e.g. `Bench Press 15° — 60kg 3x10`) plus the structured `sets` count and
   `restSec` between sets. Keep the name and the `"60kg 3x10"`-style detail
   within the watch-safe budget ("Watch display" → "Text budget" above) —
   this is the text that actually lands on screen (on set steps and on the
   prep screen before each exercise, where you check the weight while
   walking to the station), so don't let it drift long over a mesocycle.
   `sets`/`restSec` are what let the watch build one step per set and one
   rest step between sets.
3. Call `push_strength_guide` once per session in the split, each as its
   own Guide:
   - title: the session name from `PROGRAM.md` (e.g. `"PUSH A"`) — plain, so
     they're distinguishable at a glance in the watch's Guide list, no day names
   - date: today's date for all of them (they're not day-locked; the user picks
     which one to do)
   - exercises: the list from `plan-week.md` for that session, `{name, detail,
     sets, restSec}` — `detail` is the pre-formatted `"60kg 3x10"` style
     string (weight and sets — it's shown on the prep screen too), `sets` is
     the number of sets, `restSec` is the rest between sets in seconds (the
     countdown length; not used between exercises, where the prep stopwatch
     is self-paced).
   - `restMode`/`lapGranularity`: omit both to get the recommended defaults
     (countdown rest between sets, one lap per set — see "Watch display" →
     "What's configurable" above). Only pass them when the user explicitly
     asks for a self-paced stopwatch between sets or a shorter Guide list
     over per-set data.
4. Tell the user the sessions will appear on the watch after their phone's
   next normal Suunto app sync — no manual pinning needed in testing. If one
   doesn't show up, they can open the Suunto app > their watch > SuuntoPlus
   Guides and pin it manually there.

## `/suunto-gym today`

1. Pull recovery signal via the suunto MCP tools: `get_recovery` and `get_sleep`
   for yesterday/today.
2. Recovery gate:
   - HRV Balance clearly below the user's recent baseline, or sleep score low
     → cut planned intensity 20-30% (drop a set, or reduce load ~10%), tell the
     user why in one line.
   - Normal or good recovery → run the session in `PROGRAM.md` as planned.
3. Show today's session: exercise, sets×reps, target weight (last logged weight
   + progression rule if criteria met, otherwise same weight).
4. Include the session's warmup and cool-down from `gym/warmup-cooldown.md`
   (keyed by session name), plus any standing pre-session check the user has
   set up there. If the user wants it on their phone, send the session +
   warmup/cooldown via the signal MCP's `send_note_to_self`.
5. If the user says an exercise/machine/rack isn't available (gym's busy,
   traveling, home setup missing something): substitute a same-muscle-group
   alternative on the spot — don't just drop the exercise. Log the swap in
   `plan-week.md` for that session so `/suunto-gym review` sees what was actually
   trained, not the original plan.

## `/suunto-gym log`

1. Ask what happened if not already given. If the user gives weights/reps
   without an effort rating, ask how hard the last set felt (RPE 1-10, or
   "easy/moderate/hard/max effort" translated to a number) — it's the
   difference between "hit the reps easily" and "barely made it," which
   changes whether to progress next time.
2. Call:
   ```
   python3 ~/.claude/skills/health-skill/scripts/care_workspace.py log-workout \
     --root <root> --person-id <id> --text "<freeform description, e.g. 'bench 60kg 3x8 RPE 7'>"
   ```
   This parses exercises/weights/sets/reps/RPE and detects PRs automatically.
3. If the user mentions pain (not just soreness) that's new or recurring,
   don't just swap the exercise — flag it into health-skill's review flow
   (this is a medical signal, not a training decision):
   ```
   python3 ~/.claude/skills/health-skill/scripts/care_workspace.py add-note --root <root> --person-id <id> --text "<pain description>"
   ```
   Still swap/regress the exercise in `PROGRAM.md` for next session as a
   precaution, but say explicitly that persistent pain should go to a
   professional — don't diagnose it.

## `/suunto-gym review`

Weekly. Read the logged workouts (`run-summary` for cardio trend deltas;
scan `HEALTH_PROFILE.json` workouts for lifts, including `rpe` per exercise
when present) plus `PROGRAM.md`'s progression rules:
- Which lifts are progressing, which have stalled 2+ sessions
- **Autoregulate using RPE when it's available**, not just weight/reps:
  reps hit + RPE ≤7 → progress. Reps hit but RPE 9-10 (grinding) → hold, even
  though the rep target was technically met. Reps missed → hold or regress.
  No RPE logged → fall back to reps-only (the old rule still applies).
- Volume trend (sets per muscle group per week)
- Suunto recovery trend for the week (average HRV/sleep vs. prior week) —
  for context on *why* a lift stalled, not as the progression trigger itself
  (that's performance data's job; recovery data's job is the daily gate)
- Concrete next-week adjustment: hold, progress, or deload — write it into
  `PROGRAM.md` and summarize in one short paragraph, not a wall of stats.

## Watch sync

Plan → watch: `push_strength_guide` (see `/suunto-gym plan` above), one Guide per
session: a prep stopwatch before each exercise, then one watch step per set
with a rest step between sets.

Everything below describes the default `restMode: "countdown"` +
`lapGranularity: "perSet"` combo ("Watch display" → "What's configurable"
above). With `restMode: "stopwatch"` the lap pattern is identical — the
rest-end lap just comes from the user's press instead of automatically. With
`"perExercise"` there's only one lap for the prep and one for the exercise,
so skip step 2 below and just log against the whole exercise.

Watch → Claude: no explicit done/skip is returned by the Guide API — only
laps land in the synced workout's data, and every prep, set and rest is its
own lap:
- Prep ends on a lap press (the user is set up) → marks prep-end/set-1-start.
- A set ends on a lap press (the reps are done) → marks set-end/rest-start,
  or, after an exercise's last set, set-end/next-prep-start.
- A countdown rest ends on its own, and the next set step logs an automatic
  lap the instant it begins → marks rest-end/set-start.
- Net: an exercise with N sets is 2N laps in order — prep, set 1, rest 1,
  set 2, …, rest N−1, set N.

`/suunto-gym log` reconstructs what happened from this stream:
1. Group the synced workout's laps by exercise in plan order, consuming
   2 × `sets` laps per exercise (from that session's `plan-week.md`).
2. Within each exercise's group, the first lap is the prep (walk to the
   station and set up the weight — its duration and the HR during it show
   how much rest the user took between exercises), then set and rest
   alternate — pair them up in that order to reconstruct actual per-set and
   per-rest duration, and use `get_workout_fit`'s HR samples windowed by lap
   timestamps to read per-set effort and per-rest recovery.
3. If the lap count for an exercise doesn't come to 2 × `sets` (skipped
   exercise, extra laps, watch not synced mid-session), don't guess — ask
   the user to confirm what was actually done for that exercise rather than
   assuming the lap stream matches the plan.

## Boundaries

- Training programming only. Anything medical (persistent pain, dizziness,
  injury requiring diagnosis) routes to health-skill's `triage` command —
  don't give medical advice here.
- Never invent 1RMs or weights the user hasn't reported — ask.
- Progression is conservative by default: small increments, deload before
  the user burns out. This is not a "no pain no gain" coach.
