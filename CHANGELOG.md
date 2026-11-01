# Changelog

Releases of `skiplink/action`. Pin `@v1` to follow every compatible release, or an
exact tag such as `@v1.0.0` to stay put.

## v1.0.0 — 2026-11-01

First public release.

- Scans one or more URLs with the hosted Skiplink API and fails the job on violations
  not in the committed baseline (`fail-on: new`), or on new violations at or above an
  impact (`critical`, `serious`, `moderate`, `minor`), or never.
- Posts one pull-request comment and edits it on every run, plus inline annotations
  and a job summary.
- Outputs `passed`, `new-violations`, `report` and `exit-code`, so an outage (`3`) can
  be handled differently from a regression (`1`).
- All inputs reach the shell through the environment, never by template
  interpolation.
- Runs `@skiplink/cli` 1.0.0 or later through `npx`; pin it with `cli-version`.
