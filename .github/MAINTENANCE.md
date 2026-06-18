# status-page maintenance (SEC-44)

This repo is an [Upptime](https://upptime.js.org) status page that monitors
`https://trent.ai` and publishes `status.trent.ai`. As part of **SEC-44** it was
hardened away from stock Upptime defaults. Read this before editing the
workflows.

## What changed vs. stock Upptime

- **No more `GH_PAT`.** All workflows run on the built-in, per-run
  `GITHUB_TOKEN` with explicit least-privilege `permissions:` blocks. There is
  no long-lived personal access token to steal.
- **Self-upgrade workflows removed** — `update-template.yml`, `updates.yml`,
  `setup.yml`. These were the only workflows that needed the broad PAT (they
  rewrite files under `.github/workflows/`, which requires the `workflow` token
  scope that `GITHUB_TOKEN` deliberately lacks). They also auto-pulled upstream
  Upptime code daily from a mutable `@master` ref and ran it with the PAT.
- **Actions are SHA-pinned** (`actions/checkout`, `upptime/uptime-monitor`,
  `peaceiris/actions-gh-pages`) with the human-readable version in a trailing
  comment.
- The `# Do not edit / will be overwritten` banner from stock Upptime **no
  longer applies** — nothing regenerates these files now.

## Trade-off

We gave up Upptime's unattended self-upgrade (which auto-pulled `@master` and ran
it with the PAT) in exchange for removing the org-wide token. Monitoring, graphs,
and the status site are unaffected.

## Keeping the pinned actions current — automated via Dependabot

Detection is **not** manual. `.github/dependabot.yml` enables the
`github-actions` ecosystem, so Dependabot:

- opens a **grouped weekly PR** bumping the pinned SHAs
  (`actions/checkout`, `upptime/uptime-monitor`, `peaceiris/actions-gh-pages`),
  updating both the SHA and the `# vX.Y.Z` comment, and
- opens an **immediate PR** if a security advisory is published for any of them.

Dependabot only rewrites the `uses:` line, so the `permissions:` blocks, the
`GITHUB_TOKEN` wiring, and the de-templating above are all preserved.

**Merging is one click.** This repo has no PR-triggered CI gate and no branch
protection, so we deliberately do **not** auto-merge — a quick human glance is
the only check between an upstream release and production. Per Dependabot PR
(~weekly): skim the linked release notes for breaking changes or a suspicious
surprise release, confirm the SHA is from the official tag, then squash-merge.
(Total: roughly 30 seconds/week. If you'd rather go fully hands-off, see the
auto-merge note below.)

### Major version bumps need a manual port

Dependabot bumps the action reference but never restructures the workflow files.
A **major** Upptime bump (e.g. v1 → v2) can change commands/inputs or add new
workflows; treat those PRs as a manual task: read the upstream migration notes,
hand-port any structural changes, and re-apply this hardening.

### Optional: fully hands-off (auto-merge)

If the ~30s/week is still unwanted, add a `dependabot/fetch-metadata` workflow
that auto-merges `semver-patch` + `semver-minor` action bumps and leaves majors
for review. Accept the tradeoff: with no CI gate, an auto-merged bump reaches
production unreviewed. Recommended only alongside the post-merge health-check
workflow (curl `status.trent.ai` + assert the last `uptime.yml` run succeeded).

## Token permissions per workflow

| Workflow | Permissions | Why |
|----------|-------------|-----|
| `uptime.yml` | `contents: write`, `issues: write` | commit history/api data; open/close incident issues |
| `response-time.yml` | `contents: write` | commit response-time data |
| `graphs.yml` | `contents: write` | commit generated graph images |
| `summary.yml` | `contents: write` | commit the README summary table |
| `site.yml` | `contents: write` | push the built site to `gh-pages` |

> Note: the repo's default workflow-token permission is `read`; the explicit
> `permissions:` blocks above grant exactly what each job needs. If a future
> Upptime feature needs more, the job will fail with a clear permissions error —
> add the minimum scope it names.
