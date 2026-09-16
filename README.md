# Suunto Gym Skill

A Claude Code skill that plans real strength-training programs and pushes
them straight to your Suunto watch. It adapts on two axes: before training,
your recovery data (HRV, sleep) gates how hard today should be; after
training, what actually happened — weights and reps hit, effort (RPE), PRs,
even lap-inferred completion from the watch — decides whether next week
progresses, holds, or deloads.

Not a generic workout generator — it writes real progressive-overload
programming (specific exercises, weights, sets, reps, and a progression rule
per lift), gates intensity against your HRV/sleep, and keeps training history
unified with your broader health record via
[health-skill](https://github.com/googlarz/health-skill).

## What it needs

- [suunto-mcp](https://github.com/googlarz/suunto-mcp) — connects your Suunto
  watch to Claude, and provides `push_strength_guide` (puts your plan on your
  wrist, one watch step per set plus rest) and `get_recovery`/`get_sleep`
  (the recovery-gating data)
- [health-skill](https://github.com/googlarz/health-skill) — the person
  workspace this skill stores its data in, so training stays unified with
  injuries, conditions, and the rest of your health record

## Install

```bash
git clone https://github.com/googlarz/suunto-gym.git ~/.claude/skills/suunto-gym
```

Then in Claude Code:

```
/suunto-gym setup
```

## Commands

| Command | What it does |
|---------|-------------|
| `/suunto-gym setup` | Interviews you once (goal, equipment, experience, injuries not already in health-skill), writes a real split program |
| `/suunto-gym plan` | Refreshes the week and pushes every session in the split to your watch |
| `/suunto-gym today` | Checks your recovery, gates intensity if it's poor, shows today's session |
| `/suunto-gym log` | Logs what happened including RPE, detects PRs, flags recurring pain to health-skill instead of guessing at it |
| `/suunto-gym review` | Weekly progression check — which lifts are moving, which stalled, next week's adjustment |

## Dashboard

A demo of what a `/suunto-gym dashboard` command would show once real data exists.
Sample data throughout, clearly marked — not yet a real command, needs
`/suunto-gym setup` to have actually run first.

**Up next** leads with today's session and the recovery-gate decision,
**logged** sessions show deviations (swaps, skips) inline instead of burying
them in a full exercise list, and **eight-week trends** cover progression
per lift with RPE, bodyweight and active constraints from health-skill, and
recovery vs. the daily session call:

![Training board dashboard](docs/dashboard.png)

Full interactive version (fonts, animation, collapsible cards): download
[`docs/dashboard-demo.html`](docs/dashboard-demo.html) and open it locally.

## Watch sync

Plan → watch is a real, working loop through the SuuntoPlus Guide API — see
suunto-mcp's `push_strength_guide` docs for exactly how, and its one honest
limitation (no live push; delivery rides your phone's normal Suunto sync).
Each set and rest period is its own step on the watch, so Watch → Claude
comes back via a per-set/per-rest lap pattern in the synced workout, which
`/suunto-gym log` cross-checks against what was planned to reconstruct
per-set duration and effort.

## Boundaries

Training programming only. Persistent pain, injury, or anything medical
routes to health-skill's triage — this skill never plays physio.

## License

MIT
