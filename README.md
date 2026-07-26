# Tawa Platform — Remediation & Audit Reports

![topic: audit-reports](https://img.shields.io/badge/topic-audit--reports-blue)
![topic: remediation-plan](https://img.shields.io/badge/topic-remediation--plan-blue)
![topic: delivery-platform](https://img.shields.io/badge/topic-delivery--platform-blue)
![topic: arabic](https://img.shields.io/badge/topic-arabic-blue)
![topic: documentation](https://img.shields.io/badge/topic-documentation-blue)
![topic: android](https://img.shields.io/badge/topic-android-blue)
![topic: engineering-reports](https://img.shields.io/badge/topic-engineering--reports-blue)
![license: All Rights Reserved](https://img.shields.io/badge/license-All%20Rights%20Reserved-lightgrey)

This repository is an archive of technical audit, remediation-planning, and execution
reports produced during ongoing engineering work on the **Tawa** (توا) delivery
platform (`tawanow.com`), spanning roughly **2026-05 through 2026-07**.

Reports are written in **Arabic**, as self-contained HTML files (light/dark theme,
right-to-left layout) — each one documents a single audit pass, remediation plan, or
build/review/deploy execution cycle for a specific numbered phase or report series.

## What's in here

- **Numbered report series** (`450`–`564` and others) — HTML reports covering codebase
  audits, defect findings, remediation plans, and execution/closure reports. Report
  numbers are sequential but not necessarily contiguous; later reports supersede earlier
  findings where noted, but nothing here is edited after publication — a correction or
  follow-up is always a new, separately-numbered file.
- `561-tawa-remediation-build-spec-v2.md` — a machine-actionable build spec (113+
  tickets across 8 risk-ordered waves) distilled from the audit report series, written
  for direct implementation by an engineering/agent workflow.
- Screenshot folders (`*-لقطات`, `customer_test_screenshots`, `shots_demo`,
  `agentic-evidence`, etc.) — visual evidence captured during live verification passes
  referenced by specific reports.
- `artifacts/` — production mobile app release builds (`.apk`/`.aab` for `arm64-v8a`
  and `armeabi-v7a`, full and lite variants) plus `SHA256SUMS.txt` for integrity
  verification. The two `.aab` files are tracked via **Git LFS**.
- `tawa_agent_exploration_final_package.zip` — a packaged snapshot from an earlier
  exploration pass.

## Nature of this archive

These are internal working documents, not a curated release changelog. They include
detailed discussion of known defects, hardening gaps, and remediation status at the
time each report was written — some of which may since have been fixed (check the
report's own date and any later report that references it) and some of which may
still be open. Treat report content as a point-in-time engineering record, not a
current security or production-readiness statement.

## Downloads

| File | Variant | Link |
|---|---|---|
| `app-arm64-v8a-fullproduction-release.apk` | arm64-v8a, full | [Download](https://github.com/abd2508/tawa-remediation-reports/raw/master/artifacts/app-arm64-v8a-fullproduction-release.apk) |
| `app-arm64-v8a-liteproduction-release.apk` | arm64-v8a, lite | [Download](https://github.com/abd2508/tawa-remediation-reports/raw/master/artifacts/app-arm64-v8a-liteproduction-release.apk) |
| `app-armeabi-v7a-fullproduction-release.apk` | armeabi-v7a, full | [Download](https://github.com/abd2508/tawa-remediation-reports/raw/master/artifacts/app-armeabi-v7a-fullproduction-release.apk) |
| `app-armeabi-v7a-liteproduction-release.apk` | armeabi-v7a, lite | [Download](https://github.com/abd2508/tawa-remediation-reports/raw/master/artifacts/app-armeabi-v7a-liteproduction-release.apk) |
| `app-full-production-release.aab` | Android App Bundle, full (LFS) | [Download](https://github.com/abd2508/tawa-remediation-reports/raw/master/artifacts/app-full-production-release.aab) |
| `app-lite-production-release.aab` | Android App Bundle, lite (LFS) | [Download](https://github.com/abd2508/tawa-remediation-reports/raw/master/artifacts/app-lite-production-release.aab) |

Verify each download against `artifacts/SHA256SUMS.txt` after downloading — see below.

## Verifying the release binaries

```bash
cd artifacts
sha256sum -c SHA256SUMS.txt
```
