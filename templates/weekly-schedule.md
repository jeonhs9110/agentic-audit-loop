# Weekly schedule template

Adapted from the source YouTube video's orchestrator-on-cron pattern. Use this to run the audit loop on a repeating cadence.

**Not recommended until you have 5+ rounds of experience with the manual pattern.** Automated rounds without established rhythm produce noise, not signal. The orchestrator prompt at [`prompts/orchestrator-agent.md`](../prompts/orchestrator-agent.md) already stops after one round for human review — this template adds the cron layer on top.

---

## Cron cadence options

Pick one:

### Weekly (recommended for B2C storefronts)

```
0 8 * * MON     # Every Monday 8am local
```

### Bi-weekly (recommended for B2B SaaS admin surfaces)

```
0 8 * * MON     # Every Monday 8am; orchestrator internally checks whether the last round was < 10 days ago and skips if so
```

### Monthly (recommended for internal tools apps)

```
0 8 1 * *       # First of every month at 8am
```

### On release (recommended for multi-tenant platforms)

Not a cron. Triggered by:
- CI hook on `git tag v*`
- Webhook from the release-candidate branch creation

---

## Scheduled orchestrator prompt

Add this to the top of `prompts/orchestrator-agent.md` when running on a schedule:

```
You are running as a scheduled agent. Before starting the round:

1. Check the last merged PR from this repo. If it's less than 5 days old,
   log "recent round exists; skipping" and STOP. Do not start a round.

2. Check the merge queue. If more than 2 audit-loop PRs are unmerged,
   log "merge backlog; skipping" and STOP. The reviewer bottleneck is
   real; don't add to the pile.

3. Check the budget. If token spend this month exceeds ${{BUDGET_CEILING}},
   log "budget ceiling reached; skipping" and STOP.

4. If all three checks pass, proceed with the standard orchestrator round.

Never run more than one round per cron trigger. Human gates all merges.
```

---

## Example: GitHub Actions setup

If you're using GitHub Actions to trigger the orchestrator on a schedule:

```yaml
name: audit-loop-weekly

on:
  schedule:
    - cron: '0 8 * * MON'   # Every Monday 8am UTC
  workflow_dispatch:         # Also allow manual trigger

jobs:
  audit:
    runs-on: ubuntu-latest
    permissions:
      contents: write        # For pushing branches
      pull-requests: write   # For opening PRs
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Set up LLM CLI
        # ... your LLM CLI setup (Claude Code, Codex, or API-based script)

      - name: Run orchestrator
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          BUDGET_CEILING: 500  # USD per month
        run: |
          # Invoke your orchestrator with prompts/orchestrator-agent.md
          # ...

      - name: Notify Slack on PR open
        if: success()
        # ... your notification step
```

**Important security note.** The workflow needs `contents: write` and `pull-requests: write` — meaning if the workflow is compromised, an attacker gets those. Rotate the API key regularly. Consider a dedicated deploy key.

---

## Example: local cron (single-user machine)

If running on a workstation for a personal project:

```bash
# ~/.crontab entry
0 8 * * MON  cd ~/my-webapp && claude --headless < /path/to/orchestrator-prompt.txt >> /var/log/audit-loop.log 2>&1
```

**Caveats.** The workstation must be on and awake at 8am Monday. Better to use GitHub Actions or a small cloud instance for reliability.

---

## Cost estimation

Assume:
- 1 round per week = 4 rounds/month (or 4.3 if you care)
- $6-18 per orchestrated round (per [`docs/orchestrator.md`](../docs/orchestrator.md))
- $20-80/month for a weekly cadence at typical Sonnet pricing

At $500/month budget ceiling, weekly runs use ~15% of the ceiling. Rest is slack for peak periods (Black Friday, release crunch) or ad-hoc rounds triggered by incidents.

---

## Monitoring the automated loop

Set up alarms for:

- **PR merge latency.** If the average merged-after-open time exceeds 3 days, reviewers are overloaded — pause the loop.
- **Skipped registry growth.** If skipped items grow faster than they shrink for 3 consecutive months, either commit to paying down the debt or acknowledge the loop is finding items you can't act on.
- **Token spend.** Standard budget alarms at 50%, 75%, 100% of monthly ceiling.
- **CRITICAL find rate over time.** If it drops to zero for 3 consecutive rounds, the loop has caught what it can catch. Rotate angles, reduce cadence, or stop.

The loop is a safety net, not a lifestyle. Automated cadence is a tool for maintaining vigilance — not a substitute for it.
