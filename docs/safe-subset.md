# The safe-subset synthesis rubric

This is the highest-judgment stage in the loop. Two audit agents just handed you 20-30 findings. You're going to apply maybe 15. Which 15?

## The three-bucket sort

Read every finding, put it in one of three buckets:

### Bucket 1: Apply now

A finding qualifies if ALL of these are true:

- **Self-contained.** The fix touches N files but doesn't require changing anything you don't understand yet.
- **No new dependencies.** No `npm install`, no `pip install`, no new SDK. (Exception: if the audit specifically suggested adding a well-established library like DOMPurify and you've decided that's this round's big move, that's fine. But then that's the ONLY new-dep item this round.)
- **No schema/migration changes.** Adding a column, changing a type, dropping an index — all defer.
- **No infra changes.** IAM grants, DNS updates, load-balancer config, environment variable additions on the deploy side — all defer (add the env to `.env.example` yes; wait for the deploy team to actually set it, defer).
- **Reversible.** If the fix is wrong, `git revert` cleans it up. Fixes that permanently mutate data (deleting rows, dropping tables) always defer.
- **You'd be comfortable defending it in code review.** If you have to write "I'm not sure this is right but the LLM said so", it's not safe.

### Bucket 2: Defer with reason

A finding here is REAL and IMPORTANT — just not this round. Every deferred finding needs a specific reason:

- `needs migration` — schema change, backfill, table add
- `needs infra grant` — IAM policy, DNS, load balancer, CDN behavior
- `needs product decision` — a UX choice or business rule that isn't yours to make
- `big refactor — separate PR` — the fix requires touching a large enough surface that mixing it with other fixes hurts reviewability
- `depends on Skipped item X from round Y` — you can't fix A until B ships

Deferred items go into the PR body's **Skipped** section with the reason. Next round's audit prompt says "don't re-report the Skipped registry", and the discovery stage can skip files where the only findings would be in this bucket.

### Bucket 3: Reject

Not every finding is worth doing. Reject when:

- **LOW severity + polish.** Style choices, stale comment cleanup, dead-code that isn't causing bugs — real but not worth reviewer bandwidth this round.
- **False positive.** The LLM misread the code. This happens. Don't apply it. Optionally add a code comment explaining why the pattern that looked wrong is actually intentional, so the NEXT round's auditor doesn't re-flag it.
- **The fix is worse than the bug.** The audit says "add fallback for X" but X is guaranteed by an upstream invariant. Adding the fallback creates a code path that will never execute — and someday someone will add real logic to it not realizing it's unreachable.
- **Speculative.** "This could theoretically fail at scale if..." with no concrete evidence. Real defects have real scenarios. Speculation is noise.

Rejected findings don't need to appear anywhere. They didn't survive verification; they're gone.

## Applying the sort in practice

Read all findings top to bottom. For each one, ask three questions:

1. **Would applying this fix add a comment starting with "I'm not sure"?** If yes → defer or reject.
2. **Would applying this fix require touching a file/system I don't fully understand?** If yes → defer.
3. **Would a reviewer look at this diff and ask "why is this in the same PR as the others"?** If yes → defer to a focused PR.

Two out of three questions returning "yes" means defer. All three → reject.

## The counterintuitive part

The instinct at round 3 is to apply everything the audit found because "the LLM is right and I should trust it." At round 8 the instinct flips: "the audit is over-cautious, I know better." Both instincts are wrong at those rounds.

Trust in the pattern comes from **consistency**. Apply the rubric the same way at round 3 that you'd apply it at round 30. The safe-subset gate is the reason the loop is worth running — an open loop without the gate is where regressions come from.

## Handling agent disagreements

Sometimes the two parallel agents flag the same file with contradictory advice. Agent A says "add a NOT NULL constraint on this column"; Agent B says "this column is intentionally nullable because we defer email verification".

Read the code. The code is truth. Not the audit.

If the code doesn't answer it clearly, defer both findings and add a comment explaining the ambiguity. Future rounds can revisit with more context.

## Handling audit fatigue

By round 6-7, you'll start seeing findings that feel repetitive. "Another silent-swallow catch". "Another `SELECT *`". This is fine — the loop is doing its job. Apply them the same way you would have at round 1.

The moment you catch yourself thinking "I don't want to write another commit message for another silent-swallow fix", stop the loop for that angle. Retire the prompt, pick a different angle, come back to this one in a month.
