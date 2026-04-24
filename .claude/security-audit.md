# GitHub Actions & Supply Chain Security Audit

Tracking doc for the audit of `agent-vault`'s CI/CD and supply chain.
Branch: `claude/audit-github-actions-security-FN0nY`

Legend: `[ ]` todo · `[~]` in progress · `[x]` done · `[!]` finding · `[-]` N/A

---

## 1. GitHub Actions workflow hardening

- [ ] Action pinning: all third-party actions pinned to full commit SHA (not floating tags)
- [ ] Official vs third-party inventory: list every non-`actions/*` / non-`github/*` action used
- [ ] `permissions:` block present at top-level + tightened per-job (least privilege on `GITHUB_TOKEN`)
- [ ] `pull_request_target` / `workflow_run` usage audited; no `checkout` of untrusted PR head with secrets
- [ ] Script-injection sinks: no untrusted `${{ github.event.* }}` / `github.head_ref` interpolated into `run:` blocks
- [ ] Fork-PR secret exposure: publish / signing / cloud creds not reachable from fork PRs
- [ ] `environment:` gating on release / publish jobs with required reviewers
- [ ] `concurrency:` set where needed (release, deploy)
- [ ] Self-hosted runners not used for public-event workflows
- [ ] `actions/cache` keys not attacker-controllable; no cache sharing between trusted & untrusted jobs
- [ ] Artifact upload/download flows — no `pwn-request` pattern (PR build artifact consumed by privileged workflow)

## 2. Release & signing pipeline

- [ ] Enumerate publish targets (GitHub Releases, Docker registry, npm, Homebrew, etc.)
- [ ] Build provenance attestations (`actions/attest-build-provenance` or SLSA) on binaries & images
- [ ] npm publish: token scope, OIDC `id-token: write`, `--provenance` flag, 2FA
- [ ] Docker image signing (cosign) + SBOM (syft) + immutable tags
- [ ] Release trigger surface: tag protection, who can cut a release, required reviews
- [ ] Release-only secrets scoped to the release job (not exposed to CI jobs)
- [ ] `.goreleaser.yml`: checksums, signatures, SBOM, `-trimpath`, reproducible flags

## 3. Supply chain — dependencies

- [ ] `go.mod` / `go.sum`: replace directives reviewed, Go toolchain pinned, `-mod=readonly`
- [ ] Node SDK: lockfile present, `npm ci` in CI, no hostile install scripts in deps
- [ ] Frontend `web/`: lockfile + Dependabot + audit coverage
- [ ] `.github/dependabot.yml` covers: gomod, npm (web), npm (sdks/node), github-actions, docker
- [ ] Vuln scanning in CI: `govulncheck`, `npm audit` / `osv-scanner`
- [ ] `skills-lock.json`: source provenance + integrity check of embedded skills

## 4. Container & install supply chain

- [ ] `Dockerfile` + `Dockerfile.goreleaser`: base images pinned by digest, non-root user, minimal final image
- [ ] `install.sh`: HTTPS-only, checksum/signature verification, hardcoded release source
- [ ] Release assets: `checksums.txt`, `.sig`, SBOM published and verified by `install.sh`

## 5. Repo & org-level controls (best-effort from repo contents)

- [ ] Branch protection on `main` (inferable from workflow `if:` guards / required checks)
- [ ] Tag protection / signed commits signals in repo
- [ ] `CODEOWNERS` file covering workflows, release config, crypto/auth/oauth/session
- [ ] Secret scanning / push protection hints (e.g., `.gitleaks`, pre-commit)
- [ ] Any fine-grained PATs or deploy keys referenced in workflows

## 6. Project-specific sensitive paths

- [ ] Workflows touching `internal/crypto`, `internal/ca`, `internal/auth`, `internal/oauth`, `internal/session` with fork-PR code + secrets
- [ ] Embedded skill docs (`cmd/skill_cli.md`, `cmd/skill_http.md`) tamper path
- [ ] `web/` build → `go:embed` → Go binary: frontend supply-chain reaches the binary

---

## Findings

_(populated as the audit progresses)_

---

## Newly added tasks

_(append here when scope expands)_
