# Copilot Coding Agent Onboarding

This repository defines a composite GitHub Action that adjudicates Codex and Claude review outputs on pull requests.

## Action contract

- Entry point: `action.yml`
- Runtime: composite action + bash + `gh` CLI
- Core flow:
  1. Resolve PR number from input or event payload.
  2. Gate on open/non-draft PR and required check completion.
  3. Parse Codex/Claude verdict comments.
  4. Resolve unresolved review threads when accepted.
  5. Approve and enqueue via merge queue.

## Editing constraints

- Keep action bash POSIX-safe and idempotent.
- Preserve current output names.
- Do not introduce polling or delay commands.
- Keep required-check semantics explicit and configurable.
