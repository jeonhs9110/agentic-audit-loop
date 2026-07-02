# Architecture

How the pieces fit together. This document is for people who want to fork the pattern, adapt it, or understand it deeply enough to explain to a team.

---

## System overview

```
                            ┌──────────────────────┐
                            │       Human          │
                            │  (sets goal,         │
                            │   gates merge)       │
                            └──────────┬───────────┘
                                       │
                                       │
                            ┌──────────▼───────────┐
                            │   Orchestrator       │
                            │  (optional; can be   │
                            │   the human or an    │
                            │   agent)             │
                            │                      │
                            │  - Reads memory      │
                            │  - Picks angles      │
                            │  - Spawns auditors   │
                            │  - Synthesizes       │
                            │  - Applies fixes     │
                            │  - Verifies          │
                            │  - Opens PR          │
                            └────┬──────────┬──────┘
                                 │          │
                       spawns in │  parallel│
                                 │          │
                    ┌────────────▼─┐      ┌─▼────────────┐
                    │  Auditor A   │      │   Auditor B  │
                    │              │      │              │
                    │ Reads angle  │      │ Reads angle  │
                    │ prompt from  │      │ prompt from  │
                    │  prompts/    │      │  prompts/    │
                    │  angles/*.md │      │  angles/*.md │
                    │              │      │              │
                    │ Audits target│      │ Audits target│
                    │ codebase for │      │ codebase for │
                    │ one angle    │      │ one angle    │
                    │              │      │              │
                    │ Returns 10-15│      │ Returns 10-15│
                    │ findings     │      │ findings     │
                    └──────┬───────┘      └──────┬───────┘
                           │                     │
                           └──────────┬──────────┘
                                      │
                          ┌───────────▼───────────┐
                          │   Target codebase     │
                          │  (the app being       │
                          │   audited — this repo │
                          │   never mutates it    │
                          │   directly; the       │
                          │   orchestrator does)  │
                          └───────────┬───────────┘
                                      │
                                      │
                          ┌───────────▼───────────┐
                          │     Memory            │
                          │                       │
                          │  - PR body trajectory │
                          │  - Skipped registry   │
                          │  - Optional           │
                          │    .audit-loop/       │
                          │    directory          │
                          │                       │
                          │  (Read at Round N+1   │
                          │   discovery stage)    │
                          └───────────────────────┘
```

Two auditors spawn in parallel per round, each on a different angle. Their findings are synthesized (by the orchestrator or the human) into a safe subset, applied to the target codebase, verified, and shipped as one PR. Memory lives in the PR body and travels with git history.

---

## Component responsibilities

### The human

- Sets the initial goal ("audit this webapp for defects across N rounds")
- Reviews and gates every PR before merge
- Decides when to stop the loop (usually when CRITICAL find rate drops to zero)
- Manages the budget and cost dashboards
- Ultimately owns any risk that a fix introduces

### The orchestrator (optional)

- Reads memory (previous PR bodies, skipped registry)
- Picks two audit angles for this round based on rotation
- Spawns two auditors in parallel with the corresponding prompts
- Synthesizes findings into safe-subset buckets (Apply now / Defer / Reject)
- Applies fixes; runs verification
- Opens a PR with the standard body shape
- **Stops** — hands back to the human for merge review

Can be an agent (following `prompts/orchestrator-agent.md`) or the human directly. Same responsibilities either way.

### The auditors (specialists)

- Each spawned with one angle-specific prompt from `prompts/angles/*.md`
- Read the target codebase (files, greps, occasional bash)
- Return 10-15 concrete file:line findings ranked by severity
- Never mutate the target codebase — read-only role
- Never talk to each other — parallelism without coordination is the point

### The target codebase

- The app being audited (a separate repo from this one)
- Never modified by the auditors directly; only by the orchestrator/human after synthesis
- Optionally hosts an `.audit-loop/` directory for richer memory (see [`docs/memory.md`](docs/memory.md))

### Memory

- Persistent state that survives across rounds
- Two forms:
  - **Git-native**: PR body trajectory table + Skipped registry
  - **Optional**: `.audit-loop/` directory in the target repo with `trajectory.md`, `skipped.md`, `learnings.md`, `rules.md`, `custom-angles/`
- Read at the discovery stage of Round N+1 to inform angle selection + avoid re-reporting deferred findings

---

## Data flow (one round)

```
┌────────────────────────────────────────────────────────────────┐
│ 1. DISCOVERY                                                    │
│                                                                 │
│    Orchestrator reads memory:                                   │
│    - Last N PR body trajectory tables                           │
│    - Last N PR body skipped registries                          │
│    - .audit-loop/rules.md (if exists)                           │
│                                                                 │
│    Selects angles: (A, B) satisfying rotation + non-overlap     │
│                                                                 │
│    Spawns auditor A with prompts/audit-agent.md + angle A       │
│    Spawns auditor B with prompts/audit-agent.md + angle B       │
│    (both in parallel)                                           │
└────────────────────────┬───────────────────────────────────────┘
                         │
                         ▼
┌────────────────────────────────────────────────────────────────┐
│ 2. PLANNING (synthesis)                                         │
│                                                                 │
│    Orchestrator collects findings from both auditors.           │
│    Applies safe-subset rubric per docs/safe-subset.md:          │
│                                                                 │
│    Bucket 1: Apply now                                          │
│    Bucket 2: Defer with reason                                  │
│    Bucket 3: Reject                                             │
│                                                                 │
│    Cap Bucket 1 at 20 findings max.                             │
└────────────────────────┬───────────────────────────────────────┘
                         │
                         ▼
┌────────────────────────────────────────────────────────────────┐
│ 3. EXECUTION                                                    │
│                                                                 │
│    Orchestrator applies Bucket 1 fixes to target codebase.      │
│    Batches by file. Leaves WHY comments on non-obvious fixes.   │
│    Cross-references each fix pattern across the codebase        │
│    (one CSRF gap usually implies more).                         │
└────────────────────────┬───────────────────────────────────────┘
                         │
                         ▼
┌────────────────────────────────────────────────────────────────┐
│ 4. VERIFICATION                                                 │
│                                                                 │
│    Runs typecheck + lint + affected tests.                      │
│    Must all pass before PR opens.                               │
│    Failure → fix or revert; do not paper over.                  │
└────────────────────────┬───────────────────────────────────────┘
                         │
                         ▼
┌────────────────────────────────────────────────────────────────┐
│ 5. ITERATION                                                    │
│                                                                 │
│    Commits with structured message (templates/commit-message.md)│
│    Pushes to a branch                                           │
│    Opens PR with body from templates/pr-body.md:                │
│      - Findings grouped by angle                                │
│      - Test plan checkboxes                                     │
│      - Skipped registry (Bucket 2) with reasons                 │
│      - Trajectory table                                         │
│                                                                 │
│    STOPS. Human gates merge.                                    │
└────────────────────────────────────────────────────────────────┘
```

Once the human merges, the trajectory + Skipped are canon in git history. Round N+1's discovery re-reads them.

---

## Why parallel auditors instead of one

**One auditor with a bigger prompt**: seems simpler, but hits diminishing returns on prompt size. LLM attention degrades with context length. A single audit trying to cover 5 angles finds fewer real defects than 5 focused audits with narrow angle prompts.

**Sequential auditors**: doubles wall-clock time with no accuracy benefit. Auditors don't share context anyway (each reads target files independently), so there's no state to inherit.

**Two parallel auditors on distinct angles**: the sweet spot. Broader surface, no coordination overhead, ~50% of one auditor's cost per audit (parallelism efficiency).

**Three or more parallel auditors**: triples cost. The third auditor mostly finds the same things the first two do. Marginal find rate drops sharply. Two is the pragmatic ceiling.

---

## Why the safe-subset filter is non-negotiable

Applying every finding an audit produces is what turns closed loops into regressions. The safe-subset rubric is the trust anchor:

- **Every finding in Apply-now is defensible in code review.** If you can't defend it, defer it.
- **Every finding in Defer has a specific reason.** "Needs migration" is a real reason. "Later" is not.
- **Rejected findings are silently dropped.** No obligation to record. No obligation to justify.

Without this filter, a single bad round poisons the well for every subsequent round. The reviewer stops trusting the pattern, the merge queue stalls, and the loop dies.

---

## Extension points

Places to customize the pattern for your codebase:

1. **Custom angles.** Drop new prompts into `prompts/angles/` or a project-local `.audit-loop/custom-angles/` directory.
2. **Custom orchestrator.** Fork `prompts/orchestrator-agent.md` and add project-specific rules ("never modify migrations after they're applied to prod").
3. **Custom memory shape.** Add a `.audit-loop/` directory to your target repo with `learnings.md` + `rules.md` (see [`docs/memory.md`](docs/memory.md)).
4. **Custom verification steps.** Add `pnpm typecheck && pnpm lint && pnpm test:e2e:smoke` (or your equivalent) to your local orchestrator prompt's Stage 4.
5. **Custom scenarios.** Add scenarios to `docs/scenarios.md` (or a project-local doc) with your team's specific done conditions.

The core pattern (parallel auditors → safe-subset → apply → verify → PR → merge → repeat) doesn't need to change. Everything else is customization.

---

## Non-goals

- **Autonomous unattended operation.** The pattern requires human merge review. Running it without merge review turns closed loops into open loops with unbounded cost and blast radius.
- **A framework or SDK.** No install. No runtime. Just prompts and templates. Deliberate — LLM CLIs come and go; portable prompts survive vendor churn.
- **AI model recommendations.** The prompts work on any frontier LLM. Different models have different cost/quality curves; pick one that fits your budget.
- **Replacement for human review.** The safe-subset step is filter, not gospel. A human still needs to review the PR before merge.
