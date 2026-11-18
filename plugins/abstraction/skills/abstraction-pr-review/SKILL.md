---
name: abstraction-pr-review
description: Get Abstraction's review of your change and iterate on it — pre-review a pushed branch before opening a pull request, or review a pull request, then fix the findings, mark the ones you leave, and review again. Use when asked to review a branch or PR with Abstraction, to check a change before opening a PR, or to address Abstraction's review findings.
---

# Abstraction PR review

Abstraction reviews a change against the whole analysed codebase: it ranks the
issues it finds, says which functions the change reaches, and runs the
workspace's checks. Three tools drive it:

| Tool | Use |
|---|---|
| `request_pr_review` | Start a review of a PR (`pr_url`) or a branch (`repository` + `branch`, optional `base_branch`), or re-run one after pushing. |
| `get_pr_review` | Read a review, or follow one still running. |
| `update_pr_review_issues` | Update a review's issues: mark them fixed, invalid, won't fix or a duplicate, or reopen one. |

The first two take the same target. `repository` is `owner/name` or the GitHub URL;
read it from the `origin` remote.

## The loop

1. **Push the branch.** The review reads GitHub, not the working tree: commits
   that aren't pushed aren't reviewed.
2. **Request the review.** Before a PR exists, name the branch:

   ```json
   {"repository": "acme/api", "branch": "fix-retry-loop"}
   ```

   That is a **pre-review**: it is yours alone (`pre_review: true`), and none of
   its findings reach GitHub. When the branch already has an open PR, the same
   call reviews the PR instead, and the entry carries its `pr_number`.
3. **Wait for it.** By default the call holds until the review finishes,
   sending its progress ("Reviewing functions (34/77)") as it runs, and answers
   with the findings. A review takes minutes.
4. **Fix and push again.** A re-run updates the same review: `new` marks the
   findings this run first reported, and `resolved_since_last_run` lists the
   ones your fixes made go away.
5. **Mark what you did not fix**, then repeat from step 2. See
   [Updating issues](#updating-issues): a re-run skips what you marked
   invalid, won't fix or a duplicate, and checks what you marked fixed.
6. **Open the PR** when the review is clean enough. The pre-review becomes the
   PR's review, and the PR's first review starts fresh, told which pre-review
   findings you dismissed in the app.

## Waiting

`wait` defaults to true and holds the call for up to 3 minutes
(`wait_seconds` shortens it). A wait that runs out answers with the review
still `in_progress`, its `progress` headline and percentage, and a message
saying so. The review keeps running: call `get_pr_review` with the same target
to pick it up again. Cancelling the call stops the wait, never the review.

The wait is short because a claude.ai connector cuts a tool call off at four
minutes, without an error, and the caller then waits minutes more for nothing.
A review that takes longer needs a second `get_pr_review`.

With `wait: false`, `request_pr_review` answers as soon as the review starts.

## Reading the result

Each workspace reviewing the repository answers with one entry:

- `in_progress` / `progress` — still running, and how far it is.
- `verdict`, `summary`, `impact` — the outcome and the change's reach.
- `issues` — every finding, each with its `state`, `verdict`, `severity`,
  `file_path`, lines and the `issue_id` to rule by. The `ACTIVE` ones come
  first, ranked and with the reviewer's `function_review` write-up; then the
  `RESOLVED` ones, with the `resolution` or `verdict_reason` that closed them.
  Pass `states: ["ACTIVE"]` to list only what is left to fix, or filter by
  `verdicts`. Fix by severity, highest first.
- `checks` — each check's verdict and summary.
- `stale: true` — commits were pushed after the reviewed head; request again.

`deferred: true` means the review waits for the new head to be indexed first;
call `get_pr_review` later. On a PR that was pre-reviewed, the findings are the
PR's own; pass `include_pre_review: true` to also list the pre-review's in
`pre_review_findings`.

## Updating issues

`update_pr_review_issues` takes the entry's `report_id` and a list of
updates, each naming an issue by its `issue_id` and giving it a verdict:

```json
{
  "report_id": 1234,
  "updates": [
    {"issue_id": 4211, "verdict": "fixed"},
    {"issue_id": 4212, "verdict": "invalid", "reason": "The caller holds mu, see line 40."},
    {"issue_id": 4213, "verdict": "duplicate", "duplicate_of": 4211}
  ]
}
```

| Verdict | Use | Needs |
|---|---|---|
| `fixed` | You fixed it. The re-run checks the claim and reopens the issue if it still stands. | |
| `invalid` | The issue is wrong. | `reason` |
| `wont_fix` | Real, but deliberately left. | `reason` |
| `duplicate` | The same problem as another issue of the review. | `duplicate_of` |
| `open` | Reopen an issue marked before. | |

The updates land together or not at all; a refused one names the issue and
nothing is updated. Your verdicts are your agent's: a person's later verdict
overrules them. On a PR the issue's GitHub thread is resolved or reopened to
match; updates to a pre-review never reach GitHub. Issues in
`pre_review_findings` are frozen and can't be updated. Updating takes an
editor or admin of the review's workspace.
