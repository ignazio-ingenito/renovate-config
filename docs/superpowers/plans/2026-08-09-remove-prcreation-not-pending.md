# Immediate Renovate PR Creation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Remove the shared `prCreation: "not-pending"` override so consumer pull requests are created immediately after Renovate internal filtering succeeds.

**Architecture:** Change only the owner preset in `default.json`; all consumers then inherit Renovate's default `prCreation: "immediate"`. Preserve strict internal filtering, release-age thresholds, and automerge rules, then publish the reviewable change without merging it.

**Tech Stack:** Renovate JSON presets, `jq`, official `renovate-config-validator`, Git, GitHub CLI.

---

### Task 1: Prove the current preset violates the desired contract

**Files:**
- Inspect: `default.json`
- Inspect: `automerge.json`

- [ ] **Step 1: Assert the desired property state**

Run:

```bash
jq -e '.internalChecksFilter == "strict" and has("prCreation") == false and ([.packageRules[].minimumReleaseAge] == ["7 days", "14 days", "30 days"])' default.json
```

Expected: exit 1 because `default.json` still contains `prCreation`.

- [ ] **Step 2: Record the preserved automerge contract**

Run:

```bash
jq -e '.packageRules[0].matchUpdateTypes == ["patch", "digest"] and .packageRules[0].automerge == true and .packageRules[0].automergeType == "pr"' automerge.json
```

Expected: exit 0 and output `true`.

### Task 2: Apply the minimal owner-preset change

**Files:**
- Modify: `default.json:9`

- [ ] **Step 1: Remove the non-default PR timing override**

Change:

```json
  "internalChecksFilter": "strict",
  "prCreation": "not-pending",
  "packageRules": [
```

to:

```json
  "internalChecksFilter": "strict",
  "packageRules": [
```

- [ ] **Step 2: Re-run the desired-contract assertion**

Run:

```bash
jq -e '.internalChecksFilter == "strict" and has("prCreation") == false and ([.packageRules[].minimumReleaseAge] == ["7 days", "14 days", "30 days"])' default.json
```

Expected: exit 0 and output `true`.

- [ ] **Step 3: Confirm the automerge contract remains unchanged**

Run:

```bash
jq -e '.packageRules[0].matchUpdateTypes == ["patch", "digest"] and .packageRules[0].automerge == true and .packageRules[0].automergeType == "pr"' automerge.json
```

Expected: exit 0 and output `true`.

### Task 3: Validate syntax, semantics, and containment

**Files:**
- Validate: `default.json`
- Validate: `automerge.json`

- [ ] **Step 1: Parse both files as strict JSON**

Run:

```bash
jq empty default.json automerge.json
```

Expected: exit 0 with no output.

- [ ] **Step 2: Run the official Renovate preset validator**

Run:

```bash
npx --yes --package renovate renovate-config-validator default.json automerge.json
```

Expected: exit 0 and validation success for both files. If the CLI accepts only one path, invoke it once per file and require both invocations to exit 0.

- [ ] **Step 3: Verify the implementation diff is exactly one property removal**

Run:

```bash
git diff --check
git diff -- default.json automerge.json
git diff --numstat -- default.json automerge.json
```

Expected: `git diff --check` exits 0; `default.json` has `0` additions and `1` deletion; `automerge.json` has no diff.

- [ ] **Step 4: Commit the behavior change**

Run:

```bash
git add default.json
git commit -m "fix(renovate): restore immediate PR creation"
```

Expected: one commit containing only the `default.json` property removal.

### Task 4: Review and publish without merging

**Files:**
- Review: `docs/superpowers/specs/2026-08-09-remove-prcreation-not-pending-design.md`
- Review: `docs/superpowers/plans/2026-08-09-remove-prcreation-not-pending.md`
- Review: `default.json`

- [ ] **Step 1: Review the complete branch diff and commits**

Run:

```bash
git diff --check origin/main...HEAD
git diff --stat origin/main...HEAD
git diff origin/main...HEAD
git log --oneline origin/main..HEAD
```

Expected: only the approved design, implementation plan, and one-property behavior change; no unrelated modifications.

- [ ] **Step 2: Push the branch**

Run:

```bash
git push -u origin docs/remove-prcreation-not-pending-design
```

Expected: the remote branch is created and upstream tracking is configured.

- [ ] **Step 3: Open a draft pull request**

Run:

```bash
gh pr create --repo ignazio-ingenito/renovate-config --draft --base main --head docs/remove-prcreation-not-pending-design --title "fix: restore immediate Renovate PR creation" --body-file /tmp/renovate-pr-body.md
```

The PR body must summarize the incorrect original assumption, preserved age and automerge contracts, validation evidence, consumer timing impact, rollback, and the separate unresolved digest-age policy.

Expected: a draft PR URL. Do not merge the PR.
