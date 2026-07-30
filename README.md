# p6m7g8-actions/ai-adjudicator

- [p6m7g8-actions/ai-adjudicator](#p6m7g8-actionsai-adjudicator)
  - [Usage](#usage)
  - [Verdict contract](#verdict-contract)
    - [Author gate](#author-gate)
    - [Fail loud on an unparseable verdict](#fail-loud-on-an-unparseable-verdict)
  - [Inputs](#inputs)
  - [Outputs](#outputs)

## Usage

```yaml
name: Auto PR Orchestrator

on:
  pull_request_target:
    types: [opened, synchronize, reopened, ready_for_review, edited]
  workflow_run:
    workflows: ["Build", "Claude Code Review"]
    types: [completed]

jobs:
  orchestrate:
    runs-on: ubuntu-latest
    steps:
      - name: Adjudicate
        uses: p6m7g8-actions/ai-adjudicator@main
        with:
          gh_token: ${{ secrets.P6_A_GH_TOKEN }}
```

## Verdict contract

Adjudication reads one voice: the Claude review posted by
[`p6m7g8-actions/claude`](https://github.com/p6m7g8-actions/claude), which is the
producer of this contract. Codex is retired, so there is no second reviewer to
reconcile against and nothing that can disagree.

The producer ends every review with exactly these three lines:

```text
Review-Lane: code
Overall: patch is correct
Findings (total): 0
```

- `Overall: patch is incorrect` is the negative form.
- `Findings (total):` carries a non-negative integer.
- The three lines are the **last** lines of the comment, not the first.
  `claude-code-action` with `track_progress: true` writes its own
  `**Claude finished @user's task in Xm Ys**` header into the comment before the
  review body, so the start of the comment is not the producer's to control. The
  adjudicator therefore anchors on the three lines wherever they appear and
  never on the first line, and no leading comment prefix is required.
- Lines are matched as trimmed literals. Wrapping any of them in bold, a code
  fence, a list marker, or a table breaks the match on purpose, rather than
  being silently tolerated.

`Review-Lane:` carries the lane name. `required_lanes_csv` lists the lanes that
must each report `Overall: patch is correct` before the verdict is `accept`, so
the planned `security` and `conventions` lanes drop in by setting that input,
with no change to this action. Each lane is adjudicated from the most recent
allowlisted comment that carries its `Review-Lane:` line.

### Author gate

A comment counts as a verdict only when its author login is an **exact** member
of `allowed_bot_logins`. Body text alone never qualifies, so a third party
cannot forge a verdict by pasting well-formed contract lines onto a public
pull request.

Comments are read over the REST API rather than `gh pr view --json comments`.
The GraphQL shape `gh` uses renders a GitHub App author with the `[bot]` suffix
stripped, so the Claude app arrives as `claude`, and `github.com/claude` is a
real, separate human account. Over REST the same author is `claude[bot]`, a form
no user account can hold because `[` is not a legal login character. Exact-match
allowlisting is only sound over REST, which is why that is what this action
reads. Adding a bare `claude` to `allowed_bot_logins` would undo the gate.

### Fail loud on an unparseable verdict

Two distinct situations both yield `iterate`, and they are reported differently:

| Situation | Verdict | Reported as |
| --- | --- | --- |
| No allowlisted comment carries the lane yet | `iterate` | quiet, via `iterate_comment` |
| Lane present, contract line absent or malformed | `iterate` | loud, see below |

Loud means all three of these:

1. An `::error` annotation in the run log naming the lane and the missing line.
2. A PR comment naming the lane, the missing line, and the expected contract.
3. The lane listed in the `unparseable_lanes` output.

The second case is loud because a bare `iterate` under a bounded fix loop burns
every iteration before anyone reads the run. `comment_on_unparseable` controls
that comment independently of `comment_on_iterate`.

## Inputs

- `gh_token` (required): token for PR read/write and GraphQL thread resolution.
- `repo`: target repository, defaults to `${{ github.repository }}`.
- `pr_number`: explicit PR number; auto-detected from `pull_request` or `workflow_run` event when empty.
- `required_checks_csv`: required check names, default `build,Lint PR title,claude-review`.
- `required_lanes_csv`: review lanes that must each report `Overall: patch is correct`,
  default `code`. The value matches what the producer writes on the `Review-Lane:` line.
  An empty list is treated as a misconfiguration and never accepts.
- `allowed_bot_logins`: comma-delimited exact bot logins whose review comments count as verdicts,
  default `claude[bot],github-actions[bot],p6m7g8-automation`. Author login must match a list entry
  exactly; a matching comment body from any other author is ignored. `claude[bot]` is the login the
  Claude review action posts under, so removing it makes every verdict fail closed.
- `iterate_comment`: comment body when verdict is iterate and the verdict parsed cleanly.
- `comment_on_iterate`: `true|false`, default `true`.
- `comment_on_unparseable`: `true|false`, default `true`. Posts the fail-loud comment when a
  lane's verdict comment exists but cannot be parsed.
- `approve_on_accept`: `true|false`, default `true`.
- `enqueue_on_accept`: `true|false`, default `true`.

## Outputs

- `pr_number`: resolved PR number.
- `proceed`: whether required checks are complete and successful.
- `verdict`: `accept|iterate|skip`.
- `unparseable_lanes`: comma-delimited lanes whose verdict comment could not be parsed; empty
  when every required lane either parsed cleanly or has not reported yet.
- `resolved_threads`: number of review threads auto-resolved.
- `review_decision`: final GitHub `reviewDecision`.
- `final_guard_ok`: `true` if approval/enqueue actions were allowed.
