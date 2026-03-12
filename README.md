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
