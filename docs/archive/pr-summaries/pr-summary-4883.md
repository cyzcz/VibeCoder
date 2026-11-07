## Summary

`normaliseLogExcerpt` in `worker/deno/lib/auto_fix_attempt_tracker.ts` now
replaces compound minute and hour duration tokens (`2m14s`, `3m 45.5s`,
`1h02m03s`) with `<dur>` alongside millisecond and second durations (`12ms`,
`41s`). `computeFailureSignature` therefore produces the same signature when two
Deno test runs fail with the same excerpt except for a minute-length duration on
the summary line. Closes #4883.

Version bump: `VERSION` is not changed by this PR.

## Spec

### Intent and Rationale

- `deno test` prints minute-length durations (`(2m14s)`) on its summary line
  whenever a test run takes at least one minute, and the log excerpt keeps the
  log tail.
- `normaliseLogExcerpt` previously matched only `/\b\d+(?:\.\d+)?\s?m?s\b/gi`,
  leaving `2m14s` intact and changing the failure signature on every push.

### Essential Design Decisions

- The duration pattern is extended to
  `/\b(?:\d+h\s?)?(?:\d+m\s?)?\d+(?:\.\d+)?\s?m?s\b/gi`, which matches optional
  hour (`\d+h\s?`) and minute (`\d+m\s?`) prefixes before the seconds or
  milliseconds term while still requiring a trailing `s` or `ms` unit.
- Existing `ms` and `s` duration matching is preserved unchanged.

### Undiscoverable Facts

- None.

## Evidence

Backend-only change; no UI.

- `deno test tests/auto_fix_attempt_tracker_test.ts` in `worker/deno`: 13
  passed, 0 failed.
- `docs/workflows/ci-fix.md` (section
  `Attempt-cap and deferral state live on
  the pull request`, lines 340–375)
  read; it describes `computeFailureSignature` at the workflow level without
  listing regex tokens, so no documentation file needed updating.

## Test Plan

- `worker/deno/tests/auto_fix_attempt_tracker_test.ts`:
  - `auto_fix_attempt_tracker - normalisation strips minute and hour durations`:
    verifies `2m14s` vs `2m31s`, `1h02m03s` vs `2h15m49s`, `3m 45.5s` vs
    `9m 01.2s`, and `12ms` vs `41s` normalise to `<dur>`.
  - `auto_fix_attempt_tracker - signature is stable across Deno minute-length test durations`:
    verifies `computeFailureSignature` returns identical signatures for
    `FAILED | 17451 passed | 2 failed (2m14s)` and
    `FAILED | 17451 passed | 2 failed (2m31s)`.
- No second copy of `normaliseLogExcerpt` exists in the repository (checked via
  grep).
