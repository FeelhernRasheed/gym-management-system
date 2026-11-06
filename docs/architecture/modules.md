# Module Catalog — Gym Management System

**Status:** Proposed module decomposition (Phase 2). No Django app has been created yet; names below are planned packages for implementation in later phases, each mapping onto one or more of the entities defined in [`../database/database-design.md`](../database/database-design.md).

---

## 1. Module List

| # | Module (proposed app name) | Core entities owned | Depends on |
|---|---|---|---|
| 1 | `accounts` | `User`, `Role` | — (foundation) |
| 2 | `audit` | `AuditLog` | `accounts` (actor FK) |
| 3 | `members` | `Member` | `accounts` (staff actors recorded via audit) |
| 4 | `memberships` | `MembershipPlan`, `Membership` | `members`, `accounts` |
| 5 | `billing` | `Payment`, `Receipt` | `memberships`, `members`, `accounts` |
| 6 | `attendance` | `AttendanceSession` | `members`, `memberships` (eligibility), `access_control` (biometric events), `accounts` |
| 7 | `access_control` | `BiometricDevice`, `BiometricEnrollment`, `AccessDevice`, `AccessAttempt` | `members`, `memberships` (eligibility), `attendance`, `accounts` |
| 8 | `trainers` | `Trainer`, `TrainerAssignment` | `accounts`, `members` |
| 9 | `workouts` | `WorkoutPlan`, `Exercise`, `WorkoutPlanExercise`, `TrainingSchedule` | `accounts`, `members`, `trainers` |
| 10 | `equipment` | `EquipmentCategory`, `Equipment`, `MaintenanceRecord` | `accounts` |
| 11 | `expenses` | `ExpenseCategory`, `Expense`, `ExpenseApproval` | `accounts`, `equipment` (category upkeep records feed maintenance costs) |
| 12 | `notifications` | `Notification`, `NotificationDelivery` | all emitting modules (event producers) |
| 13 | `reporting` | none (read-only aggregations over the above) | most modules (read) |

> Dependency direction is one-way (left → right). Circular dependencies are prohibited; cross-module facts are emitted as audit/notification events rather than direct writes.

## 2. Module Responsibilities & Role Access

### 2.1 accounts
- **Responsibilities:** custom staff `User` model (username, contact data, status), `Role` lookup seeded with the two confirmed staff login roles only (ADMIN, RECEPTIONIST), staff account lifecycle (create/activate/deactivate), password policy via Django validators.
- **Owners:** ADMIN manages all staff accounts and permissions. Users authenticate to the API/admin; each role binds to one of: ADMIN, RECEPTIONIST. **Trainers and accountants are business records managed by ADMIN and hold no login accounts or role bindings** (AD-13; see [`../progress/phase-3-decisions.md`](../progress/phase-3-decisions.md)).
- **Access rules:** auth events are audited; passwords never stored in logs or custom code paths.

### 2.2 audit
- **Responsibilities:** single append-only `AuditLog` covering authentication events, user/role changes, member/membership changes, payment actions, expense decisions, trainer assignments, attendance sessions, biometric results, door access attempts, administrative overrides.
- **Access:** ADMIN reads; other roles cannot browse the log. No update/delete UI. Secrets and raw biometric data never enter the log (enforced at the emitting service layer).

### 2.3 members
- **Responsibilities:** member registration (auto GMID, required/optional fields), profile management (photo upload, contact data), member lookup, member-side notification events.
- **Access:** RECEPTIONIST (create/update), ADMIN (full). Two-staff-role rule: no other access scopes exist (§2.1).

### 2.4 memberships
- **Responsibilities:** plan catalog (`MembershipPlan` — ADMIN-managed availability and pricing), membership enrollment per member + plan, lifecycle state machine (`PENDING_PAYMENT → ACTIVE → EXPIRED`), snapshotting (agreed price, start/end dates), renewal (new membership, manual), outstanding balance computation.
- **Access:** ADMIN (plans: CRUD; memberships: full), RECEPTIONIST (select plans, register memberships, renewals, view status/outstanding balances; not price changes).

### 2.5 billing
- **Responsibilities:** payments per membership (full & partial), receipt generation with unique numbering, reversal handling, received-by accountability, live outstanding balance derivation.
- **Access:** RECEPTIONIST (record full and partial payments at front desk; issue receipts), ADMIN (full; audited reversals).

### 2.6 attendance
- **Responsibilities:** single visit sessions (check-in/check-out, elapsed time), biometric-triggered auto records (from `access_control`), manual receptionist-assisted operations (authorized, reason-bearing, audited), duplicate/precondition guards, attendance history.
- **Access:** RECEPTIONIST (manual operations, authorized; member check-in/check-out monitoring and history views), ADMIN (full).

### 2.7 access_control
- **Responsibilities:** device registry (biometric devices, door/turnstile controllers), biometric enrollment mapping (member ↔ template reference ↔ device ↔ modality — members only), staff card/PIN access events (individually assigned cards or PINs; ADMIN-managed issuance/revocation; attempts captured in `AccessAttempt` with `identity_kind = STAFF`), verification + authorization decisioning (`AccessAttempt`), unlock-command contract toward the future device gateway, audited receptionist overrides, and fail-closed offline behavior (confirmed 2026-11-06): the gateway holds a **secure offline access cache** whose entries expire and buffers events for **synchronization** on recovery (no entitlement cache in PostgreSQL — see database §3).
- **Access:** ADMIN (full control + overrides review; device registry, access-control configuration, staff card/PIN issuance), RECEPTIONIST (assisted biometric enrollment via authorized workflows, audited manual override — no device/configuration access). This module owns the *hardware boundary specification* — no hardware code in this phase.

### 2.8 trainers
- **Responsibilities:** trainer records (business data only — **no login accounts, no credentials**), member ↔ trainer assignments with append-only history, assignment reporting (current + past).
- **Access:** ADMIN (full: trainer record lifecycle create/update/activate/deactivate, assign/unassign, change/remove assignments), RECEPTIONIST (read-only views of a member's current trainer — member-facing context, no assignment changes).

### 2.9 workouts
- **Responsibilities:** exercise catalog, workout plans composed of exercises (sets/reps/guidance), member plan assignment, training schedule slots, progress/history.
- **Access:** ADMIN (full: plans, exercise library, schedules are ADMIN-managed; no object-level ownership scoping remains since trainers hold no accounts), RECEPTIONIST (none in v1).

### 2.10 equipment
- **Responsibilities:** category registry, item inventory (asset tag, serial, purchase data, condition/status: AVAILABLE / UNDER_MAINTENANCE / DAMAGED / RETIRED), maintenance schedules, repair & service history, maintenance costs.
- **Access:** ADMIN (full), RECEPTIONIST (read — availability info when assisting members).

### 2.11 expenses
- **Responsibilities:** category registry, expense recording by **ADMIN** (consolidated duty), approval workflow (`PENDING_APPROVAL → APPROVED | REJECTED`) with immutable decision records, category totals.
- **Access:** ADMIN (record, review, approve/reject with reason; the recorded expense must be decided by a **different** ADMIN user — BR-EXP-04 object rule), RECEPTIONIST and others: none (financial restriction).

### 2.12 notifications
- **Responsibilities:** event ingestion (registration, payment confirmation, expiry reminders, renewal, attendance confirmation, trainer assignment, announcements), in-app message storage, SMS/email channel orchestration, per-channel delivery tracking (queued/sent/failed), channel enable/disable configuration.
- **Access:** ADMIN (configure), staff in-app inbox by role targeting; system emits for members without requiring member login.

### 2.13 reporting
- **Responsibilities:** daily payment summaries, weekly/monthly revenue, membership payment reports, outstanding balances, expense reports, equipment maintenance costs, P&L, dashboards/KPIs. Read-only module; never mutates financial data.
- **Access:** ADMIN (all reports — financial, operational and management), RECEPTIONIST (basic operational summaries only, e.g., today's attendance — restricted by permission matrix; never sensitive system-wide financial reports).

## 3. Implementation Order (proposed for later phases)

1. **accounts + audit** — staff model, roles, audit plumbing (foundation; security invariants live here first).
2. **members** — registration and GMID issuance.
3. **memberships + billing** — plans, lifecycle, payments, receipts (the commercial core).
4. **attendance** — sessions + manual operations (manual path works before any hardware).
5. **access_control (manual/software scope)** — enrollment mapping, decisions, audit trail; device gateway contract defined but not implemented until hardware is selected.
6. **trainers + workouts** — trainer records and ADMIN-managed assignment and training content (no trainer login onboarding).
7. **equipment** — inventory and maintenance.
8. **expenses** — approval workflow.
9. **notifications** — in-app first, then SMS/email pending provider approval.
10. **reporting** — once source modules exist.
11. **biometric/hardware integration** — strictly behind device-gateway decisions (manufacturer, model, protocol) and safe-failure policy approval.

## 4. Non-Modules (deliberately excluded in v1)

| Excluded | Reason |
|---|---|
| Branch management | Confirmed: single location |
| Member self-service portal/accounts | Not required initially; `Member` is a non-login entity (see `system-architecture.md` AD-02) |
| Automatic charging / auto-renewal | Confirmed: manual renewal only |
| Custom-duration plans | Confirmed: the five fixed plans only |
| Raw biometric data storage | Forbidden by requirements; device-managed templates only |
