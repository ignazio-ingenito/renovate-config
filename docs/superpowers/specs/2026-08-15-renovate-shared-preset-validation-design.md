# Renovate Shared-Preset Validation Design

## Context

Wave `ignazio-ingenito/developer-workspace#33` Phase 2 finding F1 identified that `renovate-config` validates `default.json` and `automerge.json` as if they were global self-hosted Renovate configuration.

The repository README defines both files as shared Renovate presets. The current workflow runs:

```bash
npx --yes --package renovate@44.30.3 -- renovate-config-validator default.json automerge.json
```

Renovate's validator supports `--no-global` for validating non-global configuration such as shared presets. The current command therefore uses the right validator but the wrong validation context.

## Goal

Validate the shared presets in their actual non-global context while preserving the existing validator, Renovate version pin, workflow structure, triggers, permissions, and runtime.

## Approved approach

Change exactly one command in `.github/workflows/validate-renovate.yml`:

```bash
npx --yes --package renovate@44.30.3 -- renovate-config-validator --no-global default.json automerge.json
```

Keep unchanged:

- `renovate@44.30.3` pin;
- `renovate-config-validator` as the single validation owner;
- `ubuntu-latest` runner;
- `actions/checkout@v7` and `actions/setup-node@v7` in this finding;
- Node 22;
- workflow triggers and permissions;
- both preset filenames.

## Verification

The change is complete when:

1. the PR product diff contains only `.github/workflows/validate-renovate.yml` plus the approved design/plan documents;
2. the validator command contains `--no-global` exactly once;
3. the Renovate package pin remains `44.30.3`;
4. no other workflow behavior changes;
5. the natural PR validation run succeeds on the exact head without reruns.

## Non-goals

- No GitHub Actions SHA-pinning remediation; that remains finding F2.
- No Renovate version upgrade.
- No preset content changes.
- No new wrapper, script, validator, test framework, or required check.
- No broader workflow cleanup.
