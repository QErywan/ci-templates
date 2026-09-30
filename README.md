# ci-templates

Reusable GitHub Actions workflows for Python repos managed with [uv](https://docs.astral.sh/uv/).

| Workflow | What it does |
| --- | --- |
| `ci.yml` | `uv sync --locked`, then `ruff check`, `ruff format --check` and `pytest`, with the uv cache enabled. |
| `review.yml` | Claude reviews the PR if the repo has an `ANTHROPIC_API_KEY` secret. Otherwise it passes with a "skipped" notice. |
| `automerge.yml` | Once `ci` and `review` pass, it squash-merges the PR and deletes the branch, unless the PR has a `hold` label. |

## Use it

Save this as `.github/workflows/ci.yml` in any repo:

```yaml
name: CI
on:
  push: { branches: [main] }
  pull_request: { types: [opened, synchronize, reopened, ready_for_review, unlabeled] }
permissions: { contents: write, pull-requests: write }
jobs:
  ci: { uses: QErywan/ci-templates/.github/workflows/ci.yml@main }
  review: { uses: QErywan/ci-templates/.github/workflows/review.yml@main, secrets: inherit }
  automerge: { needs: [ci, review], uses: QErywan/ci-templates/.github/workflows/automerge.yml@main }
```

The repo needs a `uv.lock`, with `ruff` and `pytest` in its dev dependencies.

- **Only CI:** keep the `ci` job and set `permissions: { contents: read }`.
- **Turn on review:** add an `ANTHROPIC_API_KEY` repository secret. No workflow change is needed.
- **Pause a merge:** add a `hold` label to the PR (create it once with `gh label create hold`). Removing the label reruns the checks and then merges the PR. To use a different label, pass `with: { hold-label: do-not-merge }` to the `automerge` job.

## How automerge works

Private repos on the free plan have no branch protection, so GitHub's native auto-merge isn't available. The `automerge` job runs only after the `ci` and `review` jobs pass (`needs:`), then:

1. Re-reads the PR from the API. It skips the merge if the PR is closed, a draft or labelled `hold`, or if new commits arrived after the checks started.
2. Squash-merges the PR, pinned to the SHA that was tested (`sha=`), with the PR title as the commit title.
3. Deletes the head branch.

A skip is a passing job with a notice explaining why.

Caveats:

- **Workflow changes stay manual.** GitHub may refuse `GITHUB_TOKEN` merges of PRs that change `.github/workflows/`, so automerge skips those PRs. Merge them yourself.
- **No CI run on `main` after an automerge.** A merge made with `GITHUB_TOKEN` doesn't trigger `push` workflows.
- **No automerge for fork PRs.** They get a read-only token, so automerge ignores them.

## Notes

- **The review is advisory.** If the Claude step errors, the job still passes, so a flaky API call can't block a merge. The review is also skipped for drafts and for label-only events (`unlabeled`).
- **`@main` tracks the latest templates.** Pin a tag or SHA instead if you want callers to be unaffected by changes here.
- **This repo is public** so that private repos can call these workflows. Keep secrets out of it.
