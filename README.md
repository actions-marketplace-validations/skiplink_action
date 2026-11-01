# Skiplink Accessibility Gate

Fail the pull request on **new** accessibility violations — not on the ones that were
already there.

Point any scanner at a site that already exists and you get a wall of red: the
[average home page has 56 detectable WCAG failures](https://webaim.org/projects/million/).
Nobody fixes 56 issues to unblock a deploy, so the check gets deleted within a week.
This Action runs [axe-core](https://github.com/dequelabs/axe-core) against your URLs,
compares the result with a baseline committed to your repository, and fails only on
what is new. Turning it on costs nothing on day one, and the next regression fails the
build.

- **One step of YAML.** No dashboard to log into.
- **A pull-request comment that updates itself** instead of piling up, plus inline
  annotations and a job summary.
- **Baselines keyed on URL path, not hostname**, so a preview deployment with a fresh
  hostname on every pull request keeps matching.
- **Pinned engine.** The axe-core version ships with every result, so a change in
  findings can always be attributed.

```yaml
# .github/workflows/accessibility.yml
name: Accessibility
on: pull_request

jobs:
  a11y:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write        # for the report comment
    steps:
      - uses: actions/checkout@v4
      - uses: skiplink/action@v1
        with:
          urls: https://staging.example.com
          baseline: .skiplink/baseline.json
          token: ${{ secrets.SKIPLINK_TOKEN }}
```

## Setup

1. **Get an API key.** Create a free account at
   [skiplink.dev/signup](https://skiplink.dev/signup/) — 500 scans a month, no card.
2. **Store it as a secret.** Repository → Settings → Secrets and variables → Actions →
   New repository secret, named `SKIPLINK_TOKEN`.
3. **Record a baseline** of what is already there, once, from your machine:

   ```bash
   export SKIPLINK_TOKEN=skl_live_…
   npx @skiplink/cli scan https://staging.example.com \
     --baseline .skiplink/baseline.json --update-baseline
   git add .skiplink/baseline.json
   git commit -m "Record the accessibility baseline"
   ```

   Read the diff before committing. Every entry is a violation the gate will now
   ignore, and reviewing that list is the point of keeping it in the repository.

4. **Add the workflow above.** Without `baseline:` the gate fails on every violation it
   finds, which on an existing site means every build.

## Inputs

| Input | Default | Description |
|---|---|---|
| `urls` | — | **Required.** URLs to scan, one per line or space-separated. Must be reachable from the public internet. |
| `token` | — | **Required.** Your Skiplink API key. Pass it from a secret. |
| `baseline` | — | Path to the committed baseline file. Violations listed in it do not fail the build. |
| `fail-on` | `new` | `new` fails on anything not in the baseline. `critical`, `serious`, `moderate` or `minor` fail on new violations at or above that impact. `never` reports without failing. |
| `standard` | `wcag22aa` | `wcag22aa`, `wcag21aa`, `wcag21a`, `wcag2aa`, `wcag2a`, `section508`, or `all` (includes best-practice rules). |
| `wait-for` | — | CSS selector to wait for before scanning, for pages that render late. |
| `comment` | `true` | Post the report as a pull-request comment, editing the same comment on every run. Needs `pull-requests: write`. |
| `cli-version` | `latest` | Version of [`@skiplink/cli`](https://www.npmjs.com/package/@skiplink/cli) to run. Pin it for reproducible builds. |
| `api` | — | API base URL. Leave unset for the hosted API. |

## Outputs

| Output | Description |
|---|---|
| `passed` | `true` when the gate passed. |
| `new-violations` | Number of violations not in the baseline. |
| `report` | Path to the Markdown report on the runner. |
| `exit-code` | `0` passed · `1` gate failed · `2` misconfigured · `3` could not complete. |

A scan that could not run — the site was down, the URL was refused — fails the step
with "could not complete", never as a passing gate. `exit-code` lets a workflow treat
an outage differently from a regression.

## Examples

### Several pages

```yaml
      - uses: skiplink/action@v1
        with:
          urls: |
            https://staging.example.com/
            https://staging.example.com/pricing
            https://staging.example.com/checkout
          baseline: .skiplink/baseline.json
          token: ${{ secrets.SKIPLINK_TOKEN }}
```

### Preview deployments

Baseline entries are keyed by path, so a baseline recorded against one preview URL
keeps working on the next one:

```yaml
      - id: deploy
        run: echo "url=$(./deploy-preview.sh)" >> "$GITHUB_OUTPUT"
      - uses: skiplink/action@v1
        with:
          urls: ${{ steps.deploy.outputs.url }}
          baseline: .skiplink/baseline.json
          token: ${{ secrets.SKIPLINK_TOKEN }}
```

### Block only on serious problems

```yaml
      - uses: skiplink/action@v1
        with:
          urls: https://staging.example.com
          baseline: .skiplink/baseline.json
          fail-on: serious
          token: ${{ secrets.SKIPLINK_TOKEN }}
```

### Report without blocking

While a team is getting used to it:

```yaml
      - uses: skiplink/action@v1
        with:
          urls: https://staging.example.com
          fail-on: never
          token: ${{ secrets.SKIPLINK_TOKEN }}
```

The report still appears on the pull request and in the job summary; nothing goes red.

### Check production on a schedule

```yaml
on:
  schedule:
    - cron: '0 6 * * 1-5'

jobs:
  a11y:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: skiplink/action@v1
        with:
          urls: https://www.example.com
          baseline: .skiplink/baseline.json
          comment: false
          token: ${{ secrets.SKIPLINK_TOKEN }}
```

### Use the result in a later step

```yaml
      - uses: skiplink/action@v1
        id: a11y
        continue-on-error: true
        with:
          urls: https://staging.example.com
          baseline: .skiplink/baseline.json
          token: ${{ secrets.SKIPLINK_TOKEN }}
      - if: steps.a11y.outputs.exit-code == '3'
        run: echo "Scanner unavailable — retry later rather than blocking the merge."
```

## Pages this cannot reach

The hosted scanner fetches each URL from our infrastructure, so it only sees pages on
the public internet that load without signing in. For pages behind a login, on
`localhost`, or part-way through a flow, run the same engine inside your existing tests
with [`@skiplink/playwright`](https://www.npmjs.com/package/@skiplink/playwright) or
[`@skiplink/cypress`](https://www.npmjs.com/package/@skiplink/cypress). Both write to the
same baseline file.

## Notes

- **Node 20 or later** must be on the runner. `ubuntu-latest`, `macos-latest` and
  `windows-latest` already have it; on self-hosted runners add `actions/setup-node`.
- **Pull requests from forks** get a read-only token, so the comment is skipped. The
  job summary still carries the full report.
- This is a composite Action that runs the `@skiplink/cli` npm package. Everything it
  executes is readable in [`action.yml`](action.yml) — there is no bundled `dist/`.

## What a passing run means

Automated testing detects a subset of WCAG success criteria — published estimates
range from roughly 13% to 57% depending on how coverage is counted. A passing run means
no automatically detectable violations were found. It does not mean a page conforms to
WCAG, nor that it complies with the ADA, Section 508 or the European Accessibility Act.
Manual testing by a qualified reviewer is required for a conformance claim.
[What a scan does and does not tell you](https://skiplink.dev/legal/accuracy/).

## Links

[Documentation](https://skiplink.dev/docs/) ·
[CLI](https://www.npmjs.com/package/@skiplink/cli) ·
[Pricing](https://skiplink.dev/pricing/) ·
[Status](https://skiplink.dev/status/) ·
[Report an issue](https://github.com/skiplink/action/issues) ·
[Security](https://skiplink.dev/legal/security/)

MIT licensed. axe-core is © Deque Systems, Inc. and used under MPL-2.0; AXE and
AXE-CORE are trademarks of Deque Systems, Inc. Use of axe-core does not imply
endorsement by Deque.
