---
name: fix-omb-bug
description: End-to-end fix for a reported OpenMausBot bug — regression test, fix, build on the correct host, live verification, PR. Use whenever Kate describes a bot misbehaving (screenshots not rendering, a coordinate_bots spawn_error, a lost local patch after an update, a bot crashing, wrong config after migration) even if she doesn't name this skill. Do not use for feature requests, refactors, or anything without an observable broken behavior.
---

# Fix an OpenMausBot bug end-to-end

Covers the whole loop for a bug report — root cause, fix, build, deploy, live
proof — without making Kate run commands or babysit it. Stops at named
checkpoints instead of running fully unsupervised: past sessions lost hours to
wrong-host builds, wrong-branch commits, and broken wait loops (see
[[feedback-partial-apk-regresses-other-fixes]], [[gr9-omarchy-migration-plan]]
in memory) — those are judgment calls a test suite can't catch, so a human
checkpoint sits before the expensive step.

## 0. Scope check

Read the bug report back in one sentence and confirm it's actually a bug (a
broken behavior with a before/after), not a feature ask. If it's a feature,
stop and say so — this skill is not for that.

## 1. Host and branch — state before touching code

Determine and say out loud, before any edit:
- **Which host does the fix run on?** Android bug → build/adb on this Mac,
  deploy to Kate's phone. Desktop bug touching Electron/native-control/ydotool
  → GR-9 (Omarchy), per [[gr9-omarchy-migration-plan]] — cross-building from
  the Mac silently fails or produces the wrong binary (bundled CUA staging is
  linux/x64-only). Server/shared-logic bug with no host-specific surface →
  either, prefer the Mac (faster iteration).
- **Which branch?** Fresh off `origin/main` unless the bug is already being
  tracked on an open PR branch — check `gh pr list` for one first. Never
  build a fix on top of an unrelated in-flight branch without saying so.

If genuinely ambiguous, ask Kate rather than guessing — a wrong-host build
wastes the whole cycle.

## 2. Failing regression test first

Write a test that reproduces the bug and fails against current code. No fix
before this exists and is confirmed failing (root-cause discipline, not
symptom-patching — see AI-Brain's CLAUDE.md "Debugging technical bugs").

## 3. Fix, iterate to green

Fix the root cause. Re-run the new test plus the existing suite for the
touched module until both pass. Three failed attempts in a row means stop and
report back, not "one more try" (same rule as AI-Brain's debugging protocol).

## 4. Build on the correct host

Run the actual build for that host (`pnpm build:cua:linux` on GR-9 over SSH,
Gradle `assemblePreview` for Android, etc.). For any step that could run long,
poll real process/job state every 30s (`gh run view`, `adb devices`, a PID
check) — never a bare `pgrep` guess (see [[openmausbot-driver-model-mcp-gotchas]]
on why that's unreliable) — and if it's still running past 5 minutes, say so
instead of going silent.

## 5. Verify live — real proof, not a green build

- Android: `adb install`, then a real screenshot of the fixed behavior
  (`mcp__Claude_Code_iOS_Simulator__control` is for the simulator only — for
  Kate's real device use `adb shell screencap` + pull, as done in this
  session's KAT-32 check).
- GR-9/desktop: use the `verify-omb` skill's canonical fake-engine harness
  (`docs/verification/README.md`) rather than improvising raw API calls or
  driving Kate's live instance.
- A passing test suite is necessary, not sufficient — don't report fixed
  without one of the above.

## 6. Check for upstream-update collisions

If the fix touches a file that tracks an upstream OpenMausBot release (not
Kate's own bot config), check whether the next `git pull`/release would
silently overwrite it. If so, either open an upstream PR (preferred — see
recent precedent: PRs #1903, #1942) or record the local patch in
`wiki/log.md` (AI-Brain) so it survives a future update.

## 7. STOP — checkpoint before anything user-visible

Before opening a PR or pushing to a branch Kate will see, stop and show her:
which host/branch, the diff summary, the regression test, and the live-proof
screenshot/log. Get a go-ahead. This is the one deliberate unsupervised-loop
break — skipping it is the thing that turns a good fix into a surprise commit
on the wrong branch.

## 8. Ship

Commit on the confirmed branch, open a PR (title + summary + test plan, same
shape as #1942), and report back in 3 plain-English sentences: what was
broken, what fixed it, and anything still unresolved. No jargon dump of every
command run.
