# renovate-config

Shared Renovate presets for repositories owned by `ignazio-ingenito`.

## Presets

- `github>ignazio-ingenito/renovate-config` — default policy with Dependency Dashboard, strict internal checks, immediate PR creation, and minimum release ages of 7 days for patch, 14 days for minor, and 30 days for major updates. Docker updates use `timestamp-optional`: when a registry exposes a release timestamp the normal cooldown applies; when it does not, the update is not blocked indefinitely only because its age cannot be determined.
- `github>ignazio-ingenito/renovate-config:automerge` — extends the default preset and enables PR automerge for patch and digest updates, except Docker updates without a measurable release age, which remain manual.

`prCreation` is intentionally `immediate`: the shared preset must also work in repositories that only run CI for pull requests or protected branches and do not execute checks on `renovate/**` branches. Minimum release age remains the freshness gate when the datasource provides a supported release timestamp.

Repository-specific manager ownership should be declared in the consuming repository when Renovate shares responsibility with another updater such as Dependabot.
