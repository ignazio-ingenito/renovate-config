# renovate-config

Shared Renovate presets for repositories owned by `ignazio-ingenito`.

## Presets

- `github>ignazio-ingenito/renovate-config` — default policy with Dependency Dashboard, strict internal checks, immediate PR creation, and minimum release ages of 7 days for patch, 14 days for minor, and 30 days for major updates of external dependencies. Docker updates use `timestamp-optional`: when a registry exposes a release timestamp the normal cooldown applies; when it does not, the update is not blocked indefinitely only because its age cannot be determined.
- `github>ignazio-ingenito/renovate-config:automerge` — extends the default preset and enables PR automerge for patch and digest updates, subject to the same freshness and ownership rules.

`prCreation` is intentionally `immediate`: the shared preset must also work in repositories that only run CI for pull requests or protected branches and do not execute checks on `renovate/**` branches. Minimum release age remains the freshness gate for external dependencies when the datasource provides a supported release timestamp.

## OCI CalVer

The proprietary OCI producers consumed by Homelab use immutable calendar versions in the form `YYYY.MM.DD_<run_number>`. `default.json` is the authoritative package list for this rule and applies the same regex ordering to both direct GHCR names and the canonical Harbor `private-ghcr` consumer names.

These **internal OCI releases are explicitly exempt from `minimumReleaseAge`**. Homelab is also a development environment and must be able to consume a newly produced internal CalVer immediately after the producer's own build/test/security gates succeed. Applying the external 7/14/30-day freshness delay to our own images would add latency without providing the intended upstream-stabilization benefit.

The exemption only removes the age wait. It does not weaken the rest of the supply-chain contract: internal releases remain immutable CalVer artifacts, traceable to SHA/digest and subject to the producer, Harbor and consumer/runtime controls that apply to the workload.

Mutable aliases such as `latest`, branch tags or other release tags are not the operational GitOps version source when an immutable CalVer is available.

## Repository self-management

This repository intentionally does **not** use Renovate to manage its own dependencies. The absence of a top-level `renovate.json` is therefore expected, and Mend Renovate reporting that Renovate is disabled because no configuration file exists is not an actionable repository problem by itself.

GitHub Action references used by this repository are owned by Dependabot through `.github/dependabot.yml`. Introducing Renovate self-management requires a new explicit requirement or approved decision; the Mend notification alone is not sufficient justification.

## Ownership when Dependabot and Renovate coexist

Repository-specific manager ownership is declared in the consuming repository only when two updaters would otherwise overlap.

The standard split adopted during Wave `ignazio-ingenito/developer-workspace#33` is:

- Dependabot may own GitHub Action references (`github-actions`, `depType: action`), which are committed as immutable SHAs with a readable version comment;
- when that split is used, Renovate is disabled **only** for `depType: action`, not for the whole `github-actions` manager;
- Renovate remains owner of `depType: uses-with`, including explicit tool versions such as `aquasecurity/trivy-action` `with.version`;
- the shared 7/14/30-day minimum release age remains the freshness gate for normal **external** upgrades managed by Renovate;
- proprietary OCI packages listed in the CalVer rule are the intentional exception and may be proposed immediately;
- do not duplicate these cooldowns in consumers or force a cross-repository upgrade merely to make all repositories show the same external version on the same day.

A repository may bypass the external cooldown only for a concrete, documented exception such as a verified security gate with an upstream fix. The standing internal-OCI exemption is defined centrally in `default.json` and does not need to be re-declared by consumers.
