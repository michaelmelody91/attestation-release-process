# attestation-release-process

A self-contained **proof-of-concept** (POC) that demonstrates an end-to-end
container attestation process driven by a **Release Please** PR lifecycle.

---

## How it works

```
 Developer pushes feat/fix commits to main
          │
          ▼
 release-please.yml ──► opens / updates Release Please PR
          │
          ▼ (Release Please PR is opened or updated)
 rc-attest-on-release-please-pr.yml
   │
   ├─ build_and_push_rc   – build RC image → push to GHCR as `vX.Y.Z-rc` → capture sha256 digest
   ├─ attest_build        – SLSA build-provenance attestation (signed, pushed to GHCR)
   ├─ attest_tests        – run tests → test-results attestation  (signed, pushed to GHCR)
   ├─ attest_pr_summary   – release PR CI summary attestation     (signed, pushed to GHCR)
   └─ verify_and_comment  – verify all 3 attestations → post PR comment with digest
          │
          ▼ (Release Please PR is merged → release is cut)
 release-please.yml  release_evidence job
   │
   ├─ find RC digest from merged PR comment
   ├─ promote `vX.Y.Z-rc` → `vX.Y.Z` release tag (re-tag, same digest – no rebuild)
   ├─ attest release promotion (custom predicate recording the re-tag)
   ├─ verify all 4 attestations (3 from RC carry over + 1 promotion)
   └─ upload compliance-evidence-*.md + attestation bundles to GitHub Release
```

---

## Repository structure

| Path | Purpose |
|------|---------|
| `Dockerfile` | Minimal Alpine image; the build artifact under attestation |
| `.release-please-config.json` | Release Please configuration (simple release type) |
| `.release-please-manifest.json` | Current version tracked by Release Please |
| `.github/workflows/release-please.yml` | Creates Release Please PRs; cuts releases; attaches evidence |
| `.github/workflows/rc-attest-on-release-please-pr.yml` | RC build + 3 attestations + verification + PR comment |

---

## Workflows

### `release-please.yml`

**Triggers:** `push` to `main`, `workflow_dispatch`

| Job | When | What |
|-----|------|------|
| `release_please` | every push to main | Runs `googleapis/release-please-action@v4` in manifest mode |
| `release_evidence` | only when a release is cut | Promotes RC image (`vX.Y.Z-rc` → `vX.Y.Z`, same digest – no rebuild), attests the promotion, verifies all 4 attestations, uploads evidence to GitHub Release |

### `rc-attest-on-release-please-pr.yml`

**Triggers:** `pull_request` targeting `main` (opened / synchronize / reopened)

> **Only runs on the Release Please PR**, scoped by two layers:
> 1. **Trigger-level `paths` filter** – the workflow only fires when `.release-please-manifest.json` is in the PR diff (Release Please always modifies this file)
> 2. **Job-level `if` guard** – defence-in-depth check on PR title (`chore(main): release`) or head branch (`release-please`)

| Job | Purpose |
|-----|---------|
| `build_and_push_rc` | Build RC image from PR head SHA; push to GHCR as `vX.Y.Z-rc`; output digest |
| `attest_build` | `actions/attest-build-provenance@v2` – SLSA provenance |
| `attest_tests` | `actions/attest@v2` – custom test-results predicate |
| `attest_pr_summary` | `actions/attest@v2` – release PR CI summary (mocked for POC) |
| `verify_and_comment` | Verify all attestations; post PR comment with digest + verify command |

---

## Release Please

### Triggering a release

1. Land commits on `main` using [Conventional Commits](https://www.conventionalcommits.org/):
   - `feat: …` → minor bump
   - `fix: …` → patch bump
   - `feat!: …` or `BREAKING CHANGE:` footer → major bump
2. Release Please opens a PR titled **`chore(main): release X.Y.Z`** with the updated
   `CHANGELOG.md` and `.release-please-manifest.json`.
3. Merge the PR → Release Please creates the GitHub Release and git tag.

### What Release Please produces

- A GitHub Release tagged `vX.Y.Z`
- An updated `CHANGELOG.md`
- An updated `.release-please-manifest.json` (version bump)
- *(Via `release_evidence` job)* Release assets:
  - `compliance-evidence-<digest12>.md` – markdown attestation verification report
  - `*.jsonl` – signed in-toto attestation bundle files

---

## Verifying attestations

After any workflow run, verify the image's attestations with the `gh` CLI:

```bash
# Replace <digest> with the sha256 value from the PR comment or Release page.
gh attestation verify \
  oci://ghcr.io/<owner>/attestation-release-process/poc@<digest> \
  --owner <owner>
```

The command verifies all stored attestations against Sigstore's public
transparency log and exits non-zero if any verification fails.

---

## Extending the POC

| Replacement point | File | Step to modify |
|-------------------|------|----------------|
| Real test suite | `rc-attest-on-release-please-pr.yml` | `attest_tests` → `Run tests` |
| Real included-PR lookup | `rc-attest-on-release-please-pr.yml` | `attest_pr_summary` → `Collect included PR CI data` |
| Different registry | Both workflows | `IMAGE_NAME` env var + login action |
| Custom predicate types | Both workflows | `predicate-type:` inputs on `actions/attest` steps |
| Promotion predicate type | `release-please.yml` | `release_evidence` → `Generate release-promotion predicate` |