# Tawa Platform Remediation — Build Spec v2 (for automated implementation)

**Audience:** an autonomous coding agent or engineering team implementing this plan directly.
**Not a narrative document** — each item below is a self-contained ticket: current behavior,
target behavior, exact change, prerequisites, acceptance test. Prose is minimized.

**Repo:** `/home/a/Desktop/APP/platform` · **Branch:** `remediation/report557-phases0-6` ·
**Baseline commit for all file:line citations below:** `d9c5d38d` (verified fresh against this
commit; re-verify before executing anything if HEAD has moved further — this repo has an active
parallel workstream, confirmed twice in this exercise: 6bbeaad→d9c5d38d added 14 commits between
report drafts).

**Source lineage:** report 558 (audit verification, 292 findings) → report 559 (v1 execution
plan, 112 fix-clusters) → report 560 (v2 re-verification against d9c5d38d, 11 corrections + 1 new
cluster + 2 new techniques) → **this document** (v2 build spec, corrections applied).

**Governance constraints — read before touching anything:**
- `contracts/feature_flags.yaml` / `contracts/launch_gates.yaml` SHA256 hashes are pinned in
  `CLAUDE.md` under FIN-COR. Any edit to `feature_flags.yaml` (e.g. B3's new Trade flag) **must**
  re-baseline the hash in the same change, or `contract_flag_hash_check.py` fails every
  subsequent phase.
- `wallet_cash_out` and `production_payment_enabled` MUST remain `false`. No ticket below touches
  either.
- No user-visible string literal changes without explicit operator sign-off (project rule). Any
  ticket that changes displayed text is flagged `[NEEDS OPERATOR SIGN-OFF: STRING CHANGE]`.
- No commits to `backend/alembic/`, `backend/app/models/`, `backend/tests/`, `contracts/`,
  `docker-compose.yml` without this being an explicitly authorized backend/infra phase (per
  CLAUDE.md Forbidden Zones) — this spec assumes such authorization exists for the phase it's
  scoped to; confirm before executing if unsure.
- Do not create git commits unless explicitly instructed by the operator for this work (per
  CLAUDE.md's no-commit-rule for this project).

**Ticket status legend:** all 112 original tickets + E15 = 113 tickets as of `d9c5d38d`
(re-verified, zero fixed by intervening commits). **114 as of 2026-07-26**: report 564's
build/review/scenario execution of this document closed 5 (B4, B6, D1, D2, G-G — see report
564 for full build/review/deploy detail) and discovered 1 new ticket not in the original
plan, **D11** (concurrent-offer cap not re-enforced at accept time — see Wave 2, after D3).
Risk tiers: `CRITICAL` / `HIGH` / `MEDIUM` / `LOW`.

---

## 0. Hard sequencing constraints (read first, apply throughout)

These are cross-cutting rules that override ticket order below if they conflict. Referenced by
number (`SEQ-01` etc.) from individual tickets.

- **SEQ-01**: Never flip a lenient CI/gate check to strict enforcement in the same change that
  fixes/prunes the pre-existing failures it would newly catch. Land strictness first against a
  verified-clean baseline, OR fix backlog first then flip strictness — never both in one PR.
  Applies to: A1, A7(a), A7(e).
- **SEQ-02**: A6 (`tawanow` → `_PRODUCTION_STRICT_ENVS`) requires a live dry-run against
  Production's actual env vars before merging — highest blast-radius single-line change in this
  spec. Same discipline for B7 (MinIO hard-stop — verify Production `.env` has real creds first)
  and A3 (health-check extension — ship the producer/heartbeat and confirm it's flowing on all 4
  targets before the consumer/`/ready` check goes live anywhere).
- **SEQ-03**: Any CHECK/FK constraint on a live table needs an exhaustive live-data audit
  (`SELECT DISTINCT <col>, count(*)`) run independently against Production, test1, test/.194
  before adding the constraint. Use `ADD CONSTRAINT ... NOT VALID` + separate `VALIDATE
  CONSTRAINT` in a maintenance window — never a blind validating `ADD CONSTRAINT`. Applies to:
  B10 (User/Role/Device.status — 3 columns, NOT 4, see correction below), C7 (`order_state`),
  C14 (`InventoryItem` Float→Numeric), E12 (Settlement period ordering), E11 (wallet account
  status).
- **SEQ-04**: C1 and C2 (idempotency-hash field additions) ship together. C1 is client-only-safe
  (server hash already live). C2 is a genuine backend hash-shape change — deploy **client-first,
  backend-second** (new fields optional/backward-compatible) to minimize retry-window mismatches.
- **SEQ-05**: C4 and C12's "156" sub-finding share one root cause (redundant early
  `_resolve_delivery` call) and one code location — implement as a single change.
- **SEQ-06**: C14 (Float→Numeric migration) ships **before** C10 (atomic inventory decrement) —
  writing new Decimal arithmetic against a still-Float column raises a Python `TypeError` the
  first time the code path executes.
- **SEQ-07**: D1+D2 ship as one PR against `create_offer`. D3 (shared
  `check_driver_dispatch_eligibility`) depends on both being finalized, and its
  override-escape-hatch shape (who can override, is a reason mandatory) must be resolved with the
  operator **before merging**, not just before deploying.
- **SEQ-08**: B4, B6, D1's SPLIT-awareness all narrow currently-broad access. Each needs a
  shadow-mode observation window (reuse `permission_shadow.py`) before enforcement, not a direct
  cutover.
- **SEQ-09**: B3 (new Trade feature flag) changes `feature_flags.yaml`'s SHA256 — re-baseline
  FIN-COR's pinned hash in `CLAUDE.md` in the **same** change.
- **SEQ-10**: B8 must NOT touch D-60 (COD-OTP threshold) warn-only behavior — leave it off until
  the unrelated group/split-order OTP-reuse bug is fixed first (see CLAUDE.md "Operator
  Activations Pending").
- **SEQ-11**: E1 (Trade invoice maker-checker) and B15 (SUPER_ADMIN dual control) both require
  confirming ≥2 real staff hold the relevant permission/role **before** shipping — otherwise the
  fix becomes a permanently-stuck workflow or an unusable emergency path.
- **SEQ-12**: H4's credit-limit enforcement (Trade) requires backfilling every existing buyer a
  credit policy row (even "unlimited") **before** activating the enforcement check — cannot be
  the same deploy step. Single highest live-production regression risk in this spec.
- **SEQ-13**: H5 (Trade notifications via EventOutbox) ships after H2 (state-machine hardening),
  staged OPS-first then buyer-facing, default off/digest for existing users.
- **SEQ-14**: F1 (outbox post-commit fix) must land before any `notifications_outbox_enabled=True`
  flip anywhere — the relay's throughput ceiling (~400/min, single Redis lock, 60s post-boot
  sleep) needs load-testing against real volume first.
- **SEQ-15**: F2 (KMS fail-closed) and F9's secret-rotation-overlap fix touch the same function
  family (`_get_pos_secret`/`verify_pos_webhook`) — review together so fail-closed logic doesn't
  block the legitimate previous-secret fallback during rotation overlap.
- **SEQ-16**: F5 (locale-aware push copy) and F8 (bypass-allowlist consolidation) touch the same
  function (`_dispatch_channels`) — implement as one coordinated changeset.
- **SEQ-17**: I4 (mobile outbox price-revalidation) needs **zero backend coordination** — the new
  field is additive/backward-compatible in both directions, `/carts/quote` already exists and is
  live, and `revalidate_cart_pricing` already treats a missing price field as "skip check." No
  rollout-order hazard in either direction — the one exception to SEQ-14-style caution in this
  spec.
- **SEQ-18**: I4 and I5 (legacy outbox ownership) share the same file/code path and ship together.
- **SEQ-19**: J1 (OpenAPI coverage) and J5 (pos_router error envelope, now confirmed **23** raw
  `HTTPException` sites not 8 — see correction) execute together, module-by-module; this also
  overlaps F9's structured-envelope fix — implement the POS error envelope once, not twice.
- **SEQ-20**: J6's polymorphic `target_type` columns need a registry table populated with every
  distinct existing value **before** a CHECK constraint goes live.
- **SEQ-21**: J6's UUID-native migration must migrate a table's PK and every FK-referencing column
  in the **same** migration/PR — a half-migrated state breaks FK constraints on type mismatch.
- **SEQ-22**: J3's fenced-lock rollout is safe as a single-phase change **only because the
  platform runs one backend instance per environment today** (confirmed:
  `rate_limiter.py` documents this explicitly). If the platform ever goes multi-replica, split
  fenced-acquire and fenced-release into two separate releases.
- **SEQ-23**: J8's admin-allowlist hard-block needs a pre-merge audit against all 4 live deploy
  targets — **correction (v2): the empty-allowlist case is silently open today with zero logging**
  (not "warn-only" as originally assumed), strengthening the case for this fix, not weakening it.
- **SEQ-24**: G-G's maker-checker coverage expansion (+ G-O's H-09 fold-in) needs confirmation
  that a second distinct approver identity exists per domain (KYC review, feature-flag
  activation, OPS-account approval) **before** the gate goes live.
- **SEQ-25**: G-A's MinIO purge and G-B's Trade-obligation eligibility check must run in the
  existing order (eligibility check first, destructive cleanup second) — confirmed already
  correct in current code (`anonymize_user()` calls eligibility before any destructive action
  unless `force=True`). Do not let any future edit reorder this.
- **SEQ-26**: Trade Cluster H1 (idempotency-key extension) ships alone, first — nothing else in
  the Trade plan depends on it, and it de-risks every other Trade cluster's retry/duplicate-
  submission edge cases for free.
- **SEQ-27**: Group-Order Gap 1 (hash fix) lands before Gaps 2-3's tests are written — tests must
  prove the fixed behavior, not the pre-fix behavior.
- **SEQ-28**: B10's 042 (TOTP encryption) and B7 (MinIO/KMS infra) share dependency on a
  provisioned KMS key — both are safe to ship as inert scaffolding before the key exists.
- **SEQ-29**: C6, 066, and 173 (file/table-splitting refactors) must never be combined with a
  correctness fix in the same diff. Land as a series of small PRs, test-suite-green-verified
  after each.
- **SEQ-30**: Product/policy decisions requiring **operator sign-off before any code is written**
  (not just before deploy): B5 (narrow ADMIN inheritance), B9's `registration_require_otp` flip,
  C13's one-staff-one-store relaxation, D9 (pending-review gate before auto-suspend), D5
  (background location tracking + Apple Developer enrollment), E5 (notify finance before COA
  reclassification, not after).

---

## 1. Wave 0 — Ship immediately, zero precondition

| ID | Title | File(s) | Risk |
|----|-------|---------|------|
| H1 | Extend `_GUARDED_PATHS` in `idempotency_middleware.py` to Trade PO/approve/reject/invoice/shipment paths | `backend/app/core/idempotency_middleware.py:56-61` | HIGH (cheap, high value — do first per SEQ-26) |
| H4-doc | Fix `contracts/internal_api.md:361` — still says Trade invoices "post NO ledger entry"; false since commit `7fe8a64f` | `contracts/internal_api.md` | LOW |
| I1 | Add bounded timeout (~2.5-3s) to mobile force-update version check; add 3rd gate state `versionCheckDone` | `apps/mobile/src/app/_layout.tsx:49-60,130-134` | LOW |
| I2 | Freeze mobile OpenAPI codegen against a committed local snapshot instead of live `tawanow.com` URL | `apps/mobile/package.json:73` | LOW |
| I6 | Fetch schedule-lead-time config from server instead of hardcoding 45/60/120 | `apps/mobile/src/app/checkout.tsx:39-44` | LOW |
| I7 | Replace `Date.now()+Math.random()` idempotency key with `Crypto.randomUUID()` (expo-crypto already a dependency) | `apps/mobile/src/app/checkout.tsx:72` | LOW |
| I4-partial | Add `expectedUnitPriceMinor` to mobile `CreateOrderInput.items` — reactivates dormant backend price-increase guard, zero backend work (SEQ-17) | `apps/mobile/src/services/outbox.ts`, `checkout.tsx` | HIGH |
| E4 | Copy `wallet_service.get_or_create_wallet`'s SAVEPOINT+IntegrityError pattern into `ledger_service.get_or_create_account` | `backend/app/modules/payments/ledger_service.py:35-55` (pattern ref: `wallet_service.py:111-152`) | HIGH |
| E6 | `reserve_wallet`: raise `WalletReservationAmountMismatchError` if replayed amount differs from existing reservation | `backend/app/modules/payments/wallet_service.py:341-350` | HIGH |
| E7 | Fix dispute-state filter: `.where(FinancialDispute.state.in_(("OPEN","UNDER_REVIEW")))` not `== "OPEN"` | `backend/app/modules/payments/wallet_service.py:277-284` | HIGH |
| B1 | Remove `"DELETED"` from `reinstate_driver`'s allowed source states; add explicit driver-role-existence check | `backend/app/modules/operations/operations_router.py:6940,6974` | HIGH |
| B2 | Remove raw-`provider_id` fallback in staff registration; require `staff_invite_code` unconditionally (confirm no live UI depends on bare-UUID path first) | `backend/app/modules/identity/registration_router.py:787-861` | HIGH |
| D10 | Delete duplicate line `audit_s = audit_session or self.session` (appears twice consecutively) | `backend/app/modules/dispatch/dispatch_service.py:459-460` | LOW |
| G-N-partial | Add real alerting (not just `logger.warning`) to audit's no-session background-write failure path | `backend/app/core/audit.py` (no-session branch) | MEDIUM |
| J2 | Replace `f"db: {exc}"` / `f"redis: {exc}"` in `/ready` with fixed generic strings + server-side `logger.warning` | `backend/app/health.py:17-18,25-26` | MEDIUM |
| J4 | Regenerate `contracts/schema.sql` from live ORM/Alembic output instead of hand-maintained PostGIS/ENUM-based file | `contracts/schema.sql` | LOW |
| F3 | Add `correlation_id`/`causation_id` columns to `EventOutbox`/`NotificationRecord`, threaded via contextvar | `backend/app/models/notification.py` | LOW |
| F7 | Add `attempt_no`, `provider_message_id` (partial unique index) to `NotificationDeliveryLog`; parse FCM `name` field | `backend/app/models/notification.py:97-118`, `push_provider.py:36-39,135` | LOW |
| C9 | Track per-debt success/failure in sequential debt-repayment loop instead of one generic error | `web/src/pages/customer/CheckoutPage.tsx` (`payDebtsAndRetry`) | LOW |
| C11 | Add real Postgres concurrency stress test for promo usage-limit atomic UPDATE (already correctly implemented, untested under load) | new `backend/tests/` file, Postgres tier | LOW |
| C15 | Consolidate web/mobile local pricing-preview calculators into one shared module (no correctness fix, friction reduction only) | `web/`, `apps/mobile/` | LOW |
| A4 | Build self-hosted `gh`-CLI merge-gate script using GitHub Checks API — confirmed both classic branch protection AND rulesets API return 403 on this repo's plan tier | new `deploy/scripts/check_pr_mergeable.sh` | LOW |
| A10-partial | Track OpenAPI gap burn-down incrementally (see §6 modern-technique note on snapshot-ratchet pattern) | CI config | LOW |

---

## 2. Wave 1 — Live-data audit REQUIRED before shipping (highest blast radius)

**Do not execute any of these without first running the audit step listed.**

### A6 — `tawanow` → `_PRODUCTION_STRICT_ENVS` `[CRITICAL]`
- **File:** `backend/app/config.py:28` — `_PRODUCTION_STRICT_ENVS: frozenset[str] = frozenset({"production", "release"})`
- **Change:** add `"tawanow"` to the frozenset.
- **Effect:** promotes `warning()`-only checks (OTP-provider validity, CR-04 admin-IP-allowlist)
  to boot-time hard failures for the live `tawanow` env specifically.
- **MANDATORY PRE-STEP:** SSH into Production, inspect real env vars (or build a
  `python -m app.config --validate-only` dry-run entrypoint and run it against a copy of
  Production's real `.env`). Confirm a clean boot BEFORE landing this line. If a real gap is
  found, fix it in a **separate, prior** deploy — never bundle the gap-fix and this
  reclassification in the same deploy (SEQ-02).
- **Note:** does NOT affect D-59/60/61 (MFA/COD-OTP/outbox) — those are gated on separate opt-in
  flags, unaffected by this change (see B8).
- **Acceptance:** Production boots cleanly post-change; a deliberately-tampered `OTP_PROVIDER=stub`
  now refuses to boot instead of silently degrading.

### B7 — MinIO hard-stop on default credentials `[CRITICAL]`
- **Files:** `backend/app/modules/files/file_router.py:39-59`, `backend/app/config.py`
- **Change:** centralize a boot-time hard-stop in `config.py`'s validation pass (mirror the CR-04
  pattern) for strict envs when MinIO creds equal `minioadmin`/`minioadmin123`.
- **MANDATORY PRE-STEP:** verify via direct SSH/`docker exec env` that Production's real `.env`
  sets non-default `MINIO_ROOT_USER`/`MINIO_ROOT_PASSWORD` — do NOT trust the compose file (it
  doesn't set them; relies on server `.env`). If not set, set real credentials first, verify
  upload/download still works end-to-end, THEN land the hard-stop in a separate deploy.
- **Also confirm:** test1/test/.194's `APP_ENV` values are NOT accidentally caught by
  `_PRODUCTION_STRICT_ENVS` — they intentionally run loopback-only MinIO with dev defaults per
  CLAUDE.md Safety Rules.
- **Acceptance:** Production boots with real creds; a future `.env` regression that drops the
  vars now fails to boot instead of silently using weak defaults.

### C7 — `order_state` CHECK constraint `[CRITICAL]`
- **File:** `backend/app/modules/orders/state_transitions.py` + new Alembic migration (renumber
  per correction below — pick `ab69+`)
- **Change:** `ALTER TABLE orders ADD CONSTRAINT ... CHECK (order_state IN (<27 known values>))`
- **MANDATORY PRE-STEP:** run `SELECT DISTINCT order_state, count(*) FROM orders` independently
  against Production, test1, test/.194. If zero violations on all three, use
  `ADD CONSTRAINT ... NOT VALID` then a separate `VALIDATE CONSTRAINT` in a maintenance window.
  Never a blind validating `ADD CONSTRAINT` on first attempt (SEQ-03).
- **Side effect:** any OPS/support script writing an undocumented sentinel state value will now
  be blocked — confirm no runbook depends on this before shipping.
- **Acceptance:** constraint validates cleanly on all 3 targets; a new code path writing an
  unlisted state value now raises `IntegrityError` at insert time.

### B10-status (CORRECTED SCOPE) — CHECK constraints on status columns `[HIGH]`
- **CORRECTION (v2):** original ticket claimed 4 columns lack a CHECK constraint
  (`User.status`/`Role.status`/`Device.status`/`Address.status`). **`Address.status` already has
  one**: `backend/app/models/user.py:226` —
  `CheckConstraint("status IN ('ACTIVE', 'ARCHIVED')", name="ck_addresses_status")`.
  **Correct scope: 3 columns only** — `User.status` (line 65), `Role.status` (line 121),
  `Device.status` (line 144).
- **Reuse:** `ck_addresses_status` and `ck_users_kyc_tier` as in-repo precedents.
- **MANDATORY PRE-STEP:** live-data audit on all 3 columns × 4 targets before adding constraints
  (SEQ-03).

### C14 — `InventoryItem` Float → Numeric(10,3) `[MEDIUM]`
- **File:** `backend/app/models/catalog.py` (`available_quantity`, `low_stock_threshold`)
- **MUST ship before C10** (SEQ-06) to avoid Decimal/float mixing `TypeError`.
- **Also:** add real FKs to `SubstitutionOffer.order_line_id`/`offered_by` after a data-integrity
  audit.

### E12 — Settlement period CHECK `[MEDIUM]`
- **File:** `backend/app/models/payment.py:274-275` (`Settlement.period_start`/`period_end`)
- **Change:** `CHECK(period_start <= period_end)` after live-data audit (SEQ-03).
- **Migration number:** renumber to `ab69+` (see correction below).

### J8 — Admin IP allowlist hard-block `[MEDIUM]`, CORRECTED SEVERITY
- **File:** `backend/app/modules/*/ip_allowlist.py` (`AdminIPAllowlistMiddleware`)
- **CORRECTION (v2):** the empty-allowlist case is **silently open with zero logging** (not
  "log-warn" as originally stated) — line 39-40 returns `await call_next(request)`
  unconditionally with no log line at all in that branch. The only `logger.warning` in the file
  (line 51) fires on a *blocked* IP, not on empty-allowlist.
- **MANDATORY PRE-STEP:** audit all 4 live deploy targets — if any relies today on an empty
  allowlist, a hard boot-block breaks its next deploy (SEQ-23).

---

## 3. Wave 2 — Shadow-mode-first (narrowing broad access)

### B4 — Trade resource-scope `[HIGH]`
- **File:** `backend/app/modules/identity/dependencies.py:560` (`require_trade_permission`)
- **Change:** add a `scope` parameter naming which path/body field to check against the acting
  user's assigned scope. **Requires a new data-model concept**: an OPS-staff-to-supplier scope
  table (mirror `OperationalProfile.provider_id`'s existing pattern).
- **MANDATORY SEQUENCE (SEQ-08):** (1) ship scope-check machinery in shadow mode first (reuse
  `permission_shadow.py`), (2) observe a real window in Production, (3) only flip to enforcing
  once shadow logs show zero/authorized cross-scope activity, (4) if wide cross-supplier activity
  is normal, pair enforcement with an entitlement-grant pass first — never enforce then grant.
- **Also needed:** an explicit `admin:all`-style bypass for SUPER_ADMIN/platform-wide roles, or
  the fix locks out the platform owner.

### B6 — `require_role` → `require_permission` migration `[HIGH]`
- **File:** `backend/app/modules/operations/operations_router.py`
- **CORRECTION (v2):** confirmed count is **110/8** (not 111/9 as briefly miscounted during
  verification) — matches original audit exactly.
- **Plan (not a single patch):** (1) strengthen CI ratchet to a shrinking budget file (not just
  no-growth) — ship immediately, zero risk; (2) convert call-sites incrementally, financial/
  driver-status/PII-touching endpoints first; (3) each conversion needs shadow-mode logging first
  (same risk pattern as B4); (4) layer resource-scope on top afterward.

### D1+D2 — Cash-limit + concurrent-cap fix `[HIGH]`, single PR (SEQ-07)
- **D1 file:** `backend/app/modules/dispatch/dispatch_service.py:377-397` (`create_offer`)
  - Add `order_has_cash_leg()` / `prospective_cash_due_minor()` helpers (reuse `collect_cash`'s
    exact wallet/points/card attribution math, don't reimplement).
  - Check `driver.cash_outstanding_minor + prospective_due > limit` (currently only checks
    standing balance, `>=`, ignoring the new order's own amount).
  - Make `cash_allowed` gate check `payment_method in ("CASH","SPLIT")`, not `== "CASH"` only.
- **D2 file:** same file, line 445 (`_accepted_count` query)
  - Join to `Order.order_state`, filtering to `_ACTIVE_DELIVERY_ORDER_STATES` — **reuse the
    already-existing `busy_driver_ids_subquery()` at line 84**, do NOT add a new terminal offer
    state (it's read as "current/historical assignment" in ~20 other call sites: trip history,
    access control, settlement, deletion eligibility, trust score).
- **Pre-deploy audit:** `SELECT count(*) FROM drivers WHERE cash_allowed=false` cross-checked
  against currently-ACCEPTED SPLIT offers — these drivers were never actually gated before; this
  fix makes them newly ineligible.
- **Acceptance:** driver at 195,000/200,000 outstanding taking a 30,000 CASH order is now blocked
  pre-acceptance (was previously allowed, ending at 225,000 post-collection).

### D3 — Unified `reassign_driver` gate `[HIGH]`, depends on D1+D2 (SEQ-07)
- **File:** `backend/app/modules/operations/operations_router.py:982` (`reassign_driver`)
- Extract `check_driver_dispatch_eligibility()` covering status/available, cash-limit
  (prospective+SPLIT-aware), `cash_allowed`, `errand_allowed`, `dispatch_blocked_until`,
  concurrent-cap. Both `create_offer` and `reassign_driver` call it.
- **MUST include** an `override_gates: list[str]` param + mandatory `reason` for
  `reassign_driver`'s emergency-override case — today's gate-bypass is (accidentally) OPS's only
  emergency escape hatch; removing it with no replacement breaks a legitimate workflow.
- **Resolve override-hatch UX with operator before merging**, not just before deploy.
- **Test:** 6 gates × 2 call sites = 12 parametrized tests asserting both paths reject the same
  fixture identically.

### D11 (NEW — discovered 2026-07-26 during report 564's build/review/scenario execution of
D1+D2, not in original plan) — concurrent-offer cap not re-enforced at accept time `[HIGH]`
- **Files:** `backend/app/modules/dispatch/dispatch_service.py` — cap check lives only in
  `create_offer` (~line 471, the D2-fixed query); `respond_to_offer` (line 1329) never
  re-checks it before setting `offer.state = "ACCEPTED"` (~line 1385).
- **Background:** D2 (this same file, above) fixed the cap *count* to exclude offers whose
  order already reached a terminal state — a real, confirmed improvement, verified correct
  by an independent scenario/experiment agent during report 564's execution. That same
  agent, running real experiments (not just code reading) against both SQLite and a real
  Postgres instance, found a second, distinct gap in the same mechanism: the cap is
  evaluated only at offer-*creation* time (`create_offer`), never re-checked when a driver
  *accepts* an offer (`respond_to_offer`). Because `SENT` offers are deliberately exempt
  from the cap count (by design, to allow batching), a driver can simultaneously hold
  several `SENT` offers — an ordinary, unremarkable dispatch state, not an edge case — and
  accept all of them, ending up over the configured cap with no timing/race trick required.
- **Reproduced (2026-07-26, both deterministically and under genuine concurrency):**
  (a) sequential, no concurrency at all: seed a driver at cap−1 (2 of 3) `ACCEPTED`, create 2
  more `SENT` offers via real `create_offer()` calls, `respond_to_offer(ACCEPT)` both in
  turn → both return `state=ACCEPTED`; direct DB count afterward = 4 active `ACCEPTED`
  offers against a configured cap of 3. (b) genuinely concurrent, real Postgres, two
  independent connections via `asyncio.gather`: two concurrent `create_offer` calls at the
  N−1 boundary both succeed as `SENT` (expected — exempt by design), then two concurrent
  `respond_to_offer(ACCEPT)` calls on those two offers, with real row locks, both succeed;
  direct DB count afterward = 4 (cap = 3).
- **Not a regression from D1/D2 or from report 564's batch** — `respond_to_offer` was not
  touched by either ticket; this gap pre-dates this remediation window and is unrelated to
  D1's shadow-mode change. Flagged here as a genuine, real-world hole in the *existing*
  concurrent-cap mechanism that D2's own fix made newly visible/relevant.
- **Work required:** re-check the driver's live active-offer count (the same
  `_ACTIVE_DELIVERY_ORDER_STATES`-joined query D2 now uses) inside `respond_to_offer`,
  under the same row lock already taken there (`with_for_update()` on the offer, line
  ~1339) or an equivalent driver-level lock, before flipping `offer.state` to `ACCEPTED` —
  reject with the same `DRIVER_CONCURRENT_CAP_EXCEEDED`-style error `create_offer` already
  raises. Needs its own live-data audit (SEQ-03 discipline) of how many drivers today
  legitimately hold multiple simultaneous `SENT` offers before landing, since tightening
  accept-time enforcement could newly block a currently-working batching flow.
- **Acceptance:** the two reproductions above (sequential and genuinely-concurrent) both
  end with the driver's true active-offer count never exceeding the configured cap.

### G-G — Maker-checker coverage expansion `[HIGH]`
- **CORRECTION (v2):** file path is `backend/app/modules/ads/ads_service.py`, not
  `app/modules/marketing/ads_service.py`.
- Migrate `ads_service.py:264,337`'s hand-rolled `SELF_APPROVAL` checks onto
  `require_not_self_approval` (already defined in `authorization_helpers.py`).
- Add `require_not_self_approval()` calls at ~6 currently-unprotected toxic combinations (KYC
  reviewer+approver, feature-flag author+activator, others named in the audit).
- **MANDATORY PRE-STEP (SEQ-24):** confirm a second distinct approver identity exists per domain
  before flipping any of these on — retrofitting onto thin-staffed workflows (e.g. one person
  currently does both KYC review and approval) turns a correctness fix into an operational block.

---

## 4. Wave 3 — Financial integrity (includes new Wave-3 addition: E15)

### E1 — Trade invoice maker-checker `[HIGH]`
- **File:** `backend/app/modules/trade/po_router.py:512-682` (`create_invoice`)
- Add `TradeInvoice.ledger_status` (NOT_APPLICABLE/PENDING_APPROVAL/POSTED),
  `pending_entries_json`, `approved_by_user_id` — **keep separate from existing `status`** field
  (which carries the MATCHED/DISCREPANCY three-way-match verdict).
- On `verdict == "MATCHED"` (line 629): stop posting inline at line 658; persist computed entries
  to `pending_entries_json`; require a second distinct staffer via new `/invoice/approve`
  endpoint (mirror `approve_manual_journal`'s shape exactly).
- **Migration:** backfill existing `MATCHED`+`journal_id`-set rows to `ledger_status="POSTED"`,
  `approved_by_user_id=created_by_user_id` (same compromise the team already made once for
  `ManualJournal`'s C-11 migration — reuse it).
- **CORRECTED migration number:** original plan proposed `ab61` — now taken by real
  `ab61_trade_supplier_catalog.py`. **Use `ab69+`.**
- **MANDATORY PRE-STEP (SEQ-11):** confirm ≥2 staff hold `trade:invoice_manage` before shipping —
  otherwise every future MATCHED invoice gets permanently stuck at PENDING_APPROVAL.
- **Note:** `ab68` migration already promoted `trade:invoice_manage` to Tier-S in
  `permissions_catalog` — confirms (doesn't contradict) this ticket's assumption it's already
  OPS-only.

### E3 — Journal Header (report-only) `[MEDIUM]`
- New `JournalHeader` table, `status` always `POSTED`, backfilled via `GROUP BY journal_id`. Zero
  risk to existing postings.
- **Migration number correction:** was `ab62`, now taken by `ab62_trade_inventory_reservation.py`
  — use `ab69+`.

### E5 — COA_MAP entries for Trade `[MEDIUM]`
- **File:** `backend/app/modules/finance/coa_mapping.py:27-63`
- Add `TRADE_BUYER_RECEIVABLE`/`TRADE_SUPPLIER_PAYABLE`/`TRADE_REVENUE` entries.
- **Operator sign-off required before deploy (SEQ-30):** report-side only, zero ledger-balance
  risk, but reclassifies existing Trade postings in reports immediately — notify finance first.

### E8 — Settlement/LedgerEntry FK `[MEDIUM]`
- Split `Settlement.beneficiary_id` into typed nullable FKs mirroring `CashHandover`'s pattern
  (`driver_id`/`agent_id` at lines 244/247). **Leave `LedgerEntry.reference_id` polymorphic as-is
  — correct by design, do not add a single FK there.**
- **Migration number correction:** was `ab63`, now taken by `ab63_trade_reservation_expiry_index.py`
  — use `ab69+`.

### E15 (NEW — discovered during v2 verification, not in original plan) `[HIGH]`
- **File:** `backend/app/modules/finance/reconciliation_service.py` —
  `check_wallet_ledger_invariant()`
- **Background:** commit `f0983cf0` fixed one direction of a wallet/ledger balance-vs-journal
  drift (referral-accrual journals post a bookkeeping-only CR leg without touching
  `WalletAccount.balance_minor`). The function's own updated docstring admits a **second,
  opposite-direction drift source remains unresolved** (historical/seeded wallet balances with
  no funding journal), and explicitly states this guard is "not safe to wire to any live alert."
- **Work required:**
  1. Identify and fix the opposite-direction drift source (historical/seeded balances lacking a
     funding journal).
  2. Build a real-data verification step: seed a known referral volume, confirm remaining drift
     matches an expected baseline, before wiring `check_wallet_ledger_invariant` to any live
     OPS/finance alert.
- **Acceptance:** guard produces zero false-positives across both known drift directions on a
  realistic data seed; only then connect it to an alerting channel.

### G-I — Trade commission freeze `[HIGH]`
- **File:** `backend/app/modules/trade/po_router.py:633`
- Add `commission_bps`/`commission_fixed_minor` to `TradePurchaseOrder`; resolve once at
  CONFIRMED transition, persist; `create_invoice` reads the frozen rate.
- **MANDATORY bridge fallback:** `if po.commission_bps is None: resolve at invoice time (today's
  behavior)` for POs already CONFIRMED-but-not-invoiced at migration time — skipping this either
  crashes in-flight invoicing or silently treats NULL as 0% commission.

### H4-credit — Trade buyer credit limit `[CRITICAL]`
- New `TradeBuyerCreditPolicy` table (buyer_provider_id, limit_minor, status) + pre-PO-submission
  check comparing outstanding `TRADE_BUYER_RECEIVABLE` against the limit, reject with 409 if
  exceeded.
- **ABSOLUTE PREREQUISITE (SEQ-12):** every existing active buyer MUST be backfilled a policy row
  (even "unlimited"/NULL) in a separate, prior deploy step — **enforcement and backfill cannot be
  the same deploy.** This is the single highest live-production regression risk in this entire
  spec (real buyers, real receivables, real revenue).
- Also: split `trade:access` into `trade:access` (operational) + `trade:financial_read` (invoice/
  receipt reads) — requires re-granting `trade:financial_read` to every OPS role that currently
  reads via the broad grant.

### H4-doc, H4-creditnote — see Wave 0 (doc fix) and below
- Add `credit_note_of_invoice_id` (self-FK) + `CREDIT_NOTE` type + `POST
  /trade/invoices/{id}/credit-note` posting a reversing journal. `[MEDIUM]`

---

## 5. Wave 4 — Notification/POS reliability

### F1 — Post-commit notification dispatch `[HIGH]`
- **File:** `backend/app/modules/notifications/notification_service.py:428-443`
- Make `EventOutbox` insert unconditional regardless of `notifications_outbox_enabled`; add a
  SQLAlchemy `after_commit` session-event listener scheduling an immediate one-shot drain of
  *this row* (preserves today's low latency when the flag is off). When flag is `True`, skip the
  immediate task, let the 15s relay pick it up.
- **MANDATORY PRE-STEP (SEQ-14):** before ever flipping `notifications_outbox_enabled=True`
  anywhere, load-test the relay (single Redis lock, `limit=100`/15s tick, 60s post-boot sleep,
  ~400 events/min ceiling) against real production volume — flipping without this risks delaying
  time-sensitive sends (`dispatch.offer_received`, 60s expiry window) past their deadline.
- **MUST confirm and exclude:** grep for OTP send call sites — if OTP routes through `send()`, it
  needs a `bypass_outbox` param forcing the inline (post-commit) path unconditionally, since
  relay-delayed OTP breaks login/checkout, not just degrades UX.

### F2 — KMS fail-closed `[CRITICAL]`
- **File:** `backend/app/modules/integrations/pos_router.py:65-88` (`_get_pos_secret`)
- Change: on `KmsEnvelopeError` when an encrypted secret exists, raise `PosSecretUnavailableError`
  → clean `503 POS_WEBHOOK_SECRET_UNAVAILABLE` (distinct from `401
  POS_WEBHOOK_INVALID_SIGNATURE`). Only providers with **no** encrypted secret at all (never
  migrated) legitimately still read plaintext — not a fallback, the sole source of truth for them.
- **Add break-glass:** new `pos_webhook_kms_bypass_until` column, SUPER_ADMIN-gated endpoint,
  mandatory written justification, `AuditLog` row + ops alert, time-boxed. Categorically
  different from today's bug (unconditional silent fallback) — this is opt-in, expiring, traceable.
- **Review together with F9's rotation-overlap fix (SEQ-15)** — don't let fail-closed logic block
  the legitimate previous-secret fallback during rotation.

### F4 — Real DB-level outbox locking `[HIGH]`
- **File:** `backend/app/modules/sse/outbox_relay.py:44-60,75`
- **CORRECTION:** module docstring claims "FOR UPDATE SKIP LOCKED" — the actual query has **no
  such clause**, just a plain `select().order_by().limit()`. Add
  `.with_for_update(skip_locked=True)` — gate behind a dialect check (`!= "sqlite"`), since SQLite
  doesn't support it and the test suite is SQLite-backed.
- Change `_dispatch_channels` to return `DispatchOutcome(any_attempted, any_succeeded, results)`;
  treat "attempted but zero succeeded" as a failure entering the existing retry/dead-letter path
  — but distinguish "no eligible channel" (legitimate no-op) from "channel eligible and failed"
  to avoid flooding dead-letter with false positives.

### F5+F8 — Locale-aware push copy + bypass-list consolidation `[MEDIUM]`, one changeset (SEQ-16)
- **File:** `notification_service.py:456-457` (auto-derived title), `:221` (bypass frozenset),
  `:575-583/:608-619/:681-688` (3 separate channel-eligibility sets)
- Build `{ar,en,fr}` push-copy dict keyed by `event_type`, locale from `User.preferred_locale`.
- Move bypass frozenset to a DB-backed toggle (fail-closed).
- Consolidate 3 channel-eligibility sets into one co-located mapping.
- **New technique (v2 discovery):** once built, add a backend ratchet test mirroring web's new
  `i18n-missing-keys.test.ts` (commit `d4366810`) asserting every `NOTIFICATION_EVENTS` key has
  ar/en/fr entries — reuse the pattern, don't invent a new completeness check.

### F6 — Fail-closed channel-toggle read `[HIGH]`
- Bounded single retry (50ms) then fail CLOSED (not open) on DB error reading channel-enabled
  flag. Applies to both EMAIL (`:627-641`) and WHATSAPP (`:690-706`) blocks.
- **DONE** (batch F4/F6/E3/G-H, report 566, commit `c56117fc`). Also closed the F4×F6
  interaction gap found by that batch's scenario/experiment agent: when EMAIL was the *only*
  eligible channel and its toggle read genuinely failed, the old code correctly failed CLOSED
  but never recorded an attempt, so F4's `any_attempted` stayed False and the notification was
  marked delivered with zero retry despite nothing being sent. `_read_channel_toggle` now
  returns `(enabled, read_failed)` and both EMAIL/WHATSAPP call sites record a failed attempt
  when `read_failed` is True.

### F10 (NEW — discovered 2026-07-27, adversarial reviewer on report 566's batch) — EMAIL
dispatch exception safety `[MEDIUM]`, non-blocking, no ticket yet built
- `notification_service.py`'s `_dispatch_channels` docstring says "Never raises" (a contract
  F4/F6/`outbox_relay.py` now depend on), but unlike the WHATSAPP block, the EMAIL block has
  no try/except around it — `_render_email_payload`'s `format_map` on a malformed operator
  template, or the `session.get(User)` call, can still raise. Currently dormant: with
  `notifications_outbox_enabled=False` (today's default everywhere), an EMAIL-path exception
  propagates up through `send()`/`send_order_event()` into the calling business transaction
  (order state change, dispatch accept) instead of being caught and recorded as a failed
  attempt like every other channel. Fix: wrap the EMAIL branch in its own try/except mirroring
  the WHATSAPP block (`:690-706`-era code). Must land before `notifications_outbox_enabled` is
  ever flipped true (that flip already needs F1's own ticket first, so this doesn't block
  anything currently live). A secondary, lower-priority reviewer note from the same pass: audit
  whether the WHATSAPP exception path's audit-log write covers every raise site the same way
  the EMAIL fix above should.
- **DONE** (2026-07-27, commit `bb73a700` — see report 567). Wrapped the EMAIL branch in its own
  try/except mirroring WHATSAPP's. Post-review correction (found independently by both the
  standing advisor and the independent adversarial reviewer on THIS ticket's own review
  pass): the first version of the fix set `_email_attempted` right before `send_email`
  (mirroring WHATSAPP's `_wa_attempted` placement exactly) — but EMAIL, unlike WHATSAPP, is
  sometimes the SOLE eligible channel (e.g. `auth.password_changed`), so a persistent
  `_render_email_payload` bug (malformed operator template) or a recipient-lookup DB error
  would be caught, logged, and then silently dropped with `any_attempted=False` — the exact
  silent-delivery-with-nothing-sent class F4/F6 exist to prevent. Fixed by flipping
  `_email_attempted` as soon as the channel is confirmed enabled, before the recipient
  lookup, so both raise sites are retry/dead-letter-eligible. `_dispatch_channels`'s
  docstring also softened from an unqualified "Never raises" to the actual guarantee
  (EMAIL/WHATSAPP guarded, PUSH/SMS/lookup DB errors are not — see F11 below).

### F11 (NEW — discovered 2026-07-27, filed while closing F10) — PUSH/SMS dispatch
exception safety `[LOW]`, non-blocking, no ticket yet built
- Same shape as F10, one tier down in severity/likelihood. `_dispatch_channels`'s PUSH block
  (`send_push` call + the `DeviceToken` select feeding it) and SMS block (`session.get(User)`
  + `send_sms`), plus the CUSTOMER/DRIVER preference lookups upstream of PUSH
  (`CustomerPreferences`/`DriverPreferences`, ~lines 577-599), have no try/except — a DB
  error in any of them still propagates out of `_dispatch_channels`, up through
  `send()`/`send_order_event()`, into the caller's business transaction, exactly like
  EMAIL did before F10. Not fixed as part of F10 to keep that ticket scoped to the asymmetry
  it was actually filed for (matches this batch's own J3-deferral precedent: don't grow a
  ticket mid-flight to cover every call site touching the same file). Fix, when built: same
  pattern as F10/WHATSAPP — wrap each block in its own try/except, record a failed attempt
  only once the channel is confirmed eligible (not merely "about to call the provider"),
  matching F10's corrected `_email_attempted` placement rationale, not WHATSAPP's original
  (narrower) one.
- **IMPORTANT, proven empirically during F10's own build (real-Postgres
  experiment, not code reading): try/except ALONE is NOT sufficient for the DB-read sites
  (the `DeviceToken` select feeding PUSH, SMS's `session.get(User)`, and the
  `CustomerPreferences`/`DriverPreferences` lookups). On Postgres, an uncaught DB error
  inside a plain try/except still leaves the enclosing transaction aborted
  (`InFailedSqlTransactionError`) — the Python exception is caught, but every subsequent
  statement on that same session (including the IN_APP dedup SELECT and the final `flush()`
  at the end of `_dispatch_channels`) then raises anyway, so the "failed attempt" never
  actually reaches the returned `DispatchOutcome`. SQLite (the unit-test dialect) does NOT
  abort-poison transactions, so this is invisible to the test suite alone — it only showed
  up under a real Postgres experiment. Each of those DB-read sites must be wrapped in
  `async with self._session.begin_nested():` (the SAVEPOINT pattern, same as
  `_read_channel_toggle` and F10's EMAIL-recipient-lookup fix), not bare try/except. See
  `notification_service.py`'s EMAIL block (post-F10) for the exact pattern to copy.**

### F9 — POS webhook hardening `[HIGH]`, overlaps J5/J1 (SEQ-19)
- **193:** unify ~23 raw `HTTPException(detail=...)` sites into one `PosWebhookError` envelope
  `{code, message}`. **`[NEEDS OPERATOR SIGN-OFF: breaking wire-contract change for POS vendors
  parsing `detail` as plain string]`.**
- **194:** length/regex-validate `X-Event-Id` (column is `String(100)`; an over-long header
  currently reaches INSERT and raises unmanaged `DataError`) — return clean 422 instead.
- **195:** synthetic per-provider `actor_id` (e.g. `pos-integration:{provider_id}`) instead of
  reusing `provider_id` — **verify `actor_id` isn't a strict FK to `users` first.**
- **196:** add `pos_webhook_secret_encrypted_previous` + expiry columns; verify tries current
  secret first, falls back to previous within a configurable (e.g. 24h) overlap window.
- **088:** track and surface `skipped_items: [{item_id, reason}]` instead of silent `continue` +
  blanket `"PROCESSED"`.

---

## 6. Wave 5 — Privacy / OPS

### G-A — anonymize_user() completeness `[HIGH]`
- **File:** `backend/app/modules/identity/anonymization.py:79-89`
- Delete `DeviceToken` rows; bulk-update `NotificationRecord` to
  `title_key/body_key="[erased]", template_vars={}`; best-effort MinIO purge via new
  `purge_user_files()` that **never raises**.
- Add `pending_file_erasures` table (user_id, object_key, first_failed_at) + retry sweep for
  MinIO purge failures — don't accept silent one-shot best-effort as sufficient for a
  legally-mandated erasure claim.
- **Order invariant (SEQ-25):** eligibility check (G-B) runs before this destructive step —
  already correct in current code, do not reorder.

### G-B — Trade-obligation eligibility check `[HIGH]`
- **File:** `backend/app/modules/identity/deletion_eligibility.py`
- Block deletion if any owned provider has a `TradePurchaseOrder` in
  `AWAITING_SUPPLIER_CASH_APPROVAL`/`CONFIRMED`, or an unresolved (`status="OPEN"`) WorkItem for
  `trade_invoice_discrepancy`. Do NOT block on `TradeReservation` (self-resolves via existing
  30-min expiry, no money/documents at stake). Fail-closed on check error
  (`TRADE_CHECK_UNAVAILABLE`), same idiom as the existing dispute/chargeback check.
- **Side effect:** newly blocks deletions that used to silently succeed — notify whoever owns
  self-service deletion UX before shipping; add new error-code→message mappings for
  `TRADE_PO_OPEN`/`TRADE_INVOICE_DISCREPANCY_OPEN`/`TRADE_CHECK_UNAVAILABLE`.

### G-C — File-access reveal-gate + generic errors `[CRITICAL]`
- **File:** `backend/app/modules/files/file_router.py:174,247,277,79`
- Extend the existing D-50 reveal-with-reason mechanism (`_pii_reveal_active`, currently used for
  phone/email) to file downloads for OPS/Compliance/Super roles. Consider extracting to
  `app/core/reveal_gate.py` as a shared module.
- Replace `f"Storage unavailable: {exc}"` / `f"File not found: {exc}"` with fixed generic
  messages + server-side logging.
- Log the swallowed `S3Error` in `_ensure_bucket` instead of silent `pass`.
- **CRITICAL side effect:** any existing OPS UI linking directly to a file download (KYC review
  screen, e-contract viewer) without going through the reveal-first flow breaks with 403 the
  moment this ships — **update every such entry point in the same PR, not after.**

### G-E — CP complaint-count filter bug `[LOW/MEDIUM]` — **DONE** (confirmed 2026-07-27, report 566)
- **File:** `backend/app/modules/admin/control_panel_router.py:1576`
- Fix: derive `count_q` from the same filtered base query as the item list (currently
  `select(func.count()).select_from(Complaint)` with no `.where()`).
- Also add the 9 missing `domains` dict entries so CP Landing's 15 requested domains match what
  the backend actually returns (currently only 6).
- **Confirmed already fully fixed in a prior session** (both the count_q filter parity and all
  15 domain entries, the latter with an explicit code comment citing "G-E") — found already-done
  during report 566's triage, no new code needed. 126 related tests pass.

### G-H — WorkItem case-management fields `[MEDIUM]`
- Additive columns: `assigned_to_user_id`, `severity`, `sla_due_at`, `team`, `version`
  (optimistic-lock). Widen status CHECK to add `IN_PROGRESS`. Resolve endpoint gains
  `WHERE version = expected_version` — stale/conflicting resolve gets clean 409.

### G-N — Audit hash-chain lock hold-time + alerting `[HIGH]`
- Do NOT shard the hash chain (would weaken tamper-evidence — a real trade-off, not a missed
  optimization). Instead: (1) review call sites to ensure `_chain_link` is invoked as late as
  possible before commit, minimizing GLOBAL-row lock hold time; (2) add real alerting to the
  no-session background-write failure path (currently just `logger.warning`, no re-raise, no
  alert — see Wave 0 ticket).

---

## 7. Wave 6 — Trade completion

### H2 — State-machine guardrails `[MEDIUM]`, plus new addendum
- Add `received_quantity <= po_line.quantity` guard on goods-receipt insert (currently allows
  silent overreceipt).
- Link `TradeInvoice` to its resolving `WorkItem` (nullable `work_item_id`) so resolved
  discrepancies stop showing as permanently open — invoice row itself stays immutable.
- **NEW (v2 discovery):** Trade currently has FOUR different state-transition-enforcement shapes:
  `TradeReservation`/`TradeShipment` status ARE enforced (in `reservation_service.py`/
  `po_router.py`); `TradeSupplier`/`TradeProduct` status transitions are comment-documented but
  **not enforced at all**. Build ONE shared module
  `backend/app/modules/trade/state_transitions.py` exposing
  `allowed_transition(entity_type, from_status, to_status) -> bool` (dict-of-dicts), used
  uniformly by all four call sites — mirrors exactly the consolidation pattern
  `backend/app/core/item_access.py` (landed this window, commits `a25dbaf7`/`2f477e2d`) just
  proved out for item-purchasability checks.

### H3 — Buyer UX completion `[LOW]`
- Add a review-before-submit step + per-action busy state (not one shared boolean across
  unrelated actions) to `ProviderTradeBrowse.tsx`. Migrate off raw `useState`/`useEffect` onto the
  existing `@tanstack/react-query` pattern (already used elsewhere, e.g. `useMyPermissions.ts`).
- Add a `change-request` endpoint (cancel + resubmit atomically, `superseded_by_po_id` linkage)
  instead of forcing manual cancel-then-resubmit.
- **`[NEEDS OPERATOR SIGN-OFF: STRING CHANGE]`** for any i18n work here.
- **Context note:** Retail's `CartContext.tsx` was actively touched in the intervening 14 commits
  (F-14a-adjacent) — re-diff against its current shape before designing Trade's review step so
  the two patterns don't diverge on day one.

### H5 — Trade event/notification lifecycle `[MEDIUM]`, depends on H2 (SEQ-13)
- Reuse `EventOutbox` (`backend/app/models/audit.py:119`) exactly as
  dispatch/payments already do — write a row in the same transaction as each state-changing
  handler in `po_router.py`/`purchase_router.py`. No new relay/pub-sub infrastructure.
- **VERIFY BEFORE WIRING:** confirm `outbox_relay.py`'s dispatch logic fails soft on an
  unrecognized `event_type` (not yet independently re-checked as of this spec — read that file
  before implementation, do not assume).
- **MANDATORY staged rollout:** OPS-facing events first (lower volume, clearer value), buyer-
  facing later, default off/digest preference for existing users — total silence since launch
  flipped to instant flood reads as a regression without staging.

### H6 — Governance/observability `[LOW]`
- Fix `/trade/overview` to return real aggregate counts (currently hardcoded
  `{"status": "not_yet_implemented"}` at `trade_router.py:45` despite the module being fully
  built).
- Batch-load PO lines to fix N+1 in `po_service.py:30` (`serialize_purchase_order`).
- Add cursor pagination, orphan-detection job for WorkItem↔TradeInvoice, PostgreSQL lifecycle
  test, Playwright E2E for the full buyer→OPS→receipt→invoice journey.
- **Sequencing note:** `web/src/api/client.ts` was mid-edit in the working tree as of this spec's
  writing (uncommitted, per `git status`) — confirm whether that edit touches list-endpoint
  calling conventions before implementing the pagination change here.

---

## 8. Wave 7 — Mobile + remaining Web/DB hardening

### I3 — Mobile i18n hardening `[MEDIUM]`
- Move all hardcoded Arabic strings out of `checkout.tsx` (lines 39-44, 187, 191, 195, 208, 216,
  226, 236, 269, 301, 311, 314, 320, 324, 326, 333-336, 345, 356, 399, 401, 413) into `ar.ts`/
  `en.ts` (+ `fr.ts` if in scope — currently only `ar`/`en` exist, no French locale
  infrastructure on mobile at all).
- Add `messageEn` mirror to mobile's `ApiError`/`normalizeError` (currently structurally
  Arabic-only — confirmed this exact bug class has now been fixed on web **twice**, at `6bbeaad`
  and again this window at `8d17e240`/`a1c62b1e`, with zero equivalent fix reaching mobile either
  time — strengthens priority, does not change scope).
- **NEW (v2 discovery) — add a mobile i18n-missing-key ratchet**, mirroring web's
  `i18n-missing-keys.test.ts` + `scan-missing-keys.ts` (commit `d4366810`) 1:1: a regex scan over
  `apps/mobile/src/**/*.{ts,tsx}` for un-i18n'd literals, diffed against
  `flattenKeys(ar) ∪ flattenKeys(en)`, snapshotted so today's backlog is grandfathered but no new
  raw string can land going forward. Mobile currently has **zero** such regression guard.
- **`[NEEDS OPERATOR SIGN-OFF: STRING CHANGE]`**

### I4-full — Mobile outbox hardening `[HIGH]`, ships with I5 (SEQ-18)
- **File:** `apps/mobile/src/services/outbox.ts` (141 lines)
- Add `queuedAtIso` + 2h TTL; on replay, expired entries → new `EXPIRED` disposition instead of
  silent stale-price send.
- Before replay: call the already-live, read-only `POST /carts/quote` for revalidation; if
  `unavailable_items` non-empty, quarantine as `QUARANTINED_STALE`.
- Cap queue at 10 entries; reject enqueue beyond cap with explicit error (new "outbox full" UI
  branch needed in checkout.tsx's offline-save path).
- Add `attempts` counter + exponential backoff (`min(2^attempts*30s, 30min)`).
- **New consolidated review screen required** (not raw `Alert.alert`) — quarantined entries need
  a "Confirm & send at current prices" / "Discard" UI, or the fix just becomes silent order-loss
  from the user's perspective.

### I5 — Legacy outbox ownership quarantine `[MEDIUM]`, ships with I4
- **File:** `outbox.ts:63-88` (`read()`'s migration branch, line 71)
- Replace auto-attribution to "whoever is currently signed in" with a `LEGACY_UNKNOWN_OWNER`
  sentinel forcing `QUARANTINED_OWNERSHIP` status + explicit user confirmation.

### J1+J5 — OpenAPI coverage + API hygiene `[MEDIUM]`, execute together (SEQ-19)
- **CORRECTED counts (v2):** `pos_router.py` has **23** raw `HTTPException` sites (not 8 as
  originally miscounted — grounding read stopped partway through the file). Buried-import count
  in `operations_router.py` is **331** (not 336). Pagination has **≥9 distinct (default, cap)
  pairs** (not 6) — `le=1000`(599), `le=200`(758,7465,7777,8155,8474), `le=500`(3945),
  `le=50`(6162), `le=90`(598), `le=86400`(645), `le=365`(6380).
- Per-module incremental OpenAPI documentation (20-30 PRs, risk×traffic ordered, `pos_router.py`
  first since it overlaps F9's error-envelope work — implement the envelope once, in one PR that
  does both).
- Shared `pagination.py` constants module, migrated opportunistically (never a blanket
  cap-lowering sweep).
- **New technique (v2):** the RATCHET-5 committed-snapshot pattern (simpler than the existing
  git-worktree/BASE-diff pattern used by `openapi-ratchet`/`api-contract-ratchet`) is a good fit
  for tracking this burn-down — see Wave-0's A10-partial ticket.

### J3 — Fenced background-loop locks `[HIGH]`
- **File:** `backend/app/main.py` (27 `asyncio.create_task` loops, confirmed still 27 as of
  `d9c5d38d`)
- New `sweep_lock.py`: `acquire_fenced` returns a unique token; `release_fenced` does a Lua
  compare-and-delete; `still_holds()` checked immediately before each loop's DB commit.
- New `sweep_heartbeat.py`: pings a Redis key per iteration; `/ready` gets a new **non-gating**
  `warnings` array surfacing stale heartbeats.
- **Confirmed safe as single-phase** because the platform runs one instance per environment today
  (SEQ-22) — if it ever goes multi-replica, split acquire/release into two releases.

### J7 — OTel metrics, CORRECTED premise `[LOW]`
- **CORRECTION (v2):** `otel-collector-config.yaml` already has a `metrics:` pipeline configured
  (receivers: `[otlp]`) — the plan's original contingency ("verify the collector isn't
  trace-only") is moot, it already isn't. **The actual gap:** every pipeline's exporter is
  `[debug]` only — metrics are received/logged but never shipped to a durable backend
  (Prometheus remote-write etc.) for dashboards/alerting. This is a smaller, different gap
  (export target, not receiver capability) and may be in the deploy/infra-config lane already
  flagged as out of scope.
- Add label dimensions (tenant/feature/correlation_id) to the 2 existing custom counters
  (`rule_engine.py`, `transfer_metrics.py`); inject `trace_id` into structured logs.

### J9 — Naming/style cleanup `[LOW]`
- `/health/ready`'s `"product": "Libya Delivery Platform"` → `"Tawa"`.
  **`[NEEDS OPERATOR SIGN-OFF: STRING CHANGE]`**
- Convert `CheckoutPage.tsx`'s confirmed **110** inline `style={{}}` usages to shared components,
  section-by-section, E2E smoke-test after each section (SEQ-29 discipline).
- Mechanical `LoadingSpinner` swap for the 5 provider pages currently missing it (confirmed:
  12 of 33 provider pages use it today).

### Remaining Wave-7 tickets (lower priority, no new corrections)
C1, C2 (idempotency hash fields, ship together per SEQ-04), C3 (promo hard-fail — **CORRECTED
STATUS**: the general `CODE_NOT_FOUND`/expired/ineligible-code silent-fail-open at
`order_service.py:1358-1368` is still 100% unfixed; only a narrow customer-per-limit
preview/actual sub-case was closed by commit `a46f82b3` — do not treat C3 as done), C5, C6 (defer
structural split, SEQ-29), C8, C10 (after C14, SEQ-06), C12, C13, C16, D4/D5/D6/D7/D8/D9 (D6: a
frontend SSE-reconnect fix landed adjacent to this — `OpsFleetMap.tsx` — but does NOT add the
missing server-side audit-log call this ticket requires), B3, B5 (operator decision, SEQ-30), B8
(excludes D-60, SEQ-10), B9 (operator decision for the OTP flip half), B11, B12, B13, B14, B15
(operator decision for dual-control half, SEQ-11), E2 (docs-only, no code change — deliberate
design), E9, E10, E11, E13, E14, G-D, G-F, G-J, G-K, G-L, G-M, G-O, J6 (per-column decision, no
single template — see SEQ-20/21), J10.

Full technical detail for each of these (exact current-code quotes, multi-scenario run-time
simulations, side-effect analysis) is preserved in report 559 (`559-الخطة-التنفيذية-...html`) in
the same folder — this build spec supersedes only the specific corrections listed per-ticket
above; everything else in 559 stands as originally designed and was independently re-verified
against `d9c5d38d` with zero drift.

---

## 9. Group-Order / Bundled-Cart — separate track, NOT gated behind the waves above

**Framing:** two unrelated flags share the word "group."
- `group_order_enabled` — one customer, multiple providers, one atomic multi-sub-order checkout.
  Substantially built, well-tested (20+ tests). **Verdict: NOT_READY_MAJOR_GAPS.**
- `group_order_participant_split` — true multi-person bill-splitting. **Not built at all** — no
  data model, no caller ever invokes its two pure stub functions. **Verdict:
  NOT_READY_ARCHITECTURAL_REDESIGN_NEEDED** — do not schedule as a hardening pass; this is
  ground-up feature design.

### GC-1 — Fix under-hashed group idempotency key `[CRITICAL, blocks unseal]` — **DONE** (commit `6ba22499`, confirmed 2026-07-27, report 566)
- **File:** `backend/app/modules/carts/service.py:146-178` (`_hash_sub_orders`)
- **Current:** hashes only `provider_id`, `payment_method`, per-item catalog/variant/qty/modifier
  fields.
- **Target:** match `orders/idempotency.py:21-83`'s `compute_request_hash()` field list exactly —
  add `address_id`, `fulfillment_type`, `scheduled_for`, `tenders`, `promo_code`. **Better:** call
  `compute_request_hash` directly instead of maintaining a second, independently-drifting hash
  implementation.
- **Confirmed still present unmodified** as of `d9c5d38d` — the intervening 14 commits' cart
  changes were purely additive (`_unavailable_cart_items()` on the `/carts/quote` path) and never
  touched this function.
- **Concrete failure without this fix:** a `checkout-group` request that commits server-side but
  loses its response to the client (realistic on unreliable mobile networks), followed by the
  customer changing address/tender split and resubmitting with the same key, silently returns the
  OLD group with OLD values — no error, no signal.

### GC-2 — Add group-conflict/hash-mismatch test coverage `[HIGH]` — **DONE** (commit `6ba22499`)
- Zero tests exist for `GroupIdempotencyConflictError` or a same-key/different-value replay.
- **SEQ-27: implement after GC-1**, so tests prove the fixed behavior. The new
  `test_promo_per_customer_limit_preview_and_actual_parity` (added this window, unrelated to
  group-checkout) is a usable *structural template* for a preview/actual-parity-style test here.
- **Confirmed done**: `tests/test_r516_cart_foundation.py` covers
  `test_fix6_reused_key_different_payload_conflicts` plus 4 dedicated
  `test_gc1_{address,tenders,promo_code,scheduled_for}_change_same_key_conflicts` tests. 10/10 pass.

### GC-3 — Add true-concurrency double-submit test `[MEDIUM]` — **DONE** (commit `6ba22499`)
- `carts/router.py:186-205`'s `IntegrityError` handler (unique constraint collision) is
  comment-documented only, no automated test simulating two simultaneous requests with the same
  idempotency key.
- **Confirmed done**: `tests/test_gc3_group_checkout_integrity_race.py` exists, is in
  `scripts/run_pg_tests.sh`'s curated real-Postgres list, and drives genuine concurrent requests.

### GC-4 — Promo silent-fail-open in group checkout `[LOW, latent]`
- Backend behavior (`order_service.py:1323-1327`, applies per-sub-order via
  `carts/service.py:368`) is real but currently unreachable — `CartGroupCheckoutPage.tsx` has no
  promo-code input UI at all (confirmed via full-file read + grep). **Not a blocker to unseal,
  but must be fixed before anyone adds a promo field to this page** — flag explicitly in any
  future PR that does so.

### No action needed (verified correct, do not touch)
- `distributeGroupTenders` (`web/src/api/carts.ts:91-118`) — integer waterfall, deterministic
  provider-sorted order, no division — mathematically sound, confirmed unchanged in this window.
- Atomicity, live-state derivation, DB-level kill switch — all independently verified correct by
  direct code read.

---

## 10. Appendix — Modern techniques reference (apply where cited above)

**A. Committed-snapshot i18n ratchet** (source: `web/src/test/i18n/i18n-missing-keys.test.ts` +
`scan-missing-keys.ts`, commit `d4366810`). Simpler than git-worktree/BASE-diff ratchets
(`openapi-ratchet`, `api-contract-ratchet`) — one committed JSON snapshot, "can only shrink,"
runs identically locally and in CI. **Apply to:** A7(d)/A10(a) OpenAPI/baseline-failure burn-down
trackers, F5 push-copy locale completeness, I3 mobile hardcoded-string ratchet (mobile currently
has zero equivalent).

**B. "Unwired shared module, then migrate callers" two-commit pattern** (source:
`backend/app/core/item_access.py`, commits `a25dbaf7` + `2f477e2d`). Land a new shared
predicate/module in one commit with zero callers wired to it yet; migrate call sites onto it in a
follow-up commit. Proven low-risk in this exact codebase this week. **Apply to:** C1/C2/C5/C15's
proposed shared idempotency-signature / pricing-preview package (web+mobile), H2's proposed
`state_transitions.py` for Trade.

**C. `require_not_self_approval` as the single maker-checker primitive** (already exists in
`authorization_helpers.py`, used by refund/chargeback/manual-journal). Do not build a second
implementation (as `ads_service.py` currently does) — consolidate onto this one. **Apply to:**
G-G's full toxic-combination coverage expansion.

---

*Build spec generated from reports 558→559→560. Every file:line citation in this document was
independently re-verified against commit `d9c5d38d` by 11 parallel forensic passes on 2026-07-25.
Re-verify against current HEAD before executing if further commits have landed — this repo has an
active parallel workstream, confirmed twice during this exercise.*
