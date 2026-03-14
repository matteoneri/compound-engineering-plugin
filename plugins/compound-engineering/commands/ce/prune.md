---
name: ce:prune
description: Detect and triage stale docs/solutions/ entries whose referenced code has changed
argument-hint: "[optional: path to specific doc or category to check]"
---

# /ce:prune

Audit the `docs/solutions/` knowledge base for stale entries and interactively triage them.

## Purpose

Over time, documented solutions become outdated as code is refactored, dependencies change, and architecture evolves. The `learnings-researcher` agent surfaces these docs during `/ce:plan` as if they were current. `/ce:prune` detects stale entries and lets you decide what to do with them.

**Why prune?** Stale documentation is worse than no documentation — it gives wrong advice with the authority of institutional knowledge.

## Usage

```bash
/ce:prune                              # Scan all docs/solutions/
/ce:prune docs/solutions/performance-issues/  # Scan a specific category
```

## Execution Flow

### Phase 1: Prerequisites

Check that the environment supports staleness detection:

```bash
# Must be a git repository
git rev-parse --is-inside-work-tree 2>/dev/null
```

**If not a git repo:** Exit with message:
```
/ce:prune requires a git repository with commit history. Not a git repository.
```

**If docs/solutions/ does not exist:** Exit with message:
```
No docs/solutions/ directory found.

To start building your knowledge base, run /ce:compound after solving non-trivial problems.
The staleness detector will help keep it accurate over time.
```

**If docs/solutions/ is empty (no .md files):** Exit with message:
```
docs/solutions/ exists but contains no documentation files. Nothing to prune.
```

### Phase 2: Scan

Invoke the staleness-detector agent to scan the knowledge base:

```
Task staleness-detector(docs/solutions/)
```

If a specific path was provided as argument, pass it to the agent to narrow the scan.

**Wait for the agent to complete** before proceeding to Phase 3.

### Phase 3: Report Summary

Present the scan results summary:

```
Staleness scan complete.

Scanned: X documents
Potentially stale: Y
Unable to verify: Z (no file references)
Current: W

[If Y == 0]: All documents are current. No action needed.
[If Y > 0]: Ready to triage Y stale entries. Proceed?
```

If no stale entries, exit here.

### Phase 4: Interactive Triage

Present stale entries one at a time, ordered by staleness severity (high first).

For each stale entry, use **AskUserQuestion** to present the triage options:

```
question: "[1/Y] docs/solutions/category/filename.md\nDate: YYYY-MM-DD (N days ago)\nStale: path/to/file.rb (N commits), path/to/other.rb (deleted)\n\nWhat should we do with this entry?"
header: "Prune"
options:
  - label: "Keep"
    description: "Mark as reviewed today. Add last_reviewed: YYYY-MM-DD to frontmatter. No other changes."
  - label: "Update"
    description: "Display the doc and update it to reflect current code. Updates date to today."
  - label: "Archive"
    description: "Move to docs/solutions/.archived/. Excluded from future learnings-researcher searches."
  - label: "Delete"
    description: "Remove permanently via git rm."
```

**Action implementations:**

#### Keep
Read the file, add or update `last_reviewed: YYYY-MM-DD` in the YAML frontmatter using the Edit tool. This resets the staleness clock — the entry will not be flagged again until referenced files accumulate more changes.

#### Update
1. Display the full document content to the user
2. Use AskUserQuestion: "What changed? Describe the update needed, or say 'edit manually' to skip."
3. If the user describes changes: apply edits to the doc using the Edit tool
4. Update the `date:` field in frontmatter to today's date
5. If the user says "edit manually": print the file path and move to the next entry

#### Archive
```bash
# Preserve directory structure under .archived/
RELATIVE_PATH="<path relative to docs/solutions/>"
ARCHIVE_DIR="docs/solutions/.archived/$(dirname "$RELATIVE_PATH")"
mkdir -p "$ARCHIVE_DIR"
mv "<full_path>" "$ARCHIVE_DIR/"
```

The `.archived/` directory is excluded from `learnings-researcher` searches by convention — files there do not appear during `/ce:plan`.

#### Delete
Use AskUserQuestion for confirmation:
```
question: "Permanently delete docs/solutions/category/filename.md? This will run git rm."
header: "Confirm"
options:
  - label: "Yes, delete"
    description: "Remove the file permanently. Git history preserves it if needed later."
  - label: "No, skip"
    description: "Keep the file and move to the next entry."
```

If confirmed:
```bash
git rm "<path>"
```

### Phase 4b: Batch Operations

**If more than 10 stale entries remain** after triaging the first 3 individually, offer batch operations:

```
question: "N entries remaining. Apply a batch action to all remaining?"
header: "Batch"
options:
  - label: "Continue one by one"
    description: "Keep triaging entries individually."
  - label: "Keep all remaining"
    description: "Add last_reviewed to all remaining entries."
  - label: "Archive all remaining"
    description: "Move all remaining to .archived/"
```

### Phase 5: Summary

After all entries are triaged, present the final summary:

```
Pruning complete.

  Kept (reviewed): X
  Updated: Y
  Archived: Z
  Deleted: W
  Skipped: V

[If Y > 0]: Some docs were updated. Consider running /ce:compound if any
solutions need to be fully rewritten with current context.

[If Z > 0]: Archived docs are in docs/solutions/.archived/ and excluded
from learnings-researcher. Restore with: mv docs/solutions/.archived/<path> docs/solutions/<path>
```

### Phase 6: Critical Patterns Check

If the staleness-detector reported stale critical patterns:

```
Additionally, N critical pattern(s) in docs/solutions/patterns/critical-patterns.md may be outdated:

- Pattern 3: "Documented in:" link broken (file deleted)
- Pattern 7: CORRECT code block not found in codebase

Critical patterns are read on EVERY /ce:plan. Stale patterns here are the most damaging.
Edit critical-patterns.md to update or remove these patterns.
```

Display the file path and offer to open it for editing.

## Preconditions

<preconditions enforcement="hard">
  <check condition="git_repository">
    Must be inside a git repository with commit history.
  </check>
  <check condition="docs_solutions_exists">
    docs/solutions/ directory must exist with at least one .md file.
  </check>
</preconditions>

## Related Commands

- `/ce:compound` — create new solution docs (the input to this system)
- `/ce:review` — runs staleness-detector as advisory agent alongside code review
- `/ce:plan` — consumes docs/solutions/ via learnings-researcher (benefits from pruning)
