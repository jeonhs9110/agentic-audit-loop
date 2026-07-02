# The Loop, in detail

The pattern in one paragraph: **for each round, spawn two parallel LLM audit agents on fresh non-overlapping angles; each returns 10-15 concrete file:line findings ranked by severity; you synthesize the safe subset (deferring anything that needs a migration, infra grant, or big refactor); you apply the fixes; you verify with typecheck + lint; you commit and open a PR whose body carries a trajectory table and a skipped-items registry that survive to the next round; a human gates the merge; then you rotate angles and repeat.**

The rest of this document expands each stage.

## Stage 1: Discovery

Two parallel audit agents, non-overlapping angles.

**Why two?** One agent leaves too much on the table — even a great audit prompt has blind spots. Two agents with distinct angles multiplies coverage without redundant cost. Three or more hits diminishing returns fast and the synthesis step gets harder.

**Why non-overlapping?** If both agents audit "perf", they find the same findings. Waste. If one audits perf and the other audits DB queries, you cover twice the surface for the same number of tool calls. Pair angles that are naturally distinct:

- perf-cwv + db-query-hygiene
- admin-csrf + env-config-safety
- seo-meta-i18n + pii-privacy
- user-content + perf-cwv

Don't pair angles that touch the same files (e.g. don't pair "SEO" and "product-detail perf" — they'll both crawl the same product route and duplicate work).

**Why fresh?** Repeating the same angle two rounds in a row surfaces the polish findings the first round skipped. That's fine occasionally, but at 10-round scale you want to rotate through the 7-10 angles in [`prompts/angles/`](../prompts/angles/) rather than double-dipping.

## Stage 2: Planning (safe-subset synthesis)

This is the highest-judgment stage. Not every finding should be applied this round.

Read every finding and sort them into three buckets:

- **Apply now.** Safe, self-contained, high value. No new dependencies, no migrations, no schema changes, no risky refactors.
- **Defer with reason.** Real findings that need a migration / infra grant / cross-team coordination / product decision. These go into the PR body's Skipped section so future rounds pick them up.
- **Reject.** LOW-severity polish that isn't worth the reviewer's time this round. Findings that would introduce a bigger regression than they fix. Findings the LLM misread the codebase on.

Rule of thumb: **if applying a finding would require you to add a comment starting with "I'm not sure this is right", defer it.** The safe-subset stage is where you preserve trust in the pattern. One risky fix that breaks prod undoes ten rounds of goodwill.

See [`safe-subset.md`](safe-subset.md) for the concrete rubric.

## Stage 3: Execution

Apply the fixes. Parallelize where safe.

**Batch by area.** Edits to the same file should be in one edit. Edits to related files (e.g. all the admin CSRF fixes) should be a contiguous batch you can reason about as a group.

**Cross-reference before editing.** If the audit found one CSRF gap, grep for the same pattern across the codebase before ending the batch. Half the value of a single-round audit is the "and one more" cascade.

**Write comments that survive.** For non-obvious fixes, leave a one-line comment that says WHY (the constraint, the invariant, the past incident). Not what the code does (the code says what it does). This is the memory that helps the next round's auditor not re-flag the same code as a defect.

## Stage 4: Verification

Non-negotiable: typecheck + lint pass before you open the PR.

Optional but valuable:

- Type-narrower coverage checks (`tsc --strict` diff)
- Focused test runs on files touched
- Manual smoke test on the changed feature
- The `verify` skill (in Claude Code) drives affected flows end-to-end

The PR body's Test Plan should list every check you ran (checked) and every check you're delegating to the reviewer (unchecked). Honesty is the point — a checked box says "I ran this and it passed", not "the reviewer should run this".

## Stage 5: Iteration

The Skipped registry is the memory. Every deferred finding gets a line with a reason.

Next round's Discovery stage reads the registry (either you paste it into the audit prompt, or the audit agent grep's the last few merged PR bodies) and knows what's already been flagged. The audit prompt should say: "don't re-report anything from the Skipped registry — I know about those."

The trajectory table is orientation. At round 3 it's overkill. At round 10 it lets you see:

- Which angles you've covered (rotate to fresh ones)
- Whether your CRITICAL find rate is dropping (a sign to stop the loop)
- Whether one specific angle is over-producing polish (retire that prompt)

## Human-gated merge

Every PR waits for you (the human) to merge. This is what makes the loop "closed" rather than "open" — cost is bounded, blast radius per round is one PR.

The temptation to autoflow (merge → next round automatically) is real and worth resisting until you have enough rounds behind you to trust the pattern. Even then, keep a merge gate on any round that touches auth, payment, or PII.

## When to stop

The pattern has a natural end. Stop when:

- CRITICAL findings dry up (usually after 5-10 rounds on a stable codebase)
- The Skipped registry is dominated by items that need product/schema/infra decisions rather than code changes
- Marginal cost per finding rises above marginal value
- Your team asks you to stop
- You hit a budget ceiling

The loop is a safety net, not a lifestyle. Ship it, keep it, and pull it out again when your codebase drifts.
