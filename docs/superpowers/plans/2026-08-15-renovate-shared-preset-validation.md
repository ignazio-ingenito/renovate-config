# Renovate Shared-Preset Validation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Validate the repository's shared Renovate presets in non-global context by adding `--no-global` to the existing validator command.

**Architecture:** Keep the existing workflow, Renovate version pin, runner, actions, Node version, triggers and permissions unchanged. Modify only the validator invocation so `default.json` and `automerge.json` are validated as shared presets rather than global self-hosted config.

**Tech Stack:** GitHub Actions, Node.js 22, `renovate-config-validator` from `renovate@44.30.3`.

## Global Constraints

- Scope is only Wave #33 Phase 2 finding F1.
- Keep `renovate@44.30.3` unchanged.
- Keep both preset files unchanged.
- Do not touch GitHub Actions SHA pinning; that is F2.
- Do not add wrappers, scripts, validators, tests, required checks or workflow layers.
- Use only the natural PR validation run; no reruns without explicit authorization.

---

### Task 1: Validate shared presets in non-global mode

**Files:**
- Modify: `.github/workflows/validate-renovate.yml`
- Verify: `default.json`
- Verify: `automerge.json`

**Interfaces:**
- Consumes: existing `renovate-config-validator` invocation and pinned Renovate package.
- Produces: validation of both files using `--no-global`.

- [ ] **Step 1: Change exactly one command**

Replace:

```bash
npx --yes --package renovate@44.30.3 -- renovate-config-validator default.json automerge.json
```

with:

```bash
npx --yes --package renovate@44.30.3 -- renovate-config-validator --no-global default.json automerge.json
```

- [ ] **Step 2: Verify diff scope**

Expected product diff:

```text
.github/workflows/validate-renovate.yml: one token-group addition, --no-global
```

Fail review if the Renovate pin, actions, Node version, runner, triggers, permissions or preset files change.

- [ ] **Step 3: Open one draft PR tied to Wave #33**

Record exact base/head and state that F2 action pinning is explicitly out of scope.

- [ ] **Step 4: Let natural validation run once**

Expected evidence:

```text
Validate Renovate presets: SUCCESS
run_attempt: 1
```

Diagnose failures from the existing run; do not rerun automatically.

- [ ] **Step 5: Exact-head review**

Confirm:

```text
- product diff is only --no-global
- renovate@44.30.3 unchanged
- default.json/automerge.json unchanged
- no new tool or workflow layer
- no F2 changes
```

- [ ] **Step 6: Merge only after explicit authorization**

Use squash merge with `expected_head_sha`, then verify `main` and branch deletion.
