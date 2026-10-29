# check-review

`check-review` is an agent skill for interactively triaging the comments of
a GitHub PR review and turning them into a mitigation plan.

## What It Does

- Locates the review to triage: review URL, PR number, or a local report
- For every comment, shows the code context, any metadata, the verbatim
  comment, and a short opinion — with minimal cognitive load
- Asks for a decision on every comment: apply, select for further analysis,
  skip with comment, or skip without comment
- Compiles every decision into a mitigation plan to execute later
- Commit strategy follows review ownership: self-review fixes are folded
  into the existing commits (amend / fixup + autosquash); handling other
  reviewers' comments creates new commits

Triage is read-only: no code edits, no GitHub writes. The plan is executed
in a separate step, only when you ask for it.

## Repository Layout

```text
check-review/
├── SKILL.md
└── README.md
```

## Install

Clone the skill into your agent's skills directory:

```sh
git clone git@github.com:gfouillet/check-review-skill.git ~/.agents/skills/check-review
```

| Harness         | Directory                                            |
|-----------------|------------------------------------------------------|
| Agents standard | `~/.agents/skills/`                                  |
| OpenCode        | `~/.agents/skills/` or `~/.config/opencode/skills/`  |
| Claude Code     | `~/.claude/skills/`                                  |
| Copilot CLI     | `~/.copilot/skills/`                                 |

In OpenCode you can also call it through a command wrapper in
`~/.config/opencode/commands/check-review.md`:

```markdown
---
description: Triage review comments and build a mitigation plan
---
Load the following skill:
skill({ name: "check-review" })

Apply this skill to the target: $ARGUMENTS
```

## Requirements

- `gh` CLI authenticated against GitHub
- A local checkout of the repository helps for code context; the skill
  degrades gracefully to GitHub API content reads

## Usage

```
Check the review on PR 22403
```

```
/check-review https://github.com/juju/juju/pull/22403#pullrequestreview-1234567890
```

```
/check-review ~/.cache/juju-pr-review/pr-22403-report.md
```

## See Also

- [verify-review](https://github.com/gfouillet/verify-review-skill) — verify
  that review comments you gave were handled or answered
- [juju-pr-review](https://github.com/gfouillet/juju-pr-review) — produce a
  structured review of a PR (its report is a valid input here)
