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

We gave up Upptime's unattended weekly self-upgrade in exchange for removing the
org-wide PAT. Monitoring, graphs, and the status site are unaffected. The only
cost is that **Upptime version bumps are now a manual, reviewed action.**

## How to update Upptime (manual)

Do this ~quarterly, or sooner if upstream publishes a security advisory
(watch <https://github.com/upptime/upptime/releases>):

1. Review the release notes for the new version and any breaking changes.
2. Regenerate the workflows locally with a token that has `workflow` scope
   (your own `gh auth` session works), or hand-port the changes.
3. **Re-apply this hardening** to the regenerated files: scoped `GITHUB_TOKEN`,
   `permissions:` blocks, SHA pins, and re-delete the self-upgrade workflows.
4. Bump the pinned SHAs to the new release (verify the tag → SHA and signature),
   updating the trailing `# vX.Y.Z` comments.
5. Open a PR; review the diff before merging.

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
