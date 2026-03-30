# create-issue-on-failure

A GitHub composite action that automatically creates a separate GitHub Issue per failed step with failure details when a workflow job fails.

## Usage

Add this action as the last step in your job, with `if: failure()` so it only runs when something goes wrong:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      issues: write
      actions: read
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Run tests
        run: make test

      - name: Create issue on failure
        if: failure()
        uses: DmitriyLewen/create-issue-on-failure@main
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

When steps fail, the action creates one issue per failed step. For example, if `Run tests` fails:

**Title:** `ci: step Run tests failed in workflow My CI`

**Body:**
```
## Workflow Failure Report

- **Workflow:** My CI
- **Failed step:** Run tests
- **Triggered by:** push
- **Actor:** octocat
- **Commit:** abc1234...
- **Run URL:** https://github.com/owner/repo/actions/runs/123456789
```

If multiple steps fail, a separate issue is created for each one. Duplicate issues (same title, already open) are skipped automatically.

## Permissions

The action requires `issues: write` permission to create issues and `actions: read` to look up the failed step names. Set them at the job level:

```yaml
jobs:
  build:
    permissions:
      issues: write
      actions: read
```

Or at the workflow level to apply to all jobs:

```yaml
permissions:
  issues: write
  actions: read
```

> **Note:** If your repository has default workflow permissions set to "read-only" in Settings → Actions → General, you must explicitly grant `issues: write` in the workflow file.

The action requires `jq` to be available on the runner. It is pre-installed on all GitHub-hosted runners (`ubuntu-*`, `macos-*`, `windows-*`). For self-hosted runners, install it manually.

The `GH_TOKEN` environment variable must be set to a token with issue creation access. `secrets.GITHUB_TOKEN` works for repositories within the same organization. For cross-repository issue creation, use a Personal Access Token (PAT) with `repo` scope.

## Troubleshooting

### `Resource not accessible by integration` or `403 Forbidden`

The token does not have permission to create issues.

- Make sure `permissions: issues: write` is set at the job or workflow level.
- Check that the repository's default workflow permissions allow issue creation: **Settings → Actions → General → Workflow permissions**.

### `Bad credentials` or `401 Unauthorized`

The `GH_TOKEN` environment variable is missing or invalid.

Make sure you pass the token explicitly:

```yaml
- uses: DmitriyLewen/create-issue-on-failure@main
  if: failure()
  env:
    GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### Issues are not created (action is skipped)

The `if: failure()` condition only triggers when a **previous step** in the same job has failed. The action will be skipped if:

- All steps passed (expected behavior).
- The job was cancelled — use `if: failure() || cancelled()` to also catch cancellations.
- The failing step has `continue-on-error: true` — the job is not considered failed in that case.

### `Issues are disabled for this repository`

Issues must be enabled in the repository settings: **Settings → Features → Issues**.

### When are new issues created despite existing ones

The action matches open issues by exact title (`ci: step <name> failed in workflow <workflow>`). If the issue was closed or the title format changed, a new issue will be created.

> **Note for users upgrading from v1 (single issue per workflow):** The issue title format changed from `ci: workflow <name> failed` to `ci: step <name> failed in workflow <name>`. Previously created open issues will not be recognized as duplicates and new issues will be created on the first run after upgrade.

### Action creates issues even on pull requests from forks

Fork PRs run with a read-only `GITHUB_TOKEN` and cannot create issues. The `gh api` call will fail silently or return a 403. To avoid this, add a check:

```yaml
- uses: DmitriyLewen/create-issue-on-failure@main
  if: failure() && github.event.pull_request.head.repo.full_name == github.repository
  env:
    GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```
