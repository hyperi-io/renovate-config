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
- Grouped PRs by ecosystem (Rust, Python, TypeScript, Go, Docker, GitHub
  Actions), by the annotated `custom.regex` pins, and by the Kubernetes /
  IaC managers (helm, terraform)
- Coupled pairs grouped so they move together: kafka client+broker,
  ClickHouse client+server, AWS SDK + S3-compatible servers, testcontainers
  and the images it supplies defaults for
- **PR-only. NOTHING automerges** -- `:automergeDisabled` is in `extends`
- Actions SHA pinning
- `fix(deps):` commit prefix, so merging a dependency PR actually ships
  (see *Why the prefix is `fix(deps):`* below)
- PR rate limiting (5/hour, 10 concurrent)
- **7-day cooldown for external dependencies; same-org HyperI packages
  bypass** (see *Supply-chain cooldown policy* below)

## Why the prefix is `fix(deps):`

semantic-release's default rules release on `fix` / `feat` / `perf` and
nothing else. A `chore(deps):` bump therefore lands on main and never ships
-- and because the stack deploys by digest, an unreleased bump is an
undeployed one. The CVE PRs this preset works hardest to raise early were
exactly the ones that reached main and stopped there.

Release count does not scale with PR count. semantic-release analyses every
commit since the last tag and cuts ONE version at the highest bump, so a
cycle with eight grouped `fix(deps):` merges produces one patch release, not
eight. The only new releases are the cycles where dependencies were the only
change -- which is the gap being closed.

This follows the existing org convention rather than inventing one: a
security patch already ships as `fix(security):` (see `hyperi_ci.release_rules`).
The way to make something releasable is to type it `fix`, not to teach the
analyser new types -- hyperi-ci's tagger config carries no custom
`releaseRules` on purpose.

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

One root setting and three packageRule blocks in `default.json`:

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
   `"matchPackageNames": ["ghcr.io/hyperi-io/**"]` ->
   `"minimumReleaseAge": "0 days"` -- backstop for any GHCR HyperI
   image that lacks `org.opencontainers.image.source` labels.
4. `vulnerabilityAlerts: { "minimumReleaseAge": "0 days" }` nested
   override -- CVE fixes bypass the cooldown. No automerge: they are
   PR-only like everything else.

Verified firing, not assumed: `brace-expansion@5.0.8` was raised 5.7 days
after publication and `next-auth@4.24.15` at 6.3 days, both inside the
cooldown.

The `security:minimumReleaseAgeNpm` preset is explicitly ignored via
`ignorePresets` so the global 7-day rule applies uniformly across all
datasources (without this, the nested npm-only 3-day preset would
silently override the root for npm packages).

### How it interacts with merging

Nothing automerges -- `:automergeDisabled` is in `extends`, and that is org
policy. The cooldown only delays Renovate from PROPOSING the update; a human
reviews and merges every PR, security and same-org included. Net effect for
external deps: the PR appears 7 days after upstream release. For same-org
HyperI deps: on the next Renovate scan after publication.

### A second cooldown will defeat this one

The cooldown here has a security exception. A cooldown configured in the
PACKAGE MANAGER does not, so it silently outlives the bypass: Renovate
raises the CVE PR early and the package manager then refuses to install the
package until its own gate expires, leaving the PR red for the difference.

dfe-ui hit exactly this -- `npmMinimalAgeGate: '7d'` in `.yarnrc.yml` against
a CVE PR raised at 5.7 days, failing `YN0016 ... quarantined`. Do not add a
package-manager age gate on top of this preset; it duplicates a control that
already exists and strips its one exception.

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

## Git submodules

Renovate ships the `git-submodules` manager **disabled** -- it is opt-in beta.
A repo that vendors a submodule therefore gets no PRs for it, and nothing
reports the silence. This preset enables it for the whole org.

That default cost us. dfe-schemas is consumed as a submodule by dfe-engine,
dfe-loader and dfe-fetcher; with nothing watching, the three pins drifted to
three different commits -- engine current, loader 26 behind, fetcher 27. The
stale two were missing the `_org_id` rename on `detection_checkpoint` and the
fix for JSON columns that cannot be `Nullable`, so two apps were building
against a schema definition the third had already corrected.

**Every consumer must name a branch in `.gitmodules`:**

    [submodule "schemas"]
        path = schemas
        url = https://github.com/hyperi-io/dfe-schemas.git
        branch = main

Renovate tracks the branch stated there. It also makes
`git submodule update --remote` agree with the bot, so a human and Renovate
move the pointer to the same place.

**A bot to move it, a gate to prove it moved.** Renovate raises the PR; it
does not stop one repo sitting on an unmerged bump for a month. Where several
repos must agree on one submodule commit, pair this with a drift check that
fails CI when they diverge -- dfe-infra's `check_submodule_drift.py` is the
reference implementation.

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
