---
name: review-verify-uncommitted
description: Standalone review of uncommitted changes across sessions, using a fresh reviewer and a separate verifier. No implementation-session summary; stops at verdicts.
argument-hint: "[optional path; default: all uncommitted changes]"
disable-model-invocation: true
---

Run a standalone review of uncommitted work, then independently verify its findings. Use two separate fresh agents sequentially and wait for each. Do not review or verify the code yourself, and do not edit it during this skill.

Review all staged and unstaged changes against `HEAD`, plus untracked files. An optional path narrows that scope. Exclude previous commits, including unpushed commits. If nothing is uncommitted in scope, report nothing to review; do not fall back to a commit range.

This workflow does not depend on an implementation session. Do not prepare a summary of this thread's work or retrieve earlier conversations. The diff, applicable repository instructions, surrounding code, tests, and documented contracts provide the review context. Include explicit requirements supplied with this invocation, if any, without inventing missing requirements, approvals, authorship, or rationale.

## Starting the agents

Start each agent with no conversation history, on the current model, at high effort. Give it the absolute path of its role file in the `references/` folder of the sibling `review-verify` skill (`../review-verify/references/` from this file), plus only that step's inputs.

In Claude Code, use the Agent tool with the `general-purpose` type and `run_in_background: false`. In Codex, use `spawn_agent` with `fork_turns: "none"`, then wait for it. If the host requires a delegation tool, use its fresh-child equivalent and follow its rules. Never create top-level threads or shell-launched agents. If fresh agents are unavailable, say so and stop.

## 1. Review

Start a fresh agent with the `code-reviewer.md` role.

Give it only the repository's absolute working directory and this target:

> Review uncommitted changes only: staged changes, unstaged changes, and untracked files. Optional path restriction: TARGET. Exclude previous commits. No implementation-session context is supplied.

Replace `TARGET` with the path from this invocation's arguments (`$ARGUMENTS` in Claude Code, the text after the skill name in Codex), or `none` for all uncommitted work. If the reviewer returns no findings, tell the user and stop.

## 2. Verify

Start a second fresh agent with the `review-verifier.md` role.

Give it:

- The same repository and review scope.
- The review findings, verbatim.
- Any requirements explicitly supplied with this invocation, quoted verbatim; otherwise state that no implementation requirements were supplied.

Tell it to verify against the actual code and repository contracts without assuming the user approved the existing behavior. Missing session history alone does not make a finding `UNPROVEN` or `YOUR CALL`; identify the specific unresolved fact or product decision when using either verdict.

## 3. Report

Keep the verifier's full report in this thread, but show only:

1. One sentence counting the verdicts.
2. Each entry from its `## Plain summary` section as written, ordered `FIX` (in the recommended implementation order), `YOUR CALL`, `UNPROVEN`, `LEAVE`, `NOT REAL`. Number the entries continuously from 1 across all groups.
3. Any evidence you already have against a verdict, after that finding, in plain words.
4. A closing note that the evidence is available on request.

All uncommitted changes in the requested scope are part of this review, regardless of which session produced them. Do not label findings as outside this thread's work. Do not fix anything until the user asks.
