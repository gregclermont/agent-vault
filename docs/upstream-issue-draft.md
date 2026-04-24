# Draft: opening issue for Infisical/agent-vault

Paste the body below into a new issue on `Infisical/agent-vault`.

---

## Title

```
Supply-chain hardening shortlist — would any of these PRs be welcome?
```

## Body

````markdown
Hi 👋, prospective user here doing a pre-adoption security review of `agent-vault`. I spent a few hours auditing the CI/CD setup, release pipeline, and install path, especially in light of recent supply-chain incidents.

Before I send anything as PRs, I wanted to check which (if any) you'd welcome. Each proposal below is self-contained and each would be a separate small PR.

## Small PRs I'd happily send

1. **Dependabot cooldown** — catches freshly-published malicious versions before they land in auto-PRs.
2. **`npm install` → `npm ci` in `release-node-sdk.yml`** — installs strictly from the lockfile, closing the window between what CI tested and what ships to npm. The exact vector the Axios compromise exploited.
3. **Downgrade `contents: write` → `contents: read`** on the node-sdk publish workflow — checkout needs `contents: read`; OIDC publish needs `id-token: write`; nothing in the job writes to the repo. (Can't *remove* the `contents:` line entirely — any declared `permissions:` block forces unspecified permissions to `none`, which would break checkout.)
4. **`persist-credentials: false`** on `actions/checkout` — found by `zizmor` (artipacked). No `git push` happens in any workflow, so the `GITHUB_TOKEN` doesn't need to be left in `.git/config`.
5. **`--proto '=https'`** on `install.sh` curl calls — defense-in-depth against HTTP redirect downgrade.
6. **Pin Dockerfile base images by digest** and enable Dependabot's `docker` ecosystem — floating tags (`alpine:3.21`, `node:22-alpine`, `golang:1.25-alpine`) are mutable upstream; a retag or compromise lands silently in the next build. Digest pinning (`@sha256:...`) makes base-image changes opt-in; the Dependabot docker ecosystem keeps the pins fresh so they don't rot.
7. **`actions/attest-build-provenance` after GoReleaser** — enables `gh attestation verify agent-vault_*.tar.gz --repo Infisical/agent-vault` for users with zero extra tool install.
8. **`docker_signs` block in `.goreleaser.yml`** — cosign is already installed in the release workflow; it's currently only used to sign `checksums.txt`.

## Bigger but still small-ish

9. **Checksum + cosign verification in `install.sh`** — the current installer downloads and executes the binary without integrity verification, despite `.goreleaser.yml` already publishing a signed `checksums.txt`. Largest single impact of any change here; happy to send as its own PR after (7) lands so it can prefer `gh attestation verify` when available.
10. **Add `govulncheck` and `osv-scanner` jobs to `ci.yml`** — advisory-mode to start, tighten to blocking later. Covers all three lockfiles (`go.sum`, `web/package-lock.json`, `sdks/sdk-typescript/package-lock.json`).

## Things that need maintainer action (flagging, not PR'ing)

These are settings / external-infra changes only you can make. Listing so they're on your radar; happy to defer on any of them:

- **Immutable Releases** (repo settings → Releases) — single checkbox that prevents the Trivy-class (Mar 2026) and tj-actions-class (Mar 2025) asset-swap attacks by making release tags and assets platform-level immutable after publication. Complements cosign by making the platform itself enforce tamper-evidence.
- **Tag protection ruleset** on `v*` / `node-sdk/v*.*.*` pushes with `creation` / `deletion` / `non_fast_forward` rules.
- **DNSSEC on `agent-vault.dev`** — zone is currently unsigned (signed NSEC3 denial from the `dev.` TLD confirmed). One click in Cloudflare + DS record at the registrar. Important given the install-path trust reliance on the domain.
- **`GO_RELEASER_GITHUB_TOKEN`** — read directly as a static secret (`release.yml:60`); no `actions/create-github-app-token` step anywhere in the workflow, which rules out a GitHub App installation token (App tokens expire in ~60 min and can't be pre-stored statically). That leaves classic PAT or fine-grained PAT. If classic, migrating to a fine-grained PAT scoped to `Infisical/homebrew-get-cli` `contents:write` (or better, a GitHub App) would be a major blast-radius reduction. If it's already a scoped fine-grained PAT, current posture is fine — useful to confirm either way.

## A few questions before I start

- **Bundling preference** — happy with 10 separate small PRs, or would you rather a couple of "security hardening batch" PRs grouped by concern (e.g. one for Dependabot-related, one for permissions)?
- **Order preference** — I'd suggest starting with 1-5 (truly tiny wins, easy review), then 6-8 (still small, need one look), then 9-10 (slightly bigger). Open to any order you prefer.
- **Overlap** — if any of this is already planned or WIP on your end, just say so and I'll drop it.

Thanks for the project!
````

---

## Notes for you (not part of the issue body)

- **The `Immutable Releases` bullet is the single biggest "checkbox with outsized effect"** in the whole list. If they only do one thing from the maintainer-action section, that's the one to nudge.
- **No video needed** — Infisical's video requirement per their contrib guide is for functional features. Workflow/config PRs have never needed one; the trimmed draft drops the explicit ask on this, fine.
- **You trimmed the CAA bullet and the maintainer-bypass bullet** from the earlier version. Both are in the audit doc if you want to bring them up later once trust is built, but they're the ones most likely to prompt a defensive response in a first-contact issue — reasonable to defer.
- **You also dropped the audit-doc link**. Keeps the issue focused on the PRs being offered. If a maintainer asks "what else is in there?" you can drop the permalink in a follow-up comment.
