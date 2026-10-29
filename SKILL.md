---
name: check-review
description: >
  Interactively triage the comments of a GitHub PR review and produce a
  mitigation plan. Use when asked to handle review comments, check a review,
  triage review feedback, prepare mitigations for review comments, or go
  through a review comment by comment. Trigger on: "check the review",
  "handle the review comments", "triage the review", "prepare mitigations",
  "go through the review", "address the review comments". Also consumes
  review reports produced by the juju-pr-review skill. Works on any GitHub
  repository.
---

# Check Review Skill

Go through every comment of a GitHub PR review one by one, help the user
understand each comment with minimal cognitive load, record a decision per
comment, and compile all decisions into a mitigation plan that is executed
later.

## Hard Rules

- Triage is read-only: do not edit files, do not post comments, do not
  mutate the PR or the repository while triaging. The plan is executed
  later, in a separate step, only if the user asks for it.
- Show the verbatim text of every comment — never paraphrase what the
  reviewer wrote.
- Ask the user for a decision on every comment. Never decide on their
  behalf and never silently skip a comment.
- If the input is a local report file instead of a GitHub review, triage
  its findings the same way.

## Input Contract

The target can be:

- A review URL:
  `https://github.com/<owner>/<repo>/pull/<N>#pullrequestreview-<id>`
- A PR URL or PR number, optionally as `<owner>/<repo>#<N>`; if the PR has
  several reviews, use the most recent one unless the user says otherwise
- A path to a local review report (for example the report produced by the
  `juju-pr-review` skill) — triage its Finding Summary table
- No argument: detect the PR of the current branch; if it has a recent
  review, use it; otherwise ask the user which review to triage

If the repository cannot be determined from the input or the current
checkout, ask the user.

## Workflow

### Phase 1 — Collect the comments

From a GitHub review:

```sh
gh api repos/<owner>/<repo>/pulls/<N>/reviews      # locate the review
gh api repos/<owner>/<repo>/pulls/<N>/comments --paginate
```

Keep comments whose `pull_request_review_id` matches the target review.
Replies are comments whose `in_reply_to_id` matches one of those — a reply
from the author may already resolve a comment; note it and confirm with the
user. Also read the review body itself (general comments that are not
attached to a line).

From a local report: read the report file and extract one entry per finding
(severity, file:line, description, suggested fix).

### Phase 2 — Per-comment interactive loop

For each comment, in review order:

1. Gather and display the context:
   - The code around the comment (if it is attached to source code): at
     least 10 lines around the commented line, extended back to the method
     signature when possible. Annotate the displayed code to highlight the
     relevant part. The goal is to make the comment understandable without
     burning the user's cognitive load.
   - Any metadata: severity, suggested fix, related findings.
   - The verbatim comment.
   - Your opinion of the comment, in no more than 5 lines: is it valid, how
     important is it, roughly how much work a fix would be.
2. Prompt the user for a decision. Use your harness's interactive question
   mechanism (OpenCode `question` tool, Claude Code `AskUserQuestion`; if
   none exists, print numbered options and wait for the reply). Offer these
   options and recommend one when you can:
   - **Apply** — there is a quick fix for the comment
   - **Select for further analysis** — the comment needs investigation
     before it can be decided
   - **Skip with comment** — no code change; draft a reply to the reviewer
   - **Skip without comment** — no action at all
3. Record the decision plus any user notes, then move to the next comment.
   Keep momentum: gather context for the next comment while the current one
   is still fresh.

### Phase 3 — Compile the mitigation plan

Once every comment has a decision, gather them into a single plan that will
be executed later:

- **Fixes to apply** — per file: comment, location, what to change
- **Items for further analysis** — what to investigate and why
- **Replies to draft** — a draft reply per skipped-with-comment item (draft
  only; do not post anything)
- **Skipped** — items with no action

Present the plan and stop there unless the user asks you to execute it
(see Execution Rules below).

## Execution Rules

The plan is applied only when the user asks for it. How changes are
committed depends on whose review is being handled:

### Self-review — fold fixes into the existing commits

When the triaged review is the user's own review of their own work (or a
local report produced by the `juju-pr-review` skill in self-review mode),
do not create new "address review" commits. Keep the history clean by
amending the existing ones:

- Single-commit branch: apply the fixes and `git commit --amend` into the
  existing commit.
- Multi-commit branch: stage each fix as `git commit --fixup=<sha>` of the
  commit that introduced the code, then fold them with
  `GIT_SEQUENCE_EDITOR=true git rebase -i --autosquash <base>`.
- Rewriting history is destructive. Before rebasing: confirm the working
  tree is clean, the branch belongs to the user, and a force-push is
  acceptable — use `--force-with-lease`, never plain `--force`. If other
  reviewers have comments on the PR, fall back to new commits unless the
  user explicitly accepts the rewrite.

### Someone else's review — new commits

When handling a review from another reviewer on the user's PR, create new
commits with clear messages addressing the comments. Never amend or
rebase: the reviewer needs to see the delta as new commits.

## Harness Adaptation

- **Subagents**: the loop is interactive, so run it in the main session. If
  your harness supports subagents with a model choice, you may delegate the
  read-only context gathering (step 1) to a fast, cheap model and keep the
  decision prompts in the main session.
- **Interactive questions**: use the harness's native mechanism (OpenCode
  `question` tool, Claude Code `AskUserQuestion`); fall back to numbered
  options printed in the chat.

## See Also

- `verify-review` — check that review comments *you* gave were handled
- `juju-pr-review` — produce the review report this skill can consume
