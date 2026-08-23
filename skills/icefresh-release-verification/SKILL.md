---
name: icefresh-release-verification
description: "IceFresh.kz release verification and repair playbook derived from the RC1.6.x path: exact-candidate identity, development fixes, security gates, independent QA, responsive/UI checks, CRM/finance reconciliation, production GO/NO-GO, and post-release verification."
risk: medium
source: project-derived
---

# IceFresh Release Verification

## Purpose

Use this skill for any IceFresh.kz release candidate, hotfix, regression, production-readiness check, or release-blocking defect.

This skill captures the practical lessons learned from the IceFresh RC1.6.x path so future releases do not repeat the same mistakes.

The objective is simple:

1. fix only the real problem;
2. prove the exact candidate contains the fix;
3. independently verify the candidate;
4. verify security when relevant;
5. verify the real user-facing scenario;
6. verify CRM/finance/business consequences;
7. require explicit production GO;
8. verify production after release.

Do not skip a gate merely because a previous step says PASS.

## Core principle

A release is not "ready" because Development says it is ready.

A release is ready only when:

- the exact candidate is identified;
- required build/test checks pass on that candidate;
- independent QA verifies the changed behavior and regressions;
- security-impacting changes pass Security;
- critical user-facing flows pass in the real interface;
- business rules such as revenue, debt, status, and order integrity are correct;
- no unresolved release blocker remains;
- Admin gives explicit GO when production publication is required.

## Roles

### Development
Owns diagnosis, implementation, local verification, candidate creation, and traceability.

### Security
Independently verifies security-impacting changes.

### QA
Independently verifies the exact candidate and refuses to trust Development evidence blindly.

### UX/UI
Verifies responsive behavior, layout, modal/card overflow, usability, and visual acceptance.

### CRM & Orders
Verifies order lifecycle, item integrity, statuses, deletion/edit behavior, customer-facing order semantics, and audit expectations.

### Finance
Verifies totals, paid/debt values, revenue recognition, cancellation handling, and double-counting risks.

### Admin / Control Plane
Coordinates dependencies, keeps production closed until gates pass, and issues final GO/NO-GO.

## Standard release chain

Use this default chain:

`Issue -> Development Fix -> Exact Candidate -> Security if needed -> Independent QA -> UX/UI/CRM/Finance targeted checks as relevant -> Production Readiness -> Admin GO -> Production Deployment -> Post-Deploy Smoke Check`

Do not parallelize dependent gates when one result can invalidate the next.

## Stage 1 — Freeze the problem statement

Before changing code, write the defect in observable terms.

A good defect statement includes:

- where it occurs;
- exact screen or flow;
- device/viewport if relevant;
- expected behavior;
- actual behavior;
- business impact;
- what must not be broken by the fix.

Example:

- Bad: "order modal broken"
- Good: "authenticated desktop order editor at 1440px has horizontal overflow; product, quantity, price, line total and remove action must remain fully inside the modal; no backend logic may change."

This prevents broad, risky fixes.

## Stage 2 — Reproduce before fixing

Development must reproduce the defect before claiming a fix.

Record enough evidence to prove the original issue was understood.

For UI/responsive issues check at minimum:

- narrow mobile width;
- tablet width around 768px when the defect is responsive;
- desktop around 1440px when the defect is desktop-specific.

For order/data issues use a known control order when available.

## Stage 3 — Define invariants

Before implementation, state what must remain true.

For IceFresh multi-item orders, important invariants include:

- every order item is preserved unless explicitly removed;
- editing one item must not silently delete another;
- save operation is atomic enough to avoid partial corruption;
- stale/old data must not overwrite newer data silently;
- totals equal the sum of all item rows;
- paid and debt remain consistent;
- a cancelled order must not become received revenue;
- multi-item orders must not be double-counted;
- UI and backend must show the same values.

For pure UI fixes:

- backend behavior must remain untouched;
- no unrelated source changes;
- responsive fix must not break mobile or desktop layouts.

## Stage 4 — Implement the smallest safe fix

Prefer a narrow change over a broad rewrite.

Rules:

- do not touch backend/security code for a CSS-only defect;
- do not rewrite order-saving logic when the defect is purely layout;
- do not mix unrelated improvements into a release blocker hotfix;
- avoid changing product behavior during a release-stabilization branch;
- if a fix expands scope, re-evaluate which gates must run again.

## Stage 5 — Development self-check

Before creating a release candidate, Development should run all checks appropriate to the project.

Typical checks:

- dependency installation from lockfile;
- lint;
- typecheck;
- build;
- unit/integration tests;
- browser smoke test;
- relevant targeted regression test;
- console errors/warnings check;
- changed-file review.

Passing local checks is necessary but not sufficient for release approval.

## Stage 6 — Create an exact release candidate

The candidate must be treated as a fixed object that will not be silently modified after QA begins.

Record:

- exact filename;
- exact byte size;
- SHA-256 or equivalent checksum;
- source branch;
- source commit;
- candidate version;
- where the candidate is stored.

### Why this matters

QA must test the same bytes that may later be approved.

If a candidate changes, it is a new candidate and prior verification may no longer apply.

## Stage 7 — Candidate identity verification

Independent QA must verify candidate identity before unpacking/testing when a packaged artifact is part of the release process.

Check:

- filename;
- byte size;
- checksum.

If any identity value differs, stop and classify it as a different candidate.

Never silently accept "almost the same" artifact.

## Stage 8 — Exact-artifact build verification

A major lesson from RC1.6.x:

It is not enough to know that the repository builds. The exact release candidate must have trustworthy build evidence when the release protocol requires packaged-artifact verification.

Required evidence may include:

- clean dependency install;
- lint PASS;
- typecheck PASS;
- build PASS;
- build-dependent tests PASS.

If these checks cannot be performed because the runner cannot download dependencies or access a private artifact, report that as a release blocker instead of inventing PASS evidence.

Do not modify source inside the candidate just to make the build check pass.

## Stage 9 — Compare scope of changes

When a hotfix follows a previously accepted candidate, compare the new candidate to the previous one.

Goal: prove that only intended files changed.

Examples:

- pure CSS overflow fix should ideally show only CSS/layout-related changes;
- if backend, migrations, auth, database permissions, or save logic changed unexpectedly, stop and expand Security/QA scope.

A small, proven diff can avoid unnecessary gates, but only when evidence confirms security/backend scope is untouched.

## Stage 10 — Security gate decision

Run a new Security Gate when changes touch or may affect:

- authentication;
- authorization;
- Supabase RLS;
- database grants;
- RPC/server save operations;
- backend APIs;
- secrets/environment configuration;
- migrations;
- CSP/XSS protections;
- input validation affecting security;
- data-integrity protections.

### Security checks learned from multi-item order fixes

Security should verify:

- user cannot bypass row-level permissions;
- save RPC cannot write data the caller should not control;
- stale/partial snapshots do not silently overwrite correct data;
- failed writes rollback cleanly where expected;
- removal/re-add of order items does not leave inconsistent orphan state;
- permissions/grants did not widen accidentally.

### When a new Security Gate may be unnecessary

A new Security Gate can usually be skipped for a proven presentation-only change such as CSS/layout if:

- diff confirms only presentation files changed;
- no build/runtime/security config changed;
- no backend/auth/database code changed;
- the previous relevant Security Gate is still applicable.

Document why Security was not repeated.

## Stage 11 — Independent functional QA

QA must independently reproduce the required scenario using the exact candidate.

QA should verify:

- the original defect is fixed;
- adjacent behavior still works;
- no new visible errors;
- console does not show new relevant errors;
- data remains correct after refresh/reload;
- rollback/error behavior is safe where relevant.

QA verdicts should be clear:

- PASS
- RETURN TO DEVELOPMENT
- BLOCKED BY ENVIRONMENT / EVIDENCE

Do not blur a blocked test into PASS.

## Stage 12 — Multi-item order regression protocol

This is a permanent IceFresh critical regression suite.

For a multi-item order:

1. open an order with at least 3 items;
2. verify all rows and values load correctly;
3. edit quantity and/or price of one row;
4. save;
5. reload;
6. confirm all untouched rows still exist;
7. remove one row, e.g. 3 -> 2;
8. save and reload;
9. confirm exactly one intended row was removed;
10. add a row back, e.g. 2 -> 3;
11. save and reload;
12. confirm all 3 rows persist;
13. verify totals after each step;
14. verify paid/debt values;
15. attempt an outdated-data save when the application supports conflict protection;
16. confirm old data cannot silently overwrite newer state;
17. verify failure does not leave partial data corruption.

This suite must remain part of future order-editor releases.

## Stage 13 — Control-order financial reconciliation

Use a known order as a control case whenever finance/order logic is modified.

For the historic control scenario used in IceFresh:

- bag1 = 100 x 523 = 52,300
- bag2 = 200 x 855 = 171,000
- cup250 = 150 x 304 = 45,600
- total = 268,900 KZT
- paid = 0
- debt = 268,900 KZT
- status = New

Checks:

- total equals sum of all `order_items`;
- paid is correct;
- debt = total - paid when that is the governing rule;
- unpaid New order is not treated as received cash revenue;
- cancelled order is not recognized as received revenue;
- changing quantity/price recalculates total/debt correctly;
- finance UI and backend agree;
- multi-item order does not cause double counting.

The exact values above are a regression fixture, not a universal business rule for all orders.

## Stage 14 — CRM/order-lifecycle verification

CRM/Orders should verify:

- order status transitions are valid;
- cancelled state is respected everywhere;
- edit/delete/archive permissions match roles;
- removed order items are actually removed and do not reappear;
- audit/history behavior is acceptable;
- order list and order editor show consistent status and totals;
- customer/order relationships remain intact.

## Stage 15 — UI/UX responsive verification

Two concrete failures from the RC1.6.x path are permanent regression cases.

### Case A — desktop order modal overflow

At desktop width around 1440px verify:

- no horizontal overflow;
- modal content width is contained;
- product field remains inside the dialog;
- quantity remains inside;
- price remains inside;
- line total remains inside;
- remove control remains inside;
- «Сумма позиции» does not cross the right boundary;
- no clipped required control;
- scrollWidth <= clientWidth for the relevant container when measurable.

### Case B — product catalog tablet overflow

At width around 768px verify:

- product cards stay inside the viewport/container;
- grid does not create horizontal page scroll;
- text/buttons/images remain usable;
- layout still works on narrower mobile;
- layout still works on desktop after the tablet fix.

### Minimum responsive matrix

For changed customer/admin UI, prefer checking:

- ~390px mobile;
- ~768px tablet;
- ~1440px desktop.

Add more breakpoints when the design warrants it.

## Stage 16 — Browser/user-flow verification

A release-blocking user-facing defect must be verified through the actual authenticated interface when authentication is part of the flow.

For order editor:

- log in as the correct role;
- open the real editor;
- perform edit/remove/add/save;
- refresh page;
- confirm values remain correct;
- check responsive layout;
- check console for relevant errors.

A database-level PASS alone is not enough to close a UI incident.

## Stage 17 — Cross-layer consistency

Before release, compare values across the layers relevant to the change.

Examples:

- UI order total vs backend-calculated total;
- UI status vs stored status;
- Finance dashboard vs order data;
- CRM order item count vs backend order item count;
- paid/debt shown in UI vs persisted values.

If layers disagree, release remains blocked even if one layer is correct.

## Stage 18 — Release blocker discipline

At any moment there should be a clearly identified current release blocker.

Do not keep old solved blockers marked active.

Do not mark downstream tasks `Needs Admin` simply because they are waiting for technical work.

Use dependency chains such as:

`Development fix -> Security -> QA -> UX/Finance/CRM -> Admin release gate`

Only unblock the next stage after the prerequisite actually passes.

## Stage 19 — GO / NO-GO rules

### GO is allowed only when

- all P0/release-blocking defects are closed;
- exact candidate identity is known;
- required build/test evidence exists;
- required Security Gate passed;
- independent QA passed;
- targeted UX/UI checks passed;
- CRM/Finance checks passed when relevant;
- real user-facing critical flow passed;
- no unresolved conflicting Accepted Decision exists;
- rollback path is known when required;
- Admin explicitly authorizes production publication.

### NO-GO when

Any mandatory gate is incomplete, blocked, failed, or based on unverified evidence.

"Almost ready" is still NO-GO.

## Stage 20 — Production deployment

Do not deploy production during verification unless the user explicitly authorizes it and all release gates are satisfied.

Before deployment verify:

- correct project/environment;
- correct candidate/version;
- production environment variables/config are expected;
- no accidental preview-only settings;
- rollback target is known.

Record the deployed version/commit/artifact identity.

## Stage 21 — Post-deploy smoke test

Immediately after production release, perform a short high-value verification.

At minimum for a release touching orders:

- site loads;
- authentication works;
- order list loads;
- known control order opens;
- critical edit flow works;
- save persists after refresh;
- totals/status remain correct;
- no obvious console/runtime failure;
- no new horizontal overflow on changed screens.

If production behavior differs from the verified candidate, treat it as a new incident.

## Stage 22 — Incident closure

Close a release incident only when:

- root defect is fixed;
- fix is present in the approved candidate;
- independent verification passed;
- production behavior is confirmed when deployed;
- dependent Finance/CRM/UX consequences are resolved;
- no remaining linked blocker exists.

Do not close merely because Development task is Done.

## Key lessons from the IceFresh path

### Lesson 1 — Exact candidate matters

Testing repository source and testing the release ZIP are not always equivalent. When a release is artifact-based, the exact artifact must be identified and verified.

### Lesson 2 — A functional PASS can coexist with a release NO-GO

The multi-item logic can work correctly while the release is still blocked by missing build evidence or UI failure.

### Lesson 3 — Fix scope controls re-testing scope

If only CSS changed and this is independently proven, do not unnecessarily restart backend/security verification. If backend/security files changed, reopen the appropriate gates.

### Lesson 4 — User-visible defects require user-visible verification

Backend/database correctness does not prove the UI works.

### Lesson 5 — Responsive bugs require exact breakpoint retests

A desktop fix may break tablet; a tablet fix may break mobile. Always recheck neighboring breakpoints.

### Lesson 6 — Multi-item orders are an aggregate

Treat an order and its items as one logical aggregate. Saving one row must not silently truncate the rest.

### Lesson 7 — Stale data must fail safely

An old editor state must not silently overwrite newer changes.

### Lesson 8 — Cancellation affects finance

Cancelled orders must not inflate received revenue. Status logic and financial logic must be tested together.

### Lesson 9 — Do not double count multi-item orders

Finance must sum order items once and recognize the order according to the business rule, not count both item totals and order total independently.

### Lesson 10 — Independent QA is actually independent

QA should re-check artifact identity, run tests itself where possible, and return `BLOCKED` when independent evidence is unavailable rather than copying Development's PASS.

### Lesson 11 — Admin should decide only real decisions

Technical waiting is not an admin decision. `Needs Admin` belongs on actual GO/NO-GO, risk acceptance, or business-policy choices.

### Lesson 12 — Keep one chain, not many duplicate tickets

Reuse the existing issue/task chain and attach new evidence to it. Duplicate tasks hide the real blocker.

## Repair loop

When any gate fails, use this loop:

1. record exact failure;
2. keep production NO-GO;
3. identify the smallest responsible scope;
4. route to the correct agent;
5. implement the smallest safe fix;
6. create a new exact candidate if candidate bytes changed;
7. decide which prior gates remain valid and which must repeat;
8. run independent retest;
9. continue only after PASS.

Never patch a candidate in place after independent QA has started.

## Re-test scope decision table

### CSS/layout only

Repeat:

- Development build/self-check as appropriate;
- independent QA for changed screen;
- UX/UI responsive matrix.

Security may remain valid if diff proves no security/backend/config changes.

### Frontend behavior/business logic

Repeat:

- Development checks;
- independent QA;
- affected CRM/Finance/UX flows;
- Security if permissions/auth/input trust boundaries are affected.

### Backend/RPC/database

Repeat:

- Development checks;
- Security;
- independent QA;
- data integrity regression;
- CRM/Finance checks where relevant.

### Auth/RLS/grants/secrets/migrations

Repeat:

- Security mandatory;
- independent QA;
- affected role/access scenarios;
- deployment/runtime configuration checks.

## Owner-facing reporting

For the project owner, hide unnecessary technical noise.

Use:

### Что произошло
Plain-language result.

### Что это значит
Whether the project moved forward or remains blocked.

### Что дальше
Next responsible step.

### Нужно ли что-то от вас
`Да` or `Нет`.

Examples of translations:

- checksum/hash -> «контрольная сумма, подтверждающая, что файл не изменился»
- exact artifact -> «точно тот файл версии, который будем выпускать»
- regression test -> «проверка, что исправление не сломало старые функции»
- E2E -> «полная проверка сценария глазами пользователя»
- RPC -> «серверная команда сохранения данных»
- stale snapshot -> «устаревшая копия данных, которую нельзя позволить сохранить поверх новой»

Technical identifiers belong in Fibery/GitHub reports unless the user requests them.

## Suggested Fibery task/report behavior

When a release report arrives:

- attach it to the existing release/incident task;
- update current blocker;
- create no duplicate task if an existing one covers the same fix;
- set next technical task `Ready` or `In Progress` only when prerequisites allow;
- keep downstream tasks `Blocked` until their gate opens;
- set `Needs Admin=true` only for final GO/NO-GO or real policy/risk choices;
- link relevant Accepted Decisions;
- preserve historical evidence.

## Release completion checklist

Before saying "готово к production", confirm:

- [ ] Problem reproduced and clearly defined
- [ ] Fix scope is minimal and understood
- [ ] Candidate filename recorded
- [ ] Candidate byte size recorded
- [ ] Candidate checksum recorded
- [ ] Branch/commit trace recorded
- [ ] Build/lint/typecheck/tests passed as required
- [ ] Exact candidate independently verified
- [ ] Security passed when required
- [ ] Original defect independently retested
- [ ] Relevant regression suite passed
- [ ] 390px responsive check passed when relevant
- [ ] 768px responsive check passed when relevant
- [ ] 1440px responsive check passed when relevant
- [ ] Authenticated critical flow passed when relevant
- [ ] Multi-item integrity passed when relevant
- [ ] CRM lifecycle check passed when relevant
- [ ] Finance reconciliation passed when relevant
- [ ] Cancelled-order revenue treatment passed when relevant
- [ ] No double counting found
- [ ] No unresolved blocker remains
- [ ] Admin GO received
- [ ] Production deployed from approved candidate
- [ ] Post-deploy smoke test passed

If any mandatory box is unchecked, the correct status is not final GO.

## Companion skill

Use together with `icefresh-control-plane`:

- `icefresh-control-plane` decides who acts next and maintains Fibery coordination;
- `icefresh-release-verification` defines how fixes and release candidates must be verified before production.
