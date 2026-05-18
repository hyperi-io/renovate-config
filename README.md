# renovate-config

Organisation-wide [Renovate Bot](https://docs.renovatebot.com/) configuration presets for `hyperi-io`.

## Usage

Add this to your repo's `renovate.json`:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["github>hyperi-io/renovate-config"]
}
```

This extends from `default.json` in this repo, which provides:

- Weekly update schedule
- Dependency dashboard issue in each repo
- Grouped PRs by ecosystem (Rust, Python, TypeScript, Go, Docker, GitHub Actions)
- Automerge for patch/digest updates and GitHub Actions
- Docker digest pinning and Actions SHA pinning
- `chore(deps):` commit prefix (no semantic-release version bump)
- PR rate limiting (5/hour, 10 concurrent)
- **7-day cooldown for external dependencies; same-org HyperI packages
  bypass** (see *Supply-chain cooldown policy* below)

## Supply-chain cooldown policy

External dependencies (anything not published by `hyperi-io/`) carry a
mandatory **7-day cooldown** before Renovate proposes them. The intent
is to block the fast-moving compromised-release class of supply-chain
attack that dominated 2026: Trivy (March), LiteLLM (March), axios
(March), xinference (April), TanStack / Mini Shai-Hulud worm (May),
sui-execution-cut (May). An analysis of ten prominent 2026 attacks
found eight had exploitation windows under one week; a 7-day cooldown
blocks roughly 80-90 percent of that class without touching legitimate
patch flow.

### Who waits, who doesn't

| Package source | Cooldown | Why |
|---|---|---|
| External registries (crates.io, PyPI, npm, etc.) | **7 days** | External attacker model applies |
| Same-org HyperI packages (`github.com/hyperi-io/*`) | **0 days** | We publish these through our own CI gates and release process; the external-attacker threat model doesn't apply |
| HyperI GHCR containers (`ghcr.io/hyperi-io/*`) | **0 days** | Same as above; matched by package pattern as a backstop for images missing the source label |
| CVE / vulnerability alerts | **0 days** | Security patches ship immediately regardless of source |

### How it's configured

Three packageRule blocks in `default.json`:

1. Global `"minimumReleaseAge": "7 days"` at root with
   `"minimumReleaseAgeBehaviour": "timestamp-required"` (Renovate
   41.150.0+ safer default; deps without a publishable timestamp are
   treated as not yet aged).
2. `"matchSourceUrls": ["https://github.com/hyperi-io/**"]` ->
   `"minimumReleaseAge": "0 days"` -- catches HyperI org packages
   across every Renovate datasource that reads `repository` /
   `Project-URL: Source` / `repository.url` from package metadata,
   plus GitHub Actions whose org is in the action ref.
3. `"matchDatasources": ["docker"]` +
   `"matchPackagePatterns": ["^ghcr\\.io/hyperi-io/"]` ->
   `"minimumReleaseAge": "0 days"` -- backstop for any GHCR HyperI
   image that lacks `org.opencontainers.image.source` labels.
4. `vulnerabilityAlerts: { "minimumReleaseAge": "0 days", "automerge":
   true }` nested override -- CVE fixes bypass the cooldown.

The `security:minimumReleaseAgeNpm` preset is explicitly ignored via
`ignorePresets` so the global 7-day rule applies uniformly across all
datasources (without this, the nested npm-only 3-day preset would
silently override the root for npm packages).

### How it interacts with automerge

`automerge: true` on patch + digest updates still applies. The
cooldown delays Renovate from PROPOSING the update; once the PR opens,
it merges normally if CI passes. Net effect for external deps: patch
PRs land 7 days after upstream release. For same-org HyperI deps:
they land on the next Renovate scan after publication.

### What the cooldown doesn't fix

Cooldowns are one layer. They block fast-moving registry-poisoning
attacks. They do **not** address:

- Typosquatting (need behavioural scanning -- Socket.dev, GuardDog,
  Snyk, or an internal proxy with quarantine)
- Long-term maintainer compromise (xz-utils style)
- Zero-days in deps already installed (need `cargo audit` /
  `pip-audit` / `npm audit` / `govulncheck` -- which the HyperI CI
  quality stage already runs)
- Historical commits that introduced a now-known-malicious package
  (need `git-scrub` for incident-response history rewrites)

The cooldown is one layer of a layered defence -- prevention +
detection + historical cleanup. Each addresses a different time
window.

### Sources

- [Renovate -- Minimum Release Age](https://docs.renovatebot.com/key-concepts/minimum-release-age/)
- [cooldowns.dev](https://cooldowns.dev/)
- [Yossarian -- We should all be using dependency cooldowns](https://blog.yossarian.net/2025/11/21/We-should-all-be-using-dependency-cooldowns)
- [Andrew Nesbitt -- Package Managers Need to Cool Down (March 2026)](https://nesbitt.io/2026/03/04/package-managers-need-to-cool-down.html)
- [Datadog Security Labs -- the case for cooldowns post-axios](https://securitylabs.datadoghq.com/articles/dependency-cooldowns/)

## JFrog Private Registry Access

For repos that use JFrog, add `hostRules` to the repo-level `renovate.json`:

```json
{
  "extends": ["github>hyperi-io/renovate-config"],
  "hostRules": [
    {
      "matchHost": "https://hypersec.jfrog.io",
      "hostType": "cargo",
      "username": "artifactory@hypersec.io",
      "encrypted": {
        "password": "<encrypted-token>"
      }
    }
  ]
}
```

Encrypt tokens at https://app.renovatebot.com/encrypt.

## Overriding

Repos can override any setting by adding it to their own `renovate.json` alongside the `extends`.

## Licence

Proprietary — HYPERI PTY LIMITED
