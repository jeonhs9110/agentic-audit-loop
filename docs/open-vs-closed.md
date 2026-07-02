# Open vs closed loop — cost management

The single biggest failure mode of this pattern is unattended cost. Read this before running.

## Two shapes

**Open loop.** "Go find bugs, apply fixes, keep going until you can't find any more." The agent orchestrates its own rounds, its own angle rotation, its own safe-subset decisions, and its own merges. Zero human gate.

**Closed loop.** Bounded per round. Two audit angles, safe-subset apply, one PR. Human gates the merge before the next round starts.

## Cost math

A typical deep audit agent, running a real audit against a real codebase, will fire 50-90 tool calls (reads + greps + occasional bash). Each returns file content into context. Rough cost per audit: **100-160K tokens.**

**Closed loop, one round:**
- 2 audits in parallel: 200-320K tokens
- Your synthesis + fix-apply cycle: 50-150K tokens (varies with codebase familiarity)
- Verification + PR body: 10-30K tokens
- **Total per round: 260-500K tokens.**

At current Claude Sonnet pricing (~$3/$15 per million in/out), a single closed round is roughly $5-15. A 10-round campaign is $50-150. Predictable.

**Open loop, one week unattended:**
- No natural stopping point
- Agents can re-audit files they've already covered
- No safe-subset gate — risky fixes ship and cause regressions
- No PR-body memory — findings loop back through discovery
- **Total: unbounded. Reported cases: $1,000-10,000 per week.**

## When each fits

**Closed loop fits when:**
- You have a budget you'd like to keep track of
- Your codebase serves real users and regressions are expensive
- You want to review what changed before it merges
- You want the pattern to be sustainable across weeks/months

**Open loop fits when:**
- You have a research budget and no production consequences
- You're benchmarking agent capability at extended-horizon tasks
- You're OK with the agent making design decisions you'd disagree with
- You have a reliable rollback strategy at the deployment layer

For a real product, closed loop is the answer. Open loop is a research pattern with a research-lab budget.

## Concrete guardrails for closed loop

If you're running this yourself, set these up before the first round:

1. **Two-audit ceiling per round.** Don't spawn three. Don't spawn one and re-run it with a different prompt.
2. **Angle rotation.** Rotate through your available angles rather than double-dipping. The `prompts/angles/` folder gives you 7 to start.
3. **Round-count budget.** Decide up front how many rounds this campaign will run. 5 rounds for a small codebase, 10 for a bigger one. Stop when you hit the number, not when the LLM stops finding things.
4. **Skipped-item audit.** Every 3-4 rounds, look at the Skipped registry and decide: are these items ever going to get done, or are you just accumulating them? If accumulating, either commit to doing them or delete them.
5. **Cost dashboard.** Watch your LLM provider's spend page. Set an alarm at 50%, 75%, 100% of your target budget.

## Signs to stop

Stop the loop when any of these fire:

- **CRITICAL find rate drops to zero for 2 consecutive rounds.** The remaining findings are polish; you're paying token cost for diminishing returns.
- **Same angle produces > 80% deferred findings.** That angle has surfaced everything code-only can catch; the rest is infra/schema/product.
- **Your team asks you to stop.** Loops that surprise reviewers destroy the trust that made the pattern work.
- **Budget alarm fires.** Hard limit. Don't rationalize past it.
- **You catch yourself thinking "the next round will find something big."** No it won't. That's a sunk-cost fallacy.

## What to do with the budget you didn't spend

The best outcome of a closed-loop campaign isn't the fixes — it's what you didn't have to pay for. If your 10-round budget was $150 and you only spent $80 because CRITICAL findings dried up at round 6, you saved $70 that can go into:

- Real security review (pentest of the auth surface)
- Real code review of the biggest recent feature
- Documentation the LLM can't write for you
- Infrastructure improvements

The loop is a tool. It's not the whole toolbox.
