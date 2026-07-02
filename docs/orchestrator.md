# The orchestrator pattern

For teams running the audit loop at scale, the loop itself can be orchestrated by an agent. This document explains when to use it, how it interacts with the closed-loop discipline, and where its failure modes live.

## What the video shows

Ben Stevenson's video demonstrates the pattern with a pickleball-store growth example. One orchestrator agent delegates work to three specialists:

- **Builder** — creates a marketing quiz
- **Scout** — researches content opportunities on Reddit and competitor sites
- **Growth** — turns scout output into ready-to-ship marketing artifacts

The orchestrator reads a `next_steps.md` memory file, delegates in parallel, awaits results, synthesizes them into a unified action plan, updates memory, and either loops again or hands back to the human.

Same shape, different specialists — that's the pattern we're adapting.

## Applied to auditing

The audit-loop orchestrator does the same thing, with different specialists:

- **Auditor A** — runs one audit angle (e.g. perf-cwv) using the prompt in `prompts/angles/perf-cwv.md`
- **Auditor B** — runs a second, non-overlapping angle (e.g. db-query-hygiene)
- **(Optional) Fixer** — applies the safe subset of findings

The orchestrator:

1. Reads memory (last N merged PR bodies from this repo — trajectory + skipped registry)
2. Picks two audit angles based on rotation + coverage gaps
3. Spawns Auditor A and Auditor B in parallel
4. Awaits both, synthesizes findings into safe-subset buckets
5. Applies fixes (or delegates to Fixer)
6. Runs verification
7. Opens a PR with the standard trajectory + skipped-items body
8. **Stops** — hands back to the human for merge review

The orchestrator prompt template is [`prompts/orchestrator-agent.md`](../prompts/orchestrator-agent.md).

## When to use orchestration

**Good fit:**

- You've run 5+ closed-loop rounds manually and understand what the pattern produces
- You have a merge review discipline in place (someone always reviews, no auto-merge)
- You have cost dashboards + alarms configured on your LLM provider
- Your codebase is mature enough that CRITICAL findings are rare and the rounds are mostly polish-tier

**Bad fit:**

- You've never run a closed-loop round manually — you don't yet know what "safe subset" looks like for your codebase
- No merge-review discipline; auto-merging is on the table
- No budget monitoring; token spend is a mystery
- Your codebase is early / actively-refactored — findings will be stale before the next round starts

## When to STOP the orchestrator

Hard stop conditions:

- **Two consecutive rounds produce zero CRITICAL findings.** The loop has caught what it can catch. Rotate angles or halt.
- **Skipped registry grows faster than it shrinks.** You're producing debt instead of paying it down.
- **Cost dashboard fires an alarm.** Budget ceiling is a ceiling.
- **Merge queue accumulates unreviewed PRs.** The human gate is the whole point. If PRs pile up, orchestration is producing noise the human can't absorb.
- **Reviewer complaint.** Reviewers who can't keep up become skeptical of the pattern. Preserve trust.

## The orchestrator's biggest failure mode

**It picks the same angle twice.** Every round should hit two DIFFERENT angles. If the orchestrator picks perf-cwv two rounds in a row, the second round finds only the polish the first round skipped. Rotate.

Enforce rotation in the orchestrator prompt: "Do NOT pick angles covered in the last 3 rounds."

## Angle rotation strategy

With 15 angles, rotation can look like:

| Round | Angle A | Angle B | Rationale |
|---|---|---|---|
| 1 | perf-cwv | db-query-hygiene | Frontend + backend perf |
| 2 | admin-csrf | env-config-safety | Auth + config surface |
| 3 | seo-meta-i18n | pii-privacy | Discovery + compliance |
| 4 | accessibility | error-observability | UX quality + operator quality |
| 5 | user-content | mobile-responsive | UGC surface + mobile UX |
| 6 | api-design | auth-flow | API + auth completeness |
| 7 | test-coverage | dependencies-supply-chain | Verification + supply chain |
| 8 | devops-cicd | (any that produced high-severity findings previously) | Ops + revisit |
| 9+ | rotate freshest angles | ... | ... |

At round 8 you've covered every angle at least once. Rounds 9+ revisit the highest-yield angles or introduce custom ones.

## Custom angles

The orchestrator's angle picker should also read `prompts/angles/custom/*.md` if the directory exists. Teams add codebase-specific angles here — for example, a Shopify app team might add `angles/custom/shopify-webhook-idempotency.md`.

## Memory shape

The orchestrator reads memory from the PR body trajectory table. See [`memory.md`](memory.md) for the exact schema and how to structure custom memory files if you want richer state than a git-history-derivable table.

## Cost profile

An orchestrated round costs slightly more than a manual round due to the orchestrator's own token spend (~30-80K tokens for memory read + planning + synthesis). Total:

- Manual closed round: 260-500K tokens (~$5-15 at Sonnet pricing)
- Orchestrated round: 320-600K tokens (~$6-18 at Sonnet pricing)

The extra cost buys you consistency across rounds — the orchestrator applies the safe-subset rubric identically every time, whereas a human's judgment drifts.

## Don't orchestrate to save time

The orchestrator makes the loop MORE consistent, not faster. A manual round takes ~1-2 hours end to end (mostly reading findings + applying fixes). An orchestrated round takes ~30-60 minutes (the orchestrator + specialists run mostly unattended). But the merge-review step still takes as long — because a reviewer still has to read every diff.

If your bottleneck is reviewer time, orchestration doesn't help. It might hurt (more PRs to review). Fix the reviewer bandwidth first, then consider orchestration.
