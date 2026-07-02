# Memory: how the loop remembers

The single reason the loop scales beyond one round is **memory that survives across runs**. This document specifies what memory the pattern uses, where it lives, and how to structure it if you want richer state than the default.

## The three memory surfaces

### 1. The trajectory table (per PR body)

Lives at the bottom of every round's PR body. Shape:

```markdown
## Trajectory

| Round | PR | Findings | Pattern |
|---|---|---|---|
| 1 | #123 | 12 | perf + DB |
| 2 | #124 | 14 | auth + env |
| 3 | this | 11 | SEO + PII |
```

**Purpose:** at round N, a reviewer or the next orchestrator can see which angles have been covered and roughly what the yield looks like. Trends emerge — an angle producing 14 findings in round 2 and 3 in round 5 signals the angle is saturating; time to rotate.

**Discoverability:** git-native. Anyone with `git log` can reconstruct the trajectory.

### 2. The Skipped registry (per PR body)

Also at the bottom of every round's PR body. Shape:

```markdown
## Skipped (deferred with reason)

- **DOMPurify migration for user-content HTML** — needs new dependency; parser rewrite. Handoff item.
- **Streaming CSV export + users(created_at) index** — needs ReadableStream refactor + migration.
- **Keyset pagination on posts board** — bigger UX + component change.
```

**Purpose:** every deferred finding gets a line so future rounds can pick it up. Reasons are concrete — vague reasons ("later") defeat the pattern.

**Reading strategy:** the orchestrator's memory read should extract Skipped items from the last N PR bodies and pass them as "don't re-report these" context to the audit agents. Otherwise the agents keep flagging the same known-deferred items.

### 3. Round-scoped short-term memory (within one round)

Lives in the audit agent's own working context. When agent A reads a file and finds a defect, agent B in the same round might re-read the same file and duplicate the finding. This is fine occasionally — the orchestrator's synthesis step deduplicates.

**Purpose:** parallelism without coordination overhead. Two agents running in parallel with no shared state is simpler than two agents with shared state; the deduplication cost is small.

## What memory is NOT

**Not a chat history.** Chat history is inside-conversation state that dies at the end of the round. Memory has to survive.

**Not a knowledge graph.** Knowledge graphs are the future but not this pattern's present. The trajectory table + Skipped registry are enough for the loop to work.

**Not a database.** Some teams reach for a proper DB (Postgres, SQLite) to track memory. Overkill. The git history IS the DB — free, distributed, auditable.

**Not the LLM's own memory (Claude projects, ChatGPT memories).** Those are per-user state, not per-project state. If a different reviewer picks up round 9, they wouldn't have the previous 8 rounds' context.

## Optional richer memory: `.audit-loop/` directory

If you want more than the PR body carries, add a `.audit-loop/` directory to your target project (checked into git). Recommended shape:

```
.audit-loop/
├── trajectory.md              # append-only log of rounds
├── skipped.md                 # append-only log of deferred items
├── learnings.md               # patterns that recur across rounds
├── rules.md                   # codebase-specific audit rules
└── custom-angles/             # project-specific audit angle prompts
    └── shopify-webhooks.md
```

**`trajectory.md`** — append a section per round: date, angles, PR link, findings count, patterns spotted. Longer-form than the PR body table; useful for a monthly retro.

**`skipped.md`** — append a section per round: findings deferred with reason. When the reason is resolved ("migration shipped"), strike through the item. This is where you spot Skipped-registry rot ("we've been deferring this for 6 rounds").

**`learnings.md`** — codify patterns. "Round 5 found 3 CSRF gaps because a shared route helper didn't call the guard. Add to `custom-angles/shared-helper-csrf.md`." Learnings become new angle prompts.

**`rules.md`** — codebase-specific rules. "Never open a PR that adds new AWS resources; those are handled by the infra team." "Never modify migration files after they've been applied to a production environment."

**`custom-angles/`** — codebase-specific audit angle prompts. Same format as `prompts/angles/*.md` from this repo, tailored to patterns unique to your codebase.

## Reading memory in an audit agent

If your codebase uses the `.audit-loop/` pattern, include this snippet in the audit-agent prompt (before the checklist):

```
Before hunting for defects, read `.audit-loop/rules.md` and `.audit-loop/skipped.md`
in the target repo. Do NOT report any finding that is already in the skipped
registry with a valid deferral reason. Do NOT recommend changes that violate a
rule in the rules file.
```

For codebases without the pattern, skip this — the trajectory + Skipped in the PR body carry the essentials.

## Memory hygiene

The Skipped registry can grow indefinitely. Prune it every 5-10 rounds:

- **Delete items resolved.** A migration shipped that made the skipped item moot — delete the line.
- **Delete items rejected in retrospect.** "We considered X and decided not to do it" — delete + note the decision in `learnings.md`.
- **Promote items to actual tickets.** For a skipped item that WILL be done but not by the loop (needs product decision), copy it to your team's ticket tracker and delete from the registry.
- **Never let the registry cross 50 items.** Above that, the audit agent's context window can't hold it all + do a useful audit.

## Memory as handoff artifact

The 10-round trajectory table + Skipped registry is often the highest-quality documentation of what changed on a codebase over a quarter. When the loop wraps up, the PR bodies become the artifact you hand to the next team.

Better than a changelog: the WHY is embedded in the finding descriptions, the WHAT is in the diff, and the WHAT-WAS-DEFERRED is in the Skipped registry.
