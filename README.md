# renovate-config

Shared Renovate presets for repositories owned by `ignazio-ingenito`.

## Presets

- `github>ignazio-ingenito/renovate-config` — default policy with Dependency Dashboard, strict internal checks, immediate PR creation, and minimum release ages of 7 days for patch, 14 days for minor, and 30 days for major updates. Docker updates use `timestamp-optional`: when a registry exposes a release timestamp the normal cooldown applies; when it does not, the update is not blocked indefinitely only because its age cannot be determined.
- `github>ignazio-ingenito/renovate-config:automerge` — extends the default preset and enables PR automerge for patch and digest updates, except Docker updates without a measurable release age, which remain manual.

`prCreation` is intentionally `immediate`: the shared preset must also work in repositories that only run CI for pull requests or protected branches and do not execute checks on `renovate/**` branches. Minimum release age remains the freshness gate when the datasource provides a supported release timestamp.

## Ownership when Dependabot and Renovate coexist

Repository-specific manager ownership is declared in the consuming repository only when two updaters would otherwise overlap.

The standard split adopted during Wave `ignazio-ingenito/developer-workspace#33` is:

- Dependabot may own GitHub Action references (`github-actions`, `depType: action`), which are committed as immutable SHAs with a readable version comment;
- when that split is used, Renovate is disabled **only** for `depType: action`, not for the whole `github-actions` manager;
- Renovate remains owner of `depType: uses-with`, including explicit tool versions such as `aquasecurity/trivy-action` `with.version`;
- the shared 7/14/30-day minimum release age therefore remains the freshness gate for normal Trivy binary upgrades;
- do not duplicate these cooldowns in consumers or force a cross-repository upgrade merely to make all repositories show the same version on the same day.

A repository may bypass the cooldown only for a concrete, documented exception such as a verified security gate with an upstream fix. The exception is local and does not modify this shared policy.
