# security-baseline

Shared security checks for personal repos. Each repo has a small caller workflow that points here, so tool versions and settings are changed in one place.

## What runs, and when

| Tool | When | Blocks the build? | Scope |
|---|---|---|---|
| gitleaks (pre-commit) | Every local `git commit` | Yes, the commit is refused | Staged changes |
| gitleaks (CI) | push, pull_request | Yes | New commits only (PR range, or the push range; new branches scan commits that are not on the default branch yet) |
| trivy | push, pull_request, weekly | Only fixable HIGH/CRITICAL (`--ignore-unfixed`) | Dependencies (lockfiles) and misconfigurations (Dockerfile, compose, Terraform, Kubernetes, ...). Secret scanning is left to gitleaks. |
| semgrep | push, pull_request | No (report-only) | `p/default` community rules, official container, metrics off |
| nuclei | weekly, only where you set targets | No (report-only) | Domains you own. Rate-limited to 10 req/s, intrusive/dos/fuzz templates excluded |

## Where results show up

- **Public repos:** SARIF is uploaded to the GitHub code scanning page (Security > Code scanning).
- **Private repos:** results go to the job summary of each run and are kept as an artifact for 14 days. Uploading SARIF to code scanning on a private repo needs GitHub Advanced Security, so it is skipped automatically. Visibility is detected at run time.

## Adding a repo

1. Copy `templates/security-caller.yml` to `.github/workflows/security.yml` in the repo.
2. Copy `templates/pre-commit-config.yaml` to `.pre-commit-config.yaml`, then run:

   ```sh
   pre-commit install
   pre-commit run --all-files
   ```

   The gitleaks hook is the `golang` variant. pre-commit downloads its own Go toolchain the first time, so you do not need Go or Docker installed.
3. If the repo has no real code (docs, static assets), set `run-semgrep: false` in the caller.

The calling job must grant `contents: read` and `security-events: write`. The template already does this.

## Inputs (`security.yml`)

| Input | Default | Meaning |
|---|---|---|
| `run-gitleaks` | `true` | Secret scan on push / PR |
| `run-trivy` | `true` | Dependency + misconfiguration scan |
| `run-semgrep` | `true` | Static analysis, report-only |
| `trivy-severity-gate` | `HIGH,CRITICAL` | Severities that fail the build (fixable ones only) |
| `semgrep-config` | `p/default` | Semgrep ruleset |
| `nuclei-targets` | `""` | URLs you own. When set, nuclei also runs on the weekly schedule |

`nuclei.yml` takes `targets` (required), `rate-limit` (default 10), `severity` and `exclude-tags`.

## Enabling nuclei for a repo

Only do this for a domain you own and have deployed. Add the targets to the caller:

```yaml
    with:
      nuclei-targets: |
        https://example-you-own.com
```

## Tuning

- **A false positive from gitleaks:** add its fingerprint (printed in the CI log or by `gitleaks git --redact`) to `.gitleaksignore` in that repo, with a `#` comment line above it saying why. If the finding is a real secret, rotate the credential first. Removing it from git history is a separate and destructive step.
- **An accepted trivy finding:** add the ID (for example `CVE-2026-12345` or `AVD-DS-0002`) to `.trivyignore` in that repo, with a comment saying why and when to recheck.
- **Noisy semgrep rules:** add `# nosemgrep: <rule-id>` on the line, or add paths to `.semgrepignore`.
- **Upgrading a tool:** change the version and the pinned `sha256` in `env:` at the top of the workflow. Take the hash from the release's `checksums.txt`. Actions are pinned to commit SHAs, with the version in a comment. Update both together.

## Access

Other repos can only call these workflows if this repo allows it.

- **Public `security-baseline`:** any repo can call it. Public callers (for example `momentum-backtest`) require this, because a private reusable workflow cannot be called from a public repo.
- **Private `security-baseline`:** go to Settings > Actions > General > Access and choose "Accessible from repositories owned by the user 'aaronyarm'". This works for private callers only.
