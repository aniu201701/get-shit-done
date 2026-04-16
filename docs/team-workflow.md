# GSD Team Workflow

## Team Structure

```
Agent Repo                              Frontend Repo
┌────────────────────────────┐          ┌────────────────────────────┐
│  Unit 1                    │          │  Unit 1                    │
│  ├── Driver (coordination) │          │  └── FE                    │
│  ├── BE                    │          │                            │
│  └── Algorithm             │          │  Unit 2                    │
│                            │          │  └── FE                    │
│  Unit 2                    │          └────────────────────────────┘
│  ├── Driver (coordination) │
│  ├── BE                    │
│  └── Algorithm             │
└────────────────────────────┘
```

- 2 repos: Frontend, Agent
- 2 units, each with 1 FE, 1 BE, 1 Algorithm (BE + Algorithm work on Agent repo)
- Each unit has a Driver who breaks down requirements and assigns tasks
- Driver's work happens offline (meetings, discussions) — not through GSD

## Prerequisites (One-Time Setup)

### Enable Team Mode

```bash
# Option A: During project init
/gsd:new-project    # → select "Team Mode"

# Option B: Existing project
/gsd:settings       # → Team Mode → Yes
```

This enables:
- `unique_milestone_ids` — prevents file collisions across developers
- `push_branches` — pushes milestone branches to remote
- `pre_merge_check` — validates before merge
- `.planning/.gitignore` — tracks shared docs, ignores personal state

## Workflow

### Phase 1: Requirements Breakdown (Driver, Offline)

Driver leads team discussion to produce:

| Output | Description |
|--------|-------------|
| Phases | How many phases, what each phase contains |
| Assignment | Who owns each phase |
| Dependencies | Which phases depend on others |
| Timeline | Deadlines per phase |
| Merge order | Who merges first, who merges later |

Driver records the meeting (or takes notes), then uses AI to generate a structured context document:

```markdown
<!-- docs/MILESTONE-CONTEXT-v1.1.md -->

# Milestone v1.1: Smart Recommendation System

## Objective
Build real-time personalized recommendation with user profiling and ranking model.

## Phase Breakdown

### Phase 1: User Service API (Owner: BE)
- User profile CRUD API
- Behavior log collection endpoint
- Dependencies: none (can start immediately)
- Deadline: 4/3

### Phase 2: Ranking Model (Owner: Algorithm)
- Training pipeline for ranking model
- Online inference service
- Dependencies: Phase 1 (needs user profile API schema)
- Deadline: 4/5

### Phase 3: Recommendation Dashboard (Owner: FE, Frontend Repo)
- Recommendation performance dashboard
- A/B experiment configuration page
- Dependencies: Phase 2 (needs inference API)
- Deadline: 4/7

## Merge Order
1. BE (Phase 1) → milestone/v1.1
2. Algorithm (Phase 2) → rebase → milestone/v1.1
3. Integration testing on milestone/v1.1
4. milestone/v1.1 → main
5. FE (Phase 3) → Frontend repo main (after Agent repo API is deployed)

## Timeline
- Development: 3/27 - 4/5
- Integration: 4/6 - 4/7
- Release: 4/8
```

Driver commits this to the repo:

```bash
git checkout main && git pull
git checkout -b milestone/v1.1-smart-rec
cp MILESTONE-CONTEXT-v1.1.md docs/
git add docs/MILESTONE-CONTEXT-v1.1.md
git commit -m "docs: add milestone v1.1 context"
git push -u origin milestone/v1.1-smart-rec
```

### Phase 2: Individual Development (Each Developer)

Each developer treats their assigned phase as a **standalone GSD milestone**.

#### BE starts Phase 1:

```bash
# Branch off the milestone branch
git fetch origin
git checkout milestone/v1.1-smart-rec
git checkout -b milestone/v1.1-smart-rec/phase-1-user-service

# Read the context document to understand the full picture,
# then start GSD — scope to own phase only
/gsd:new-milestone
# When Claude asks what this milestone is about:
# "Refer to docs/MILESTONE-CONTEXT-v1.1.md, I'm responsible for Phase 1: User Service API"

# Normal GSD flow
/gsd:discuss-phase
/gsd:plan-phase
/gsd:execute-phase
# (repeat for each sub-phase if needed)
/gsd:verify-work
```

#### Algorithm starts Phase 2 (in parallel):

```bash
git fetch origin
git checkout milestone/v1.1-smart-rec
git checkout -b milestone/v1.1-smart-rec/phase-2-ranking-model

/gsd:new-milestone
# "Refer to docs/MILESTONE-CONTEXT-v1.1.md, I'm responsible for Phase 2: Ranking Model"

/gsd:discuss-phase
/gsd:plan-phase
/gsd:execute-phase
/gsd:verify-work
```

#### FE starts Phase 3 (Frontend Repo, in parallel):

```bash
# In Frontend repo
git checkout main && git pull
git checkout -b milestone/v1.1-smart-rec/phase-3-dashboard

/gsd:new-milestone
# "Refer to docs/MILESTONE-CONTEXT-v1.1.md (in Agent repo), I'm responsible for Phase 3: Dashboard"

/gsd:discuss-phase
/gsd:plan-phase
/gsd:execute-phase
```

### Phase 3: Merge to Milestone Branch (Per Merge Order)

Follow the merge order defined by the Driver.

#### Step 1: BE merges first

```bash
# On branch milestone/v1.1-smart-rec/phase-1-user-service
/gsd:merge-milestone milestone/v1.1-smart-rec
```

What happens:
1. Fetch latest `milestone/v1.1-smart-rec`
2. No new commits from others → clean merge
3. Squash merge BE's work into `milestone/v1.1-smart-rec`
4. Push

#### Step 2: Algorithm merges second

```bash
# On branch milestone/v1.1-smart-rec/phase-2-ranking-model
/gsd:merge-milestone milestone/v1.1-smart-rec
```

What happens:
1. Fetch latest `milestone/v1.1-smart-rec` (now contains BE's code)
2. **Step 2.5 triggers**: rebase onto BE's changes
3. If conflicts: AI-assisted resolution (reads both sides' .planning/ context)
4. After rebase: Algorithm's branch now includes BE's code
5. Squash merge into `milestone/v1.1-smart-rec`

### Phase 4: Integration Testing

Integration happens on the `milestone/v1.1-smart-rec` branch, which now contains both BE and Algorithm code.

```bash
git checkout milestone/v1.1-smart-rec
# Run tests, start services, verify integration
npm test
npm run dev
```

If issues are found:
- Fix directly on `milestone/v1.1-smart-rec`
- Or create a short-lived fix branch, merge back

### Phase 5: Merge to Main

Before merging, check if main has moved (other Unit may have shipped):

```bash
git checkout milestone/v1.1-smart-rec
/gsd:merge-milestone main
```

What happens:
1. **Step 2.5**: fetch main, detect if other milestones have been merged
2. If main has new commits: rebase `milestone/v1.1-smart-rec` onto main
3. `.planning/` conflicts:
   - Ephemeral files (STATE.md, locks, etc.) → auto-resolved
   - Shared artifacts (PROJECT.md, ROADMAP.md) → semantic merge strategy
4. Code conflicts → AI-assisted resolution, team confirms
5. Squash merge to main
6. Push

### Phase 6: Frontend Integration

After Agent repo is deployed with the new APIs:

```bash
# In Frontend repo
# FE can now test against real APIs
/gsd:verify-work

# Merge to main
/gsd:merge-milestone main
```

## Cross-Unit Coordination

When two Units work in parallel on the same repo:

```
Agent Repo main ──────────────────────────────────────────────────
  │                    ↑                              ↑
  ├── milestone/v1.1 (Unit 1) ───────────────────────┘|
  │   ├── phase-1-user-service (BE)                   │
  │   └── phase-2-ranking-model (Algo)                │
  │                                                    │
  └── milestone/v1.2 (Unit 2) ────────────────────────┘
      ├── phase-1-payment (BE)
      └── phase-2-fraud-detection (Algo)
```

**Merge order between Units** is coordinated by Drivers:
- Unit that finishes first merges to main first
- Unit that merges second: rebase picks up the first Unit's code
- If there are cross-Unit code conflicts (shared config, routes, schemas): Drivers arrange a joint session where team members resolve conflicts together — AI merges `.planning/` files, humans + AI merge code

## Branch Naming Convention

```
milestone/v{version}-{name}                          # Team milestone branch (Driver creates)
milestone/v{version}-{name}/phase-{n}-{description}  # Individual developer branch
```

Examples:
```
milestone/v1.1-smart-rec                             # Unit 1 milestone
milestone/v1.1-smart-rec/phase-1-user-service        # BE's work
milestone/v1.1-smart-rec/phase-2-ranking-model       # Algorithm's work

milestone/v1.2-payments                              # Unit 2 milestone
milestone/v1.2-payments/phase-1-payment-api          # BE's work
milestone/v1.2-payments/phase-2-fraud-detection      # Algorithm's work
```

## GSD Commands Per Role

| Role | Phase | Command |
|------|-------|---------|
| Driver | Create milestone branch | `git checkout -b milestone/v1.1-xxx` |
| Driver | Push context document | `git add docs/MILESTONE-CONTEXT-v1.1.md && git push` |
| Developer | Start own phase | `git checkout -b milestone/v1.1-xxx/phase-n-yyy` → `/gsd:new-milestone` |
| Developer | Plan | `/gsd:discuss-phase` → `/gsd:plan-phase` |
| Developer | Execute | `/gsd:execute-phase` (loop) |
| Developer | Verify | `/gsd:verify-work` |
| Developer | Merge to milestone | `/gsd:merge-milestone milestone/v1.1-xxx` |
| Developer/Driver | Integration test | Manual testing on milestone branch |
| Developer/Driver | Merge to main | `/gsd:merge-milestone main` |

## File Structure After All Merges

```
.planning/
  PROJECT.md              # Semantic-merged from all developers' milestones
  ROADMAP.md              # Semantic-merged, all phases marked complete
  MILESTONES.md           # All milestone entries preserved
  config.json

  phases/
    1-abc123-PLAN.md      # BE's phase plan (unique ID: abc123)
    1-abc123-SUMMARY.md
    1-def456-PLAN.md      # Algorithm's phase plan (unique ID: def456)
    1-def456-SUMMARY.md

docs/
  MILESTONE-CONTEXT-v1.1.md   # Driver's original context (preserved for reference)
```

## FAQ

**Q: What if Algorithm needs BE's API schema before BE is done?**
A: Algorithm can temporarily merge BE's branch into their own for testing:
```bash
git fetch origin
git merge origin/milestone/v1.1-smart-rec/phase-1-user-service
# Test against BE's current code
# Continue development
```

**Q: What if the merge order needs to change mid-sprint?**
A: The merge order is just a convention. Any developer can merge at any time. The later merger will rebase and handle conflicts. The "order" just minimizes who has to deal with conflicts.

**Q: What if two developers need to change the same file?**
A: This is handled by merge-milestone's AI-assisted conflict resolution:
1. Step 2.5 rebase detects the conflict
2. AI reads both sides' .planning/ context to understand intent
3. AI proposes a resolution with confidence level
4. Developer confirms or provides guidance

**Q: Can a developer work on multiple phases?**
A: Yes. Create multiple branches, one per phase:
```bash
milestone/v1.1-smart-rec/phase-1-user-service
milestone/v1.1-smart-rec/phase-4-notification-service
```
Merge them one at a time to `milestone/v1.1-smart-rec`.

**Q: What if Driver wants to track overall progress?**
A: Check the milestone branch:
```bash
git checkout milestone/v1.1-smart-rec
git log --oneline
# See which phases have been merged
# Check ROADMAP.md for phase completion status
```
