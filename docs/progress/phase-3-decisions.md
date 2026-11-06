# Phase 3 — Role Consolidation Decision Record

- **Date:** 2026-11-06
- **Type:** Documentation-only correction phase. **No code, no models, no migrations, no database access, no dependency changes, no commits.**
- **Trigger:** Confirmed business requirement: the Gym Management System must have exactly **two staff login roles — ADMIN and RECEPTIONIST**. No separate TRAINER or ACCOUNTANT login roles or staff accounts exist.

---

## 1. Approved Decisions (register)

| # | Decision | Where applied |
|---|---|---|
| P3-01 | Staff login roles = exactly two: ADMIN, RECEPTIONIST. Seeded `Role` rows contain only these codes; no additional login role or third-party user hierarchy may be introduced | permissions §1/§3; modules §2.1; database 2.1, §5; system-architecture AD-13 |
| P3-02 | **No TRAINER login role.** Trainers are ADMIN-managed business records (`Trainer`, formerly `TrainerProfile`) with no credentials. Trainer assignment, workout plans, exercise library and training schedules are managed by ADMIN. No separate trainer authentication model exists | business-rules BR-TRN-01..04, BR-WKO-01/03; database 2.5; erd §1/§2; modules §2.8/§2.9; system-architecture §6.3 |
| P3-03 | **No ACCOUNTANT login role.** Financial records, expense recording, review, and approval/rejection are ADMIN duties. The recorder≠approver object rule (BR-EXP-04) is preserved: the user who records an expense can never approve it; a different ADMIN user decides | business-rules BR-EXP-01/04; modules §2.11; permissions §2/§4.2/§5; database 2.7; system-architecture §6.4 |
| P3-04 | Trainer/database integrity: trainer business entities (`Trainer`, `TrainerAssignment`, `TrainingSchedule`) and all trainer relationships remain; only the login linkage (`User` FK) was removed. `TrainerAssignment` and `TrainingSchedule.trainer_id` now reference the standalone `Trainer` record; cascade clause (profile extension) replaced by PROTECT master-data policy | database 2.5, §4.1; erd §1/§2 |
| P3-05 | Authorization guarantee: backend authorization design must prevent every RECEPTIONIST path to ADMIN-only operations (staff management, system settings, financial approvals, biometric device configuration, sensitive system-wide reports) and must preserve full auditability of financial and administrative actions | permissions §3/§4.2; modules §2.2/§2.13 |
| P3-06 | `AuditLog` event types (including `TRAINER_ASSIGNED`) and notification trigger events are unchanged — trainer assignment remains a reportable, audited business event even though trainers do not log in | database 2.8/2.9; business-rules BR-AUD-01, BR-NTF-02 |

## 2. Impact Summary (before → after)

| Concern | Before | After |
|---|---|---|
| Login roles | 4 (ADMIN, RECEPTIONIST, TRAINER, ACCOUNTANT) | 2 (ADMIN, RECEPTIONIST) |
| Trainer identity | FK to `User` (role TRAINER) with OneToOne `TrainerProfile` | Standalone `Trainer` business record, own identity fields, no `User` FK, no credentials |
| Trainer assignments | Admin-driven; trainers could view own assignments | Admin-driven; RECEPTIONIST has read-only member-facing views; no trainer principal exists |
| Workout plans / schedules | Created by trainers for assigned members (object-level ownership) | Created and managed by ADMIN |
| Expense recording | ACCOUNTANT records; ADMIN approves/rejects | ADMIN records; a **different** ADMIN user approves/rejects (BR-EXP-04 object rule) |
| Financial reports | ADMIN + ACCOUNTANT | ADMIN only; RECEPTIONIST gets a basic daily operational summary |
| Entity count | 27 | 27 (`TrainerProfile` retitled `Trainer`; count unchanged) |
| Trainer/accounting functionality | Present | **Present — preserved, not deleted**; exercised by ADMIN |

## 3. Documents Updated

| File | Changes |
|---|---|
| `docs/architecture/permissions.md` | Two-role table, role matrix (ADMIN/RECEPTIONIST columns only), restriction list, auth-architecture planner (role gates, financial gate, recorder≠approver), guardrails |
| `docs/architecture/modules.md` | `TrainerProfile`→`Trainer`, two-role account module, per-module access lists, expenses/workouts/trainers/reporting rewording |
| `docs/architecture/business-rules.md` | BR-TRN-01..04, BR-WKO-01/03, BR-EXP-01/04 amended; domain convention now lists WKO; header records the amendment |
| `docs/architecture/system-architecture.md` | Module map entities, §6.3 trainer management, §6.4 expense flow, §7 authorization, AD-03 amended, **AD-13 added**, terminology ("Subscription"), related-documents link |
| `docs/database/database-design.md` | `Role` seed (2 values), `Trainer` standalone entity block, `TrainerAssignment`/`TrainingSchedule` FK retargeted to `Trainer`, `WorkoutPlan.created_by` note, `Expense.recorded_by` note, candidate evaluation, delete policy, seed data |
| `docs/database/erd.md` | `TRAINER_PROFILE`→`TRAINER` entity block, relationship edges, FK dictionary rows, no-User-Trainer-FK note |
| `docs/progress/phase-2-architecture.md` | D-03, D-14 annotated as amended; amendment log §9 added |
| `docs/progress/phase-3-decisions.md` | This record (new) |

## 4. Unresolved Decisions Register (status updated 2026-11-06)

**Resolved as CONFIRMED by the second amendment (§7):** PostgreSQL 18 (P3-07); KES single currency + payment methods (P3-08/09); cash-basis accounting (P3-10); membership start = full-payment date (P3-11); offline fail-closed with a secure access cache, expiry and event synchronization (P3-12); staff card/PIN physical access (P3-13).

**Still open:**

1. Biometric device manufacturer & model — BR-BIO-06.
2. Door controller protocol / gateway contract — BR-ACC-10.
3. Multiple active trainer assignments per member (primary vs co-trainers) — BR-TRN-05; still affects the partial unique index on `TrainerAssignment`.
4. Same vs separate entry/exit biometric terminals — BR-ACC-09.
5. Member phone uniqueness (BR-REG-06); minor/age policy (BR-REG-07); overpayment/credit handling (BR-MEM-06); audit retention policy (BR-AUD-05); notification provider selection (BR-NTF-04 — channels confirmed, providers not).
6. Offline-cache expiry window and refresh policy (implementation parameter accompanying P3-12).
7. Member-portal future linkage (AD-02) remains out of v1 scope by design — not a conflict.

## 5. Validation Performed (2026-11-06)

- `TRAINER`/`ACCOUNTANT` full-corpus search across `docs/`: remaining hits are exclusively (a) business-entity names (`Trainer`, `TrainerAssignment`, `TrainingSchedule.trainer_id`), (b) explicit "no login / retired role" statements, (c) audited event names (`TRAINER_ASSIGNED`), (d) amended-register annotations. No reference represents TRAINER or ACCOUNTANT as a login role.
- All 13 modules remain catalogued (`modules.md` §1).
- Trainer management, workout management, equipment management and accounting functionality confirmed present and ADMIN-managed.
- Both role-permission matrices (permissions §2; spanning modules, business rules, database and ERD references) contain only ADMIN and RECEPTIONIST columns/values.
- Cross-document consistency checks: entity names agree between `database-design.md` and `erd.md`; decision pointers (AD-13, P3-*) agree across files.
- `git diff --check` run clean (no whitespace errors).
- Second-amendment consistency: DB and ERD agree on `AccessAttempt` extensions (`identity_kind`, `staff_actor_id`, nullable biometric-device edge for staff events) and on `Membership` date semantics (start = full-payment date, set at activation); confirmed statuses (PG 18, KES, payment methods, cash basis, offline cache, staff card/PIN access) reconciled across all eight files.

## 6. Boundaries Respected

No Django applications, models, serializers, views, endpoints, migrations, database connections or writes; no dependency or environment changes; no frontend work; no Git commits or history modification. Implementation is deferred until explicit approval.

## 7. Second Amendment — 2026-11-06 (confirmed operational decisions)

Applied per user-confirmed decisions; all edits documentation-only. These resolve five formerly-[UNRESOLVED] register items and add one newly confirmed access requirement.

| # | Decision (confirmed) | Where applied |
|---|---|---|
| P3-07 | PostgreSQL 18 is the target database engine | system-architecture §4 stack table; database §6; phase-2 register item 1 |
| P3-08 | Single currency: **KES** (multi-currency out of scope) | business-rules BR-MEM-15 |
| P3-09 | Payment methods: **CASH, MOBILE_MONEY, CARD, BANK_TRANSFER** (one per payment) | business-rules BR-MEM-14; database 2.3 `Payment.method`; system-architecture §6.8 |
| P3-10 | **Cash-basis** accounting (revenue = COMPLETED payments; expense = approved decisions) | business-rules BR-FIN-05; system-architecture §6.8 |
| P3-11 | Membership starts on the **date full payment is received**; `start_date`/`end_date` set at activation, snapshots immutable thereafter | business-rules BR-MEM-13; database 2.3 `Membership` dates + §4.3; system-architecture §6.1 + terminology; permissions §4.3 |
| P3-12 | Offline access fails **closed**; secure offline access cache with entry expiry on the gateway; buffered events synchronized on recovery; cache expiry window = open implementation parameter | business-rules BR-ACC-07; system-architecture §6.5 + terminology; modules §2.7; database §3 candidates; phase-2 register item 6 |
| P3-13 | Staff physical access via **individually assigned cards or PINs** as the standard method; staff biometric enrollment optional, not prohibited (amended 2026-11-06 — when configured, it must identify the staff account (`identity_kind = STAFF`) and is never treated as member access); staff access bypasses member subscription/payment validation; entry/exit attempts captured in `AccessAttempt` (`identity_kind = STAFF`) and attributed to the individual staff `User`; issuance/revocation ADMIN-only | business-rules BR-ACC-11 + BR-BIO-01 note; modules §2.7; permissions §2; database 2.4; erd §1/§2/§3 |

Not changed (already confirmed in the documentation): one gym location; five fixed plans (DAILY, WEEKLY, MONTHLY, QUARTERLY, YEARLY); full payment before activation; manual renewal only; member fingerprint + facial recognition; in-app, SMS, email channels; emergency exit independent of backend authorization.
