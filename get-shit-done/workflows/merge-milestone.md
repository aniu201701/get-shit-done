<purpose>
Merge a milestone branch into a target branch using squash merge. For conflicts, uses AI-assisted resolution: reads both sides' .planning/ context to understand intent, proposes merged code, and asks the user to confirm or provide guidance. After all conflicts are resolved, performs a full review of the resolution before committing. Designed for team mode multi-developer workflows.
</purpose>

<process>

## 1. Detect Current State and Determine Target Branch

```bash
CURRENT_BRANCH=$(git branch --show-current)
```

Parse `$ARGUMENTS` for target branch and `--dry-run` flag.

**If target branch is provided in `$ARGUMENTS`:** Use it as `$TARGET_BRANCH`.

**If target branch is NOT provided:** Ask the user to specify.

Use AskUserQuestion:

- header: "Target Branch"
- question: "Which branch should $CURRENT_BRANCH be merged into?"
- options:
  - "main" — merge into main
  - "master" — merge into master
  - "Let me specify" — I'll provide the branch name

**If "Let me specify":** Use AskUserQuestion:
- header: "Branch Name"
- question: "Enter the target branch name (e.g., milestone/v1.1-smart-rec)"
- requireAnswer: true

Store the result as `$TARGET_BRANCH`.

**Verify prerequisites:**

```bash
# Check we're not already on the target branch
if [ "$CURRENT_BRANCH" = "$TARGET_BRANCH" ]; then
  echo "Error: Already on $TARGET_BRANCH. Switch to your milestone branch first."
  echo "  git checkout milestone/v1.1  (or your branch name)"
  exit 1
fi

# Verify target branch exists
if ! git rev-parse --verify "$TARGET_BRANCH" >/dev/null 2>&1; then
  # Try fetching from remote
  git fetch origin "$TARGET_BRANCH" 2>/dev/null
  if ! git rev-parse --verify "$TARGET_BRANCH" >/dev/null 2>&1 && \
     ! git rev-parse --verify "origin/$TARGET_BRANCH" >/dev/null 2>&1; then
    echo "Error: Target branch '$TARGET_BRANCH' does not exist locally or on remote."
    exit 1
  fi
fi

# Check for uncommitted changes
if ! git diff --quiet || ! git diff --cached --quiet; then
  echo "Warning: You have uncommitted changes."
fi
```

## 2. Show What Will Be Merged

Display a summary of changes between the current branch and the target:

```bash
# Count commits
COMMIT_COUNT=$(git rev-list --count $TARGET_BRANCH..HEAD 2>/dev/null || echo "?")

# Show code-only diff stats (exclude .planning/ ephemeral files)
echo "## Changes to merge"
echo ""
echo "Branch: $CURRENT_BRANCH -> $TARGET_BRANCH"
echo "Commits: $COMMIT_COUNT"
echo ""
git diff --stat $TARGET_BRANCH...HEAD -- \
  ':!.planning/STATE.md' ':!.planning/STATE.md.lock' ':!.planning/.lock' ':!.planning/auto.lock' \
  ':!.planning/active-workstream' ':!.planning/WAITING.json' ':!.planning/metrics.json' \
  ':!.planning/continue.md' ':!.planning/*-CONTINUE.md' \
  ':!.planning/activity/' ':!.planning/runtime/' ':!.planning/worktrees/'
```

**If `--dry-run`:** Stop here and show the summary without merging.

## 3. Confirm Merge

Use AskUserQuestion:

- header: "Merge"
- question: "Ready to merge $CURRENT_BRANCH into $TARGET_BRANCH?"
- options:
  - "Squash merge" -- Combine all commits into one clean commit on $TARGET_BRANCH (recommended)
  - "Regular merge" -- Preserve commit history (merge commit)
  - "Cancel" -- Abort

**If "Cancel":** Exit.

## 4. Sync and Execute Merge

**Fetch the latest target branch:**

```bash
git fetch origin $TARGET_BRANCH 2>/dev/null || true
```

**Switch to target branch and merge:**

```bash
git checkout $TARGET_BRANCH
git pull origin $TARGET_BRANCH 2>/dev/null || true
```

**If "Squash merge":**

```bash
git merge --squash $CURRENT_BRANCH
```

**If "Regular merge":**

```bash
git merge $CURRENT_BRANCH --no-edit
```

**Why merge instead of rebase:** GSD workflows produce many atomic commits per phase. Rebase would replay each commit individually, requiring conflict resolution at every commit boundary — this means partial context per round, repeated resolution of the same files, and excessive iterations. Squash merge (or regular merge) surfaces **all conflicts at once** against the final state of both branches, giving the AI full context to resolve everything in a single pass.

## 5. Handle Conflicts

**If merge succeeds (no conflicts):**

```bash
# For squash merge, create the commit
if [ "$MERGE_TYPE" = "squash" ]; then
  git commit -m "feat: merge milestone $CURRENT_BRANCH

Squash merge of $COMMIT_COUNT commits from $CURRENT_BRANCH.

Co-Authored-By: Claude <noreply@anthropic.com>"
fi
```

Skip to Step 6.

**If merge has conflicts:**

```bash
CONFLICTS=$(git diff --name-only --diff-filter=U)
```

### 5a. Auto-resolve .planning/ ephemeral file conflicts

Ephemeral state files are per-developer — safe to auto-resolve by accepting the target branch version (ours, since we're now on target branch):

```bash
EPHEMERAL_FILES=(
  .planning/STATE.md
  .planning/STATE.md.lock
  .planning/.lock
  .planning/auto.lock
  .planning/active-workstream
  .planning/WAITING.json
  .planning/metrics.json
  .planning/continue.md
)

for file in "${EPHEMERAL_FILES[@]}"; do
  if git diff --name-only --diff-filter=U | grep -q "^$file$"; then
    git checkout --ours "$file" 2>/dev/null && git add "$file" 2>/dev/null
  fi
done

# Auto-resolve any *-CONTINUE.md conflicts
git diff --name-only --diff-filter=U | grep '\.planning/.*-CONTINUE\.md$' | while read f; do
  git checkout --ours "$f" 2>/dev/null && git add "$f" 2>/dev/null
done
```

**IMPORTANT: Do NOT delete any `.planning/` files from the working tree.** Even if a file like `STATE.md` is not in the shared git-tracked set, the user may have it locally. Only use `git checkout --ours` to resolve conflicts — never `git rm`.

### 5b. Gather branch context before conflict resolution

Before resolving any shared planning artifact or code conflict, **verify that you have sufficient context about both branches**. This context informs all subsequent resolution steps (5c, 5d, 5e).

**Collect from both sides:**

For each branch (target and source), read the available `.planning/` artifacts:

```bash
# Target branch (main/HEAD) versions — already checked out after merge
# Source branch versions — read from the merge's MERGE_HEAD
git show MERGE_HEAD:.planning/PROJECT.md > /tmp/source-PROJECT.md 2>/dev/null
git show MERGE_HEAD:.planning/ROADMAP.md > /tmp/source-ROADMAP.md 2>/dev/null
git show MERGE_HEAD:.planning/REQUIREMENTS.md > /tmp/source-REQUIREMENTS.md 2>/dev/null
git show MERGE_HEAD:.planning/MILESTONES.md > /tmp/source-MILESTONES.md 2>/dev/null

# STATE.md and research/ may not be tracked in team mode (per-developer artifacts).
# Attempt to read, but do NOT fail if unavailable.
git show MERGE_HEAD:.planning/STATE.md > /tmp/source-STATE.md 2>/dev/null || true
```

Read the target branch versions directly from `.planning/` (current working tree, ours side).

**Graceful degradation for untracked files:**

In team mode, the following `.planning/` files are per-developer (not git-tracked): `STATE.md`, `research/`, `reports/`, `forensics/`, `debug/`, `todos/`. When these are unavailable from either branch:

- **STATE.md unavailable** → Infer progress from ROADMAP.md (phase `[x]`/`[ ]` marks) and `.planning/phases/*-SUMMARY.md` (one-liner extracts). The key information lost is `Accumulated Context` — note this gap in the context summary and ask the user to supplement if needed.
- **research/ unavailable** → Not critical for conflict resolution. Key research conclusions are typically captured in PROJECT.md (Key Decisions) and REQUIREMENTS.md (scoping decisions). Skip without warning.
- **reports/, forensics/, debug/, todos/ unavailable** → Not relevant to conflict resolution. Skip without warning.

Do NOT treat any of these missing files as an error or "unrecoverable context" — these are expected gaps in team mode.

**Build a branch context summary:**

From the collected artifacts, identify:

1. **Each branch's milestone** — version, name, goal (from PROJECT.md `Current Milestone` section)
2. **Each branch's scope** — what requirements each branch was working on (from REQUIREMENTS.md REQ-IDs)
3. **Each branch's progress** — which phases were completed, what was shipped (from ROADMAP.md phase marks; if STATE.md available, also use `Last activity` timestamp; if unavailable, infer from phase SUMMARY.md files)
4. **Relationship between the two branches:**
   - **Independent milestones** — different goals, non-overlapping requirements (e.g., v1.1-auth vs v1.2-perf)
   - **Same milestone, different workstreams** — same goal, divided scope (e.g., both working on v1.1 but different phases)
   - **Overlapping scope** — partially or fully overlapping requirements (needs careful merge)
   - **Unknown** — cannot determine from available artifacts

**Present context summary to user for confirmation:**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD > MERGE CONTEXT
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Target ($TARGET_BRANCH):
  Milestone: v[X.Y] [Name]
  Scope: [REQ-IDs or description]
  Progress: [N] phases completed

Source ($CURRENT_BRANCH):
  Milestone: v[X.Y] [Name]
  Scope: [REQ-IDs or description]
  Progress: [N] phases completed

Relationship: [independent / same-milestone / overlapping / unknown]

Conflicting files:
  .planning/: [list]
  Code: [list]
```

**Use AskUserQuestion:**

- header: "Merge Context"
- question: "Is this understanding correct? This determines how conflicts will be resolved."
- options:
  - "Correct" — proceed with conflict resolution using this context
  - "Let me clarify" — user provides additional context about the branches
  - "Show me the details" — display full diff of both sides' key artifacts

**If "Let me clarify":** Use AskUserQuestion:
- header: "Branch Context"
- question: "What should I know about these two branches? (e.g., what each branch was responsible for, whether they touch the same features, any priorities)"
- requireAnswer: true

Incorporate the user's context and re-present the summary. Loop until "Correct".

**If "Show me the details":** Display key sections from both branches' PROJECT.md and REQUIREMENTS.md, then re-ask.

**If context is unrecoverable** (e.g., one branch has no .planning/ artifacts, or artifacts are too sparse to understand intent):

```
Cannot determine branch context automatically:
- [reason: e.g., "Source branch has no REQUIREMENTS.md", "Both branches have identical PROJECT.md"]

I need your help to understand the merge intent.
```

Use AskUserQuestion:
- header: "Missing Context"
- question: "Please describe: 1) What did $TARGET_BRANCH work on? 2) What did $CURRENT_BRANCH work on? 3) Any overlapping areas?"
- requireAnswer: true

Store the gathered context as `$BRANCH_CONTEXT` for use in 5c and 5e.

### 5c. Smart-merge shared planning artifacts (command-aware)

For shared `.planning/` documents, apply the **same update logic as the GSD command that normally manages that file**, informed by `$BRANCH_CONTEXT` from step 5b. These files are structured documents — git's line-based merge often produces nonsensical results.

**General principle:** Each GSD command encodes domain knowledge about how a file should be updated. Conflict resolution should reuse that knowledge rather than applying generic merge heuristics.

**Resolution mapping — which command logic to apply:**

| Conflicting File | Command Logic | What It Does |
|------------------|---------------|--------------|
| PROJECT.md | `complete-milestone` → `evolve_project_full_review` | Full evolution review: update What This Is, Core Value, merge Validated/Active/Out of Scope requirements from both sides, combine Key Decisions, update Context |
| ROADMAP.md | `complete-milestone` → `reorganize_roadmap` | Understand both milestones' phase structures, apply milestone grouping, preserve phase status marks from both sides, re-number if collision |
| REQUIREMENTS.md | `new-milestone` Step 9 (define requirements) | Understand category grouping and REQ-ID format, merge REQ-IDs from both sides preserving categories, merge traceability tables |
| STATE.md | `new-milestone` Step 5 / `quick` Step 7 | Merge Current Position (prefer more recent activity), combine Accumulated Context from both sides, merge Quick Tasks tables |
| MILESTONES.md | `complete-milestone` → `create_milestone_entry` | Both sides may have added entries — keep all sorted by version; **same-version entries must be merged** (see below) |
| config.json | Deep-merge | Merge JSON objects: scalars prefer ours, nested objects merge keys from both sides |
| codebase/* | `map-codebase` (post-merge re-run) | Generated snapshots are stale after merge — accept theirs for now, suggest re-running `/gsd:map-codebase` in Step 6 |
| research/* | Additive | Each milestone produces independent research docs — keep both, rename on filename collision (e.g., `api-design.md` → `api-design-v1.2.md`) |
| phases/* | Keep both | Different milestones have different phase numbers — keep all plans/summaries as independent artifacts |
| milestones/* | Keep all | Archive directories are version-named — keep all entries; **same-version archives must be merged** (see below) |

**Per-file resolution flow:**

For each conflicting `.planning/` file:

1. **Read the conflict file** with markers.
2. **Read both branches' versions** (from target and source, collected in 5b).
3. **Identify which command logic applies** (per table above).
4. **Apply that command's update logic** using `$BRANCH_CONTEXT`:

   **PROJECT.md — apply `evolve_project_full_review` logic:**
   - Read both sides' phase summaries and requirements
   - "What This Is": merge descriptions if both evolved the product differently
   - "Core Value": flag to user if both sides suggest different core values
   - "Validated": union of both sides' validated requirements (each with milestone version reference)
   - "Active": union, deduplicate by REQ-ID, flag conflicts if same REQ-ID has different descriptions
   - "Out of Scope": union, preserve reasoning from both sides
   - "Key Decisions": combine tables, sort chronologically, mark source milestone for each
   - "Context": synthesize from both sides' latest state
   - "Current Milestone": use source branch's version (the one being merged in) if it's newer; otherwise flag to user
   - **Same-version milestone handling:** If both branches worked on the same milestone version, after merging ROADMAP.md (which may renumber phases), cross-reference the merged ROADMAP.md to update any phase number references in PROJECT.md (e.g., "Completed in Phase 19" → "Completed in Phase 20" if renumbered). Scan "Validated", "Key Decisions", and "Context" sections for stale phase references.

   **ROADMAP.md — apply `reorganize_roadmap` logic:**
   - Identify which phases belong to which milestone on each side
   - Preserve `[x]`/`[ ]` status marks from both sides
   - Apply milestone grouping (`<details>` sections for completed milestones)
   - If both sides added phases with colliding numbers, re-number the source branch's phases to continue from target's last phase
   - Merge the Progress table with entries from both sides

   **MILESTONES.md — apply `create_milestone_entry` logic (same-version merge):**
   - If both branches have entries for the **same milestone version** (e.g., both have v0.5.0):
     - **Title**: Combine both titles to reflect the full scope (e.g., "Markdown Rendering" + "Message Timestamps" → "Markdown Rendering & Message Timestamps")
     - **Accomplishments**: Union of both sides' accomplishment lists
     - **Metrics** (phases completed, plans executed, tasks completed): Sum counts from both branches
     - **Date/Timeline**: Use the later completion date
   - If both branches have entries for **different versions**: Keep all, sorted by version number

   **milestones/* — same-version archive merge:**
   - If both branches archived the same milestone version (e.g., both have `milestones/v0.5.0/`):
     - **REQUIREMENTS.md archive**: Merge requirements from both branches (union of REQ-IDs and categories). Update the title to reflect combined scope.
     - **ROADMAP.md archive**: Merge phase lists from both branches. Re-number if phases collide.
     - **Other archive files**: Keep both, rename on collision (append branch scope identifier)
   - If both branches archived **different versions**: Keep all — no conflict

   **phases/* + milestones/* cross-directory reconciliation:**
   - After resolving both directories independently, check for inconsistencies:
     - If one branch archived phases to `milestones/vX.Y/` but the other branch left those same phases in `phases/`, unify them: move to `milestones/vX.Y/` (the archived location takes precedence)
     - Verify that every phase referenced in the merged ROADMAP.md exists in either `phases/` (active) or `milestones/` (archived) — flag any orphaned references

   **REQUIREMENTS.md — apply define-requirements logic:**
   - Parse both sides' category groupings and REQ-ID lists
   - Merge categories: union of all categories from both sides
   - Within each category: union of REQ-IDs, preserve checkbox status from both sides
   - Traceability table: merge rows, flag if same REQ-ID has different phase mappings
   - Future Requirements / Out of Scope: union from both sides

   **STATE.md — apply state-update logic:**
   - Current Position: use whichever side has the more recent `Last activity` timestamp
   - Accumulated Context: concatenate unique entries from both sides, deduplicate
   - Quick Tasks Completed: merge table rows, sort by date
   - Blockers/Concerns: union from both sides

5. **Present the resolution proposal to user:**

```
## Conflict: $FILE_PATH
**Resolving with:** [command name] logic

### What each branch contributed:
- **Target ($TARGET_BRANCH):** [summary from $BRANCH_CONTEXT]
- **Source ($CURRENT_BRANCH):** [summary from $BRANCH_CONTEXT]

### Merged result preview:
[Show key sections of the merged file, especially areas where both branches contributed]

### Merge decisions made:
- [Decision 1: e.g., "Combined Validated requirements from both milestones"]
- [Decision 2: e.g., "Re-numbered source phases 5-7 → 8-10 to avoid collision"]
- [Decision 3: e.g., "Flagged Key Decision conflict — both sides made different choices on auth strategy"]
```

6. **Ask user to confirm** — use AskUserQuestion per file:

- header: "Resolve: [filename]"
- question: "Review the merged result above. Accept or adjust?"
- options:
  - "Accept" — apply the merged result
  - "Let me provide context" — user will explain the intent, then re-merge
  - "Skip — I'll resolve manually" — leave this file for manual resolution

**If "Accept":** Write the merged file and `git add $FILE_PATH`.

**If "Let me provide context":** Ask what needs changing, re-apply command logic with user's guidance, re-present. Loop until accepted or skipped.

**If "Skip":** Leave unresolved, continue to next file.

### 5d. Identify remaining code conflicts

```bash
REMAINING_CONFLICTS=$(git diff --name-only --diff-filter=U)
```

If no remaining conflicts, skip to Step 5f.

### 5e. AI-assisted conflict resolution (context-aware)

For each remaining conflicting **code file**, use the `$BRANCH_CONTEXT` gathered in 5b to understand intent from both sides.

For each conflicting file:

1. **Read the conflict file** — use `Read` to see the full file with `<<<<<<<` / `=======` / `>>>>>>>` markers.

2. **Leverage `$BRANCH_CONTEXT`** — you already know what each branch was building. Use this to understand:
   - Which requirements drove this code change on the target side
   - Which requirements drove this code change on the source side
   - Read relevant `*-PLAN.md` files from `.planning/phases/` if the file path appears in any plan

3. **Analyze each conflict block** — for every `<<<<<<<` ... `>>>>>>>` section, determine:
   - What the **target branch** (HEAD) change intended (informed by target's requirements/plans)
   - What the **source branch** (milestone) change intended (informed by source's requirements/plans)
   - Whether the changes are **independent** (both can coexist), **overlapping** (same goal, different implementation), or **contradictory** (mutually exclusive)

4. **Generate a resolution proposal** — present to the user:

```
## Conflict: $FILE_PATH

### Block 1 (line $LINE)

Target ($TARGET_BRANCH) intent: [description, linked to requirement/plan]
Source ($CURRENT_BRANCH) intent: [description, linked to requirement/plan]
Relationship: independent / overlapping / contradictory

Proposed resolution:
​```
[merged code]
​```

Confidence: high / medium / low
Reason: [why this merge is correct, referencing $BRANCH_CONTEXT]

---
(repeat for each conflict block in the file)
```

5. **Ask the user to confirm** — use AskUserQuestion per file:

- header: "Conflict: [filename]"
- question: "Review the proposed resolution above. Accept or provide guidance?"
- options:
  - "Accept all" — apply the proposed resolution for this file
  - "Let me provide context" — user will explain the intent, then re-analyze
  - "Skip — I'll resolve manually" — leave this file for manual resolution

**If "Accept all":** Use `Edit` to replace the conflicted content with the proposed resolution, then `git add $FILE_PATH`.

**If "Let me provide context":** Use AskUserQuestion:
- header: "Additional Context"
- question: "What should the merged result look like for this file? (e.g., keep both features, prefer one side, combine differently)"
- requireAnswer: true

Re-analyze with the user's guidance and present an updated proposal. Repeat until accepted or skipped.

**If "Skip":** Leave the file unresolved and continue to the next conflicting file.

### 5f. Review all resolved conflicts

**Before committing, perform a full review** of all files that were resolved in steps 5a-5e. This catches issues that per-file resolution might miss: inconsistencies between files, broken cross-references, or integration problems.

```bash
# List all files that were resolved (staged after conflict resolution)
RESOLVED_FILES=$(git diff --cached --name-only)
```

**Review checklist:**

1. **Cross-file consistency** — Do resolved `.planning/` files reference each other correctly?
   - **Phase number alignment**: Verify phase numbers in PROJECT.md ("Validated in Phase N"), REQUIREMENTS.md (traceability table), and ROADMAP.md (phase headings) are consistent. If ROADMAP.md renumbered phases during merge, ensure PROJECT.md and REQUIREMENTS.md reflect the new numbers.
   - **Milestone version alignment**: Verify MILESTONES.md entries reference the correct phases and that `milestones/` archive directory names match.
   - **REQ-ID alignment**: Verify REQ-IDs in REQUIREMENTS.md, ROADMAP.md phase descriptions, and PROJECT.md Validated section are consistent.
2. **Code + planning alignment** — Do resolved code files match the intent described in the resolved `.planning/` files?
3. **No orphaned conflict markers** — Verify no `<<<<<<<`, `=======`, or `>>>>>>>` markers remain in any resolved file:

```bash
grep -rn '<<<<<<<\|=======\|>>>>>>>' $(git diff --cached --name-only) 2>/dev/null && echo "WARNING: Conflict markers found!" || echo "Clean — no conflict markers."
```

4. **Import / dependency coherence** — For resolved code files, check that imports, function signatures, and shared interfaces are consistent across files from both branches.

**Present review summary:**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD > MERGE REVIEW
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Resolved files: [N] total
  .planning/: [list]
  Code: [list]

Cross-file checks:
  - [check 1]: [pass/issue found]
  - [check 2]: [pass/issue found]
  - Conflict markers: [clean/found in X files]

[Issues found? describe each]
```

**If issues found:** Fix them before proceeding — edit the affected files and `git add` them. Re-run the review checklist after fixes.

**If review passes:** Proceed to 5g (impact assessment + testing).

### 5g. Impact assessment and post-merge testing

After the review in 5f confirms no structural issues, assess the **blast radius** of the resolved conflicts and run targeted tests before committing.

**Step 1: Identify impacted areas**

From the resolved code files and `$BRANCH_CONTEXT`, determine the impact scope:

```bash
# Get list of resolved code files (exclude .planning/)
RESOLVED_CODE=$(git diff --cached --name-only | grep -v '^\.planning/')
```

For each resolved code file, trace its impact:
- **Direct dependents** — which files import/require this file?
- **API surface** — did any exported function signatures, types, or interfaces change?
- **Shared state** — did any config, schema, or shared data structure change?

```bash
# Find files that import/reference the resolved files
for f in $RESOLVED_CODE; do
  basename=$(basename "$f" | sed 's/\.[^.]*$//')
  grep -rl "$basename" --include='*.ts' --include='*.js' --include='*.py' --include='*.go' --include='*.rs' --include='*.java' . 2>/dev/null | grep -v node_modules | grep -v '.planning/'
done | sort -u
```

**Present impact summary:**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD > IMPACT ASSESSMENT
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Resolved code files: [N]
  [file list]

Impact scope:
  - Direct dependents: [N] files
  - API changes: [yes/no — list changed signatures]
  - Config/schema changes: [yes/no — list]
  - Estimated risk: [low / medium / high]

Recommended test scope:
  - [test scope description based on impact]
```

**Step 2: Run targeted tests**

Use AskUserQuestion:

- header: "Post-Merge Testing"
- question: "Run tests to verify the merge didn't break anything?"
- options:
  - "Run tests" — execute the relevant test suite
  - "Skip tests" — I'll test manually later
  - "Let me specify" — I'll provide the test command

**If "Run tests":**

Auto-detect the test framework and run the appropriate scope:

```bash
# Detect and run tests based on project type
if [ -f "package.json" ]; then
  # Node.js — check for test script
  npm test 2>&1 || true
elif [ -f "pytest.ini" ] || [ -f "setup.py" ] || [ -f "pyproject.toml" ]; then
  # Python
  python -m pytest 2>&1 || true
elif [ -f "Cargo.toml" ]; then
  # Rust
  cargo test 2>&1 || true
elif [ -f "go.mod" ]; then
  # Go
  go test ./... 2>&1 || true
fi
```

If the project has a more targeted test command (e.g., running only tests related to the impacted files), prefer that over a full test run. Check `CLAUDE.md`, `package.json` scripts, or `Makefile` for project-specific test commands.

**If "Let me specify":** Use AskUserQuestion:
- header: "Test Command"
- question: "Enter the test command to run"
- requireAnswer: true

Execute the user-provided command.

**Handle test results:**

**If all tests pass:**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD > TESTS PASSED
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

All tests passed. Safe to commit.
```

Proceed to commit.

**If tests fail:**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD > TEST FAILURES
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[N] test(s) failed:
  [failure summary]

Likely cause: [analysis based on $BRANCH_CONTEXT and resolved files]
```

Use AskUserQuestion:
- header: "Test Failures"
- question: "Tests failed after conflict resolution. How to proceed?"
- options:
  - "Fix and re-test" — I'll fix the issues, then re-run tests
  - "Commit anyway" — failures are pre-existing, not caused by merge
  - "Abort merge" — discard and return to source branch

**If "Fix and re-test":** Analyze the failures in context of the resolved conflicts. If the failure is clearly caused by the merge resolution (e.g., missing import, incompatible types), fix the issue, `git add` the fix, and re-run the tests. Loop until tests pass or user decides to commit/abort.

**If "Commit anyway":** Proceed to commit. Note in the commit message that tests had pre-existing failures.

**If "Abort merge":**

```bash
git merge --abort
git checkout $CURRENT_BRANCH
```

Exit workflow.

**If "Skip tests":** Proceed directly to commit.

### 5h. Finalize and commit

Use AskUserQuestion:

- header: "Merge Review"
- question: "All conflicts resolved and reviewed. Ready to commit?"
- options:
  - "Commit" — finalize the merge
  - "Let me check" — I want to review some files manually first
  - "Abort" — discard merge and return to source branch

**If "Commit":**

```bash
UNRESOLVED=$(git diff --name-only --diff-filter=U)
```

**If no unresolved files:** Commit the merge.

```bash
if [ "$MERGE_TYPE" = "squash" ]; then
  git commit -m "feat: merge milestone $CURRENT_BRANCH

Squash merge of $COMMIT_COUNT commits from $CURRENT_BRANCH.

Co-Authored-By: Claude <noreply@anthropic.com>"
fi
```

**If "Let me check":** Wait for the user to review. When they confirm, proceed to commit.

**If "Abort":**

```bash
git merge --abort
git checkout $CURRENT_BRANCH
```

Exit workflow.

**If some files remain unresolved after review:**

```
The following files still need manual resolution:
$UNRESOLVED

After resolving, run:
  git add <resolved-files>
  git commit
```

Stop here — do not proceed to Step 6 until all conflicts are resolved.

## 6. Post-Merge Cleanup

```bash
echo ""
echo "## Merge Complete"
echo ""
echo "Branch $CURRENT_BRANCH merged into $TARGET_BRANCH."
echo ""
git log --oneline -1
echo ""
```

**If any `codebase/*` files were conflicted or present in the diff:**

Both branches' codebase snapshots reflect their own state — the merged codebase may differ from both. Suggest refreshing:

Use AskUserQuestion:
- header: "Codebase Map"
- question: "Codebase map files may be stale after merge. Refresh now?"
- options:
  - "Re-map now" — run `/gsd:map-codebase` to refresh
  - "Skip" — I'll do it later

**If "Re-map now":** Instruct the user to run `/gsd:map-codebase` after this workflow completes (fresh context window recommended).

Use AskUserQuestion:

- header: "Cleanup"
- question: "Delete the milestone branch?"
- options:
  - "Delete branch" -- Remove $CURRENT_BRANCH (local only)
  - "Keep branch" -- Leave it for reference

**If "Delete branch":**

```bash
git branch -d $CURRENT_BRANCH
```

## 7. Push (Optional)

Use AskUserQuestion:

- header: "Push"
- question: "Push $TARGET_BRANCH to remote?"
- options:
  - "Push" -- Push to origin
  - "Skip" -- I'll push later

**If "Push":**

```bash
git push origin $TARGET_BRANCH
```

## 8. Summary

```
---

## Merge Complete

| Detail | Value |
|--------|-------|
| Source | $CURRENT_BRANCH |
| Target | $TARGET_BRANCH |
| Type | $MERGE_TYPE |
| Commits | $COMMIT_COUNT |
| Branch deleted | Yes/No |
| Pushed | Yes/No |

Next: Continue with /gsd:progress or /gsd:new-milestone
```

</process>

<success_criteria>
- [ ] Target branch determined: from $ARGUMENTS if provided, otherwise user selected/specified
- [ ] Target branch verified to exist (local or remote)
- [ ] Squash or regular merge executed (merge, NOT rebase — all conflicts surfaced at once)
- [ ] No .planning/ files deleted from working tree (ephemeral conflicts resolved via git checkout --ours, not git rm)
- [ ] .planning/ ephemeral state conflicts auto-resolved (accept ours / target branch version)
- [ ] Branch context gathered before conflict resolution: both branches' milestones, scope, progress, and relationship identified
- [ ] User confirmed branch context understanding (or provided clarification when context was incomplete)
- [ ] Shared planning artifacts resolved using command-aware logic (PROJECT.md via evolve_project_full_review, ROADMAP.md via reorganize_roadmap, REQUIREMENTS.md via define-requirements, STATE.md via state-update, etc.)
- [ ] Same-version milestone handling: if both branches completed the same milestone, MILESTONES.md entries merged (combined title/accomplishments/metrics), milestones/* archives merged, phases/* cross-reconciled, and phase numbers in PROJECT.md/REQUIREMENTS.md updated to match merged ROADMAP.md
- [ ] Code conflicts analyzed with AI using gathered branch context: intent from both branches' requirements and plans
- [ ] Resolution proposals presented per file with command logic explanation and confidence level
- [ ] User confirmed or provided guidance for each conflict
- [ ] Full review of all resolved files performed before committing (cross-file consistency, conflict markers, code+planning alignment)
- [ ] Impact assessment performed: resolved code files' dependents, API surface changes, and risk level identified
- [ ] Post-merge tests executed (or user explicitly skipped); test failures analyzed in context of merge resolution
- [ ] User confirmed review and test results before final commit
- [ ] Codebase map refresh suggested if codebase/* files were conflicted
- [ ] Unresolved files clearly listed with manual resolution instructions
- [ ] Optional branch cleanup and push
- [ ] Clear summary of what was merged
</success_criteria>
