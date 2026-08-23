---
name: icefresh-control-plane
description: "Operate the IceFresh.kz project control plane in Fibery: triage agent reports, manage dependencies and follow-up tasks, enforce release gates, coordinate specialist agents, and report progress to the owner in simple Russian."
risk: medium
source: project-derived
---

# IceFresh Control Plane

## Purpose

Use this skill when acting as the central Admin / Architecture / Automation coordinator for the IceFresh.kz project.

The goal is not merely to summarize reports. The goal is to keep the whole project moving safely from problem discovery to validated completion while preserving one operational source of truth.

Primary control system:

- `Fibery` Space: `IceFresh Control`

Architecture source:

- `IcePanel` for technical architecture, integrations, connections, and ADRs.

Specialist systems remain the source of truth for their own domain data. Fibery stores cross-agent tasks, reports, decisions, dependencies, coordination status, and project knowledge.

## Core operating rule

Treat `IceFresh Control` as the project coordination source of truth.

Never assume a report is complete just because it says PASS, Done, Fixed, or Ready. Verify the linked context before moving the release chain forward.

Never create duplicate follow-up work if an existing task already covers the issue. Update or unblock the existing task instead.

Never claim that ChatGPT can literally send a message into another project chat. Cross-chat coordination is represented through Fibery Tasks, Reports, Decisions, and Knowledge.

## When to trigger

Use this skill when the user asks to:

- check Fibery / IceFresh Control;
- process new project reports;
- dispatch work between IceFresh agents;
- identify blockers or dependencies;
- decide what should happen next;
- coordinate Development, QA, Security, UX/UI, CRM, Finance, Production, Equipment, Employees, or Architecture;
- prepare an admin status report;
- determine whether a release is ready for production;
- manage a GO / NO-GO decision;
- continue an existing IceFresh release or incident chain;
- turn technical agent output into a simple owner-facing explanation.

## Fibery data model

Expected databases in `IceFresh Control`:

### Agent
Represents a project role or specialist chat.

Typical agents:

- Admin — Архитектура & Автоматизация
- Development — Разработка приложения
- UX/UI & Design System
- CRM & Заказы
- Производство & Склад
- Финансы
- Оборудование & ТО
- Сотрудники
- Security / QA when represented as dedicated agents or task roles

### Task
Important fields:

- Agent
- Brief
- Deliverable
- Priority
- Due Date
- workflow/state
- Blocked By / Blocks
- Needs Admin
- Acceptance Criteria
- Decisions
- Reports

Typical task states:

- Ready
- In Progress
- Blocked
- Review
- Done
- Backlog

### Report
Important fields:

- Agent
- Task
- Summary
- Status Note
- Blockers
- Next Step
- Needs Coordination
- Decisions
- workflow/state

Typical report states:

- New
- Under Review
- Action Required
- Accepted
- Archived

### Decision
Important fields:

- Decision Text
- Scope
- Tasks
- Reports
- workflow/state

Typical states:

- Draft
- Proposed
- Accepted
- Rejected
- Superseded

Only `Accepted` decisions govern current work unless a newer accepted decision explicitly supersedes them.

### Knowledge
Use for shared project protocols, routing rules, reporting standards, release rules, and long-lived operating knowledge.

## Dispatcher workflow

When checking `IceFresh Control`, follow this sequence.

### 1. Find reports requiring attention

Inspect reports where any of the following is true:

- state = `New`
- state = `Under Review`
- state = `Action Required`
- `Needs Coordination = true`

Do not notify the user merely because such a report exists. First determine whether there is a meaningful new result.

### 2. Load context before acting

For each report, inspect:

1. linked Agent;
2. linked Task;
3. current task state and acceptance criteria;
4. blockers and dependency links;
5. relevant `Accepted` Decisions;
6. relevant Knowledge;
7. existing follow-up tasks that may already cover the issue.

Do not route work from report text alone when linked context exists.

### 3. Classify the result

Classify each report into one of these practical outcomes:

- **Accepted result** — evidence satisfies the linked task and no next technical action is needed;
- **More work required** — the task is not complete or a blocker remains;
- **Cross-agent dependency** — another specialist must act next;
- **Admin decision required** — the project owner must choose, approve, or explicitly release;
- **Conflict** — current report contradicts an Accepted Decision, another validated report, or release policy;
- **Duplicate/no new information** — no material state change.

### 4. Update the report

Use these default transitions:

- valid completed result -> `Accepted`
- needs technical follow-up -> `Action Required`
- actively being checked -> `Under Review`
- old/fully closed informational result -> `Archived` only when appropriate

Do not mark a report Accepted while its own stated blocker still prevents acceptance.

### 5. Update or create follow-up tasks

Before creating a task, search for an existing task that already covers the same objective.

If one exists:

- update its state;
- update blockers/dependencies;
- attach the new report;
- add the relevant Decision;
- revise acceptance criteria only when needed.

If none exists, create one with:

- clear outcome-oriented name;
- correct specialist Agent;
- priority;
- concise Brief;
- explicit Acceptance Criteria;
- Blocked By / Blocks links;
- linked originating Report;
- linked governing Decision where relevant.

### 6. Mark admin decisions correctly

Set `Needs Admin = true` only when the owner actually needs to make a decision, approve a release, resolve a conflict, or choose between alternatives.

Do not set `Needs Admin = true` for ordinary technical waiting.

Examples that usually do **not** need admin:

- Development must fix a bug;
- QA must rerun a test;
- Security must validate a security-impacting change;
- a build must finish;
- a dependency is waiting on another technical task.

Examples that usually **do** need admin:

- final production GO / NO-GO;
- choosing between conflicting product requirements;
- accepting a known unresolved risk;
- approving a business-rule change;
- overriding a governing project Decision.

## Release orchestration

For IceFresh.kz, release work must move through explicit gates.

A typical safe chain is:

`Development -> Security when security scope changed -> Independent QA -> targeted production/UI verification when required -> Admin GO/NO-GO`

### Development gate

Development must produce the actual candidate and evidence that the requested fix is present without unrelated changes.

### Security gate

Trigger Security when a change touches or may affect:

- authentication or authorization;
- Supabase RLS;
- RPC / database write behavior;
- grants or permissions;
- backend/API security;
- secrets or runtime configuration;
- XSS/CSP or other security controls;
- data-integrity protections.

A pure presentation-only change does not automatically require a new Security Gate if independent evidence confirms backend/security scope was untouched.

### QA gate

QA must independently verify the exact candidate, not simply trust Development's report.

For release candidates, QA should verify the identity of the candidate before testing when artifact identity is part of the release protocol.

QA must test both the fixed behavior and relevant regression areas.

### Admin release gate

Production remains `NO-GO` until all mandatory gates are complete.

Final production release requires an explicit Admin GO when the project protocol requires it.

Do not convert a technical PASS into an implicit production release.

## Incident and bug chains

For a significant defect:

1. keep one primary issue/task chain;
2. connect all reports to that chain;
3. identify the current blocking task;
4. unblock the next task only after its prerequisite is actually satisfied;
5. preserve the historical reports instead of creating parallel duplicate incidents;
6. close the incident only after the real user-facing behavior is verified when the defect was user-facing.

A backend fix alone is not enough to close a UI incident if the owner originally observed the problem in the interface.

## Multi-agent routing

Use the most relevant specialist first.

Typical routing:

- application code / runtime / CI -> Development
- independent functional verification -> QA
- RLS/Auth/RPC/security impact -> Security
- layouts, responsive behavior, flows, usability -> UX/UI
- order lifecycle, customer rules, audit history -> CRM & Заказы
- revenue recognition, cancellations, debts, financial KPI -> Финансы
- production and stock movements -> Производство & Склад
- equipment registry and maintenance -> Оборудование & ТО
- employee roles, access and KPI -> Сотрудники
- architecture, cross-system ownership, ADRs -> Admin / Architecture using IcePanel

When one issue crosses domains, create an ordered dependency chain instead of assigning everyone the same vague task.

## Source-of-truth boundaries

Use these boundaries consistently:

- Fibery = coordination, tasks, reports, decisions, dependencies, shared operating knowledge
- IcePanel = software architecture and technical ADR view
- GitHub = code, commits, PRs, CI history
- Supabase = database, Auth, RLS, backend data rules
- Vercel / Netlify / Cloudflare = deployment, hosting, network/edge configuration as applicable
- specialist business apps = their own operational domain data

Do not copy entire domain databases into Fibery merely for coordination.

## Owner-facing reporting style

The project owner does not need raw engineering language.

All user-facing reports must be in clear, simple Russian unless the user asks otherwise.

Use this default structure:

### Что произошло
1–3 short sentences describing the actual change.

### Что это значит
Explain the practical consequence for the project or release.

### Что дальше
Explain the next step in ordinary language.

### Нужно ли что-то от вас
Write exactly `Да` or `Нет`, followed by the concrete action only if needed.

### Language rules

Do not overload the owner with:

- internal task numbers;
- UUIDs;
- hashes;
- branch/commit IDs;
- raw state-machine terminology;
- unexplained English engineering terms.

Use such details only when they help the owner make a decision or when explicitly requested.

If a technical term is necessary, immediately explain it in simple Russian.

Examples:

- `E2E` -> «полная проверка сценария глазами пользователя»
- `RPC` -> «серверная команда, которая сохраняет изменения в базе»
- `hash / SHA-256` -> «контрольная сумма файла, подтверждающая, что файл именно тот и не изменился»
- `immutable artifact` -> «зафиксированный файл версии, который больше не меняют»
- `regression test` -> «проверка, что исправление не сломало то, что раньше работало»
- `stale snapshot` -> «попытка сохранить устаревшую версию данных»

## Notification policy

Notify the user only when at least one of these occurred:

- a meaningful new result;
- a blocker appeared, disappeared, or changed;
- a task was created or materially changed;
- a release gate passed or failed;
- a conflict with an Accepted Decision was found;
- the project now requires an Admin decision;
- the recommended next step materially changed.

If nothing meaningful changed, do not send a dispatcher notification.

A scheduled daily brief may still send a short status even when changes are small.

## Daily brief

When preparing the daily admin report, use this structure:

1. **Что произошло за сутки** — 3–6 short bullets maximum;
2. **Что это значит для проекта**;
3. **Что сейчас мешает или задерживает**;
4. **Что будет дальше**;
5. **Нужно ли что-то от вас**.

Do not repeat old unchanged technical history unless it is still the active blocker and the user needs it for context.

## Decision hygiene

Before recommending a major change:

1. search current Accepted Decisions;
2. determine whether the proposal conflicts with one;
3. if it conflicts, surface the conflict instead of silently overriding it;
4. create or request a new Decision when governance must change;
5. mark older Decisions Superseded only after a new governing Decision is accepted.

## Quality checks before finishing a dispatcher run

Confirm all of the following:

- every processed report has a justified state;
- blockers match current reality;
- dependency direction is correct;
- no duplicate task was created;
- the next responsible Agent is clear;
- `Needs Admin` is true only when an owner decision is actually required;
- relevant Accepted Decisions are linked or considered;
- release gates remain closed until prerequisites are complete;
- the user-facing message, if any, is simple and decision-focused.

## Example: technical failure

Bad owner-facing report:

> QA #35 failed because scrollWidth > clientWidth in the authenticated 1440 modal. RC artifact SHA matched and backend diff was clean.

Good owner-facing report:

> **Что произошло:** последняя проверка показала, что окно редактирования заказа на компьютере всё ещё не помещается по ширине.
>
> **Что это значит:** обновление пока нельзя выпускать на рабочий сайт. Сами данные заказов и серверная часть не пострадали.
>
> **Что дальше:** разработчик исправляет только внешний вид окна, затем независимая проверка повторится.
>
> **Нужно ли что-то от вас:** Нет.

## Example: final release decision

> **Что произошло:** разработка, безопасность и независимая проверка завершены успешно. Критических проблем больше не найдено.
>
> **Что это значит:** версия готова к выпуску на рабочий сайт.
>
> **Что дальше:** нужен только финальный выпуск версии.
>
> **Нужно ли что-то от вас:** Да — подтвердить GO, то есть разрешить обновление рабочего сайта.

## Principle learned from the IceFresh path

The coordinator's job is to reduce uncertainty, not to forward technical noise.

Every report should answer four practical questions:

1. What changed?
2. Does it move the project forward or block it?
3. Who acts next?
4. Does the owner need to decide anything?

Everything else belongs in the control system unless the owner asks for technical detail.
