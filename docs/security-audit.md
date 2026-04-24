# GitHub Actions & Supply Chain Security Audit

Tracking doc for the audit of `agent-vault`'s CI/CD and supply chain.
Branch: `claude/audit-github-actions-security-FN0nY`

Legend: `[ ]` todo · `[~]` in progress · `[x]` done · `[!]` finding · `[-]` N/A

---

## Findings summary

**35 findings.** Fix F20 first, F27 second — they unlock or undermine most of the rest.

| ID | Sev | Area | Finding |
|---|---|---|---|
| F1 | low | workflow | `contents: write` on `release-node-sdk.yml` unnecessary |
| F2 | medium | workflow | No `environment:` gating on release workflows |
| F3 | low | workflow | No `concurrency:` on release workflows |
| F4 | info | workflow | Unquoted `${GITHUB_REF_NAME#...}` shell expansion |
| F5 | medium | supply-chain | `npm install` instead of `npm ci` in publish job |
| F6 | medium | supply-chain | Missing Dependabot `cooldown:` on all ecosystems |
| F7 | low | workflow | Cache poisoning risk from shared language caches |
| F8 | low | workflow | `actions/checkout` default `persist-credentials: true` |
| F9 | **high** | release | Docker images are not cosign-signed |
| F10 | medium | release | No SBOM attached to Docker images |
| F11 | medium | release | No SLSA build provenance attestation |
| F12 | low | release | `:latest` tag is mutable (by design; document) |
| F13 | low | release | Cosign verify regex is too broad |
| F14 | medium | container | Base images not pinned by digest |
| F15 | medium | supply-chain | No vuln scanning in CI (`govulncheck`/`osv`/`trivy`) |
| F16 | medium | supply-chain | Dependabot missing `sdks/sdk-typescript` and `docker` |
| F17 | info | supply-chain | `skills-lock.json` is orphaned (dead integrity mechanism) |
| F18 | info | supply-chain | npm install scripts (esbuild/fsevents) outside lockfile |
| F19 | low | supply-chain | `go.mod` lacks explicit `toolchain` directive |
| F20 | **high** | install | `install.sh` does not verify checksums or signatures |
| F21 | medium | install | JSON parsed from GitHub API via `grep | sed` |
| F22 | low | install | Anonymous GitHub API rate limit (reliability) |
| F23 | low | install | Telemetry beacon fires before binary verification |
| F24 | info | install | `get.agent-vault.dev` root-of-trust (out-of-repo) |
| F25 | info | install | Installer doesn't fetch the SBOM |
| F26 | low | install | `curl` missing `--proto '=https'` |
| F27 | **high** | repo | `main` branch is not protected (verify upstream) |
| F28 | medium | repo | No CODEOWNERS file |
| F29 | low | repo | No signed-commit / DCO enforcement |
| F30 | medium | repo | `GO_RELEASER_GITHUB_TOKEN` likely a PAT with cross-repo scope |
| F31 | low | repo | No secret-scanning tooling in-repo |
| F32 | low | project | Sensitive-internal tests run in fork-PR CI (mitigated) |
| F33 | medium | project | Skill doc tamper path (agents trust embedded markdown) |
| F34 | medium | project | Frontend supply chain reaches Go binary via `go:embed` |
| F35 | info | project | Other `go:embed`ed content (emails, migrations, sandbox assets) |

**Three high-severity findings — fix order:**
1. **F20** (install.sh skips verification) — highest blast radius: one compromise → every install backdoored.
2. **F27** (main is unprotected) — undermines PR-review assumption that F33/F34/F28 rely on.
3. **F9** (images not signed) — parity with F20 on the container install path.

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

- [x] `Dockerfile` + `Dockerfile.goreleaser`: base images pinned by digest, non-root user, minimal final image — `[!]` F14 (digest pinning)
- [x] `install.sh`: HTTPS-only, checksum/signature verification, hardcoded release source — `[!]` see F20, F21, F24, F26
- [x] Release assets: `checksums.txt`, `.sig`, SBOM published and verified by `install.sh` — `[!]` see F20, F25

## 5. Repo & org-level controls (best-effort from repo contents)

- [x] Branch protection on `main` (inferable from workflow `if:` guards / required checks) — `[!]` see F27
- [x] Tag protection / signed commits signals in repo — `[!]` see F29, cross-ref F2
- [x] `CODEOWNERS` file covering workflows, release config, crypto/auth/oauth/session — `[!]` see F28
- [x] Secret scanning / push protection hints (e.g., `.gitleaks`, pre-commit) — `[!]` see F31
- [x] Any fine-grained PATs or deploy keys referenced in workflows — `[!]` see F30

## 6. Project-specific sensitive paths

- [x] Workflows touching `internal/crypto`, `internal/ca`, `internal/auth`, `internal/oauth`, `internal/session` with fork-PR code + secrets — `[!]` see F32
- [x] Embedded skill docs (`cmd/skill_cli.md`, `cmd/skill_http.md`) tamper path — `[!]` see F33
- [x] `web/` build → `go:embed` → Go binary: frontend supply-chain reaches the binary — `[!]` see F34, F35

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

### Section 4 — Container & install supply chain

**Positives (what's already right)**

- `Dockerfile.goreleaser` is minimal: `alpine:3.21` + `ca-certificates` + the prebuilt binary + entrypoint. Runs as `agentvault` uid 65532 (non-root) with `USER agentvault`, `VOLUME /data` for persistence, and a `HEALTHCHECK` against the in-container `/health` endpoint.
- `scripts/docker-entrypoint.sh` is a 1-line `exec` passthrough — no shell injection surface, no env-handling footguns.
- `install.sh` uses `set -e`, HTTPS-only URLs (`https://api.github.com`, `https://github.com/.../releases/download`, `https://get.agent-vault.dev`), `curl -fsSL` (fail on HTTP error, silent, show errors, follow redirects), a cleanup `trap ... EXIT` to remove tmpdirs, and `maybe_sudo` that only escalates when `INSTALL_DIR` isn't already writable.
- Installer backs up the SQLite DB (including `-wal` / `-shm`) before replacing the binary on upgrade — good operator hygiene.
- Telemetry is off-by-default-friendly: documented at the top, opt-out via `AGENT_VAULT_NO_TELEMETRY=1`, payload is only OS/arch/version/event, fired with `-m 3` (3-second cap) and `|| true` so it can't block the install.
- Only the proxy port (`14321`) is `EXPOSE`d; the MITM port (`14322`) is not declared — minor attack-surface reduction at the Docker level.

---

**F20 — `install.sh` does not verify checksums or signatures** (**high**, supply chain — single biggest install-path gap)

The release pipeline produces `checksums.txt` (sha256 per archive) and `checksums.txt.bundle` (cosign keyless signature over the checksum file via OIDC) — see `.goreleaser.yml:38-53`. Both are published as GitHub Release assets. **`install.sh` downloads neither** and installs the binary directly after extracting the tarball (`install.sh:141-156`).

Consequence: any adversary with one of the following footholds can ship a backdoored `agent-vault` to users:
1. Compromise of `get.agent-vault.dev` (the script host). The script runs before any verification happens — it *is* the verifier.
2. Compromise of the GitHub Release assets (Infisical org-member account takeover, or an unreviewed tag push per F2).
3. Any future TLS-break / proxy / CDN misconfiguration.

Given the project's purpose — a credential broker that will hold OAuth tokens, Stripe keys, encrypted CA material, etc. — this is the single most consequential finding in the audit. A compromised `agent-vault` binary is more valuable than the secrets it fronts.

**Recommendation:** add mandatory checksum verification and optional (but documented) cosign verification:

```sh
# Download checksums + signature bundle alongside the archive
curl -fSL -o "${TMP_DIR}/checksums.txt" \
    "https://github.com/${REPO}/releases/download/v${LATEST}/checksums.txt"
curl -fSL -o "${TMP_DIR}/checksums.txt.bundle" \
    "https://github.com/${REPO}/releases/download/v${LATEST}/checksums.txt.bundle"

# Verify sha256 — mandatory
cd "$TMP_DIR"
if command -v sha256sum >/dev/null; then
    grep " ${ARCHIVE}$" checksums.txt | sha256sum -c - || error "Checksum verification failed"
elif command -v shasum >/dev/null; then
    grep " ${ARCHIVE}$" checksums.txt | shasum -a 256 -c - || error "Checksum verification failed"
else
    error "Neither sha256sum nor shasum available; cannot verify download integrity"
fi

# Verify cosign signature if cosign is installed — strongly encouraged
if command -v cosign >/dev/null; then
    cosign verify-blob \
        --bundle checksums.txt.bundle \
        --certificate-identity "https://github.com/${REPO}/.github/workflows/release.yml@refs/tags/v${LATEST}" \
        --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
        checksums.txt || error "Cosign signature verification failed"
fi
```

Cross-refs F9 (image signing — same class of gap on the Docker side) and F13 (`--certificate-identity` regex should be tightened when this lands).

**F21 — JSON parsing of GitHub API response via `grep | sed`** (medium, robustness)

`install.sh:124-125`:
```sh
LATEST="$(curl -fsSL "https://api.github.com/repos/${REPO}/releases/latest" \
    | grep '"tag_name"' | head -1 | sed 's/.*"tag_name":[[:space:]]*"v\{0,1\}\([^"]*\)".*/\1/')"
```
Regex-parsing JSON is fragile — a future API response adding `tag_name` somewhere else (e.g., in `author.login` for an author named "tag_name"), or changing whitespace/escaping, silently produces an incorrect version string. More concerning for security: if the response is ever partially served (truncated, JSON injection via upstream misconfig), the extracted string could be influenced by an attacker and then embedded into a URL (`install.sh:136`) without validation.

**Recommendation:** validate that `LATEST` matches a strict semver regex before using it in URL construction:
```sh
case "$LATEST" in
    [0-9]*.[0-9]*.[0-9]*) : ;;
    *) error "Unexpected version format: ${LATEST}" ;;
esac
```
Or use `jq` if present with a `python3 -c "import json; print(json.load(...)['tag_name'])"` fallback.

**F22 — Anonymous GitHub API rate limit** (low, reliability not security)

The unauthenticated `api.github.com` call uses the 60-req/hour-per-IP limit. Users behind shared egress (NAT, corporate proxies, some CI runners) can hit 429. Consider falling back to parsing `https://github.com/${REPO}/releases/latest` (redirects to the tag page) via `Location:` header, which doesn't consume the API quota.

**F23 — Telemetry beacon fires even if the binary is broken** (low)

`install.sh:160-167,188-194` — the `agent-vault version` check runs *after* `maybe_sudo mv`, but the telemetry beacon fires regardless of whether verification (once F20 is implemented) would have failed. Order the flow as: download → verify → install → verify-runs → beacon. That way, a compromised tarball that passes checksum but fails to execute doesn't get reported as a successful install.

**F24 — `get.agent-vault.dev` is the install-path root of trust** (informational, out-of-repo)

The README and `install.sh:5` recommend `curl -fsSL https://get.agent-vault.dev | sh`. That domain is now a first-class piece of release infrastructure: whoever controls it can replace the script with anything, bypassing every other control in this audit. Can't verify from the repo, but worth confirming:

- DNSSEC enabled on `agent-vault.dev`, CAA record pinning the CA.
- HSTS + `Strict-Transport-Security` with `preload` on `get.agent-vault.dev`.
- Script is served from an immutable source — ideally a GitHub Pages / Cloudflare Workers deployment whose config is in this repo and reviewed via PR, not a mutable S3 bucket.
- Consider serving `install.sh` over `ghcr.io` / raw.githubusercontent at a pinned commit, so the `curl | sh` surface is provably the in-repo file at a specific ref.
- If the script ever changes at tag time, that change should itself be signed (or the tag protection rules from F2 cover it).

**F25 — Installer doesn't fetch the SBOM** (informational)

Goreleaser attaches a syft SBOM per archive (`.goreleaser.yml:42-43`), but `install.sh` doesn't download it or leave it on disk. Users who want to run their own SBOM audits (Vex, OSV lookup, etc.) have to manually grab it from the release page. Low-effort improvement: download `${ARCHIVE}.sbom.json` alongside and drop it next to the installed binary.

**F26 — `curl` invocations don't enforce `--proto '=https'`** (low)

All `curl` calls in `install.sh` use `-fsSL` without `--proto '=https' --proto-redir '=https'`. If GitHub's CDN ever served a redirect to a non-HTTPS mirror (historically has happened with CDN misconfigurations), `-L` would follow. Adding `--proto '=https' --proto-redir '=https'` is a single-line defence.

**Recommendation:** update every `curl` in the script:
```sh
curl --proto '=https' --proto-redir '=https' -fsSL ...
```

**Dev-image (`Dockerfile`) notes:**

- Cross-ref F14 for base-image digest pinning (`node:22-alpine`, `golang:1.25-alpine`, `alpine:3.21` ×2 — all floating).
- The local `make docker` build passes `--build-arg BUILD_DATE=$(DATE)` where `DATE := $(shell date -u ...)`, so the dev image is *not* reproducible the way the goreleaser build is (which uses `mod_timestamp: {{.CommitTimestamp}}`). This is fine for local dev but worth a comment in the Makefile so someone doesn't mistake `make docker` for a release build.
- `Dockerfile:21` latent bug (`COPY --from=frontend /internal/server/webdist ...` from a non-existent absolute path) already tracked under *Newly added tasks*.

---

### Section 5 — Repo & org-level controls

**Scope note:** GitHub MCP access in this session is limited to `gregclermont/agent-vault` — a mirror/fork of `Infisical/agent-vault`. Branch-protection and repo-setting findings below refer to the mirror; the upstream may have different settings. The upstream repo is the one users actually install from, so these findings are phrased as "verify on upstream."

**Positives (what's already right)**

- `SECURITY.md` is present with a clear vuln-reporting channel (`security@infisical.com`) and the correct "don't open public issues" guidance.
- `.github/pull_request_template.md` includes a **Security checklist** (no secrets, no unauth endpoints, input validation, OWASP top-10). Not enforced, but a good nudge during PR authoring.
- `.gitignore` explicitly blocks common secret-file patterns: `.env`, `.env.*`, `*.key`, `*.pem`, `*.p12`, `*.pfx`, `credentials.json`, `secrets.yaml`. Defence-in-depth against accidental commits.
- Only four distinct secrets are referenced across all workflows: `GITHUB_TOKEN` (built-in), `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`, `GO_RELEASER_GITHUB_TOKEN`. No deploy keys, no SSH key references, no exotic cloud creds.

---

**F27 — `main` branch is not protected** (**high**)

`mcp__github__list_branches` on `gregclermont/agent-vault` returns `"protected": false` for both `main` and the audit branch. Taken at face value, this means:
- Direct pushes to `main` are allowed (no required PR review).
- No required CI status checks before merge.
- Force-push to `main` is permitted.
- Anyone with push access can ship arbitrary code that will then be picked up by the release pipeline on the next tag.

**This is the upstream mitigation for half the findings in this audit** — F2 (release environment gating), F20 (installer verification), F33/F34 (tamper paths into embedded content) all rely on "merges to main are reviewed." Without branch protection, those controls degrade to "anyone on the commit bit can bypass everything."

**Recommendation (to confirm on upstream `Infisical/agent-vault`):**
- Require at least 1 review (ideally 2 for release-touching paths — see F28).
- Require status checks: `test`, `lint`, the planned zizmor + vuln-scan jobs from F15.
- Disallow force-push and deletion.
- Enforce linear history (no merge commits that bypass required checks via rebase races).
- Enable **tag protection rules** for `v*` and `node-sdk/v*.*.*` so only admins can create release tags (cross-ref F2).

**F28 — No CODEOWNERS** (medium)

`find` for `CODEOWNERS` returns nothing. Without it, no path-specific mandatory reviewer exists — so a workflow tweak, a `.goreleaser.yml` edit, a `cmd/skill_*.md` rewrite, or a change under `internal/{crypto,ca,auth,oauth,session}` can be approved by any contributor with review rights. Given the project's trust model (credential broker, root-CA key material), some paths should require named security-competent reviewers.

**Recommendation:** add a `CODEOWNERS` at repo root covering at minimum:
```
# Release + CI
/.github/                          @security-team @release-maintainers
/.goreleaser.yml                   @security-team @release-maintainers
/Dockerfile                        @security-team
/Dockerfile.goreleaser             @security-team
/install.sh                        @security-team
/scripts/docker-entrypoint.sh      @security-team

# Crypto & auth cores
/internal/crypto/                  @security-team
/internal/ca/                      @security-team
/internal/auth/                    @security-team
/internal/oauth/                   @security-team
/internal/session/                 @security-team

# Agent-facing contract (embedded)
/cmd/skill_cli.md                  @security-team @agent-contract-owners
/cmd/skill_http.md                 @security-team @agent-contract-owners
/cmd/run.go                        @security-team
/internal/server/persistent_instructions_admin.txt @security-team
```
Replace team handles with whatever the Infisical org uses.

**F29 — No signed-commit / DCO enforcement** (low, informational)

No DCO bot workflow, no `.github/workflows/dco.yml`, no `CONTRIBUTING.md` mention of signing. Not a strict requirement, but when combined with F27 (no branch protection) and F28 (no CODEOWNERS), it weakens the evidence chain on *who produced a given commit*. For a security-sensitive project, consider requiring signed commits (`git commit -S`) on `main` via branch protection.

**F30 — `GO_RELEASER_GITHUB_TOKEN` is likely a PAT with cross-repo scope** (medium, out-of-repo)

`release.yml:60` passes `GO_RELEASER_GITHUB_TOKEN` as `HOMEBREW_TAP_TOKEN` to goreleaser, per the (currently commented-out) `brews:` block in `.goreleaser.yml`. This is a PAT that needs write access to `Infisical/homebrew-get-cli` — i.e., a *different* repo than the workflow runs in. The default `GITHUB_TOKEN` can't reach that repo, hence the PAT.

Risks:
- Classic PATs are scoped per-user; if the owning account is compromised, so is the Homebrew tap.
- Classic PATs typically have wider scope than the one repo they're used for (reading all private repos, for example).
- Fine-grained PATs (introduced in 2022) can be scoped to a single repo but still live on a user account.

**Recommendation:** migrate to a **GitHub App** installation with `contents: write` on `Infisical/homebrew-get-cli` only. The App installation token is per-workflow-run, short-lived, and not tied to a human account. Alternatively, verify the current token is a fine-grained PAT scoped to that single repo and held on a machine account with SSO + 2FA.

**F31 — No secret-scanning tooling in-repo** (low)

No `gitleaks`, `trufflehog`, or `detect-secrets` config; no pre-commit hook wiring; no secret-scan workflow. GitHub's native **Secret Scanning + Push Protection** can cover this from the settings side — assume off until verified on upstream. Adding `gitleaks-action` (or enabling the native product) is a cheap pre-commit safety net that pairs nicely with `.gitignore`'s file-pattern blocks.

**Recommendation:** verify GitHub Secret Scanning is enabled on upstream; add `zricethezav/gitleaks-action` to CI as a pre-merge check.

---

### Section 6 — Project-specific sensitive paths

**F32 — Sensitive-internal tests run in fork-PR CI** (low, mitigated)

`internal/{crypto,ca,auth,oauth,session}/*_test.go` all run under the `test:` job in `ci.yml` on `pull_request` events, including fork PRs. An attacker submitting a fork PR with a malicious test could execute arbitrary Go code on the runner.

**Mitigation in current state:** `ci.yml` has `permissions: contents: read`, no secrets attached, no artifact uploads that feed privileged workflows (verified in Section 1). A malicious fork PR runs code on a throwaway runner with no credentials worth stealing. Risk is low.

**Residual:** a malicious test could still abuse the runner's network egress (crypto-mining, attacks on external services). StepSecurity Harden-Runner (already tracked as follow-up) closes this — *that follow-up item now covers this finding explicitly*.

**F33 — Skill doc tamper path (agent trust surface)** (medium)

`cmd/run.go:23-26`:
```go
//go:embed skill_cli.md
...
//go:embed skill_http.md
```
These are embedded into the binary and served publicly at `/v1/skills/{cli,http}` (per CLAUDE.md). Agents (Claude Code, Cursor) fetch them as the authoritative how-to-use-Agent-Vault contract — and then *act on their contents*. A malicious revision of either file could:
- Instruct agents to send credentials to an attacker-controlled endpoint.
- Describe a "test command" that exfiltrates the vault.
- Subtly change the agent's mental model so it chooses insecure defaults.

Because the content is trusted *by agents, not users*, the XSS-style review heuristics don't apply — a maintainer reviewing a skill-docs PR is checking for correctness, not necessarily for prompt-injection payloads. Classic lunar-lander problem.

**Mitigation today:** None code-side. Relies entirely on PR review. With F27 + F28 gaps, a single compromised contributor can ship this.

**Recommendation:**
- **CODEOWNERS** entries on `cmd/skill_*.md` requiring security review (covered by F28 proposal above).
- **Wire up `skills-lock.json` (F17 option b):** store the sha256 of each embedded skill doc in the lockfile; fail `make build` if the embed drifts from the lockfile without an explicit lockfile update in the same PR. Makes tampering a two-file diff (easier to spot in review) and gives reviewers a clear "integrity changed" signal.
- Consider serving skill docs with an in-binary signature the agent can verify (e.g., the build attests the skill content; the agent checks a public key). Heavy for the current scale — probably future work.

**F34 — Frontend supply chain reaches the Go binary via `go:embed all:webdist`** (medium, largest indirect surface)

`internal/server/server.go:30`:
```go
//go:embed all:webdist
```
The `web/` build output is embedded into the binary and served as the admin UI. A compromise at any of these points lands JS in every installed Agent Vault:
- Any dep in `web/package-lock.json` (or their transitives).
- A PR that modifies `web/src/` or `web/package.json`.
- A compromised `vite`/`@tanstack/react-router`/`react` or a typo-squatted dependency.
- The `esbuild` / `fsevents` install-script binaries (F18) executed during the frontend build.

The admin UI handles session cookies, proposal review (which approves credential changes), and direct credential CRUD. Injected JS in this UI can exfiltrate vault contents or approve malicious proposals.

**Mitigation today:**
- `web/` has Dependabot coverage (F16 would add `sdks/sdk-typescript` too).
- CI builds the frontend (`npm ci && npm run build`) — catches build-time errors but not malicious behaviour.

**Recommendation (layered):**
- **CODEOWNERS** on `web/` (covered by F28).
- **osv-scanner** on `web/package-lock.json` in CI (F15).
- **Content-Security-Policy** on the admin UI response headers — strict `script-src 'self'`, no `unsafe-inline`, no third-party domains. Limits what injected JS can do even if it gets in. (This is server-side code review, slightly outside the workflow-audit scope, but flagging here.)
- **Socket Firewall Free** on the `npm ci` step for `web/` (already tracked as a follow-up) blocks install-time egress from malicious transitive deps.
- Consider pinning esbuild to a specific version with `--save-exact` so post-install binary-fetching is at least deterministic.

**F35 — Other `go:embed`ed content (lower stakes but real)** (informational)

Inventory from `grep //go:embed cmd/ internal/`:
- `internal/server/persistent_instructions_admin.txt` — agent-facing admin instructions. **Same threat model as F33** (agents act on it). Should be in CODEOWNERS.
- `internal/server/*_email.html` (invite, proposal-notification, verification-code, password-reset, test) — HTML email templates. Tamper path → phishing-shaped emails from compromised binaries. Stored-XSS-adjacent if a client renders HTML.
- `internal/store/migrations/*.sql` — SQL migrations. A malicious migration runs with DB privileges on every upgrade. Integrity gated only by code review.
- `internal/sandbox/assets/{Dockerfile,init-firewall.sh,entrypoint.sh}` — used by `vault run --sandbox container`. Compromised content runs inside user sandboxes; `init-firewall.sh` sets iptables egress rules — a subtly-broken version could leak traffic past the broker.

**Recommendation:** extend F28's CODEOWNERS proposal to cover each of these (email templates, SQL migrations, sandbox assets). All are embedded-content tamper paths to a trusted binary; all deserve the same PR-review discipline as the crypto internals.

---

## Newly added tasks

- [ ] Verify tag protection rules on `v*` and `node-sdk/v*.*.*` (repo setting — may need to ask user; cross-ref F2)
- [ ] Audit the floating `version: "~> v2"` on goreleaser-action + `version: v2.11` on golangci-lint-action — consider pinning the tool binary too (low prio)
- [ ] Consider adding zizmor to CI as a recurring check (uv tool install zizmor; run against `.github/`)
- [x] Generalise F6: audit **every** package/dependency manager config in the repo for cooldown / delay settings, not just Dependabot. **Resolved:** executed `find` for Renovate (`renovate.json` / `.renovaterc*`), `.npmrc`, `tools.go`, `.pre-commit-config.yaml`, Mergify, auto-merge — none present. Dependabot is the only dep-manager config in the repo, so F6's cooldown fix is the complete remediation. Re-run this check if Renovate is ever introduced.
- [ ] Verify Docker Hub repository settings: tag immutability on versioned tags, two-factor auth on the publishing account, scoped access token for `DOCKERHUB_TOKEN` (repo:write on `infisical/agent-vault` only) — cross-ref F12
- [ ] Confirm npmjs trusted-publisher config for `@infisical/agent-vault-sdk` is scoped to `.github/workflows/release-node-sdk.yml` on this repo only
- [ ] Investigate `Dockerfile:21` — `COPY --from=frontend /internal/server/webdist ...` looks like it copies from an absolute path in the frontend stage that doesn't exist (WORKDIR is `/app`). Likely a latent build-correctness bug, out of scope for security but worth flagging separately.
- [ ] Decide resolution for F17 `skills-lock.json` — either delete or wire up the integrity check in `make build`.
- [ ] Once F15 lands, decide fail-threshold policy for vuln scanners (hard-fail on high/critical vs advisory comments on PRs).
- [ ] Verify `agent-vault.dev` / `get.agent-vault.dev` infrastructure (F24): DNSSEC, CAA records, HSTS preload, source-of-truth for the served `install.sh` (ideally an in-repo file at a pinned commit, not a mutable bucket).
- [ ] Once F20 lands, update README's install instructions to reflect the verification step + mention the optional cosign path.
- [ ] Consider offering a verification-only mode (`install.sh --verify-only`) and a per-platform install via Homebrew / a signed `.pkg` for macOS / `apt` repo for Debian — as alternatives to `curl | sh` for security-conscious users.
- [ ] Verify **upstream** `Infisical/agent-vault` branch + tag protection settings (F27 is phrased against the `gregclermont/agent-vault` mirror that this session has MCP access to). Re-check on upstream before treating F27 as actionable.
- [ ] Add Content-Security-Policy headers to the admin UI response (F34 defence-in-depth). Out of scope for workflow-audit but logged here so it isn't lost.
- [ ] Review `internal/store/migrations/*.sql` review discipline: migrations run with DB privs on every upgrade (F35). Consider signed migrations or a migration-review CODEOWNERS entry.
- [ ] Consider adding runner-level egress / install-time controls to workflows:
  - **StepSecurity Harden-Runner (Community tier)** — `step-security/harden-runner@<sha>` as the first step of every job. Monitors/restricts outbound network from the runner, detects compromised actions exfiltrating data, and records a runtime SBOM of all egress. Free for public repos. High signal for the supply-chain threat model here (credential broker with cosign keys + Docker Hub token on the runner).
  - **Socket Firewall Free (`sfw`)** — wrap `npm ci` / `npm install` steps (particularly in `release-node-sdk.yml` and the `web/` / `sdks/sdk-typescript/` installs in `ci.yml`) so malicious install-script behaviour from compromised transitive deps is blocked before reaching the network. Complements F18 (esbuild/fsevents postinstall binary fetches).
  Either one alone is useful; together they cover both runner egress and package-install egress. Evaluate cost (added runtime, potential flakes) before rolling out — start with release workflows where the blast radius is highest.
