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
  - [Breaking changes](#breaking-changes)
    - [v0.3.0](#v030)
    - [v0.2.0](#v020)

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
  with nothing after them. Acceptance is positional on those three lines and is
  decided by them alone. The rest of the body is read for one purpose only, to
  tell a misplaced contract apart from no contract at all, which changes how a
  rejection is reported and never whether one happens.
- They are at the end rather than the start because `claude-code-action` with
  `track_progress: true` writes its own
  `**Claude finished @user's task in Xm Ys**` header into the comment before the
  review body, so the start of the comment is not the producer's to control.
  Nothing anchors on the first line and no leading comment prefix is required.

Positional acceptance is a security property, not a convenience. Accepting on a
whole-body scan meant that any review which merely *quoted* the contract while
discussing it emitted a parseable verdict, and this action's own fail-loud comment
used to reproduce the contract, so one run could report a broken producer and the
next could read that comment back as a pass and auto-merge. Three guards now close
that: positional acceptance, the self-marker below, and a fail-loud comment that
describes the contract instead of reproducing it.

What does and does not accept, precisely. Every row is covered by a fixture case:

| Shape | Accepts? |
| --- | --- |
| Plain, as the last three non-empty lines | yes |
| Trailing blank lines, or CRLF endings | yes, both are normalised away |
| Leading indentation on the three lines | yes, leading whitespace is trimmed |
| Bold, a list marker, a table, or a blockquote marker | no, the line no longer starts with the keyword |
| Inside a fenced block at the end | no, the closing fence becomes the last line |
| Followed by a footer or any other line | no, and it is reported loudly |
| Anywhere earlier in the body, quoted or not | no, and it is reported loudly |

The last two rows are the ones to know about, and the reason the whole body is
scanned for one narrow purpose. `claude-code-action`'s own base prompt tells the
reviewer to put a branch link at the bottom of the comment, which directly
contradicts "nothing after the three lines". A producer that emits a perfect
contract and then obeys that instruction is the single likeliest failure, so it
must not be silent. If no candidate carries the lane in its last three lines but
some candidate names it anywhere in its body, the lane is `unparseable` and takes
the fail-loud path rather than the quiet one.

That body scan cannot fail open, because it only ever turns a rejection into a
louder rejection. **Acceptance remains positional on the last three lines and
nothing else.** The cost is that a review which merely discusses the contract in
prose also gets a loud comment; that is the right trade, since a false alarm is
visible and cheap while a silent `iterate` burns the fix loop.

`Review-Lane:` carries the lane name. `required_lanes_csv` lists the lanes that
must each report `Overall: patch is correct` before the verdict is `accept`, so
the planned `security` and `conventions` lanes drop in by setting that input,
with no change to this action.

### Freshness

This is the canonical description; `action.yml` points here rather than repeating
it.

A green `claude-review` check on the head SHA does not prove a comment was
written for that SHA. The distributed `claude-review.yml` runs the review with
`continue-on-error: true`, so the check goes green even when the review produced
nothing, and sticky comments are edited in place rather than reposted. Left
alone, that means a pass from an earlier push can be read as a pass for new code.

Two rules prevent it:

1. A comment is a verdict candidate only if its `updated_at` is at or after the
   **freshness cutoff** below. A comment last written during an earlier push
   cannot clear that bar. `updated_at` rather than `created_at`, because a sticky
   comment keeps its original `created_at` forever.
2. If the newest candidate **from the same author** as a lane's verdict carries
   no `Review-Lane:` line in its own last three lines, that lane is treated as
   having no verdict rather than falling back to the older one. Scoped to the
   same author on purpose: an unrelated allowlisted bot commenting on the PR
   must not invalidate a real verdict.

The cutoff is the earliest **check suite** `created_at` on the head SHA. Suites
are created in response to the push, so it lands just after it, and `created_at`
is server-assigned and immutable.

It has to be a value that does not move when a check is re-run. Check *run*
`started_at` does move: re-running updates the run in place, so "Re-run all jobs"
after a flake would drag the cutoff past a genuine fresh verdict and deadlock an
otherwise-green PR at quiet `iterate`. Suite `created_at` does not move, and
taking the minimum ignores any later suite a re-run adds.

A commit's GraphQL `pushedDate` would be the natural choice, being the
server-assigned push time and, unlike a commit's committer date, not settable by
the PR author. It is not usable: the field is deprecated, "no longer supported.
Removal on 2023-07-01 UTC", and returns null for every commit. Check suite
`created_at` has the same two properties and still works.

If no cutoff can be established, the action refuses and reports rather than
carrying on with the filter disabled. The same applies to an empty
`required_checks_csv` or `required_lanes_csv`: a guard that silently switches
itself off is worse than one that stops.

### Self-marker

Every body this action posts ends with `<!-- p6-adjudicator -->`, and the parser
skips any comment carrying it. Without that, the action's own comments, authored
by whatever `gh_token` resolves to and therefore potentially an allowlisted
identity, would be candidate verdicts and would shadow the real review on
re-runs. The approving review carries it too, even though review bodies are not
read, so that widening the parse scope later cannot turn an approval into a
verdict.

Each posted body also carries a `<!-- p6-adjudicator-key: ... -->` line scoped to
the head SHA, and the fail-loud and `iterate` comments skip posting when a comment
with the same key already exists. This action fires on `pull_request_target`
including `edited`, and on `workflow_run` completion of both Build and Claude Code
Review, so without dedup one push drew several identical comments and every body
edit added another. A new push produces a new key, so a genuine re-report still
happens.

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
verdict.

The default allowlist is **`claude[bot]` alone**, the producer and nothing else.
It previously also held `github-actions[bot]` and `p6m7g8-automation`, and both
were pure attack surface. `p6m7g8-automation` posts formal reviews, which are not
read, so it could never legitimately supply a verdict through the only channel
that is. `github-actions[bot]` is every workflow in the consuming repository, and
since the body requirement is three short public lines, any of those workflows
could mint an `accept` by ending a conversation comment with them. Add entries
only for identities that genuinely post review verdicts.

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
| No candidate comment mentions the lane anywhere | `iterate` | quiet, via `iterate_comment` |
| Lane named in the last three lines, but a contract line is absent, malformed, or out of order | `iterate` | loud |
| Lane named in the body but not in the last three lines | `iterate` | loud |

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
  An empty list is a misconfiguration and never proceeds.
- `required_lanes_csv`: review lanes that must each report `Overall: patch is correct`,
  default `code`. The value matches what the producer writes on the `Review-Lane:` line.
  An empty list is a misconfiguration and never accepts.
- `allowed_bot_logins`: comma-delimited exact bot logins whose pull request conversation comments
  count as verdicts, default `claude[bot]`. Author login must match a list entry exactly; a matching
  comment body from any other author is ignored. `claude[bot]` is the login the Claude review action
  posts under, so removing it makes every verdict fail closed. Formal review bodies and inline
  review comments are never read.
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
- `findings_total`: sum of `Findings (total):` across every lane that reported a parseable
  verdict, `0` otherwise. Reported for both `correct` and `incorrect` verdicts.
- `resolved_threads`: number of review threads auto-resolved.
- `review_decision`: final GitHub `reviewDecision`.
- `final_guard_ok`: `true` if approval/enqueue actions were allowed.

## Breaking changes

This action is consumed at `@main` throughout the fleet by owner policy; nothing
pins a tag internally. Tags are published, so the version numbers below are real,
but for the fleet they are advisory: a change lands for every consumer as soon as
it lands on `main`. Read this section as a changelog of things that changed
behaviour, not as an upgrade path you can defer by pinning.

### v0.3.0

- `allowed_bot_logins` default narrowed from `claude[bot],github-actions[bot],p6m7g8-automation`
  to `claude[bot]`. If you relied on a verdict from either dropped identity, pass it explicitly.
  Neither could supply a verdict through the only channel that is read unless it posted a
  conversation comment ending in the contract lines.
- An empty `required_checks_csv` now refuses instead of proceeding with nothing verified.
- A contract found in the body but not as the last three lines is now reported loudly instead of
  being treated as "no verdict yet".

### v0.2.0

- `codex_prefix` and `claude_prefix` were removed. Codex was retired fleet-wide, and the match now
  anchors on the contract lines rather than a leading prefix, so neither input has a meaning.
  GitHub Actions treats an unknown `with:` key as a warning rather than an error, so a caller still
  passing them is not failed, merely ignored: check your workflow logs for
  `Unexpected input(s)` if you set either.
- `required_checks_csv` default changed from `build,Lint PR title,claude-review,codex-review` to
  `build,Lint PR title,claude-review`. If you set it explicitly, drop `codex-review` yourself; the
  check no longer exists and waiting on it never completes.
- Verdicts are now read from the last three non-empty lines of a conversation comment. A producer
  that posted a prefixed comment, or put the contract anywhere other than the end, no longer
  produces an `accept`.
