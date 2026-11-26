# Required gate jobs in GitHub Actions

End each workflow that gates merging with one [required gate job](required-gate.yml).
The gate reports every needed job's result and succeeds only when all are
`success`. The example uses `build` and `test` as placeholders. Copy `jobs.required`
into your workflow and wire it to your real jobs.

## Wire the gate

1. List every job whose success is required in `needs`, even when another
   listed job already depends on it. `needs` exposes direct dependencies only.
   Keep the step's environment variables, log lines, and success checks in sync
   with that list. A `failure`, `cancelled`, `skipped`, or missing result must
   fail the gate. Leave `continue-on-error` off for required work.
   [GitHub Actions contexts](https://docs.github.com/en/actions/reference/workflows-and-actions/contexts#needs-context)
2. For required work that only applies to some changes, keep its job eligible
   to run. Put the applicability condition on the work's steps and add a step
   that logs a successful no-op when the work does not apply. Leave purely
   optional jobs out of `needs`.
3. Keep `if: ${{ always() }}` on the gate job and its shell step. GitHub can
   treat a skipped job as satisfying a required check. The gate must run and
   explicitly fail on a non-success result. Keep it short and self-contained
   so it can finish during ordinary cancellation. Whole-run cancellation can
   still prevent a gate from reaching its shell step. The gate sets `shell`
   and `working-directory` to override workflow `defaults.run`.
   [Required status check troubleshooting](https://docs.github.com/en/pull-requests/how-tos/merge-and-close-pull-requests/troubleshooting-required-status-checks#handling-skipped-but-required-checks),
   [Expressions](https://docs.github.com/en/actions/reference/workflows-and-actions/expressions#always),
   [Workflow cancellation](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-cancellation)
4. Set the gate's `name` to a stable name unique across all jobs in the
   repository's workflows. Use the workflow's purpose as a prefix, such as
   `CI gate` or `Docs gate`.
   [About protected branches](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches#about-branch-protection-rules)

## Run it for every change that can merge

- Keep `pull_request` unfiltered by target branch so the gate can report
  before a pull request is retargeted onto the protected branch. Changing only
  the target branch emits `edited`, which the default activity types (`opened`,
  `synchronize`, and `reopened`) omit. If a target-branch filter is unavoidable,
  set `types: [opened, synchronize, reopened, edited]`. A retarget made with
  `GITHUB_TOKEN` still creates no run. Have a user push a new commit to the
  pull request's head branch or reopen the pull request to trigger a run.
  Avoid `paths` and `paths-ignore` on the gating workflow.
  Skip directives in commit messages can also leave required checks pending.
  [Workflow events](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows#pull_request),
  [Required status check troubleshooting](https://docs.github.com/en/pull-requests/how-tos/merge-and-close-pull-requests/troubleshooting-required-status-checks#handling-skipped-but-required-checks)
- For a merge queue, also use `merge_group` with `types: [checks_requested]`.
  Make sure required jobs can run on the merge group's commit and that any
  applicability logic works without a pull request payload.
  [Merge group event](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows#merge_group)
- Pull requests opened, updated, or reopened by a workflow using `GITHUB_TOKEN`
  create runs that a user with write access must approve. For unattended
  automation, use a GitHub App installation token to open and update the pull
  request. A personal access token also triggers the runs without the
  `GITHUB_TOKEN` approval prompt.
  [Triggering a workflow from a workflow](https://docs.github.com/en/actions/how-tos/write-workflows/choose-when-workflows-run/trigger-a-workflow#triggering-a-workflow-from-a-workflow)

## Require the gate

After the gate reports successfully, add its reported check name (for example,
`CI gate`) to the target branch ruleset's required status checks. Select
**GitHub Actions** as the expected source. Require only the gate from each
gating workflow. Keep the individual job results available for diagnosis in
the workflow run.
[Available rules for rulesets](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/available-rules-for-rulesets#require-status-checks-to-pass-before-merging)
