---
name: review-verify
description: Review uncommitted changes by default in a fresh agent, then check each finding in a second fresh agent. Stops at the verdicts.
argument-hint: "[PR, branch or path to review; default: uncommitted changes only]"
disable-model-invocation: true
---

Run a review and then verify it, using a separate fresh agent for each so neither is influenced by this thread's work. Run each agent in the foreground and wait for it. Don't review or verify anything yourself, and don't edit any code during this skill.

With no target, review only uncommitted changes: staged changes, unstaged changes, and untracked files. Previous commits, including unpushed commits, are excluded unless explicitly requested. If there are no uncommitted changes, report that there is nothing to review; do not fall back to a commit range.

## Starting the agents

Start each agent with no conversation history, on the current model, at high effort. Give it the absolute path of its role file in this skill's `references/` folder, the repository's absolute working directory, and that step's inputs.

In Claude Code, use the Agent tool with the `general-purpose` type and `run_in_background: false`. In Codex, use `spawn_agent` with `fork_turns: "none"`, then wait for it, and use the text after the skill name as the target in step 2. If the host requires a delegation tool, use its fresh-child equivalent and follow its rules. Never create top-level threads or shell-launched agents. If fresh agents are unavailable, say so and stop.

## 1. Summarize this thread's work

Write a summary for the verifier in the parts below. Keep it clear throughout who decided what: the verifier treats only the user's words as requirements and checks everything else like the code.

- **Task:** one or two sentences.
- **What the user said:** every message in which the user asked for something, decided something or approved a proposal, quoted verbatim. When they approved something you proposed, quote the proposal too. Don't paraphrase these. If the conversation was summarized and you no longer have a message's exact words, say so.
- **How you read it:** where the user's words allow more than one reading, the one you implemented and the others.
- **Your own choices:** every decision the user didn't make, including known gaps you left, each with your reason. Write each in the first person, such as "I kept one shared cache rather than one per tenant, because…". Avoid words that hide who decided, such as "accepted", "deliberate" or "by design".
- **Files changed.**
- **Not checked:** what you didn't test.

If this thread changed nothing, say so. The review agent doesn't get this summary.

## 2. Review

Start a fresh agent with the `code-reviewer.md` role and this input only:

> Review target (if empty, gather the diff as your instructions describe): $ARGUMENTS

If it reports no findings, tell the user and skip step 3.

## 3. Verify

Start a second fresh agent with the `review-verifier.md` role and a prompt containing:

- What was reviewed: the target given above, or else only uncommitted changes (staged, unstaged, and untracked files), excluding previous commits.
- Your summary from step 1, labeled as the implementer's own account, to be checked rather than trusted.
- The findings, verbatim.

## 4. Report

Keep the verifier's full report in this thread for answering questions and making fixes, but don't show it to the user unless they ask. Report to the user like this:

1. Open with the outcome in one sentence, such as "Of 9 findings: 2 to fix, 1 needs your decision, 6 need no change."
2. List the findings in this order: `FIX` (in the verifier's recommended order), `YOUR CALL`, `UNPROVEN`, `LEAVE`, `NOT REAL`. For each one give the entry from the verifier's `## Plain summary` section as written. Number the entries continuously from 1 across all groups.
3. Mark any finding in a file this thread didn't change as outside this thread's work.
4. If you have evidence against a verdict, add it after that finding, in equally plain words.
5. End by saying the evidence for any finding is available on request.

Don't fix anything until the user says so.
