# Commit message template

Structured commit shape that mirrors the PR body. When landing multiple rounds, `git log --oneline` gives you the trajectory for free.

Replace all `{{PLACEHOLDERS}}` before committing.

---

```
chore(round-{{ROUND_NUMBER}}): {{SHORT_SUMMARY}}

Bundle from two fresh-angle audits: {{ANGLE_A}} and {{ANGLE_B}}.

{{ANGLE_A}} — {{COUNT_A}} fixes
- {{FIX_1_HEADLINE}}. {{ONE_SENTENCE_WHY_AND_WHAT}}
- {{FIX_2_HEADLINE}}. {{ONE_SENTENCE_WHY_AND_WHAT}}
...

{{ANGLE_B}} — {{COUNT_B}} fixes
- {{FIX_1_HEADLINE}}. {{ONE_SENTENCE_WHY_AND_WHAT}}
- {{FIX_2_HEADLINE}}. {{ONE_SENTENCE_WHY_AND_WHAT}}
...

Verified: `{{TYPECHECK_COMMAND}}` + `{{LINT_COMMAND}}` both clean.

Skipped (deferred with reason)
- {{SKIPPED_ITEM_1}} — {{REASON}}.
- {{SKIPPED_ITEM_2}} — {{REASON}}.

Trajectory
| Round | PR | Findings | Pattern |
|---|---|---|---|
| ... | ... | ... | ... |
| {{ROUND_NUMBER}} | this | {{TOTAL_FINDINGS}} | {{THIS_PATTERN}} |
```

---

## Notes on style

**Prefix with `chore(round-N)`** so `git log --grep="chore(round-"` filters the loop history cleanly.

**Short summary — 5-10 words.** The reader is scanning `git log --oneline`. "perf/CWV + DB query hygiene bundle" beats "various improvements and fixes".

**One line per fix.** Not a paragraph. The full detail lives in the code comments (which R25+ audit patterns will remind you to write). The commit is an index, not a document.

**Verified line.** State the exact commands that passed. If a reviewer wants to reproduce, they know what to run.

**Skipped items.** Same registry as the PR body. Keeping it in the commit means the memory travels with `git log` even after the PR is merged and closed.

**Trajectory table.** Overkill for round 1. Essential by round 5. Add it once and forget it.
