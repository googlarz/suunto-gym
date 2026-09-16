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

Three sessions per week: **Upper**, **Mid** (core/posterior chain/accessories),
**Legs** — not day-of-week labels. 12–16 exercises per session. User picks
which one to do on a given day; the watch Guide list is titled by split name
("UPPER", "MID", "LEGS"), not Mon/Wed/Fri, since which day maps to which
session varies week to week.

## Data model

Everything lives inside the user's health-skill person folder:

```
<health-root>/<person-id>/
├── HEALTH_PROFILE.json        # health-skill owns — injuries, conditions, meds
├── workouts[], personal_records[], workout_plans[]   # inside HEALTH_PROFILE.json,
│                                                        written via care_workspace.py
└── gym/
    ├── PROGRAM.md              # current mesocycle: Upper/Mid/Legs, exercises/sets/reps/kg — source of truth
    └── plan-week.md            # this week's 3 sessions, exact text pushed to the watch
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
4. Write `gym/PROGRAM.md` — a real mesocycle (4-8 weeks), three sessions
   (Upper / Mid / Legs), 12-16 exercises each, with starting weights, sets,
   reps, and a progression rule per lift (e.g. "+2.5kg when all sets hit top
   of rep range for 2 sessions running"). Before finalizing any exercise,
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
6. Push all three sessions to the watch (see `/suunto-gym plan` below) so they're
   ready immediately.

## `/suunto-gym plan` — weekly refresh, pushes to watch

Run whenever the mesocycle's next week changes (new working weights after
progression, a deload, an exercise swap).

1. Update `PROGRAM.md` with the new numbers/exercises for all three sessions.
2. Write `gym/plan-week.md` with the exact display text per exercise per
   session — one line each, e.g. `Bench Press 15° — 60kg 3x10`. This is
   exactly what shows on the watch, so keep it short and unambiguous.
3. Call `push_workout_guide` **three times**, once per session, each as its
   own Guide:
   - title: `"UPPER"`, `"MID"`, `"LEGS"` (plain, so they're distinguishable
     at a glance in the watch's Guide list — no day names)
   - date: today's date for all three (they're not day-locked; the user picks
     which one to do)
   - exercises: the list from `plan-week.md` for that session, `{name, detail}`
     pairs — `detail` is the pre-formatted `"60kg 3x10"` style string
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
4. If the user says an exercise/machine/rack isn't available (gym's busy,
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

Plan → watch: `push_workout_guide` (see `/suunto-gym plan` above), one Guide per
session. Watch → Claude: no explicit done/skip is returned by the Guide API —
only lap-button presses (`manualLap`), one per exercise step, land in the
synced workout's data. `/suunto-gym log` cross-checks the lap count/timing against
that session's `plan-week.md` to infer what was actually completed; ask the
user to confirm rather than assuming a lap always means "done as written."

## Boundaries

- Training programming only. Anything medical (persistent pain, dizziness,
  injury requiring diagnosis) routes to health-skill's `triage` command —
  don't give medical advice here.
- Never invent 1RMs or weights the user hasn't reported — ask.
- Progression is conservative by default: small increments, deload before
  the user burns out. This is not a "no pain no gain" coach.
