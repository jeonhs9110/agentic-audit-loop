# Angle: DevOps + CI/CD + deploy safety

Paste this as the `{{ANGLE_SPECIFIC_CHECKLIST}}` in the audit-agent prompt.

Target: CI workflows, deploy pipeline, rollback path, health checks, secrets management, infra-as-code hygiene. Not full SRE — code-adjacent hygiene that affects deploys.

## Look for

1. **Missing rollback path.** Deploy pipeline pushes to prod but no scripted way to redeploy the previous version. When something breaks, someone has to remember how to roll back.
2. **Deploy without health check gate.** New instance starts serving traffic before `/api/health` confirms DB connectivity. Bad deploy → healthy target → traffic → 500s.
3. **Health check that only checks the process.** `/api/health` returns 200 if the Node process is up, but doesn't ping the DB. Cascading failures invisible.
4. **Secrets in the repo.** `.env`, `.env.production`, `credentials.json` — anything in the working tree that a `git ls-files` finds. Even in `.gitignore` if it was previously committed and never `git rm --cached`-d, it lives in history.
5. **Secrets in CI logs.** `echo $API_KEY` in a workflow step. `curl -H "Authorization: Bearer $TOKEN"` where the token gets logged.
6. **Secrets in commit messages.** History-check for accidentally-committed secrets (grep + git-secrets tool).
7. **Long-lived credentials.** AWS access keys in env vars when EC2/ECS role could be used. Hardcoded database passwords when IAM DB auth could be used.
8. **CI running on `push` to any branch.** Unbounded — a bot pushing spam branches burns actions minutes. Restrict to `pull_request` + main branch pushes.
9. **CI without concurrency guards.** Multiple runs of the same workflow racing against each other on the same branch. Add `concurrency: { group: ..., cancel-in-progress: true }`.
10. **CI running with excessive permissions.** GitHub Actions workflow with `permissions: write-all` when it only needs `contents: read`.
11. **Third-party actions pinned to `@main` or `@v3`.** Should be pinned to a SHA (`actions/checkout@a12345...`) for supply-chain safety on the CI side.
12. **Build cache misuse.** Docker layer cache invalidated on every build because a `COPY . .` happens before `COPY package.json`. Fixable in Dockerfile ordering.
13. **Missing multi-stage Docker build.** Final image includes build tools, source maps, dev deps. Bloated + more attack surface.
14. **`latest` tag in production.** Deployed image `myapp:latest` — reproducibility gone. Pin to a specific tag/SHA.
15. **Missing container security scanning.** No Trivy / Snyk / Grype step in the build pipeline. Base image CVEs go unnoticed.
16. **Terraform state stored locally.** `terraform.tfstate` committed to the repo (contains secrets and is unlocked for concurrent operations). Should be in S3 + DynamoDB lock.
17. **Environment parity gaps.** Dev env uses PostgreSQL 15, prod uses PostgreSQL 12. Or dev uses SQLite while prod uses Postgres. Bugs hide.
18. **No automated tests on the CD side.** After deploy, smoke test runs? Or does the deploy pipeline succeed silently and rely on the operator to notice a broken deploy?
19. **Cron jobs / scheduled tasks without idempotency.** A cron that re-runs on failure — does the second run duplicate the work?
20. **`docker-compose.yml` in prod.** Great for dev; not for prod. Kubernetes / ECS / equivalent for real orchestration.
21. **Missing observability on deploys.** No structured "deploy started" / "deploy succeeded" event. Debugging "when did prod start returning 500s?" means grep-scrolling git log.

## Files to prioritize

- `.github/workflows/*.yml` (or `.gitlab-ci.yml`, `.circleci/config.yml`)
- `Dockerfile`, `docker-compose.yml`
- `infrastructure/` (Terraform, CDK, Pulumi)
- `scripts/deploy*` — any deploy scripts
- `.env.example` — should list every env var; live envs may drift
- Health check route

## Grep starters

```
uses: actions/                     # third-party action usage
permissions:                       # workflow permissions
secrets\.                          # secret references in workflows
FROM.*:latest                      # latest tag in Dockerfile
run: npm install                   # vs npm ci
```

## What NOT to report

- "Consider Kubernetes" — infra decision, not code
- "Add monitoring" — vague; must name specific missing metric/log
- "CI is slow" — perf tuning, needs specific bottleneck
