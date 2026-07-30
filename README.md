# p6m7g8-actions/ai-adjudicator

- [p6m7g8-actions/ai-adjudicator](#p6m7g8-actionsai-adjudicator)
  - [Usage](#usage)
  - [Verdict contract](#verdict-contract)
    - [Freshness](#freshness)
    - [Self-marker](#self-marker)
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
- They must be the **last three non-empty lines** of the comment, in that order,
  with nothing after them. Matching is positional on those three lines; the body
  above them is never scanned.
- They are at the end rather than the start because `claude-code-action` with
  `track_progress: true` writes its own
  `**Claude finished @user's task in Xm Ys**` header into the comment before the
  review body, so the start of the comment is not the producer's to control.
  Nothing anchors on the first line and no leading comment prefix is required.

Positional matching is a security property, not a convenience. Scanning the whole
body meant that any review which merely *quoted* the contract while discussing it
emitted a parseable verdict, and this action's own fail-loud comment used to
reproduce the contract, so one run could report a broken producer and the next
could read that comment back as a pass and auto-merge. Three guards now close
that: positional matching, the self-marker below, and a fail-loud comment that
describes the contract instead of reproducing it.

What does and does not break the match, precisely:

| Shape | Matches? |
| --- | --- |
| Plain, as the last three non-empty lines | yes |
| Trailing blank lines, or CRLF endings | yes, both are normalised away |
| Leading indentation on the three lines | yes, leading whitespace is trimmed |
| Bold, a list marker, a table, or a blockquote marker | no, the line no longer starts with the keyword |
| Inside a fenced block at the end | no, the closing fence becomes the last line |
| Anywhere earlier in the body, quoted or not | no, only the last three lines are read |
| Followed by a footer or any other line | no, and this fails closed |

The last row is the one to know about: a producer that emits the contract and
then appends a footer gets a quiet `iterate`, not a pass. The contract says
nothing may follow the three lines, and the adjudicator holds that line.

`Review-Lane:` carries the lane name. `required_lanes_csv` lists the lanes that
must each report `Overall: patch is correct` before the verdict is `accept`, so
the planned `security` and `conventions` lanes drop in by setting that input,
with no change to this action.

### Freshness

A green `claude-review` check on the head SHA does not prove a comment was
written for that SHA. The distributed `claude-review.yml` runs the review with
`continue-on-error: true`, so the check goes green even when the review produced
nothing, and sticky comments are edited in place rather than reposted. Left
alone, that means a pass from an earlier push can be read as a pass for new code.

Two rules prevent it:

1. A comment is a verdict candidate only if its `updated_at` is at or after the
   earliest check-run start on the head SHA. A comment last written during an
   earlier push cannot clear that bar. `updated_at` rather than `created_at`,
   because a sticky comment keeps its original `created_at` forever.
2. If the newest candidate **from the same author** as a lane's verdict carries
   no `Review-Lane:` line in its own last three lines, that lane is treated as
   having no verdict rather than falling back to the older one. Scoped to the
   same author on purpose: an unrelated allowlisted bot commenting on the PR
   must not invalidate a real verdict.

### Self-marker

Every comment this action posts ends with `<!-- p6-adjudicator -->`, and the
parser skips any comment carrying it. Without that, the action's own comments,
authored by whatever `gh_token` resolves to and therefore usually an allowlisted
identity, would be candidate verdicts and would shadow the real review on
re-runs.

### Author gate

A pull request **conversation comment** counts as a verdict only when its author
login is an **exact** member of `allowed_bot_logins`. Body text alone never
qualifies, so a third party cannot forge a verdict by pasting well-formed
contract lines onto a public pull request.

Only conversation comments are read: `/repos/{owner}/{repo}/issues/{n}/comments`.
Formal review bodies (`/pulls/{n}/reviews`) and inline review comments are
deliberately not read. The producer posts to the conversation, which is what
`claude-code-action` with `track_progress: true` does, and an allowlisted
identity that posts an approving review is casting an approval, not supplying a
verdict. `p6m7g8-automation` is in the default allowlist and posts formal
reviews; those bodies are invisible here, by design.

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
| No candidate comment names the lane in its last three lines | `iterate` | quiet, via `iterate_comment` |
| Lane named there, but a contract line is absent, malformed, or out of order | `iterate` | loud, see below |

Loud means all three of these:

1. An `::error` annotation in the run log naming the lane and the missing line.
2. A PR comment naming the lane and describing what was missing. It describes the
   contract rather than reproducing it, so the comment can never itself parse.
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
- `allowed_bot_logins`: comma-delimited exact bot logins whose pull request conversation comments
  count as verdicts, default `claude[bot],github-actions[bot],p6m7g8-automation`. Author login must
  match a list entry exactly; a matching comment body from any other author is ignored.
  `claude[bot]` is the login the Claude review action posts under, so removing it makes every
  verdict fail closed. Formal review bodies and inline review comments are never read.
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
