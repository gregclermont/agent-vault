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

- [x] Enumerate publish targets (GitHub Releases, Docker registry, npm, Homebrew, etc.)
- [x] Build provenance attestations (`actions/attest-build-provenance` or SLSA) on binaries & images — `[!]` see F11
- [x] npm publish: token scope, OIDC `id-token: write`, `--provenance` flag, 2FA
- [x] Docker image signing (cosign) + SBOM (syft) + immutable tags — `[!]` see F9, F10, F12
- [x] Release trigger surface: tag protection, who can cut a release, required reviews — cross-ref F2
- [x] Release-only secrets scoped to the release job (not exposed to CI jobs) — cross-ref F2
- [x] `.goreleaser.yml`: checksums, signatures, SBOM, `-trimpath`, reproducible flags — `[!]` see F13, F14

## 3. Supply chain — dependencies

- [x] `go.mod` / `go.sum`: replace directives reviewed, Go toolchain pinned, `-mod=readonly` — `[!]` see F19 (toolchain)
- [x] Node SDK: lockfile present, `npm ci` in CI, no hostile install scripts in deps — cross-ref F5, F18
- [x] Frontend `web/`: lockfile + Dependabot + audit coverage
- [x] `.github/dependabot.yml` covers: gomod, npm (web), npm (sdks/node), github-actions, docker — `[!]` see F16
- [x] Vuln scanning in CI: `govulncheck`, `npm audit` / `osv-scanner` — `[!]` see F15
- [x] `skills-lock.json`: source provenance + integrity check of embedded skills — `[!]` see F17

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

### Section 2 — Release & signing pipeline

**Publish targets (inventory)**

| Target | What ships | Config | Signing | Provenance |
|---|---|---|---|---|
| GitHub Releases | `agent-vault_{ver}_{os}_{arch}.tar.gz` for linux/darwin × amd64/arm64 + `checksums.txt` + `checksums.txt.bundle` (cosign) + SBOMs (syft, per archive) | `.goreleaser.yml` | checksum file cosign-signed (keyless OIDC) | **none** (no SLSA attestation) |
| Docker Hub | `infisical/agent-vault:{ver}-amd64`, `:{ver}-arm64`, multi-arch manifest `:{ver}` and `:latest` | `.goreleaser.yml` + `Dockerfile.goreleaser` | **not signed** | **none** |
| npm | `@infisical/agent-vault-sdk` (from `sdks/sdk-typescript/`) | `release-node-sdk.yml` | Sigstore via `npm publish --provenance` | **yes (npm-native)** |
| Homebrew tap | `Infisical/homebrew-get-cli` | `.goreleaser.yml` (commented out) | — | — |

**Positives (what's already right)**

- Go binary built with `-trimpath`, `-s -w`, `CGO_ENABLED=0` (static), and `mod_timestamp: {{.CommitTimestamp}}` — a decent step toward reproducibility.
- `checksum: sha256` + cosign `sign-blob` with `--yes` (keyless, OIDC via `id-token: write`).
- SBOMs generated for tarball archives (`sboms: [artifacts: archive]`).
- npm publish uses OIDC (no `NODE_AUTH_TOKEN` in workflow) + `--provenance` + `--access public`.
- `Dockerfile.goreleaser` runs as a dedicated non-root user (`agentvault`, uid 65532), minimal alpine base, bundles only the prebuilt binary + entrypoint.
- Verification instructions are embedded in the release footer (checksum + cosign verify-blob).
- `make web` (goreleaser `before.hooks`) correctly uses `npm ci` — so the embedded frontend is deterministic in the main release path (only the npm-SDK publish workflow is affected by F5).

---

**F9 — Docker images are not cryptographically signed** (high, supply chain)

`.goreleaser.yml:45-53` — the `signs:` block uses `artifacts: checksum`, which signs only `checksums.txt`. Docker Hub images `infisical/agent-vault:{ver}-{arch}` and the multi-arch manifests (`:ver`, `:latest`) have **no cosign signature**. A consumer pulling `infisical/agent-vault:latest` has no way to verify it came from this repo's release pipeline. Given the project's trust model (credential broker with root-CA private-key material), this is the single biggest release-integrity gap.

**Recommendation:** add a `docker_signs:` block to `.goreleaser.yml` so cosign signs each image after push — example:
```yaml
docker_signs:
  - cmd: cosign
    artifacts: images
    args:
      - "sign"
      - "--yes"
      - "${artifact}@${digest}"
```
Document the `cosign verify` invocation in README alongside the existing blob-verify snippet.

**F10 — No SBOM for Docker images** (medium, supply chain)

`sboms: [artifacts: archive]` generates SBOMs only for the tarball archives. The Docker images have no attached SBOM, so downstream consumers can't enumerate what's inside them. Syft is already installed in the release workflow — it just isn't invoked for images.

**Recommendation:** add an image-SBOM block:
```yaml
sboms:
  - artifacts: archive
  - artifacts: package    # attaches SBOM to each docker image
    documents:
      - "{{ .ArtifactName }}.spdx.sbom.json"
```
Or attach via `cosign attest --predicate sbom.json` as an OCI referrer.

**F11 — No SLSA build provenance attestation for binaries or images** (medium, supply chain)

Neither `actions/attest-build-provenance` nor `slsa-framework/slsa-github-generator` is wired up. Cosign signs the checksum blob (good) but doesn't bind the artifact to a specific *builder* (workflow ref / commit SHA / runner identity) the way SLSA provenance does. This is the gap between "this file's hash was signed by someone with our OIDC identity" and "this file was produced by this specific workflow run from this specific commit."

**Recommendation:** add `actions/attest-build-provenance@<sha>` after GoReleaser runs, targeting `dist/agent-vault_*_*.tar.gz` and the Docker digests. Update the release footer with `gh attestation verify` instructions.

**F12 — Mutable `:latest` tag** (low, by design)

`.goreleaser.yml:110-113` republishes `infisical/agent-vault:latest` every release. Standard practice but worth documenting: anyone pinning `:latest` in production is implicitly trusting every future release. The README should nudge users toward `:{version}` or, better, `:{version}@sha256:...` digest pinning. Unrelated to the signing gap in F9 — fixing F9 lets consumers pin-and-verify.

**Recommendation:** add a "Pin by digest" section to the install/Docker docs; optionally enable tag-immutability on Docker Hub for `:{version}-*` tags.

**F13 — Cosign verification regex is too broad** (low, informational)

The release footer (`.goreleaser.yml:130-134`) tells users to verify with:
```
--certificate-identity-regexp "github.com/Infisical/agent-vault"
```
This matches *any* workflow in the org/repo whose identity URL contains that substring — including hypothetical future workflows that might not be release-related. Better to pin the exact release workflow identity.

**Recommendation:** switch to:
```
--certificate-identity "https://github.com/Infisical/agent-vault/.github/workflows/release.yml@refs/tags/{{ .Tag }}"
```
(goreleaser can template the tag at release time, producing a per-release verification command.)

**F14 — Base images not pinned by digest** (medium)

`Dockerfile.goreleaser:2` (`FROM alpine:3.21`), `Dockerfile:2` (`FROM node:22-alpine`), `Dockerfile:11` (`FROM golang:1.25-alpine`), and `Dockerfile:29` (`FROM alpine:3.21`) are all pinned to floating tags. A compromised or repushed upstream tag would silently land in the release image. Equivalent to pinning a GitHub Action by `@v6` instead of `@<sha>` — the fix we've already applied on the Actions side.

**Recommendation:** pin each with `@sha256:<digest>` (resolve via `docker buildx imagetools inspect alpine:3.21`). Dependabot's `docker` ecosystem will keep them updated if added to `.github/dependabot.yml` (cross-ref Section 3).

---

### Section 3 — Supply chain (dependencies)

**Positives (what's already right)**

- `go.mod` has **zero replace directives** — no local path overrides, no forked imports, no unexpected redirects.
- Direct Go dependency list is modest (11 entries) and drawn from reputable authors (`charmbracelet/*`, `fatih/color`, `spf13/cobra`, `golang.org/x/*`, `modernc.org/sqlite`).
- CI runs `go mod tidy && git diff --exit-code go.mod go.sum` (`.github/workflows/ci.yml:35`) — catches stealth additions and uncommitted tidies.
- All three Node projects ship a `package-lock.json` (lockfile version 3).
- CI and `make web` / `make sdk-ts` both use `npm ci` (not `install`) — only the npm-publish workflow violates this (F5).
- `actions/setup-go` uses `go-version-file: go.mod`, so CI builds with the exact Go version declared (currently 1.25.0).

**Go direct-deps inventory**

| Module | Purpose | Vendor |
|---|---|---|
| `charmbracelet/huh`, `lipgloss`, `bubbletea` (indirect) | Interactive TUI | Charm (reputable) |
| `fatih/color` | Terminal colour | Long-established |
| `jedib0t/go-pretty/v6` | Table rendering | Community-vetted |
| `muesli/reflow` | Text wrapping | Charm-adjacent |
| `spf13/cobra` | CLI framework | De facto standard |
| `golang.org/x/{crypto,oauth2,term}` | std-x | Go team |
| `gopkg.in/yaml.v3` | YAML parsing | go-yaml (v3 only) |
| `modernc.org/sqlite` | Pure-Go SQLite | Pure-Go build (avoids CGO attack surface) |

No obvious typosquats or abandoned projects.

---

**F15 — No vulnerability scanning in CI** (medium, supply-chain)

`grep -rE 'govulncheck|osv-scanner|npm audit|snyk|trivy|grype'` across `.github/`, `scripts/`, `Makefile` returns zero hits. Dependabot opens update PRs, but it doesn't fail builds on known CVEs in the *current* lockfile; a critical vulnerability in a pinned transitive dep can sit unnoticed until a human checks.

**Recommendation:** add three cheap gates to `ci.yml`:
1. **`govulncheck`** (Go) — `go install golang.org/x/vuln/cmd/govulncheck@latest && govulncheck ./...`. Scans against the Go vuln DB, only flags reachable vulns.
2. **`osv-scanner`** (all lockfiles) — one pass over `go.sum`, `web/package-lock.json`, `sdks/sdk-typescript/package-lock.json`. Runs as `google/osv-scanner-action`.
3. **`trivy` or `grype`** for the `Dockerfile` images once F14 digest-pinning lands.

Either fail the build on findings above a threshold, or run as advisory but surface in PR summaries via `github/codeql-action/upload-sarif`.

**F16 — Dependabot missing coverage for `sdks/sdk-typescript` and `docker`** (medium, supply-chain)

`.github/dependabot.yml` covers `gomod /`, `npm /web`, and `github-actions /`. It does **not** cover:

- **`npm /sdks/sdk-typescript`** — the published SDK package has no auto-updates. Combined with F5 (`npm install` at publish), this SDK's transitive deps can drift undetected.
- **`docker`** — `Dockerfile`, `Dockerfile.goreleaser`. Once F14 is fixed (digest-pinning), Dependabot's `docker` ecosystem is the only low-friction way to keep those digests current.

**Recommendation:** append to `.github/dependabot.yml`:
```yaml
  - package-ecosystem: npm
    directory: /sdks/sdk-typescript
    schedule:
      interval: weekly
    commit-message:
      prefix: "deps"
    cooldown:
      default-days: 7
      semver-major-days: 14

  - package-ecosystem: docker
    directories:
      - /
      - /   # Dockerfile.goreleaser also at root
    schedule:
      interval: weekly
    commit-message:
      prefix: "deps"
```
Note: Dependabot's `docker` ecosystem reads `Dockerfile` by default; to cover `Dockerfile.goreleaser` specifically, you may need a `file` or `target-branch` override, or rename it to a pattern Dependabot auto-discovers.

**F17 — `skills-lock.json` is orphaned (dead integrity mechanism)** (informational)

`skills-lock.json` declares a `mintlify` skill with `computedHash` (sha256), but **no Go code reads the file** — the sub-agent trace confirmed skills are in fact `go:embed`ed from `cmd/skill_cli.md` / `cmd/skill_http.md` (see `cmd/run.go:23-27`) and served from embedded bytes at `/v1/skills/{cli,http}`. The lockfile is vestigial — probably left over from a planned trusted-publishing / remote-skill-fetch feature.

Consequence: a reviewer who sees `skills-lock.json` may incorrectly assume it enforces integrity of embedded skills. It doesn't. Skill integrity is currently guaranteed only by the fact that `cmd/skill_*.md` is in the git tree and embedded at build time — so the real integrity gate is *git branch protection + release provenance* (cross-ref F11).

**Recommendation:** either (a) delete `skills-lock.json` and the "computedHash" idea until a loader lands, or (b) wire it up — add a build-time check that the computed hash of each embedded `cmd/skill_*.md` matches `skills-lock.json`, failing `make build` on drift. Option (a) is cheaper given there's no remote-fetch code path.

**F18 — npm install scripts allowed in lockfiles (esbuild, fsevents)** (informational)

Both `web/package-lock.json` and `sdks/sdk-typescript/package-lock.json` include `esbuild` and `fsevents`, both with `"hasInstallScript": true`. `esbuild`'s postinstall downloads a platform-specific prebuilt binary outside the lockfile integrity scope; `fsevents` is a macOS-only native module. These are legitimate and hard to remove (esbuild is a dep of vite/tsup). But it does mean the lockfile's integrity hashes don't fully cover what lands on disk after `npm ci`.

**Recommendation:** this is inherent to the toolchain; accept and document. If desired, pin esbuild to a specific version and trust the pattern. Do **not** add `--ignore-scripts` broadly — esbuild won't function without its postinstall binary download.

**F19 — `go.mod` lacks an explicit `toolchain` directive** (low)

`go.mod:3` declares `go 1.25.0` (a minimum), without a `toolchain go1.25.x` line that would pin the exact Go toolchain version. `actions/setup-go` with `go-version-file: go.mod` resolves the `go` directive as the version to install, so CI is deterministic today — but a future Go that honors the `toolchain` directive more strictly, or a developer running `go build` locally on a different Go version, could produce slightly different output. Minor reproducibility gap.

**Recommendation:** add `toolchain go1.25.5` (or whichever patch you standardise on) to `go.mod`, and let Dependabot keep it fresh.

---

## Newly added tasks

- [ ] Verify tag protection rules on `v*` and `node-sdk/v*.*.*` (repo setting — may need to ask user; cross-ref F2)
- [ ] Audit the floating `version: "~> v2"` on goreleaser-action + `version: v2.11` on golangci-lint-action — consider pinning the tool binary too (low prio)
- [ ] Consider adding zizmor to CI as a recurring check (uv tool install zizmor; run against `.github/`)
- [ ] Generalise F6: audit **every** package/dependency manager config in the repo for cooldown / delay settings, not just Dependabot. Candidates to check: Renovate (`renovate.json` / `.renovaterc*`), npm (`package.json` → `overrides`, `.npmrc`), Go (`go.mod` `toolchain` directive, any `tools.go`), Docker base-image auto-updaters, pre-commit hook update schedules, and any third-party bot configs under `.github/`. Flag any that can auto-merge or auto-bump without a waiting window.
- [ ] Verify Docker Hub repository settings: tag immutability on versioned tags, two-factor auth on the publishing account, scoped access token for `DOCKERHUB_TOKEN` (repo:write on `infisical/agent-vault` only) — cross-ref F12
- [ ] Confirm npmjs trusted-publisher config for `@infisical/agent-vault-sdk` is scoped to `.github/workflows/release-node-sdk.yml` on this repo only
- [ ] Investigate `Dockerfile:21` — `COPY --from=frontend /internal/server/webdist ...` looks like it copies from an absolute path in the frontend stage that doesn't exist (WORKDIR is `/app`). Likely a latent build-correctness bug, out of scope for security but worth flagging separately.
- [ ] Decide resolution for F17 `skills-lock.json` — either delete or wire up the integrity check in `make build`.
- [ ] Once F15 lands, decide fail-threshold policy for vuln scanners (hard-fail on high/critical vs advisory comments on PRs).
