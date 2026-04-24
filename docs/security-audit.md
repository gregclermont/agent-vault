# GitHub Actions & Supply Chain Security Audit

Tracking doc for the audit of `agent-vault`'s CI/CD and supply chain.
Branch: `claude/audit-github-actions-security-FN0nY`

Legend: `[ ]` todo · `[~]` in progress · `[x]` done · `[!]` finding · `[-]` N/A

---

## 1. GitHub Actions workflow hardening

- [x] Action pinning: all third-party actions pinned to full commit SHA (not floating tags)
- [x] Official vs third-party inventory: list every non-`actions/*` / non-`github/*` action used
- [x] `permissions:` block present at top-level + tightened per-job (least privilege on `GITHUB_TOKEN`) — `[!]` see F1
- [x] `pull_request_target` / `workflow_run` usage audited; no `checkout` of untrusted PR head with secrets
- [x] Script-injection sinks: no untrusted `${{ github.event.* }}` / `github.head_ref` interpolated into `run:` blocks — `[!]` see F4 (minor)
- [x] Fork-PR secret exposure: publish / signing / cloud creds not reachable from fork PRs
- [x] `environment:` gating on release / publish jobs with required reviewers — `[!]` see F2
- [x] `concurrency:` set where needed (release, deploy) — `[!]` see F3
- [x] Self-hosted runners not used for public-event workflows
- [x] `actions/cache` keys not attacker-controllable; no cache sharing between trusted & untrusted jobs
- [x] Artifact upload/download flows — no `pwn-request` pattern (PR build artifact consumed by privileged workflow)

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

### Section 1 — Workflow hardening

**Positives (what's already right)**

- Every action in all three workflows is pinned to a full commit SHA with a version comment — `actions/checkout`, `actions/setup-go`, `actions/setup-node`, `golangci/golangci-lint-action`, `docker/setup-qemu-action`, `docker/setup-buildx-action`, `docker/login-action`, `sigstore/cosign-installer`, `anchore/sbom-action/download-syft`, `goreleaser/goreleaser-action`.
- `ci.yml` declares `permissions: contents: read` at top level — minimal default.
- No `pull_request_target` or `workflow_run` triggers anywhere.
- `ci.yml` uses no secrets, so fork PRs can't exfiltrate anything via the CI path.
- Release workflows trigger only on `push: tags: [v*]` / `node-sdk/v*.*.*` — not reachable from fork PRs.
- All runners are GitHub-hosted (`ubuntu-latest` / `ubuntu-22.04`); no self-hosted runners.
- No `actions/cache` keys with attacker-controlled inputs; Node cache keyed on lockfile hash via `cache-dependency-path`.
- No cross-workflow artifact consumption (no `pwn-request` pattern).

**Third-party action inventory** (non-`actions/*`, non-`github/*`)

| Action | Owner | Purpose | Notes |
|---|---|---|---|
| `golangci/golangci-lint-action` | golangci | Go linter | Reputable; fetches `golangci-lint v2.11` at runtime (floating minor — acceptable) |
| `docker/setup-qemu-action` | Docker Inc | multi-arch emulation | Official |
| `docker/setup-buildx-action` | Docker Inc | buildx | Official |
| `docker/login-action` | Docker Inc | registry auth | Official |
| `sigstore/cosign-installer` | Sigstore | cosign | Reputable |
| `anchore/sbom-action/download-syft` | Anchore | syft | Reputable |
| `goreleaser/goreleaser-action` | GoReleaser | release orchestration | Pinned SHA but `version: "~> v2"` floats the goreleaser binary itself |

---

**F1 — `contents: write` on `release-node-sdk.yml` is unnecessary** (low)

`.github/workflows/release-node-sdk.yml:11` grants `contents: write`, but the only job runs `npm publish`; it never creates a GitHub release, pushes commits, or tags. `contents: read` is sufficient.

**Recommendation:** drop to `contents: read`.

**F2 — No `environment:` gating on release workflows** (medium)

Neither `release.yml` nor `release-node-sdk.yml` declares an `environment:`. Release-critical secrets (`DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`, `GO_RELEASER_GITHUB_TOKEN`) are plain repo secrets, and npm publishing uses OIDC but is not gated. Any maintainer who can push a `v*` or `node-sdk/v*.*.*` tag can cut a release with no required reviewer.

**Recommendation:** create a protected `release` environment with required reviewers + a tag-pattern deployment branch restriction, and move these secrets/OIDC trust to that environment. Pair with repo-level **tag protection rules** on `v*` and `node-sdk/v*.*.*` so non-admins can't push release tags in the first place. (This partially overlaps with Section 5 — noted there too.)

**F3 — No `concurrency:` on release workflows** (low)

Two simultaneous tag pushes could run two releases in parallel, racing on GitHub Releases / Docker Hub / npm publish. Not a security hole on its own but creates messy state (and could interact badly with any future caching of signing material).

**Recommendation:** add `concurrency: { group: release-${{ github.ref }}, cancel-in-progress: false }` to both release workflows.

**F4 — Unquoted `GITHUB_REF_NAME` shell expansion in `release-node-sdk.yml`** (informational)

`.github/workflows/release-node-sdk.yml:38`:

```yaml
run: npm version ${GITHUB_REF_NAME#node-sdk/v} --allow-same-version --no-git-tag-version
```

`GITHUB_REF_NAME` is a runner env var (not a `${{ }}` template interpolation) so classic GitHub-Actions script injection doesn't apply, and bash does not re-scan parameter expansion for command substitution. Git also forbids spaces in tag names. Exploitability is essentially nil, but as a defense-in-depth nit the expansion should be double-quoted:

```yaml
run: npm version "${GITHUB_REF_NAME#node-sdk/v}" --allow-same-version --no-git-tag-version
```

**F5 — `npm install` instead of `npm ci` in release publish job** (medium, supply chain)

`.github/workflows/release-node-sdk.yml:35` uses `npm install`, which can resolve new versions and mutate `package-lock.json` at publish time. This widens the dependency-pinning window between what CI built/tested and what ships to npm — a classic supply-chain weakness. CI already uses `npm ci` for both `web/` and `sdks/sdk-typescript/`.

**Recommendation:** change to `npm ci`. (Cross-ref Section 3.)

---

### Cross-check: zizmor v1.24.1 (auditor persona, `--collect=all`)

Ran `zizmor --persona=auditor --collect=all .github/`. **25 findings** across 8 rules: 8 high / 10 medium / 4 low / 3 informational.

**Confirmations (zizmor corroborates my findings):**

| Manual | zizmor rule | Count | Verdict |
|---|---|---|---|
| F1 (excessive permissions) | `excessive-permissions` | 5 | **Confirmed & expanded**: zizmor flags *every* workflow-level write permission (not only the redundant `contents: write` on node-sdk). All should be moved to job-level so they're not inherited by any future sibling job. |
| F2 (no environment gating) | `secrets-outside-env` | 3 | **Confirmed**: `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`, `GO_RELEASER_GITHUB_TOKEN` all accessed outside a dedicated environment. |
| F3 (no concurrency) | `concurrency-limits` | 3 | **Confirmed** on all three workflows (I had only flagged release; CI is efficiency-only). |
| F4 (unquoted `GITHUB_REF_NAME`) | — | 0 | **Not in zizmor's scope** (its template-injection audit only covers `${{ }}`, not shell parameter expansion). Finding stands as defense-in-depth. |
| F5 (`npm install` vs `npm ci`) | — | 0 | **Not in zizmor's scope**. Finding stands as a supply-chain concern. |

**New findings from zizmor that I missed:**

**F6 — Missing `cooldown:` on all Dependabot ecosystems** (medium, supply-chain)

`.github/dependabot.yml` has no `cooldown:` block on any of the three ecosystems (gomod, npm, github-actions). Without cooldown, a freshly published malicious version could land in an auto-generated Dependabot PR within hours of upload, before public discovery. Cooldown (e.g., 7 days for patch, 14 for minor) is a cheap supply-chain mitigation.

**Recommendation:** add a `cooldown:` block per ecosystem — example:
```yaml
cooldown:
  default-days: 7
  semver-major-days: 14
```

**F7 — Cache poisoning risk on release workflows** (low, supply-chain)

`release.yml:26` (`setup-go`) and `release.yml:31`, `release-node-sdk.yml:27` (`setup-node` with explicit `cache: npm`) enable language caches. Caches populated by `pull_request` / `push: main` runs (untrusted or less-reviewed code) can be restored inside release jobs if keys overlap. For Go, the module cache is largely content-addressed (lower risk); for npm, the cache stores tarballs keyed on lockfile hash, so a different lockfile defends — but a cache-scope overlap between branches is still possible.

**Recommendation:** either disable caching on release jobs (`cache: false` / explicit `GOMODCACHE` override), or confirm cache scope is release-only.

**F8 — `actions/checkout` default `persist-credentials: true` (artipacked)** (low, informational)

All four `actions/checkout` invocations (ci.yml x2, release.yml x1, release-node-sdk.yml x1) leave the default `persist-credentials: true`, which writes the `GITHUB_TOKEN` into `.git/config` on the runner. Exploitable only if later steps upload the workspace as an artifact or a malicious step reads `.git/config` — neither currently happens — but cheap to harden.

**Recommendation:** set `with: persist-credentials: false` on every checkout (re-enable only where `git push` is needed, which is nowhere in these workflows).

**Cosmetic zizmor findings — not tracked:**
- `anonymous-definition` x3 (jobs missing `name:`) — style only.
- `undocumented-permissions` x1 (`contents: write` missing comment on node-sdk) — moot after F1 fix.

---

---

## Newly added tasks

- [ ] Verify tag protection rules on `v*` and `node-sdk/v*.*.*` (repo setting — may need to ask user; cross-ref F2)
- [ ] Audit the floating `version: "~> v2"` on goreleaser-action + `version: v2.11` on golangci-lint-action — consider pinning the tool binary too (low prio)
- [ ] Consider adding zizmor to CI as a recurring check (uv tool install zizmor; run against `.github/`)
- [ ] Generalise F6: audit **every** package/dependency manager config in the repo for cooldown / delay settings, not just Dependabot. Candidates to check: Renovate (`renovate.json` / `.renovaterc*`), npm (`package.json` → `overrides`, `.npmrc`), Go (`go.mod` `toolchain` directive, any `tools.go`), Docker base-image auto-updaters, pre-commit hook update schedules, and any third-party bot configs under `.github/`. Flag any that can auto-merge or auto-bump without a waiting window.
