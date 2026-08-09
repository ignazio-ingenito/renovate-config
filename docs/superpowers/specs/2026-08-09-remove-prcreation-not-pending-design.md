# Remove `prCreation: not-pending` Design

## Context

The shared default preset currently combines `internalChecksFilter: "strict"`
with `prCreation: "not-pending"`. The latter was added to avoid opening pull
requests before `minimumReleaseAge` had elapsed. Current Renovate semantics make
that responsibility belong to `internalChecksFilter`: with strict filtering, an
update that has not passed the age check does not get a branch or pull request.

`prCreation: "not-pending"` controls a later stage. After Renovate has created a
branch, it waits for branch checks to complete before opening the pull request.
For consumers whose checks run only on `pull_request`, this creates a dependency
cycle. Renovate falls back to opening the pull request after 24 hours because no
branch checks exist.

## Goal

Restore immediate pull-request creation for updates that have already passed
Renovate's internal filtering, while preserving the existing release-age policy
and automerge safeguards.

## Scope

- Remove only `prCreation: "not-pending"` from `default.json`.
- Keep `internalChecksFilter: "strict"` explicit.
- Keep the existing 7/14/30-day `minimumReleaseAge` rules unchanged.
- Keep `automerge.json`, including patch and digest automerge rules, unchanged.
- Validate both shared preset JSON files and inspect the resulting diff.
- Publish the change for review without merging it automatically.

## Non-goals

- Do not add or change a `minimumReleaseAge` policy for digest updates.
- Do not alter automerge eligibility or GitHub Actions workflows.
- Do not change consumer repositories as part of this change.
- Do not force or recreate existing Renovate pull requests.

## Expected Behavior

1. Renovate evaluates `minimumReleaseAge` using the existing package rules.
2. `internalChecksFilter: "strict"` suppresses branches for updates whose
   applicable age check is pending.
3. Once an update passes internal filtering, Renovate creates its branch and
   pull request in the same run using the default `prCreation: "immediate"`.
4. Consumer workflows triggered by `pull_request` start immediately.
5. Renovate automerge remains gated by consumer checks because
   `platformAutomerge` remains false and tests are not ignored.

## Consumer Impact

All repositories extending the shared default preset inherit the timing change.
Pull requests may become visible while CI is still running instead of appearing
only after branch checks finish. This is intentional. Release-age filtering is
unchanged, and repositories retain their existing CI and automerge gates.

Repositories that run checks directly on Renovate branch pushes lose the
notification-delay behavior provided by `not-pending`; they do not lose the
checks themselves. No active consumer dependency on that delay was found during
the pre-change review.

## Validation

- Parse `default.json` and `automerge.json` as JSON.
- Run Renovate's configuration validator if the repository exposes or can run
  the official validator in the existing environment.
- Confirm the diff removes exactly one property and does not change age or
  automerge rules.
- Confirm the homelab Filestash scenario is explained by the new flow: a mature
  branch gets an immediate pull request, allowing pull-request-only workflows to
  start.

## Rollback

Restore `prCreation: "not-pending"` in `default.json`. The rollback is a single
property change and does not require consumer repository changes or data
migration.

## Residual Decision

Whether digest updates need an explicit `minimumReleaseAge` is a separate policy
decision. It must not be bundled with this behavioral correction.
