# Required gate jobs in GitHub Actions

End each workflow that gates merging with one [required gate job](required-gate.yml).
The gate succeeds only when it receives at least one result and every result
is `success`. The example uses `build` and `test` as placeholders. Copy
`jobs.required` into your workflow and list your required jobs in `needs`.

## Wire the gate

1. List every job whose success is required in `needs`, even when another
   listed job already depends on it. To add an existing job to the gate, edit
   `needs`. The step reads `needs.*.result` for all direct dependencies and
   fails on `failure`, `cancelled`, `skipped`, or any other value besides
   `success`. The gate relies on GitHub supplying a documented `result` for
   each direct dependency because `needs.*.result` omits dependencies without
   that property. With one `success` and one missing `result`, the gate would
   pass. The log lists the results in an unspecified order. Use the run's job
   list to see which job reported each result. Leave `continue-on-error` off
   for required work.
   [GitHub Actions contexts](https://docs.github.com/en/actions/reference/workflows-and-actions/contexts#needs-context),
   [Object filters](https://docs.github.com/en/actions/reference/workflows-and-actions/expressions#object-filters)
2. For required work that only applies to some changes, keep its job eligible
   to run. Put the applicability condition on the work's steps and add a step
   that logs a successful no-op when the work does not apply. Leave purely
   optional jobs out of `needs`.
3. Keep `if: ${{ always() }}` on the gate job and its shell step. GitHub
   accepts `skipped` and `neutral` conclusions for required checks. The gate
   must run and explicitly fail on a non-success result. Keep it short and
   self-contained so it can finish during ordinary cancellation. Cancelling
   the entire run can still prevent the gate from reaching its shell step.
   The gate sets `shell` and `working-directory` to override workflow
   `defaults.run`.
   [Required status check troubleshooting](https://docs.github.com/en/pull-requests/how-tos/merge-and-close-pull-requests/troubleshooting-required-status-checks#handling-skipped-but-required-checks),
   [Expressions](https://docs.github.com/en/actions/reference/workflows-and-actions/expressions#always),
   [Workflow cancellation](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-cancellation)
4. Set the gate's `name` to a stable name unique across all jobs in the
   repository's workflows. Use the workflow's purpose as a prefix, such as
   `CI gate` or `Docs gate`.
   [About protected branches](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches#about-branch-protection-rules)

## Run it for every change that can merge

- Subscribe to `pull_request` activities `opened`, `synchronize`, `reopened`,
  and `edited`. Changing the base branch emits `edited`, which runs the gate
  against the new test merge commit. Title and description edits also emit
  `edited` and run the workflow. Keep the required jobs and gate eligible on
  those edits. Skipping a needed job fails the gate, and skipping the gate
  itself can satisfy the required check. If you use `cancel-in-progress: true`
  in a concurrency group shared by all runs for a pull request, verify after
  an edit that the replacement gate run finished against the intended base.
  [Workflow events](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows#pull_request),
  [Pull request webhook](https://docs.github.com/en/webhooks/webhook-events-and-payloads#pull_request),
  [Required status check troubleshooting](https://docs.github.com/en/pull-requests/how-tos/merge-and-close-pull-requests/troubleshooting-required-status-checks#handling-skipped-but-required-checks),
  [Workflow concurrency](https://docs.github.com/en/actions/how-tos/write-workflows/choose-when-workflows-run/control-workflow-concurrency)
- Like a person's edit, a title or description edit by a GitHub App or other
  automation emits `edited`. The event starts another workflow run unless the
  edit uses `GITHUB_TOKEN`. If an app writes summaries into the description,
  configure it to post them in a comment. For CodeRabbit, set
  `high_level_summary_in_walkthrough` to `true` under `reviews` in a V2
  `.coderabbit.yaml` to put the summary in the walkthrough comment. On
  CodeRabbit Cloud, the setting can come from the organization even when the
  repository has no CodeRabbit configuration file. Organization defaults can
  supply it when they are the starting configuration or when inheritance
  reaches them with the setting still unset. Organization global overrides
  can take precedence regardless of inheritance. On the first pull request,
  post `@coderabbitai configuration` as a comment and check the setting's
  value and its `# Source:` in the reply.
  [CodeRabbit PR summaries](https://docs.coderabbit.ai/pr-reviews/summaries#moving-the-summary-to-the-walkthrough),
  [CodeRabbit configuration resolution](https://docs.coderabbit.ai/configuration/resolution),
  [Check which configuration a review used](https://docs.coderabbit.ai/configuration/guides/check-effective-configuration)
- Leave `pull_request` unfiltered by base branch and avoid `paths` and
  `paths-ignore` on the gating workflow. Workflow filters and skip directives
  in commit messages can leave required checks pending. Resolve merge conflicts
  before expecting a `pull_request` run.
  [Workflow events](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows#pull_request),
  [Required status check troubleshooting](https://docs.github.com/en/pull-requests/how-tos/merge-and-close-pull-requests/troubleshooting-required-status-checks#handling-skipped-but-required-checks)
- For a merge queue, also use `merge_group` with `types: [checks_requested]`.
  Make sure required jobs can run on the merge group's commit and that any
  applicability logic works without a pull request payload.
  [Merge group event](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows#merge_group)
- With `GITHUB_TOKEN`, `opened`, `synchronize`, and `reopened` create runs that
  a user with write access must approve. A retarget using `GITHUB_TOKEN` emits
  `edited` and creates no run. After such a retarget, have a maintainer push a
  new commit to the pull request's head branch or reopen the pull request using
  their own credentials, then verify the new run against the new base. For
  unattended automation, use a GitHub App installation token or a personal
  access token to trigger runs without the `GITHUB_TOKEN` approval prompt.
  [Triggering a workflow from a workflow](https://docs.github.com/en/actions/how-tos/write-workflows/choose-when-workflows-run/trigger-a-workflow#triggering-a-workflow-from-a-workflow)

## Require the gate

After the gate reports successfully, add its reported check name (for example,
`CI gate`) to the target branch ruleset's required status checks. Select
**GitHub Actions** as the expected source. Require only the gate from each
gating workflow. Keep the individual job results available for diagnosis in
the workflow run.
[Available rules for rulesets](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/available-rules-for-rulesets#require-status-checks-to-pass-before-merging)
