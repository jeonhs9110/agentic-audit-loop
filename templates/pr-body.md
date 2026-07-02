# PR body template

Paste this as the PR body when landing a round. The trajectory table + skipped-items registry are the memory that survives across rounds and reviewers.

Replace all `{{PLACEHOLDERS}}` before submitting.

---

## Summary

Round {{ROUND_NUMBER}} bundle — two fresh-angle audits ({{ANGLE_A}} and {{ANGLE_B}}). {{TOTAL_FINDINGS}} real findings.

### {{ANGLE_A}} ({{COUNT_A}})
- **{{FINDING_HEADLINE_1}}.** {{ONE_SENTENCE_IMPACT}}
- **{{FINDING_HEADLINE_2}}.** {{ONE_SENTENCE_IMPACT}}
- ...

### {{ANGLE_B}} ({{COUNT_B}})
- **{{FINDING_HEADLINE_1}}.** {{ONE_SENTENCE_IMPACT}}
- **{{FINDING_HEADLINE_2}}.** {{ONE_SENTENCE_IMPACT}}
- ...

## Test plan

- [x] `{{TYPECHECK_COMMAND}}` clean
- [x] `{{LINT_COMMAND}}` clean
- [ ] {{MANUAL_STEP_1}}
- [ ] {{MANUAL_STEP_2}}
- [ ] {{MANUAL_STEP_3}}

## Skipped (deferred with reason)

- **{{SKIPPED_ITEM_1}}** — {{REASON}}. Handoff item.
- **{{SKIPPED_ITEM_2}}** — {{REASON}}.
- **{{SKIPPED_ITEM_3}}** — {{REASON}}.

## Trajectory

| Round | PR | Findings | Pattern |
|---|---|---|---|
| ... | ... | ... | ... |
| {{PREV_ROUND}} | #{{PREV_PR}} | {{PREV_COUNT}} | {{PREV_PATTERN}} |
| {{ROUND_NUMBER}} | this | {{TOTAL_FINDINGS}} | {{THIS_PATTERN}} |

---

## Why this shape

**Summary — grouped by angle.** Reviewers scan the two angle groups independently. Mixed lists are harder to review.

**Test plan — checkboxes.** Automated checks pre-ticked (typecheck + lint pass or the PR shouldn't exist). Manual checks unticked so the reviewer can walk through them or delegate to QA.

**Skipped items — with reason.** Every deferred finding gets a line so future rounds can pick it up. Reasons should be concrete: "needs migration", "needs infra grant from cloud admin", "needs product decision on X", "big refactor — separate PR". Vague reasons like "later" defeat the point.

**Trajectory table.** The single most valuable piece of documentation across rounds. At round 10 you can see at a glance which angles you've covered, which are due for rotation, and what the marginal find rate looks like. Show the current round + the last 2-3 for context; a longer history lives in git log.
