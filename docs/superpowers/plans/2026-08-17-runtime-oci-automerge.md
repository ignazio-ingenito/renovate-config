# Runtime OCI Automerge Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Enable Renovate PR automerge only for approved proprietary OCI runtime workloads, while keeping one-shot/migration images and external Docker packages outside this automatic promotion path.

**Architecture:** Keep `renovate-config` as the single authoritative shared policy point. Add one final, specific Docker `packageRule` to `automerge.json` that overrides the existing generic Docker automerge disablement only for the closed set of approved runtime package names. Do not add workflows, credentials, webhooks, repository-to-repository writes, or Argo CD actions.

**Tech Stack:** Renovate shared presets (`default.json`, `automerge.json`), JSON, GitHub pull requests.

## Global Constraints

- Runtime automerge applies only to `developer-workspace`, `aeris`, `skunklabs`, `baialupo.com`, `iwant`, `club-aviazione-popolare-web`, and `club-aviazione-popolare-cms`.
- Match both direct `ghcr.io/ignazio-ingenito/...` package names and canonical `harbor.lab.skunklabs.uk/private-ghcr/ignazio-ingenito/...` consumer names.
- `iwant-migrator` and `prosignal` must remain excluded from runtime automerge.
- Do not change CalVer versioning, `minimumReleaseAge`, manager ownership, `platformAutomerge`, or unrelated automerge behavior.
- Do not introduce workflows, PATs, webhooks, imperative Argo CD calls, or other custom delivery mechanisms.
- RFC-0001 remains authoritative for simplicity, scope, traceability, verification, and GitHub Actions cost controls.

---

### Task 1: Add the bounded runtime OCI automerge rule

**Files:**
- Modify: `automerge.json`

**Interfaces:**
- Consumes: existing shared preset inheritance and existing generic Docker rule that sets `automerge: false` for Docker patch/digest updates.
- Produces: one final more-specific Docker rule with the exact approved runtime package-name allowlist and `automerge: true`, `automergeType: "pr"`.

- [ ] **Step 1: Capture the current `automerge.json` as the baseline**

Read the branch version and confirm the existing order is:

```json
{
  "packageRules": [
    { "matchUpdateTypes": ["patch", "digest"], "automerge": true },
    { "matchUpdateTypes": ["minor", "major"], "automerge": false },
    {
      "matchDatasources": ["docker"],
      "matchUpdateTypes": ["patch", "digest"],
      "automerge": false
    },
    {
      "matchDatasources": ["docker"],
      "matchUpdateTypes": ["patch", "digest"],
      "matchJsonata": ["$exists(releaseTimestamp)"],
      "automerge": true
    }
  ]
}
```

The exact file contains the same logical rules plus `automergeType` where applicable. If this baseline no longer holds, stop and reconcile the plan before modifying policy.

- [ ] **Step 2: Add the final allowlist rule**

Append this rule as the final entry in `packageRules` so it is more specific and later than the generic Docker rules:

```json
{
  "matchDatasources": [
    "docker"
  ],
  "matchPackageNames": [
    "ghcr.io/ignazio-ingenito/developer-workspace",
    "harbor.lab.skunklabs.uk/private-ghcr/ignazio-ingenito/developer-workspace",
    "ghcr.io/ignazio-ingenito/aeris",
    "harbor.lab.skunklabs.uk/private-ghcr/ignazio-ingenito/aeris",
    "ghcr.io/ignazio-ingenito/skunklabs",
    "harbor.lab.skunklabs.uk/private-ghcr/ignazio-ingenito/skunklabs",
    "ghcr.io/ignazio-ingenito/baialupo.com",
    "harbor.lab.skunklabs.uk/private-ghcr/ignazio-ingenito/baialupo.com",
    "ghcr.io/ignazio-ingenito/iwant",
    "harbor.lab.skunklabs.uk/private-ghcr/ignazio-ingenito/iwant",
    "ghcr.io/ignazio-ingenito/club-aviazione-popolare-web",
    "harbor.lab.skunklabs.uk/private-ghcr/ignazio-ingenito/club-aviazione-popolare-web",
    "ghcr.io/ignazio-ingenito/club-aviazione-popolare-cms",
    "harbor.lab.skunklabs.uk/private-ghcr/ignazio-ingenito/club-aviazione-popolare-cms"
  ],
  "automerge": true,
  "automergeType": "pr"
}
```

Do not add `matchUpdateTypes`: the package's CalVer ordering is already defined centrally in `default.json`, and this rule is intended to govern the approved proprietary runtime package updates themselves rather than reclassify their version semantics.

- [ ] **Step 3: Validate the JSON syntax independently**

Run a JSON parser against both policy files:

```bash
python -m json.tool automerge.json >/dev/null
python -m json.tool default.json >/dev/null
```

Expected: both commands exit `0` with no output.

- [ ] **Step 4: Verify the allowlist and exclusions structurally**

Run a short read-only validation script:

```bash
python - <<'PY'
import json
from pathlib import Path

cfg = json.loads(Path('automerge.json').read_text())
rules = cfg['packageRules']
runtime = rules[-1]

expected = {
    'ghcr.io/ignazio-ingenito/developer-workspace',
    'harbor.lab.skunklabs.uk/private-ghcr/ignazio-ingenito/developer-workspace',
    'ghcr.io/ignazio-ingenito/aeris',
    'harbor.lab.skunklabs.uk/private-ghcr/ignazio-ingenito/aeris',
    'ghcr.io/ignazio-ingenito/skunklabs',
    'harbor.lab.skunklabs.uk/private-ghcr/ignazio-ingenito/skunklabs',
    'ghcr.io/ignazio-ingenito/baialupo.com',
    'harbor.lab.skunklabs.uk/private-ghcr/ignazio-ingenito/baialupo.com',
    'ghcr.io/ignazio-ingenito/iwant',
    'harbor.lab.skunklabs.uk/private-ghcr/ignazio-ingenito/iwant',
    'ghcr.io/ignazio-ingenito/club-aviazione-popolare-web',
    'harbor.lab.skunklabs.uk/private-ghcr/ignazio-ingenito/club-aviazione-popolare-web',
    'ghcr.io/ignazio-ingenito/club-aviazione-popolare-cms',
    'harbor.lab.skunklabs.uk/private-ghcr/ignazio-ingenito/club-aviazione-popolare-cms',
}

actual = set(runtime['matchPackageNames'])
assert runtime['matchDatasources'] == ['docker']
assert runtime['automerge'] is True
assert runtime['automergeType'] == 'pr'
assert actual == expected, (actual - expected, expected - actual)
assert not any('iwant-migrator' in p for p in actual)
assert not any('prosignal' in p for p in actual)
print('runtime OCI automerge allowlist: PASS')
PY
```

Expected output:

```text
runtime OCI automerge allowlist: PASS
```

- [ ] **Step 5: Verify that default CalVer ownership and freshness policy are unchanged**

Compare `default.json` to the branch base and confirm there is no diff:

```bash
git diff --exit-code main...HEAD -- default.json
```

Expected: exit `0`, no diff.

Also inspect the final diff and confirm the only policy behavior change is the bounded runtime allowlist in `automerge.json` plus the approved spec/plan documentation.

- [ ] **Step 6: Commit the implementation**

```bash
git add automerge.json docs/superpowers/specs/2026-08-17-runtime-oci-automerge-design.md docs/superpowers/plans/2026-08-17-runtime-oci-automerge.md
git commit -m "feat(renovate): automerge proprietary runtime images"
```

---

### Task 2: Open and verify the policy pull request

**Files:**
- Review only: `automerge.json`
- Review only: `docs/superpowers/specs/2026-08-17-runtime-oci-automerge-design.md`
- Review only: `docs/superpowers/plans/2026-08-17-runtime-oci-automerge.md`

**Interfaces:**
- Consumes: committed implementation from Task 1.
- Produces: one reviewable PR targeting `main`; no merge or CI rerun is implicitly authorized.

- [ ] **Step 1: Open a pull request**

Use title:

```text
feat(renovate): automerge proprietary runtime images
```

PR body must state:

```markdown
## Summary

- enables Renovate PR automerge only for approved proprietary OCI runtime workloads;
- covers both direct GHCR and canonical `private-ghcr` package names;
- keeps `iwant-migrator` and `prosignal` outside automatic runtime promotion;
- adds no workflow, credential, webhook or imperative deployment mechanism.

## Verification

- `automerge.json` and `default.json` parse as valid JSON;
- runtime allowlist exactly matches the approved 14 package identities;
- `iwant-migrator` and `prosignal` are absent from the automerge allowlist;
- `default.json` remains unchanged.

## Delivery behavior

Producer gate -> Renovate PR in Homelab -> required checks -> Renovate PR automerge -> Argo CD reconciliation.

No CI rerun is requested by this PR.
```

- [ ] **Step 2: Inspect the PR patch**

Confirm changed files are limited to:

```text
automerge.json
docs/superpowers/specs/2026-08-17-runtime-oci-automerge-design.md
docs/superpowers/plans/2026-08-17-runtime-oci-automerge.md
```

If any other file changed, classify it as scope expansion and remove it before proceeding.

- [ ] **Step 3: Inspect existing CI status without rerunning anything**

Read the PR/head checks. Per RFC-0001, do not rerun failed or skipped jobs automatically. If a job fails before its first step or shows infrastructure/billing symptoms, stop and report it rather than retrying.

- [ ] **Step 4: Perform final RFC-0001 completion check**

Record:

```text
SÌ — requirement satisfied: bounded runtime automerge rule exists.
SÌ — simpler solution used: existing Renovate shared preset, no new delivery mechanism.
SÌ — scope respected: runtime OCI only; one-shot/migration excluded.
SÌ — authoritative source preserved: shared Renovate preset remains the policy owner.
SÌ — verification observable: JSON parse + exact allowlist assertions + PR patch review.
NON APPLICABILE — application tests: this repository contains configuration policy, not application runtime code.
```

Do not merge the policy PR unless merge authority is separately established by the user or active project source.
