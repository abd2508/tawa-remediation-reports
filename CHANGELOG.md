# Changelog

All notable changes to this **repository itself** (its structure, metadata, and
housekeeping) — not to the underlying Tawa platform. For platform-side changes, see the
numbered report series and `561-tawa-remediation-build-spec-v2.md`.

## 2026-07-26

### Repository settings
- Repository created (private) as `abd2508/tawa-remediation-reports`.
- Visibility changed from private to **public**.
- Description set: "Tawa platform (tawanow.com) engineering audit, remediation-plan,
  and execution reports (2026-05 to 2026-07), Arabic-language HTML archive".
- Topics added: `audit-reports`, `remediation-plan`, `delivery-platform`, `arabic`,
  `documentation`, `android`, `engineering-reports`.
- History rewritten (`git lfs migrate`) to move `artifacts/app-full-production-release.aab`
  and `artifacts/app-lite-production-release.aab` onto **Git LFS** — commit hashes for
  the affected commits changed as a result; force-pushed to `master`.

### Commits
| Commit | Summary |
|---|---|
| `507d3ab` | Initial commit — report 564 (Wave2 build/review/scenario/deploy) and the D11 ticket addition to the 561 build spec. |
| `c2bd7bb` | Bulk-add the remaining ~850 pre-existing report/screenshot/supporting files from this folder's history. |
| `d597a04` | Add mobile release artifacts (`.apk`/`.aab` builds + `SHA256SUMS.txt`); tree later rewritten in place to move the two `.aab` files onto Git LFS. |
| `8d849c5` | Add `README.md` describing the archive's contents and scope. |
| `4b819f0` | Add `LICENSE` (All Rights Reserved — proprietary content, no reuse license). |
| `62f2617` | README: add topic + license badges. |
| `3bb862e` | README: add direct download links (GitHub raw URLs) for all 4 `.apk` variants and both `.aab` bundles. |
| `563c664` | README: add a Table of Contents. |
| `cf4c0fc` | README: add a Contact section (GitHub account only — no external contribution model, per `LICENSE`). |
| `27c53a9` | Add this `CHANGELOG.md`, link it from the README's Table of Contents. |
| `f6f689f` | CHANGELOG: backfill the actual commit hash for the changelog-adding commit. |
| *(next)* | README: fix the 8 badges — they were plain images with no destination link; wrap each in a link to its real `github.com/topics/<name>` page (license badge links to `LICENSE`). |

### Verification performed
- Fresh-clone checksum verification: all 6 release binaries match `SHA256SUMS.txt`
  byte-for-byte, including both LFS-tracked `.aab` files.
- README badges (7 topic + 1 license): all 8 render as valid images on the live repo
  page (`200 image/svg+xml` from shields.io).
- README Table of Contents: all anchor links confirmed against the real rendered page's
  `user-content-*` heading IDs.
- Download table: all 6 links re-verified end-to-end after every subsequent README edit
  (actual download + checksum match, not just link presence).
