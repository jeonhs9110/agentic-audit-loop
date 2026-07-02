# Angle: Dependencies + supply chain

Paste this as the `{{ANGLE_SPECIFIC_CHECKLIST}}` in the audit-agent prompt.

Target: outdated deps, deprecated packages, license conflicts, unused deps, transitive-vulnerability exposure, npm/pypi supply-chain risks.

## Look for

1. **`npm audit` / `pip check` findings on critical severity.** Run whichever is applicable and report the highest-severity items. Don't recommend blanket `--force` upgrades.
2. **Deprecated packages.** Packages with `deprecated` field in the registry. `moment` (use date-fns / Temporal), `request` (use fetch / axios), `node-sass` (use dart-sass), etc.
3. **End-of-life major versions.** Node 14/16 in engines, React 17, Next.js 12, TypeScript 4.x. Every major version drop-off leaves you outside the security-support window.
4. **Very outdated deps.** Packages that haven't been updated in the project's `package.json` in 2+ years while having active newer releases. Grep `package-lock.json` for install dates.
5. **Multiple versions of the same dep in `package-lock.json`.** `react@18.2.0` and `react@17.0.2` both installed via different transitive paths. Bundle-size penalty + runtime hazard.
6. **Unused dependencies.** Packages in `dependencies` that nothing in `src/` imports. Run `depcheck` or grep-cross-reference.
7. **Dev dependencies in `dependencies` (or vice versa).** `@types/*` in `dependencies` bloats the prod bundle. Framework CLIs in `dependencies` when they should be in `devDependencies`.
8. **License conflicts.** GPL packages in a proprietary product. AGPL packages in a SaaS. Copyleft transitive deps that require open-sourcing your code.
9. **Packages with known typosquatting risk.** `react-typing-animation` vs `react-typing-effect` — one is legitimate, one might be malicious. Run through a supply-chain risk scanner.
10. **Packages installed from GitHub URLs / non-registry sources.** `"foo": "github:acme/foo"` — no version pinning, no integrity check. Package can be swapped underneath the project.
11. **`postinstall` scripts on suspicious deps.** Any dep whose `postinstall` runs a shell script? Historic supply-chain-attack vector.
12. **Missing `package-lock.json` (or `pnpm-lock.yaml` / `yarn.lock`) in the repo.** Non-deterministic installs.
13. **`npm install` on CI without `npm ci`.** `npm ci` respects the lockfile strictly; `npm install` may drift.
14. **Framework major-version drift.** Next.js 15 with React 18 with TypeScript 5 — consistent? Or a mix of major versions?
15. **`peerDependencies` warnings ignored.** Silently-mismatched peer deps produce runtime crashes at load.
16. **Ancient `engines` field.** `"node": ">=12"` when the codebase uses features requiring Node 18+.
17. **Optional dependencies that aren't optional.** `optionalDependencies` that the code imports without a try/catch — install failure crashes the app.
18. **Dependencies pulled into the client bundle unnecessarily.** Admin-only libraries (`xlsx`, `@react-pdf/renderer`) accidentally imported by a shared util that the storefront uses.
19. **Native/binary dependencies.** `sharp`, `bcrypt`, `sqlite3` — need per-architecture builds. Deploy to ARM64 EC2 with x86_64 binaries and the app crashes at start.
20. **Renovate / Dependabot config missing.** No automated PR bump = deps drift silently.

## Files to prioritize

- `package.json` — direct deps
- `package-lock.json` / `pnpm-lock.yaml` / `yarn.lock` — inventory + transitive versions
- `Dockerfile` — install pattern (`npm ci` vs `npm install`)
- CI workflow files — deps install step
- `.github/renovate.json` or `.github/dependabot.yml` — automation config
- `LICENSE` — your project's license (informs compat check)

## Grep starters

```
"deprecated"                       # deprecated packages in lockfile
"resolved": "https://github.com    # non-registry installs
"postinstall":                     # postinstall scripts
```

## What NOT to report

- "Update all deps to latest" — dangerous; require case-by-case
- "Consider Yarn" (or Bun, pnpm) — package-manager preference
- "Deps are outdated" — vague; must name specific outdated dep + risk
- Individual CVEs of MEDIUM severity or below on transitive deps that aren't reachable from your code
