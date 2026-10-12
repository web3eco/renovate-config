# web3eco/renovate-config

Single canonical Renovate preset for the web3eco org. Consumers inherit this
policy so dependency bumps are consistent across every service repo.

## Consumers

Add one file to your repo:

```json
{
  "extends": ["github>web3eco/renovate-config"]
}
```

Last-wins merge: any key a consumer writes locally overrides this preset.

---

## What the preset does

| Area | Behaviour |
|---|---|
| **Schedule** | Dev-deps weekly (Mon 06:00 Europe/Berlin), prod-deps monthly (1st), lockfile refresh Sunday |
| **Dev-deps** | Grouped, weekly, `minimumReleaseAge: 3 days` — **manual merge, approval required** |
| **Prod-deps** | Grouped monthly, manual review |
| **github-actions** | `pinDigests: true` → rewrites `@v5` to `<full-sha>#v5`, manual merge |
| **shared-actions** | Monthly, manual review — digest bumps are security-sensitive |
| **@types/**, **eslint** | Grouped weekly, manual merge |
| **Major bumps** | Always manual, never auto-merged |
| **Vulns** | `vulnerabilityAlerts` on, OSV.dev as secondary source |
| **PR limit** | `prConcurrentLimit: 10` |
| **Branch prefix** | `renovate/` |

### Why there is no automerge

Org policy (decided 2026-10-12): **every merge requires a human approval.**
Consumer branch protection runs with `allow_auto_merge: false`, so GitHub-side
automerge is unavailable by design.

The preset therefore contains **no `automerge`, no `automergeType`, and no
`platformAutomerge` keys anywhere**, and does not extend `:automergeMinor` /
`:automergePatch` (which would set `automerge: true` implicitly).

This is deliberate rather than an oversight. Keeping `automerge: true` while
`allow_auto_merge` is `false` would mark every PR "ready to merge" and then
leave them sitting forever — the worst of both worlds. If org policy ever moves
to allowing auto-merge for low-risk updates, re-enable it here *and* flip
`allow_auto_merge: true` in the same change, so the two never disagree.

### Why `minimumReleaseAge: 3 days`

Renovate won't propose a version until it's been released ≥3 days. Catches
yank-prone releases that would otherwise land straight into `main`.

---

## Known gaps (recorded at Phase 0 reconnaissance, 2026-10-09)

These are unresolved pre-flight findings. They do not block adopting the preset,
but they do degrade behaviour until fixed.

### 1. `@web3eco/platform-team` does not exist

`gh api orgs/web3eco/teams/platform-team` → **404**.

The preset was written against `@web3eco/platform-team` in four places
(`vulnerabilityAlerts.assignees`, prod-deps reviewers, shared-actions
reviewers, major-bump reviewers). Per the plan's documented fallback, all four
currently use **`@vladimirmyshkovski`** instead.

**To do**: create the team via org admin, then `sed` the four occurrences
back to `@web3eco/platform-team` and re-run `jq -e` validation.

### 2. Renovate GitHub App is not installed on the org

`gh api orgs/web3eco/installations` lists only `netlify`, `atlassian`,
`aws-amplify-eu-central-1`. No `renovate` installation exists.

**To do** (manual UI — no CLI path):
1. https://github.com/apps/renovate → Install
2. Scope: organization `web3eco`, **All repositories**
3. Verify `web3eco/renovate-config` is included

Until this is done the preset is inert — nothing will open PRs.

### 3. `allow_auto_merge` is disabled org-wide (resolved as policy, not a gap)

Checked all 14 candidate repos: `allow_auto_merge` is `not-set` on **14/14**.
Only `blockchain`, `payments`, `referrals`, and `agent-context` have any branch
protection on `main` at all.

**Decision (2026-10-12):** this is now the intended state, not a defect. Org
policy is *"merge only with an explicit approval"*, so `allow_auto_merge: false`
everywhere and no automerge in this preset (see "Why there is no automerge"
above).

Consequence: **all Renovate PRs require a human to click Merge.** Grouping keeps
the volume tolerable (~4 groups/week instead of ~30 single-package PRs), but
nobody gets a hands-off pipeline until policy changes.

If that trade-off stops being acceptable, the change is two-sided and must be
done together:

1. Flip `allow_auto_merge: true` on the consumer's `main` branch protection.
2. Re-add `automerge`/`platformAutomerge` to the affected rules here.

### 4. Repo inventory is larger than the plan assumed

Plan expected ~30 org repos; actual count is **38**. The preset itself is
unaffected (it is org-wide), but the propagation list in the plan needs
reconciling before Phase 2. Scope was narrowed deliberately: rollout targets
the primary backend repos, not the full org. `balances` is out of scope.

---

## Updating the preset

Policy changes go through a PR to this repo. Branch naming follows the org
convention: `CU-<clickup-id>` (no pipe in branch names; the pipe separator is
PR-title only).

`default.json` is **strict JSON** — comments are forbidden. Renovate silently
fails to parse a preset containing comments and applies no manifest at all.
Validate before pushing:

```bash
# 1. strict JSON + shape
jq -e . default.json > /dev/null && echo "strict JSON OK"

# 2. full Renovate schema validation (exit 0 = no errors, no deprecations)
npx --yes --package renovate@41 renovate-config-validator default.json
```

If you need commented documentation, keep it in this README rather than in
`default.json`. (A `default.json5` source-of-truth + generated `default.json`
pair is tracked in plan 05.)

---

## Deviations from plan 02 spec (validated against Renovate 41)

The plan's illustrative `default.json` example was written against an older
Renovate. Three options were deprecated by v41 and were changed here. Each was
confirmed by bisecting the config with `renovate-config-validator`.

| Plan spec | Shipped | Reason |
|---|---|---|
| `matchPaths: []` (rule 9) | `matchFileNames: []` | `matchPaths` renamed to `matchFileNames` |
| `matchPackagePatterns` (rules 4, 5) | `matchPackageNames` | `matchPackagePatterns` deprecated; `matchPackageNames` accepts the same glob/regex strings |
| top-level `prTitle` | removed | `prTitle` is now derived from the `commitMessage*` subcomponents (`commitMessagePrefix` / `Topic` / `Action` / `Extra`), which are all still set |

Resulting PR titles are composed as
`chore(deps): <topic> <action> <extra>` → e.g. `chore(deps): vitest update to 4.2.1`.

Note: plan 05 separately found that top-level `lockFileMaintenance` is also
questionable. Validating it in isolation produced no deprecation warning, so it
was left in place — but it is a candidate for removal when plan 05's dual-file
pattern lands.

---

## Rollback

If the preset misbehaves:

1. Remove `"github>web3eco/renovate-config"` from the consumer's
   `.renovaterc.json` — or delete the file.
2. Re-enable Dependabot by uncommenting `updates:` in `.github/dependabot.yml`
   (kept disabled-but-present in each consumer as the rollback reference).
3. File a follow-up describing what broke.

Consumer `.renovaterc.json` is a local override of the preset; deleting it
reverts that repo to whatever it had before (typically Dependabot-only).

---

## See also

- Plan: `.omo/plans/02-renovate-config-extends.md`
- Deep-dive (lock files, dual-file JSON5 pattern): `.omo/plans/05-renovate-config-deep-dive.md`
- Rollout roadmap: `.omo/plans/04-integrated-rollout-roadmap.md`
- Strategy: `.omo/state/polyrepo-consistency-strategy.md`
- Session state index: `.omo/STATE.md`