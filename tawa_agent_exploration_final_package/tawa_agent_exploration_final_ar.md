# Tawa — برومبت الاستكشاف التشغيلي الحقيقي بالـAgents

> استخدم هذا البرومبت على مراحل. هذه مرحلة استكشاف وقياس وتدقيق فقط، وليست مرحلة إصلاح أو بناء ميزات جديدة.

---

## المرحلة 0 — دستور الاستكشاف العام

```text
You are Claude Code working inside the Tawa Platform repository.

MISSION:
Execute a REAL local agent-driven operational exploration and measurement audit.

This is an EXPLORATION phase, not a repair/build phase.
Do not implement product features.
Do not repair product defects unless the operator explicitly authorizes a separate repair phase.
You may create isolated audit/test harness files only under approved paths.

This is not a mock test.
This is not API-only testing.
This is not unit testing.
This is not a theoretical review.
This is not component-only frontend testing.
This is not production deployment.

The goal is to run the platform locally as a real working system, create automated agents that behave like real humans and platform operators, execute large-scale real UI/API scenarios, measure behavior precisely, and produce evidence-backed findings.

The agents must use:
- real local frontend through Playwright wherever a UI exists,
- real local backend HTTP APIs where no UI exists or for concurrency/race/load tests,
- real local PostgreSQL if available,
- real local Redis if used,
- real docker-compose/local stack if available.

STRICT RULES:

1. Exploration-only rule:
   This phase discovers, measures, and reports. It does not fix product logic.

2. Real browser first:
   Any feature with a visible UI must be tested through Playwright in a real browser.

3. No mock replacement:
   Do not use MSW, fake API handlers, mocked network responses, page.route().fulfill, hardcoded successful responses, or component-only tests as proof of real behavior.

4. No direct backend service calls:
   During scenarios, do not call backend service classes or internal functions directly.
   User actions must go through real UI or real HTTP API.

5. Database usage:
   DB writes are allowed only for deterministic seed/setup.
   DB reads are allowed for invariant verification and measurement.
   DB writes must not perform user actions.

6. Real login:
   Agents must authenticate through real login UI or real auth API.
   Do not inject tokens into localStorage/sessionStorage for real scenarios.

7. Real local infrastructure:
   Use PostgreSQL for real-operation scenarios if supported.
   SQLite-only does not qualify for real-operation verdict.

8. Evidence-first:
   Every PASS, FAIL, warning, defect, gap, governance finding, financial finding, and control-panel finding must have evidence:
   - screenshot,
   - Playwright trace,
   - video where available,
   - HAR/network transcript,
   - console logs,
   - page errors,
   - API request/response transcript,
   - DB invariant output where relevant,
   - scenario ID,
   - deterministic seed.

9. Scale targets:
   Full target: 20,000 scenarios.
   Deep audit target: 10,000 scenarios.
   Minimum broad audit: 5,000 scenarios.
   Minimum partial audit: 2,000 scenarios.
   Anything below 2,000 is PARTIAL_WITH_BLOCKERS.
   At least 70% of completed scenarios must be real browser UI scenarios unless resource limits are explicitly documented.

10. Overload/stress target:
   Include controlled overload batches to test platform behavior under heavy local load:
   - many customers browsing/checking out,
   - many providers receiving orders,
   - many staff actions,
   - many drivers competing for offers,
   - many ops/control-panel views,
   - many finance/settlement calculations,
   - many notifications/events.

11. Sharded execution:
   Do not run everything as one giant run.
   Build runners that support:
   - --total-scenarios
   - --batch-size
   - --seed
   - --role
   - --provider-category
   - --scenario-group
   - --ui-ratio
   - --load-profile
   - --resume
   - --shard-index
   - --shard-count
   - --max-failures
   - --stop-on-p0
   - --dry-run
   - --list-scenarios

12. FIN-COR protection:
   Do not enable wallet_cash_out.
   Do not enable production_payment_enabled.
   Do not alter sealed flags.
   Do not call real payment providers.
   Do not use real SMS.
   Do not use real customer data.

13. Governance awareness:
   Discover all sealed features, feature flags, launch gates, mandatory document gates, approval gates, role gates, onboarding gates, and finance/payment gates.
   Report what is locked by governance and whether UI/API correctly enforce it.

14. Control-room visibility:
   Every important business event should be visible somewhere appropriate:
   - order created,
   - provider accepted/rejected,
   - preparation started,
   - ready for pickup,
   - dispatch offer sent,
   - driver accepted/rejected,
   - pickup confirmed,
   - delivery completed/failed,
   - complaint opened,
   - refund/finance event,
   - cash exception,
   - document uploaded/rejected/approved,
   - provider/driver status change,
   - wallet/settlement/ledger event,
   - suspicious or failed action.
   If no control-room/control-panel visibility exists, report it as a gap.

15. Harness isolation:
   New files may be created only under:
   - tools/agent_scenarios/
   - web/e2e/agent-real-operation/
   - test-results/agent-real-operation/
   - app_report/

16. No unsupported verdict:
   No final positive verdict is allowed until verifier passes.
   Never claim production-ready.
   Never claim soft-launch candidate while blocking launch gates are open.
   Never claim full E2E if browser lifecycle evidence is missing.
   Never claim complete platform readiness while mobile/native status is unresolved.

FINAL REPORT:
app_report/450_AGENT_REAL_OPERATION_EXPLORATION_REPORT.md

Allowed final verdicts only:
- AGENT_REAL_OPERATION_BOOT_FAILED
- AGENT_REAL_OPERATION_NO_MOCK_PROOF_FAILED
- AGENT_REAL_OPERATION_PARTIAL_WITH_BLOCKERS
- AGENT_REAL_OPERATION_COMPLETED_WITH_CRITICAL_DEFECTS
- AGENT_REAL_OPERATION_COMPLETED_WITH_MAJOR_DEFECTS
- AGENT_REAL_OPERATION_COMPLETED_INTERNAL_PILOT_CANDIDATE
- AGENT_REAL_OPERATION_SOFT_LAUNCH_CANDIDATE
```

---

## المرحلة 1 — اكتشاف المشروع وتشغيله محلياً

```text
PHASE 1 — DISCOVERY AND REAL LOCAL BOOT

Goal:
Discover repository reality, identify run commands, boot the real local stack, and prove backend/frontend/database/Redis are actually running.

Run and capture:

pwd
git status --short
git rev-parse HEAD
git branch --show-current
git log --oneline -30
docker --version || true
docker compose version || true
node --version || true
npm --version || true
python --version || true

Inspect if present:

README.md
docker-compose.yml
.github/workflows/ci.yml
.github/workflows/tawa-smoke-run.yml
contracts/feature_flags.yaml
contracts/launch_gates.yaml
contracts/openapi.yaml
backend/app/config.py
backend/app/main.py
backend/entrypoint.sh
backend/alembic.ini
backend/alembic/versions/
web/package.json
web/playwright.config.ts
web/playwright.container.config.ts
web/playwright.staging.config.ts
mobile/FLUTTER_PLACEHOLDER.md

Boot the real local stack.

First attempt if valid:

docker compose --profile dev-minimal up --build

If another command is more correct, use it and document why.

Required boot evidence:
- docker compose config
- docker compose ps
- backend health response
- frontend HTTP response
- PostgreSQL readiness
- Redis readiness if Redis exists
- Alembic current revision and heads if available
- backend logs first 300 lines
- frontend logs first 300 lines
- first real browser screenshot of frontend root
- network response for first frontend load
- browser console log from first frontend load

Create:
- app_report/450_AGENT_REAL_OPERATION_DISCOVERY.md
- app_report/450_BOOT_EVIDENCE.md
- test-results/agent-real-operation/boot/

If boot fails, stop and write:
- app_report/450_BOOT_FAILURE_REPORT.md

Phase verdict:
- PHASE_1_BOOT_PASS
- PHASE_1_BOOT_FAIL
```

---

## المرحلة 2 — عالم Seed واسع مع حمل زائد

```text
PHASE 2 — LARGE REALISTIC SEED WORLD + OVERLOAD DATASET

Goal:
Create a deterministic large local seed world that supports broad exploration, control-panel review, financial checking, mandatory document gates, governance gates, and overload/stress scenarios.

Create:
- tools/agent_scenarios/seed_agent_world.py
- app_report/450_SEED_MANIFEST.json
- app_report/450_PROVIDER_CATEGORY_MATRIX.md
- app_report/450_MANDATORY_DOCUMENT_SEED_MATRIX.md
- app_report/450_LOAD_PROFILE_MATRIX.md

Rules:
- Idempotent.
- Local data only.
- No production data.
- No real phone numbers.
- No real SMS.
- No real payment provider.
- DB writes allowed only for seed setup.
- All later user actions must go through UI/API.
- Seed script must print created/reused counts.

Seed volume:

Customers:
- 1,000 customers total.
- 150 customers with no address.
- 250 customers with one saved address.
- 300 customers with multiple addresses.
- 100 customers with wallet balance.
- 100 customers with low/no wallet balance.
- 50 customers outside delivery radius.
- 50 customers near high-volume providers.
- Arabic names, English/French-like names, long addresses, invalid-edge addresses.

Drivers:
- 300 drivers total.
- verified active drivers.
- pending verification drivers.
- suspended drivers.
- offline drivers.
- near-provider drivers.
- far-provider drivers.
- drivers with active offer.
- drivers with expired offer.
- drivers with settlement history.
- drivers with failed delivery history if supported.

Providers:
- 150 providers total.
- At least 18 provider categories.
- At least 5 active providers per major category.
- Each category must include at least one:
  - active provider,
  - closed provider,
  - paused provider,
  - missing coordinates provider,
  - provider with active staff,
  - provider with suspended staff,
  - provider with off-shift staff,
  - provider with unavailable items,
  - provider with low-stock items,
  - provider with order-ready items.

Provider categories:
1. Restaurant
2. Cafe
3. Bakery
4. Pharmacy
5. Grocery / Mini-market
6. Supermarket
7. Water delivery
8. Gas/cylinder placeholder if supported
9. Butcher
10. Fish/seafood
11. Flowers/gifts
12. Electronics/accessories
13. Clothing
14. Cosmetics/perfume
15. Stationery/books
16. Home kitchen governed by feature flag
17. Donation/campaign governed by feature flag
18. Errand/service-like provider if supported

Provider staff:
- 600 provider staff.
- owner/admin per provider.
- full-permission staff.
- view-only staff.
- order-prep staff.
- catalog-only staff.
- staff without handoff permission.
- suspended staff.
- off-shift staff.
- staff assigned to wrong-provider boundary cases where safe.

Platform users:
- 75 ops/admin/control users:
  - super_admin
  - platform_admin
  - ops_manager
  - ops_supervisor
  - ops_operator
  - ops_viewer
  - finance_manager
  - settlement_officer
  - cash_collection_officer
  - document_reviewer
  - support_agent
  - complaint_handler
  - read_only_auditor
  - restricted_ops_user

Finance/settlement seed:
- local wallet balances.
- provider ledger data.
- driver settlement data.
- commission configuration if local.
- cash collection exceptions.
- refund-like local test data if supported.
- provider payout states.
- driver payout states.
- wallet transfer states according to feature flags.
- never enable sealed features.

Mandatory documents and verification gates:

Seed document states for providers and drivers.

Provider document states:
- no documents uploaded.
- all documents pending review.
- one required document missing.
- one required document rejected.
- all required documents approved.
- expired document if expiry is supported.
- document uploaded by wrong provider boundary case if safe.

Driver document states:
- no documents uploaded.
- license pending.
- license rejected.
- identity document pending.
- all documents approved.
- expired license if expiry is supported.
- suspended driver with approved documents.

For each mandatory document state, create accounts that test:
- onboarding blocked/allowed.
- provider can/cannot open store.
- provider can/cannot receive orders.
- driver can/cannot go online.
- driver can/cannot receive dispatch offers.
- ops can approve/reject.
- rejection reason visible.
- audit log written if implemented.
- notification generated if implemented.

Load profiles:
Create seed profiles for:
- normal load.
- peak lunch/dinner load.
- driver scarcity.
- provider overload.
- high complaint rate.
- high cancellation rate.
- many checkout attempts.
- many control-panel viewers.
- many finance events.
- many notification events.

Output manifest must include:
- counts per entity.
- counts per provider category.
- counts per user role.
- counts per document state.
- counts per finance state.
- accounts and credentials for test agents.
- feature flags snapshot.
- launch gates snapshot.
- seed commit SHA.
- seed timestamp.
- DB connection used.

Phase verdict:
- PHASE_2_SEED_WORLD_PASS
- PHASE_2_SEED_WORLD_PARTIAL
- PHASE_2_SEED_WORLD_FAIL
```

---

## المرحلة 3 — إطار Agents ومولد السيناريوهات

```text
PHASE 3 — AGENT FRAMEWORK AND SCENARIO FACTORY

Goal:
Build a reusable deterministic agent framework that can generate thousands of real-operation scenarios across roles, provider categories, UI routes, API endpoints, governance gates, finance rules, and load profiles.

Create:
- tools/agent_scenarios/AGENT_MODEL.md
- tools/agent_scenarios/agent_matrix.json
- tools/agent_scenarios/scenario_catalog.json
- tools/agent_scenarios/scenario_factory.py
- tools/agent_scenarios/run_agent_batch.py
- tools/agent_scenarios/api_agent_client.py
- tools/agent_scenarios/finance_oracle.py
- tools/agent_scenarios/control_room_oracle.py
- tools/agent_scenarios/governance_gate_oracle.py
- web/e2e/agent-real-operation/agent-fixtures.ts
- web/e2e/agent-real-operation/agent-evidence-writer.ts
- web/e2e/agent-real-operation/agent-network-recorder.ts
- web/e2e/agent-real-operation/agent-console-recorder.ts
- web/e2e/agent-real-operation/agent-db-invariants.ts

Required agents:

Customer agents:
1. CustomerAgent
2. CustomerPowerUserAgent
3. CustomerComplaintAgent
4. CustomerWalletAgent
5. CustomerCancellationAgent

Provider agents:
6. ProviderOwnerAgent
7. ProviderAdminAgent
8. ProviderCatalogManagerAgent
9. ProviderOrderManagerAgent
10. ProviderStaffAgent
11. ProviderSuspendedStaffAgent
12. ProviderOffShiftStaffAgent

Driver agents:
13. DriverAgent
14. DriverUnavailableAgent
15. DriverSettlementAgent
16. DriverFailedDeliveryAgent

Ops/control agents:
17. OpsViewerAgent
18. OpsOperatorAgent
19. OpsSupervisorAgent
20. AdminAgent
21. SuperAdminAgent
22. PlatformAdminAgent
23. ReadOnlyAuditorAgent
24. ComplaintSupportAgent
25. DocumentReviewAgent

Control panel specialist agents:
26. ControlPanelNavigationAgent
27. ControlPanelButtonAgent
28. ControlPanelPermissionAgent
29. ControlPanelDataGridAgent
30. ControlPanelUXStructureAgent
31. ControlPanelIncidentAgent
32. ControlPanelIntegrationAgent
33. ControlPanelCompletenessAgent

Finance specialist agents:
34. FinanceLedgerAgent
35. FinanceArithmeticAgent
36. FinanceCommissionAgent
37. FinanceSettlementAgent
38. FinanceWalletAgent
39. FinanceCashCollectionAgent
40. FinanceReconciliationAgent
41. FinanceProfitLossAgent
42. FinanceControlCompletenessAgent
43. FinanceGovernanceAgent

System/gate agents:
44. SystemHealthAgent
45. FeatureFlagGateAgent
46. LaunchGateAgent
47. MandatoryDocumentGateAgent
48. AccessibilityAgent
49. RTLAndLocaleAgent
50. MobileViewportAgent
51. PerformanceSmokeAgent
52. DataConsistencyAgent
53. ControlRoomObservabilityAgent

For each agent define:
- role.
- account.
- credentials.
- provider/category link if applicable.
- UI routes.
- API endpoints.
- allowed actions.
- forbidden actions.
- expected permission failures.
- scenario groups.
- evidence requirements.
- DB invariants.
- control-panel visibility expectations if relevant.
- finance expectations if relevant.

Scenario factory must combine:
- role
- provider category
- user permission
- locale
- viewport
- order type
- fulfillment type
- payment/wallet condition
- stock condition
- delivery distance
- provider status
- driver status
- staff status
- document state
- launch/feature flag status
- finance condition
- load profile
- happy path / negative path / boundary / race / recovery / overload

It must support:
--total-scenarios
--batch-size
--seed
--role
--provider-category
--scenario-group
--ui-ratio
--load-profile
--resume
--shard-index
--shard-count
--max-failures
--stop-on-p0
--dry-run
--list-scenarios

Target scenario volume:
- Full target: 20,000 scenarios.
- Deep audit target: 10,000 scenarios.
- Minimum broad audit: 5,000 scenarios.
- Minimum partial audit: 2,000 scenarios.

Do not manually write static scenarios at full volume. Build deterministic generation.

Phase verdict:
- PHASE_3_AGENT_FRAMEWORK_PASS
- PHASE_3_AGENT_FRAMEWORK_PARTIAL
- PHASE_3_AGENT_FRAMEWORK_FAIL
```

---

## المرحلة 4 — اختبارات الواجهة الحقيقية

```text
PHASE 4 — REAL BROWSER UI TEST HARNESS

Goal:
Create Playwright real browser tests for actual UI flows. No backend mocking is allowed.

Create:
- web/e2e/agent-real-operation/agent-real-operation.spec.ts
- web/e2e/agent-real-operation/customer-real-ui.spec.ts
- web/e2e/agent-real-operation/provider-real-ui.spec.ts
- web/e2e/agent-real-operation/provider-staff-real-ui.spec.ts
- web/e2e/agent-real-operation/driver-real-ui.spec.ts
- web/e2e/agent-real-operation/ops-admin-real-ui.spec.ts
- web/e2e/agent-real-operation/finance-settlement-real-ui.spec.ts
- web/e2e/agent-real-operation/control-panel-real-ui.spec.ts
- web/e2e/agent-real-operation/document-gates-real-ui.spec.ts
- web/e2e/agent-real-operation/governance-gates-real-ui.spec.ts
- web/e2e/agent-real-operation/accessibility-rtl-responsive.spec.ts

Playwright requirements:
- real frontend URL.
- real backend URL.
- no MSW.
- no page.route().fulfill.
- no fake response injection.
- no localStorage token injection for real scenarios.
- screenshots.
- traces.
- videos on failure.
- HAR/network capture.
- console error capture.
- page error capture.
- scenario_id in every test title.
- role in every test title.
- provider category in every relevant title.

Add package script if needed:
"test:e2e:agents": "playwright test web/e2e/agent-real-operation"

Required UI scenario minimums for deep run:
- Customer UI: 1,500+
- Provider owner/admin UI: 1,200+
- Provider staff UI: 1,000+
- Driver UI: 1,000+
- Ops/Admin UI: 1,200+
- Finance/Settlement UI: 1,000+
- Control Panel UI: 2,500+
- Cross-role lifecycle: 1,000+
- Accessibility/RTL/responsive: 500+
- Mandatory document gates: 500+
- Governance locked features: 500+

Critical UI flows:
- customer creates delivery order.
- provider accepts.
- provider starts preparation.
- provider marks ready.
- driver receives/accepts offer.
- driver picks up.
- driver delivers.
- customer sees delivered.
- self-pickup order lifecycle.
- provider staff self-claim lifecycle.
- provider missing coordinates path.
- out-of-stock/unavailable item path.
- cancellation path.
- complaint path.
- document approval/rejection path.
- finance/settlement visibility path.
- control-room visibility path.

Phase verdict:
- PHASE_4_REAL_UI_HARNESS_PASS
- PHASE_4_REAL_UI_HARNESS_PARTIAL
- PHASE_4_REAL_UI_HARNESS_FAIL
```

---

## المرحلة 5 — تدقيق لوحة التحكم وغرفة التحكم

```text
PHASE 5 — CONTROL PANEL AND CONTROL ROOM SPECIALIST AUDIT

Goal:
Deeply audit whether the control panel/control room can actually monitor and control everything important in the platform.

Create:
- tools/agent_scenarios/control_panel_audit_matrix.json
- tools/agent_scenarios/control_room_event_matrix.json
- web/e2e/agent-real-operation/control-panel-audit.spec.ts
- app_report/450_CONTROL_PANEL_AUDIT_REPORT.md
- app_report/450_CONTROL_ROOM_OBSERVABILITY_REPORT.md

Specialist agents:
1. ControlPanelNavigationAgent
2. ControlPanelButtonAgent
3. ControlPanelPermissionAgent
4. ControlPanelDataGridAgent
5. ControlPanelFinanceAgent
6. ControlPanelOpsCompletenessAgent
7. ControlPanelUXStructureAgent
8. ControlPanelIncidentAgent
9. ControlPanelIntegrationAgent
10. ControlRoomObservabilityAgent
11. ControlPanelDocumentGateAgent
12. ControlPanelGovernanceGateAgent

Minimum control panel/control room scenarios:
- 2,500 control-panel-specific scenarios.
- at least 70% through real browser.
- all visible buttons on key pages clicked at least once.
- all critical routes opened for each relevant role.
- all restricted roles tested against privileged routes.
- all major business events checked for visibility.
- all finance events checked for visibility.
- all document events checked for visibility.
- all governance-locked features checked for correct exposure/blocking.

Control panel report must answer:
1. Can ops fully control the platform from control panel?
2. Which operational tasks are impossible from UI?
3. Which buttons do nothing?
4. Which pages call missing/broken APIs?
5. Which pages are duplicated?
6. Which pages should be merged?
7. Which pages are confusing?
8. Which dangerous actions lack confirmation?
9. Which critical data is missing?
10. Which permissions are too broad?
11. Which permissions are too restrictive?
12. Which workflows require too many clicks?
13. Which backend features are not exposed in control panel?
14. Which controls are present but incomplete?
15. Which platform events are not visible in control room?
16. Which finance events are not visible?
17. Which document/onboarding events are not visible?
18. Which control panel areas block internal pilot?
19. Which areas block soft launch?

Mandatory document control panel audit:
- provider document page.
- driver document page.
- missing required docs.
- pending docs.
- rejected docs with reason.
- approved docs.
- expired docs if supported.
- verify operational status changes.
- verify audit log.
- verify notification if implemented.
- verify UI/backend agreement.

Phase verdict:
- PHASE_5_CONTROL_PANEL_AUDIT_PASS
- PHASE_5_CONTROL_PANEL_AUDIT_WITH_DEFECTS
- PHASE_5_CONTROL_PANEL_AUDIT_FAIL
```

---

## المرحلة 6 — المالية والحسابات الدقيقة

```text
PHASE 6 — FINANCIAL ARITHMETIC, LOGIC, AND CONTROL AUDIT

Goal:
Audit every financial behavior with strict arithmetic correctness, logical consistency, governance enforcement, and control-panel visibility.

Create:
- tools/agent_scenarios/finance_oracle.py
- tools/agent_scenarios/finance_reconciliation.py
- tools/agent_scenarios/finance_invariants.py
- app_report/450_FINANCIAL_ARITHMETIC_REPORT.md
- app_report/450_FINANCIAL_CONTROL_REPORT.md
- app_report/450_FINANCIAL_GOVERNANCE_REPORT.md

Rules:
- Use Decimal or integer minor units only.
- Do not use floating point for money.
- Record currency assumptions.
- Record rounding policy discovered from code.
- If rounding policy is unclear, report it as a defect/gap.
- Never enable sealed financial features.
- Never call real payment providers.

Finance agents:
1. FinanceArithmeticAgent
   - verifies subtotal.
   - discounts.
   - delivery fee.
   - service fee.
   - commission.
   - taxes if present.
   - wallet debit/credit.
   - refunds.
   - settlement totals.
   - rounding.

2. FinanceLedgerAgent
   - verifies ledger entries.
   - debit/credit balance.
   - no duplicate ledger movement.
   - idempotency.
   - event source consistency.

3. FinanceCommissionAgent
   - provider commission.
   - category-specific commission if present.
   - item-level commission if present.
   - delivery commission if present.

4. FinanceSettlementAgent
   - provider settlement.
   - driver settlement.
   - payout status.
   - cash collection.
   - exceptions.

5. FinanceWalletAgent
   - wallet transfers.
   - wallet locks.
   - idempotency.
   - insufficient balance.
   - threshold/step-up if implemented.
   - sealed cash-out blocked.

6. FinanceProfitLossAgent
   - platform revenue.
   - provider due.
   - driver due.
   - discounts cost owner if represented.
   - refund impact.
   - loss-making orders detection if applicable.

7. FinanceControlCompletenessAgent
   - verifies whether control panel exposes all finance events.
   - verifies whether ops/finance can understand every money movement.
   - flags missing dashboards/panels/actions.

8. FinanceGovernanceAgent
   - production_payment_enabled false must block production payments.
   - wallet_cash_out false must block cash-out.
   - peer transfer governance must be documented.
   - payment sandbox gate must be reflected.
   - legal/payment launch gates must be reported.

Required finance checks:
- subtotal = sum(items).
- total = subtotal + fees - discounts + taxes where applicable.
- ledger balances.
- provider settlement equals eligible delivered order components.
- driver settlement equals eligible delivery components.
- platform commission equals configured rate.
- refunds reverse correct components.
- duplicate idempotency does not duplicate money movement.
- failed order does not create incorrect payable.
- cancelled order has correct financial impact.
- cash collection exception is visible.
- all finance events visible to finance/control panel.
- no sealed feature is exposed as active.

Phase verdict:
- PHASE_6_FINANCE_AUDIT_PASS
- PHASE_6_FINANCE_AUDIT_WITH_DEFECTS
- PHASE_6_FINANCE_AUDIT_FAIL
```

---

## المرحلة 7 — تشغيل السيناريوهات والحمل الزائد

```text
PHASE 7 — SHARDED REAL OPERATION + OVERLOAD EXECUTION

Goal:
Run thousands of scenarios in controlled batches, including normal load and overload profiles.

Batch rules:
- default batch size: 100.
- maximum batch size: 250.
- save evidence after every batch.
- support resume.
- stop on P0 unless operator permits continuing.

Total target:
- 20,000 scenarios if resources allow.
- 10,000 deep audit.
- 5,000 broad audit minimum.
- 2,000 partial minimum.

Required distribution for 20,000 target:
- Customer scenarios: 3,000+
- Provider owner/admin scenarios: 2,500+
- Provider staff scenarios: 2,000+
- Driver scenarios: 2,000+
- Ops/admin scenarios: 2,500+
- Finance/settlement scenarios: 2,000+
- Control panel/control room scenarios: 2,500+
- API/concurrency/security scenarios: 1,500+
- Cross-role lifecycle scenarios: 1,500+
- Mandatory document/governance scenarios: 1,000+

Provider category coverage:
Every provider category must appear in:
- browse flow.
- catalog flow.
- cart/checkout flow where applicable.
- provider dashboard flow.
- ops control panel flow.
- finance/control visibility where applicable.
- at least one negative scenario.
- at least one empty/error state.

Load profiles:
1. normal
2. peak orders
3. driver scarcity
4. provider overload
5. high cancellation
6. high complaint
7. high wallet activity
8. high ops/control-room viewing
9. finance settlement burst
10. mixed chaotic day

Execution command shape:

python tools/agent_scenarios/run_agent_batch.py \
  --total-scenarios 20000 \
  --batch-size 100 \
  --seed 20260707 \
  --ui-ratio 0.70 \
  --load-profile mixed \
  --resume \
  --output-dir test-results/agent-real-operation

Each batch must write:
- test-results/agent-real-operation/raw/batch_<N>.json
- test-results/agent-real-operation/screenshots/batch_<N>/
- test-results/agent-real-operation/traces/batch_<N>/
- test-results/agent-real-operation/har/batch_<N>/
- test-results/agent-real-operation/db-invariants/batch_<N>.json
- app_report/450_BATCH_<N>_SUMMARY.md

After each batch:
- count PASS/FAIL.
- list P0/P1 immediately.
- run invariant check.
- save logs.
- save browser traces.
- save failed reproduction commands.
- record duration and performance symptoms.

Phase output:
- app_report/450_AGENT_SCENARIO_RUN_RAW.json
- app_report/450_AGENT_SCENARIO_RUN_SUMMARY.md
- app_report/450_LOAD_AND_OVERLOAD_REPORT.md

Phase verdict:
- PHASE_7_EXECUTION_PASS
- PHASE_7_EXECUTION_WITH_DEFECTS
- PHASE_7_EXECUTION_PARTIAL
- PHASE_7_EXECUTION_FAIL
```

---

## المرحلة 8 — التحليل: العيوب، الحوكمة، الإلزاميات، التحكم

```text
PHASE 8 — INVARIANTS, DEFECTS, GOVERNANCE, CONTROL ANALYSIS

Goal:
Analyze all scenario outputs and DB invariants to produce precise defect, governance, finance, control-panel, and mandatory-gate reports.

Create:
- tools/agent_scenarios/check_invariants.py
- tools/agent_scenarios/analyze_agent_results.py
- app_report/450_AGENT_INVARIANTS.md
- app_report/450_AGENT_DEFECTS.md
- app_report/450_CONTRADICTIONS_AND_GAPS.md
- app_report/450_GOVERNANCE_LOCKED_FEATURES.md
- app_report/450_MANDATORY_DOCUMENT_GATES.md
- app_report/450_CONTROL_ROOM_VISIBILITY_GAPS.md

Required reports:

1. Defects report:
   Every defect must include severity, role, scenario, reproduction, expected, actual, evidence, suspected code area, and release impact.

2. Governance locked features:
   List:
   - all sealed features.
   - all feature-flag-disabled features.
   - all launch-gate-blocked features.
   - all UI actions hidden because of governance.
   - all UI actions visible despite governance lock.
   - all APIs blocked because of governance.
   - all APIs incorrectly allowed despite governance lock.
   - all admin/ops override paths.
   - audit logs for overrides.
   - bypass risks.

3. Mandatory document gates:
   List:
   - required provider documents.
   - required driver documents.
   - required business/legal/onboarding documents.
   - document states that block operation.
   - document states that allow operation.
   - UI/backend agreement.
   - ops review capability.
   - rejection reason visibility.
   - expiry handling.
   - audit log evidence.

4. Control room visibility:
   For each important platform event, state:
   - visible yes/no.
   - where visible.
   - which role can see it.
   - delay/realtime behavior if detectable.
   - missing panel/action if any.
   - whether it blocks operations.

5. Financial correctness:
   State:
   - exact arithmetic defects.
   - rounding defects.
   - ledger imbalance.
   - settlement mismatch.
   - commission mismatch.
   - wallet/idempotency defects.
   - hidden financial events.
   - finance control-panel gaps.

6. Contradictions:
   Detect:
   - feature flag enabled but UI absent.
   - UI action visible but API blocks unexpectedly.
   - API exists but UI missing.
   - launch gate open but UI implies readiness.
   - sealed feature exposed.
   - provider category behaves incorrectly like restaurant.
   - control panel button inert.
   - backend capability has no control panel surface.
   - finance event exists but not visible to finance/control room.

Phase verdict:
- PHASE_8_ANALYSIS_PASS
- PHASE_8_ANALYSIS_WITH_DEFECTS
- PHASE_8_ANALYSIS_FAIL
```

---

## المرحلة 9 — التقرير النهائي والـVerifier

```text
PHASE 9 — FINAL VERIFIER AND REPORT

Goal:
Create an evidence verifier and final report. No claim is allowed without evidence.

Create:
- tools/agent_scenarios/verify_agent_real_operation_report.py
- app_report/450_VERIFIER_OUTPUT.txt
- app_report/450_AGENT_COVERAGE_MATRIX.md
- app_report/450_AGENT_EVIDENCE_INDEX.md
- app_report/450_AGENT_REAL_OPERATION_EXPLORATION_REPORT.md

Verifier must fail if:
- boot evidence missing.
- no-mock verification missing.
- seed manifest missing.
- raw scenario JSON missing.
- summary missing.
- defect file missing.
- evidence index missing.
- fewer than 2,000 scenarios but report claims broad exploration.
- fewer than 5,000 scenarios but report claims broad audit.
- fewer than 10,000 scenarios but report claims deep audit.
- fewer than 70% browser UI scenarios but report claims real UI coverage.
- fewer than 2,500 control-panel scenarios but report claims control panel deep audit.
- fewer than 2,000 finance scenarios but report claims finance deep audit.
- any failed scenario lacks defect entry.
- any defect lacks evidence.
- any PASS lacks evidence.
- any UI scenario lacks screenshot or trace.
- any critical role has zero UI scenarios.
- any provider category has zero scenarios.
- any open blocking launch gate and verdict says soft launch candidate.
- wallet_cash_out false but UI allows cash-out.
- production_payment_enabled false but UI allows production payment.
- Playwright not run but report claims frontend real testing.
- mocks/MSW/page.route.fulfill used.
- production-ready appears anywhere.
- E2E complete appears without lifecycle evidence.
- finance report uses float arithmetic instead of Decimal/minor units.
- mandatory document gates are not reported.
- governance locked features are not reported.
- control-room visibility is not reported.

Final report sections:
1. Executive verdict
2. Scope and non-scope
3. Exploration methodology
4. Environment and commit identity
5. Local boot proof
6. No-mock proof
7. Seed world summary
8. Load profile summary
9. Provider category matrix
10. Agent model
11. Scenario generation method
12. Total scenario statistics
13. Browser UI scenario statistics
14. API scenario statistics
15. Load/overload statistics
16. Control panel audit statistics
17. Finance audit statistics
18. Full role coverage matrix
19. Full provider category coverage matrix
20. Customer findings
21. Provider owner/admin findings
22. Provider staff findings
23. Driver findings
24. Ops/admin findings
25. Finance/settlement findings
26. Financial arithmetic findings
27. Financial control completeness findings
28. Control panel completeness findings
29. Control panel UX/navigation findings
30. Control panel button/action findings
31. Control room observability findings
32. Mandatory document gate findings
33. Governance locked feature findings
34. Dispatch/state-machine findings
35. Authorization/IDOR findings
36. Wallet/finance findings
37. UI/API mismatch findings
38. DB invariant findings
39. Launch gate findings
40. Mobile/native readiness finding
41. Confirmed working flows
42. Confirmed defects
43. Contradictions and gaps
44. Unverified areas
45. High-priority repair plan
46. Exact commands run
47. Raw evidence index
48. Verifier output
49. Final verdict
50. Operator next steps

Final verdict options:
- AGENT_REAL_OPERATION_BOOT_FAILED
- AGENT_REAL_OPERATION_NO_MOCK_PROOF_FAILED
- AGENT_REAL_OPERATION_PARTIAL_WITH_BLOCKERS
- AGENT_REAL_OPERATION_COMPLETED_WITH_CRITICAL_DEFECTS
- AGENT_REAL_OPERATION_COMPLETED_WITH_MAJOR_DEFECTS
- AGENT_REAL_OPERATION_COMPLETED_INTERNAL_PILOT_CANDIDATE
- AGENT_REAL_OPERATION_SOFT_LAUNCH_CANDIDATE

Verdict rules:
- BOOT_FAILED if boot fails.
- NO_MOCK_PROOF_FAILED if no-mock verification fails.
- PARTIAL_WITH_BLOCKERS if fewer than 2,000 scenarios.
- COMPLETED_WITH_CRITICAL_DEFECTS if any P0 exists.
- COMPLETED_WITH_MAJOR_DEFECTS if any P1 exists.
- INTERNAL_PILOT_CANDIDATE only if:
  - stack booted,
  - no-mock verifier passed,
  - at least 5,000 scenarios,
  - at least 70% browser UI,
  - no P0/P1,
  - all failures documented,
  - every critical role has UI scenarios,
  - every provider category has scenarios,
  - finance arithmetic report exists,
  - control room visibility report exists,
  - mandatory gates report exists,
  - governance report exists,
  - verifier passed.
- SOFT_LAUNCH_CANDIDATE only if:
  - all internal-pilot requirements pass,
  - at least 10,000 scenarios,
  - at least 2,500 control-panel scenarios,
  - at least 2,000 finance scenarios,
  - all blocking launch gates closed,
  - wallet/payment/legal gates resolved,
  - mobile/native strategy handled,
  - E2E lifecycle passes.

Never use production-ready.

At the end print:
- commit tested.
- stack booted yes/no.
- mocks used yes/no.
- total scenarios.
- UI scenarios.
- API scenarios.
- overload scenarios.
- control panel scenarios.
- finance scenarios.
- provider categories covered.
- passed.
- failed.
- P0/P1/P2/P3 counts.
- launch gates block release yes/no.
- governance locks found.
- mandatory gates found.
- wallet/finance safe yes/no.
- control room visibility complete yes/no.
- mobile ready yes/no.
- top 10 repairs.
- exact next command for operator.
```

---

## المرحلة 10 — أوامر التشغيل العملية

```text
PHASE 10 — OPERATOR EXECUTION PLAN

After all harness files are created, print exact commands to run in order.

Expected command sequence:

# 1. Boot
docker compose --profile dev-minimal up --build

# 2. Seed
python tools/agent_scenarios/seed_agent_world.py

# 3. Verify no mocks
python tools/agent_scenarios/verify_no_mock_runtime.py

# 4. Dry-run scenario generation
python tools/agent_scenarios/run_agent_batch.py \
  --dry-run \
  --list-scenarios \
  --total-scenarios 20000 \
  --seed 20260707

# 5. Run first small safety batch
python tools/agent_scenarios/run_agent_batch.py \
  --total-scenarios 200 \
  --batch-size 25 \
  --seed 20260707 \
  --ui-ratio 0.70 \
  --resume \
  --stop-on-p0

# 6. Run broad exploration
python tools/agent_scenarios/run_agent_batch.py \
  --total-scenarios 5000 \
  --batch-size 100 \
  --seed 20260707 \
  --ui-ratio 0.70 \
  --load-profile mixed \
  --resume \
  --output-dir test-results/agent-real-operation

# 7. Run deep exploration if resources allow
python tools/agent_scenarios/run_agent_batch.py \
  --total-scenarios 20000 \
  --batch-size 100 \
  --seed 20260707 \
  --ui-ratio 0.70 \
  --load-profile mixed \
  --resume \
  --output-dir test-results/agent-real-operation

# 8. Analyze
python tools/agent_scenarios/analyze_agent_results.py

# 9. Verify report contract
python tools/agent_scenarios/verify_agent_real_operation_report.py

# 10. Print report path
echo app_report/450_AGENT_REAL_OPERATION_EXPLORATION_REPORT.md

If any command differs because of repository reality, document the reason and use the correct command.
```
