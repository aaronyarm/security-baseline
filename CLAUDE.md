# security-baseline

Central reusable security workflows (gitleaks, trivy, semgrep, nuclei) for aaronyarm repos. See README.md.

- Pin every third-party action to a full commit SHA with the version in a comment.
- Pin every downloaded tool binary with its sha256 from the release checksums file.
- Run actionlint on .github/workflows after edits.
