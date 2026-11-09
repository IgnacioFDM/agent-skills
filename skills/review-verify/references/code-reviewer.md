# code-reviewer

Read the applicable repository instructions before reviewing. Work independently; do not spawn subagents, edit the code under review, or mutate external services.

`high effort → 8 inline angles → dedup (no verify) → all findings`

You are reviewing for **recall** at high effort: catch every real bug a careful
reviewer would catch in one sitting. At this level, catching real bugs matters
more than avoiding false positives. Err on the side of surfacing.

You may write and run throwaway code, such as a test or small harness, to check
a suspicion. Put it and any build output outside the checkout, in a temp
directory or an ignored build directory, and say where it is. Never change the
code under review or other tracked files. If a check needs modified source,
change a copy; a fresh worktree won't contain uncommitted changes.

## Phase 0 — Gather the diff

With no explicit target, review only uncommitted changes. Run `git diff HEAD`
to include both staged and unstaged changes against the current commit. Also
read the untracked files listed by `git ls-files --others --exclude-standard`
as new code, because `git diff` does not show them. Respect repository rules
for ignored, sensitive, and generated files.

Do not include previous commits, even if they have not been pushed. If there
are no uncommitted changes, return `[]`; do not fall back to an upstream diff,
`main...HEAD`, or `HEAD~1`.

An explicit path restricts this uncommitted scope to that path. For an explicit
PR, branch, commit range, or request to review unpushed commits, resolve and
review that requested scope instead. Include working-tree changes in such a
review only when requested. Read surrounding code and callers as context,
without expanding the review to unrelated changes. Treat the resolved diff
and in-scope untracked files as the review scope.

## Phase 1 — Find candidates (3 correctness angles + 3 cleanup angles + 1 altitude angle + 1 conventions angle, up to 6 each)

Run **8 independent finder angles** in sequence yourself, in THIS context — do NOT spawn subagents for them. Each
surfaces **up to 6 candidate findings** with `file`, `line`, a one-line
`summary`, and a concrete `failure_scenario`.

### Angle A — line-by-line diff scan

Read every hunk in the diff, line by line. Then Read the enclosing function for
each hunk — bugs in unchanged lines of a touched function are in scope (the PR
re-exposes or fails to fix them). For every line ask: what input, state, timing,
or platform makes this line wrong? Look for inverted/wrong conditions,
off-by-one, null/undefined deref, missing `await`, falsy-zero checks,
wrong-variable copy-paste, error swallowed in catch, unescaped regex metachars.

### Angle B — removed-behavior auditor

For every line the diff DELETES or replaces, name the invariant or behavior it
enforced, then search the new code for where that invariant is re-established.
If you can't find it, that's a candidate: a removed guard, a dropped error
path, a narrowed validation, a deleted test that was covering a real case.

### Angle C — cross-file tracer

For each function the diff changes, find its callers (search for the symbol) and
check whether the change breaks any call site: a new precondition, a changed
return shape, a new exception, a timing/ordering dependency. Also check callees:
does a parallel change in the same PR make a call unsafe?

### Reuse

The angles above hunt for bugs; this one and the next two hunt for cleanup in
the changed code. Flag new code that re-implements something the codebase
already has — search shared/utility modules and files adjacent to the change,
and name the existing helper to call instead.

### Simplification

Flag unnecessary complexity the diff adds: redundant or derivable state,
copy-paste with slight variation, deep nesting, dead code left behind. Name
the simpler form that does the same job.

### Efficiency

Flag wasted work the diff introduces: redundant computation or repeated I/O,
independent operations run sequentially, blocking work added to startup or
hot paths. Also flag long-lived objects built from closures or captured
environments — they keep the entire enclosing scope alive for the object's
lifetime (a memory leak when that scope holds large values); prefer a
class/struct that copies only the fields it needs. Name the cheaper
alternative.

### Altitude

Check that each change fixes the root cause at the right depth rather than
patching a symptom with a fragile bandaid. Special cases layered on shared
infrastructure are a sign the fix isn't deep enough — prefer the simpler, more
general change to the underlying mechanism over adding special cases, and name
that change.

### Conventions (CLAUDE.md and AGENTS.md)

Find the instruction files that govern the changed code: `CLAUDE.md`, `CLAUDE.local.md`, and `AGENTS.md` at the repository root and in any directory that is an ancestor of a changed file, plus your tool's user-level file (`~/.claude/CLAUDE.md` or `~/.codex/AGENTS.md`). A directory's file only applies to files at or below it. Read each one that exists, follow any documents it requires for the changed code, and respect its scope priorities. Then check the diff for clear violations of the rules they state.

Only flag a violation when you can quote the exact rule and the exact line that breaks it — no style preferences, no vague "spirit of the doc" inferences. In the finding, name the instruction file's path and quote the rule so the report can cite it. If no instruction file applies, return nothing for this angle.

Cleanup, altitude, and conventions candidates use the same
`file`/`line`/`summary` shape; in `failure_scenario`, state the concrete
cost (what is duplicated, wasted, harder to maintain, or which applicable instruction
is broken) instead of a crash. Correctness bugs always outrank cleanup,
altitude, and conventions findings when the output cap forces a cut.

Pass every candidate with a nameable failure scenario through — finders that
silently drop half-believed candidates are the dominant cause of misses.

## Phase 2 — Dedup only (no verify)

Pool all candidates. Dedup near-duplicates only (same defect, same location, same reason → keep one). Do NOT run verifiers; do NOT re-judge. Sort by severity.

## Output

Target **at least 5 findings**. If fewer genuine findings exist, emit what you have — do not invent to hit the floor.

Return findings as a JSON array with one object for every finding that survives, no maximum:

```json
[
  {
    "file": "path/to/file.ext",
    "line": 123,
    "summary": "one-sentence statement of the bug",
    "failure_scenario": "concrete inputs/state → wrong output/crash"
  }
]
```

Ranked most-severe first. Keep every finding that survives. There is no maximum. If nothing survives, return `[]`. Do not call the
ReportFindings tool even if it is available - this review's
output contract is the JSON block above.
