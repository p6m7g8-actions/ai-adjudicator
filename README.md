# p6m7g8-actions/ai-adjudicator

- [p6m7g8-actions/ai-adjudicator](#p6m7g8-actionsai-adjudicator)
  - [Usage](#usage)
  - [Inputs](#inputs)
  - [Outputs](#outputs)

## Usage

```yaml
name: Auto PR Orchestrator

on:
  pull_request_target:
    types: [opened, synchronize, reopened, ready_for_review, edited]
  workflow_run:
    workflows: ["Build", "Claude Code Review", "Codex Code Review"]
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

## Inputs

- `gh_token` (required): token for PR read/write and GraphQL thread resolution.
- `repo`: target repository, defaults to `${{ github.repository }}`.
- `pr_number`: explicit PR number; auto-detected from `pull_request` or `workflow_run` event when empty.
- `required_checks_csv`: required check names, default `build,Lint PR title,claude-review,codex-review`.
- `codex_prefix`: codex comment prefix, default `Codex Autonomous Review:`.
- `claude_prefix`: claude comment prefix, default `Claude Autonomous Review:`.
- `iterate_comment`: comment body when verdict is iterate.
- `comment_on_iterate`: `true|false`, default `true`.
- `approve_on_accept`: `true|false`, default `true`.
- `enqueue_on_accept`: `true|false`, default `true`.

## Outputs

- `pr_number`: resolved PR number.
- `proceed`: whether required checks are complete and successful.
- `verdict`: `accept|iterate|skip`.
- `resolved_threads`: number of review threads auto-resolved.
- `review_decision`: final GitHub `reviewDecision`.
- `final_guard_ok`: `true` if approval/enqueue actions were allowed.
