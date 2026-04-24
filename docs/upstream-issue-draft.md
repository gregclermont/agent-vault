# Draft: opening issue for Infisical/agent-vault

Paste the body below into a new issue on `Infisical/agent-vault`. Replace the `AUDIT_DOC_URL` placeholder with wherever the audit doc is publicly accessible (your fork's branch, a gist, etc.).

Format follows Infisical's contribution conventions: concise title, functional-then-technical overview, and explicit flagging of which items would be PRs vs settings changes. No video is needed for a discussion issue (their video requirement applies to feature PRs, not config/security suggestions).

---

## Title

```
Supply-chain hardening shortlist — would any of these PRs be welcome?
```

## Body

```markdown
Hi 👋 — prospective user here doing a pre-adoption security review of `agent-vault`. Credential-broker projects get extra scrutiny on my side, so I spent a few hours auditing the CI/CD setup, release pipeline, and install path, and cross-referenced findings against recent supply-chain incidents (Shai-Hulud npm worm Sep+Nov 2025, Axios npm compromise Mar 2026, Trivy/tj-actions GitHub Actions attacks Mar 2025 + Mar 2026, prt-scan campaign Mar 2026).

Full audit doc with finding IDs, severities, patches, and cross-refs: AUDIT_DOC_URL

Before I send anything as PRs, I wanted to check which (if any) you'd welcome. Each proposal below is self-contained and each would be a separate small PR — I'd rather land eight focused one-liners than one "hardening pack."

## Small PRs I'd happily send

Ordered by (security impact ÷ diff size ÷ argument surface). Any subset that sounds useful to you is fine.

1. **Dependabot cooldown** (`.github/dependabot.yml`, ~10 lines added across the 3 ecosystems) — directly addresses the Shai-Hulud-style auto-merge window. Catches freshly-published malicious versions before they land in auto-PRs.
2. **`npm install` → `npm ci` in `release-node-sdk.yml`** (1 word change) — Axios-exact defence; the vector Axios itself used was a widened dep resolution at publish time.
3. **Drop redundant `contents: write`** on the node-sdk publish workflow (1 line) — zizmor-flagged least-privilege fix; the publish job only needs `id-token: write` for OIDC.
4. **`persist-credentials: false`** on `actions/checkout` (+4 lines across 4 checkouts) — another zizmor finding (artipacked). No `git push` happens in any workflow, so the `GITHUB_TOKEN` doesn't need to be left in `.git/config`.
5. **`--proto '=https'`** on `install.sh` curl calls (~15 chars total) — defence-in-depth against HTTP redirect downgrade.
6. **Pin Dockerfile base images by digest** (4 line-replacements across `Dockerfile` and `Dockerfile.goreleaser`) — tj-actions-class defence for `alpine:3.21`, `node:22-alpine`, `golang:1.25-alpine`. Pairs nicely with enabling Dependabot's `docker` ecosystem.
7. **`actions/attest-build-provenance` after GoReleaser** (~10 lines in `release.yml`) — enables `gh attestation verify agent-vault_*.tar.gz --repo Infisical/agent-vault` for users with zero extra tool install. Materially improves the user-side verification UX and sets up the cleaner `install.sh` verify path.
8. **`docker_signs` block in `.goreleaser.yml`** (~8 lines) — cosign is already installed in the release workflow; it's currently only used to sign `checksums.txt`. Signing the Docker images themselves closes the container-install-path gap.

## Bigger but still small-ish

9. **Checksum + cosign verification in `install.sh`** — the current installer downloads and executes the binary without integrity verification, despite `.goreleaser.yml` already publishing a signed `checksums.txt`. Largest single impact of any change here; happy to send as its own PR after (7) lands so it can prefer `gh attestation verify` when available.
10. **Add `govulncheck` and `osv-scanner` jobs to `ci.yml`** — advisory-mode to start (SARIF upload to code-scanning), tighten to blocking later. Covers all three lockfiles (`go.sum`, `web/package-lock.json`, `sdks/sdk-typescript/package-lock.json`).

## Things that need maintainer action (flagging, not PR'ing)

These are settings / external-infra changes only you can make. Listing so they're on your radar; happy to defer on any of them:

- **Immutable Releases** (repo settings → Releases) — single checkbox that prevents the Trivy-class (Mar 2026) and tj-actions-class (Mar 2025) asset-swap attacks by making release tags and assets platform-level immutable after publication. Complements cosign by making the platform itself enforce tamper-evidence.
- **Tag protection ruleset** — the repo's branch protection on `main` is well-configured (5 rule types across 2 active rulesets — nice to see!), but there's no `tag`-targeted ruleset, so `v*` / `node-sdk/v*.*.*` pushes have no gate. A `target: tag` ruleset with `creation` / `deletion` / `non_fast_forward` rules would close this.
- **Remove maintainer bypass on the two active rulesets** — some recent `main` commits appear to have skipped the `pull_request` rule (committer is the maintainer rather than `web-flow`, no PR number in title). Shai-Hulud specifically exploits this pattern: compromise the maintainer identity, inherit the bypass, push-to-main.
- **DNSSEC on `agent-vault.dev`** — zone is currently unsigned (signed NSEC3 denial from the `dev.` TLD confirmed). One click in Cloudflare + DS record at the registrar. Important given the install-path trust reliance on the domain.
- **CAA records** on `agent-vault.dev` — any public CA can currently issue a cert for the zone; cert is from Google Trust Services via Cloudflare ACM. Recommend:
  ```
  CAA 0 issue     "pki.goog"
  CAA 0 issue     "letsencrypt.org"
  CAA 0 issuewild "pki.goog"
  CAA 0 issuewild "letsencrypt.org"
  CAA 0 iodef     "mailto:security@infisical.com"
  ```
  (Both CAs because Cloudflare ACM rotates between them.)
- **`GO_RELEASER_GITHUB_TOKEN`** appears to be a PAT (needs write to `Infisical/homebrew-get-cli`, which `GITHUB_TOKEN` can't reach). A GitHub App installation token scoped to that one repo with `contents:write` would remove the "PAT on a maintainer machine" risk class — the primary vector Shai-Hulud exploits.

## A few questions before I start

- **Bundling preference** — happy with 8 separate small PRs, or would you rather a couple of "security hardening batch" PRs grouped by concern (e.g. one for Dependabot-related, one for permissions)?
- **Video attachment** — your contribution guide mentions video for features. For the workflow/config changes above, attaching zizmor run output and CI logs makes more sense I think — is that acceptable, or do you still want video?
- **Order preference** — I'd suggest starting with 1-5 (truly tiny wins, easy review), then 6-8 (still small, need one look), then 9-10 (slightly bigger). Open to any order you prefer.
- **Overlap** — if any of this is already planned or WIP on your end, just say so and I'll drop it.

Thanks for the project and the SECURITY.md + clear reporting channel — happy to send whichever of the above you'd find useful, or just leave the audit doc as a reference if you'd rather handle these internally.
```

---

## Notes for you (not part of the issue body)

- **Replace `AUDIT_DOC_URL`** with a stable link to `docs/security-audit.md` on your branch. A permalink including the commit SHA is best (readers won't see it drift if you keep editing).
- **No video needed** — Infisical's video requirement per their contrib guide is for functional features. Workflow/config PRs have never needed one. The issue explicitly asks to confirm this so they can correct you if they disagree.
- **If you want to soften the "maintainer bypass" bullet** — it's phrased as an observation, not an accusation, and the Shai-Hulud framing gives the reason rather than a lecture. But if you want, you can drop that bullet entirely and just flag it in follow-up conversation once you've built trust.
- **The `Immutable Releases` bullet is the single biggest "checkbox with outsized effect"** in the whole list. If they only do one thing from the maintainer-action section, that's the one to nudge.
