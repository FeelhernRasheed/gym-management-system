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
| 8 | `trainers` | `TrainerProfile`, `TrainerAssignment` | `accounts`, `members` |
| 9 | `workouts` | `WorkoutPlan`, `Exercise`, `WorkoutPlanExercise`, `TrainingSchedule` | `accounts`, `members`, `trainers` |
| 10 | `equipment` | `EquipmentCategory`, `Equipment`, `MaintenanceRecord` | `accounts` |
| 11 | `expenses` | `ExpenseCategory`, `Expense`, `ExpenseApproval` | `accounts`, `equipment` (category upkeep records feed maintenance costs) |
| 12 | `notifications` | `Notification`, `NotificationDelivery` | all emitting modules (event producers) |
| 13 | `reporting` | none (read-only aggregations over the above) | most modules (read) |

> Dependency direction is one-way (left → right). Circular dependencies are prohibited; cross-module facts are emitted as audit/notification events rather than direct writes.

## 2. Module Responsibilities & Role Access

### 2.1 accounts
- **Responsibilities:** custom staff `User` model (username, contact data, status), `Role` lookup seeded with the four confirmed roles, staff account lifecycle (create/activate/deactivate), password policy via Django validators.
- **Owners:** ADMIN manages all staff accounts and permissions. Users authenticate to the API/admin; each role binds to one of: ADMIN, RECEPTIONIST, TRAINER, ACCOUNTANT.
- **Access rules:** auth events are audited; passwords never stored in logs or custom code paths.

### 2.2 audit
- **Responsibilities:** single append-only `AuditLog` covering authentication events, user/role changes, member/membership changes, payment actions, expense decisions, trainer assignments, attendance sessions, biometric results, door access attempts, administrative overrides.
- **Access:** ADMIN reads; other roles cannot browse the log. No update/delete UI. Secrets and raw biometric data never enter the log (enforced at the emitting service layer).

### 2.3 members
- **Responsibilities:** member registration (auto GMID, required/optional fields), profile management (photo upload, contact data), member lookup, member-side notification events.
- **Access:** RECEPTIONIST (create/update), ADMIN (full), TRAINER (read of own assigned members only via `trainers`), ACCOUNTANT (member list read for finance workflows).

### 2.4 memberships
- **Responsibilities:** plan catalog (`MembershipPlan` — ADMIN-managed availability and pricing), membership enrollment per member + plan, lifecycle state machine (`PENDING_PAYMENT → ACTIVE → EXPIRED`), snapshotting (agreed price, start/end dates), renewal (new membership, manual), outstanding balance computation.
- **Access:** ADMIN (plans: CRUD; memberships: full), RECEPTIONIST (select plans, register memberships, renewals; not price changes), TRAINER (read assigned members' membership state for training context), ACCOUNTANT (read).

### 2.5 billing
- **Responsibilities:** payments per membership (full & partial), receipt generation with unique numbering, reversal handling, received-by accountability, live outstanding balance derivation.
- **Access:** RECEPTIONIST (record payments at front desk), ADMIN (full), ACCOUNTANT (read/report; no altering payments), TRAINER (none).

### 2.6 attendance
- **Responsibilities:** single visit sessions (check-in/check-out, elapsed time), biometric-triggered auto records (from `access_control`), manual receptionist-assisted operations (authorized, reason-bearing, audited), duplicate/precondition guards, attendance history.
- **Access:** RECEPTIONIST (manual operations; authorized), ADMIN (full), TRAINER (read assigned members' history), ACCOUNTANT (none), MEMBER context shown to trainers only for assigned members.

### 2.7 access_control
- **Responsibilities:** device registry (biometric devices, door/turnstile controllers), biometric enrollment mapping (member ↔ template reference ↔ device ↔ modality), verification + authorization decisioning (`AccessAttempt`), unlock-command contract toward the future device gateway, audited receptionist overrides, safe-failure semantics (documented; code later).
- **Access:** ADMIN (full control + overrides review), RECEPTIONIST (enrollment assistance, manual override with audit), others: none. This module owns the *hardware boundary specification* — no hardware code in this phase.

### 2.8 trainers
- **Responsibilities:** trainer profile data, member ↔ trainer assignments with append-only history, assignment reporting (current + past).
- **Access:** ADMIN (assign/unassign), TRAINER (read own assignments), others (read).

### 2.9 workouts
- **Responsibilities:** exercise catalog, workout plans composed of exercises (sets/reps/guidance), member plan assignment, training schedule slots, progress/history.
- **Access:** TRAINER (create/manage plans & schedules — restricted to own assigned members), ADMIN (full), RECEPTIONIST/ACCOUNTANT (none for financial exposure reasons; read scope unnecessary in v1).

### 2.10 equipment
- **Responsibilities:** category registry, item inventory (asset tag, serial, purchase data, condition/status: AVAILABLE / UNDER_MAINTENANCE / DAMAGED / RETIRED), maintenance schedules, repair & service history, maintenance costs.
- **Access:** ADMIN (full), ACCOUNTANT (read costs for finance), RECEPTIONIST (read), TRAINER (read; trainers need to know equipment availability during planning).

### 2.11 expenses
- **Responsibilities:** category registry, expense recording (accountant), approval workflow (`PENDING_APPROVAL → APPROVED | REJECTED`) with immutable decision records, category totals.
- **Access:** ACCOUNTANT (record; never self-approve), ADMIN (approve/reject with reason; full), others (none; financial restriction).

### 2.12 notifications
- **Responsibilities:** event ingestion (registration, payment confirmation, expiry reminders, renewal, attendance confirmation, trainer assignment, announcements), in-app message storage, SMS/email channel orchestration, per-channel delivery tracking (queued/sent/failed), channel enable/disable configuration.
- **Access:** ADMIN (configure), staff in-app inbox by role targeting; system emits for members without requiring member login.

### 2.13 reporting
- **Responsibilities:** daily payment summaries, weekly/monthly revenue, membership payment reports, outstanding balances, expense reports, equipment maintenance costs, P&L, dashboards/KPIs. Read-only module; never mutates financial data.
- **Access:** ADMIN (all), ACCOUNTANT (financial reports), RECEPTIONIST (basic operational summaries only, e.g., today's attendance — restricted by permission matrix), TRAINER (none).

## 3. Implementation Order (proposed for later phases)

1. **accounts + audit** — staff model, roles, audit plumbing (foundation; security invariants live here first).
2. **members** — registration and GMID issuance.
3. **memberships + billing** — plans, lifecycle, payments, receipts (the commercial core).
4. **attendance** — sessions + manual operations (manual path works before any hardware).
5. **access_control (manual/software scope)** — enrollment mapping, decisions, audit trail; device gateway contract defined but not implemented until hardware is selected.
6. **trainers + workouts** — assignment and training content.
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
