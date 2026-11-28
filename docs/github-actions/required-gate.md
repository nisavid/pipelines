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
  configure it to post them in a comment. For CodeRabbit, set the V2 key
  `reviews.high_level_summary_in_walkthrough` to `true` to put the summary in
  the walkthrough comment. Do not add this key to a `version: 3`
  configuration file. CodeRabbit settings can come from the organization or
  a repository file, so follow CodeRabbit's documentation and change the
  setting where the repository's settings are managed. On the first pull
  request that opens after the change and gets a CodeRabbit review, wait for
  the review to finish, then check that CodeRabbit did not edit the
  description.
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
- The example triggers only on `pull_request` and `merge_group`. Keeping a
  `push` trigger on a gating workflow can add a second gate result under the
  same name to a pull request's head, and the two results can differ.
  Drop `push` from gating workflows, or move it to a separate workflow whose
  gate has its own name.
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

## Opt in to a gate-passed comment

Copy `jobs.gate-passed` from the [reference workflow](required-gate.yml) into
your workflow. Match `GATE_NAME` to the gate's name. The gate still ends the
workflow's required work. Keep the comment job out of the gate's `needs` and
out of the branch's required checks. If only the comment job fails, the pull
request shows a failed advisory check. The workflow run still succeeds, and
the gate's check stays successful. The comment reports one gate's result.
Other required checks can still be pending or failing. Anyone the comment
wakes still needs to read the pull request's checks before acting.

The job skips merge-queue runs, pull requests from forks, and pull requests
that Dependabot opened or whose run Dependabot started.

A new comment triggers notifications. Who receives them depends on
subscriptions. For example, the pull request's author can receive a
notification. Add this job only where something waits on the repository's
pull requests.
[Create an issue comment](https://docs.github.com/en/rest/issues/comments#create-an-issue-comment),
[About notifications](https://docs.github.com/en/subscriptions-and-notifications/concepts/about-notifications)

The job posts with `GITHUB_TOKEN`, so its comments start no workflow runs.
[Triggering a workflow from a workflow](https://docs.github.com/en/actions/how-tos/write-workflows/choose-when-workflows-run/trigger-a-workflow#triggering-a-workflow-from-a-workflow)

Set `GATE_PASSED_MODE` in the comment job:

- `per-head` (default) posts a new comment for each head that passes. Choose it
  when something waiting on the pull request needs a fresh comment for each
  green head.
- `sticky` creates a comment for the gate and updates it with the SHA of each
  new head that passes. Choose it to keep the result in one place. Later heads
  edit this comment and add no new one, so use `per-head` if something waiting
  on the pull request needs a new comment for each green head. Before writing,
  the job reads the pull request's current head and skips the write unless the
  read returns the run's head SHA. If the head changes between the read and
  the write, runs for different heads can still create two comments or update
  the same comment out of order.
  The job's concurrency group covers one workflow, pull request, and head.
  [Workflow concurrency](https://docs.github.com/en/actions/how-tos/write-workflows/choose-when-workflows-run/control-workflow-concurrency)

The job writes at most one per-head comment for each head SHA. A base change
leaves the head SHA unchanged, and a force-push can restore an earlier head.
In `per-head` mode, the job skips posting if it already has a comment for
that head. Switching to `sticky` can add a comment on the same head because
that mode creates its own comment if none exists. It updates an existing
sticky comment only when the comment records a different head.

Keep the comment job on `pull_request` with the same-repository check.
`pull_request_target` would run with the base repository's token on other
people's pull requests.
[Workflow events](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows#pull_request_target),
[GITHUB_TOKEN](https://docs.github.com/en/actions/concepts/security/github_token)
