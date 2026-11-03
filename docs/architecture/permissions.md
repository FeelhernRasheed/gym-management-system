# Permissions & Role-Based Access Control Design

**Status:** Authorization design (Phase 2). No implementation exists; this defines the model that DRF authorization will implement in a later phase, in line with the confirmed requirement to implement permissions through DRF authorization + role-based access control.

---

## 1. Roles

Exactly four roles exist in v1 (single gym location):

| Role | Core mandate | Financial access | Administration |
|---|---|---|---|
| ADMIN | Full system access | All | Full (staff, roles, plans, config, reports) |
| RECEPTIONIST | Front-desk operations: members, memberships, payments | Records payments in the payment flow; no financial reports beyond basic daily views | None |
| TRAINER | Assigned members' training info | None | None |
| ACCOUNTANT | Expenses + financial records/reports | Read reports; record expenses; never approve own expense | None (not even system-wide permission management) |

## 2. Role–Permission Matrix

Legend: ✔ allowed · ✖ denied · ◇ partial/conditional (scope noted) · ○ read-only

| Capability area | ADMIN | RECEPTIONIST | TRAINER | ACCOUNTANT |
|---|---|---|---|---|
| Manage staff accounts & roles | ✔ | ✖ | ✖ | ✖ |
| Assign role-based permissions | ✔ | ✖ | ✖ | ✖ |
| System configuration | ✔ | ✖ | ✖ | ✖ |
| Member registration | ✔ | ✔ | ✖ | ✖ |
| Member profile management | ✔ | ✔ | ◇ read (assigned members) | ○ |
| Member photo upload | ✔ | ✔ | ✖ | ✖ |
| Membership plan CRUD (availability & pricing) | ✔ | ✖ (cannot change prices) | ✖ | ✖ |
| Plan catalog read | ✔ | ✔ | ◇ (assigned members' plan) | ○ |
| Create membership (plan selection) | ✔ | ✔ | ✖ | ✖ |
| Membership lifecycle actions (activation via payment, expiry handling) | ✔ | ◇ (payments drive activation) | ✖ | ✖ |
| Record membership payment | ✔ | ✔ | ✖ | ✖ |
| Reverse/adjust payment | ✔ (audited reversal) | ✖ | ✖ | ✖ |
| Receipt generation & read | ✔ | ✔ (own/any receipt) | ✖ | ○ |
| Manual attendance operation | ✔ | ✔ (authorized manual ops) | ✖ | ✖ |
| Attendance history read | ✔ | ○ | ◇ (assigned members only) | ✖ |
| Biometric enrollment (member ↔ template mapping) | ✔ | ◇ (assisted enrollment, audited) | ✖ | ✖ |
| Biometric/device registry | ✔ | ✖ | ✖ | ✖ |
| Door access decision & unlock command path | ✔ (via gateway) | ◇ (audited manual override only) | ✖ | ✖ |
| Door access attempt log (operational view) | ✔ | ○ | ✖ | ✖ |
| Trainer account creation | ✔ | ✖ | ✖ | ✖ |
| Trainer assignment (assign/change/remove) | ✔ | ✖ | ✖ | ✖ |
| Workout plan create/manage | ✔ | ✖ | ◇ (only for assigned members) | ✖ |
| Training schedule manage | ✔ | ✖ | ◇ (only for assigned members) | ✖ |
| Equipment inventory CRUD | ✔ | ✖ | ✖ | ✖ |
| Maintenance schedule / repair records | ✔ | ✖ | ✖ | ✖ |
| Equipment read | ✔ | ○ | ◇ (availability for planning) | ○ (costs) |
| Expense recording | ✖ (records are accountant's job; admin may create when acting in that duty — see note) | ✖ | ✖ | ✔ |
| Expense approval / rejection | ✔ (with reason for rejection) | ✖ | ✖ | ✖ (never own expense) |
| Financial reports (revenue, outstanding, P&L, KPIs) | ✔ | ◇ basic daily summary | ✖ | ✔ |
| Send/configure notifications | ✔ (configuration) | ✖ | ✖ | ✖ |
| Audit log read | ✔ | ✖ | ✖ | ✖ |

> Note: the requirement states the accountant records expenses. ADMIN may also create an expense as a duty overlap, but the self-approval prohibition applies to whoever recorded it.

## 3. Explicit Restriction List (from confirmed requirements)

1. RECEPTIONIST cannot create/modify membership plan prices; can only select plans. → BR-MEM-02
2. TRAINER cannot access financial management or system administration. → BR-TRN-04
3. ACCOUNTANT cannot approve their own expenses. → BR-EXP-04
4. ACCOUNTANT cannot manage system-wide permissions. → role matrix
5. Full payment required before activation (no activation shortcut for any role without payment). Partial payment never counts as full payment. → BR-MEM-04/05
6. Everyone except ADMIN cannot approve/reject expenses. → BR-EXP-02
7. Non-ADMIN cannot read the audit log (recommended; see §5 boundaries). → modules.md §2.2

## 4. API Authorization Architecture

### 4.1 Authentication layer

- DRF authentication will combine: (recommended, deferred decision) session authentication for Django admin/browsable API usage and a stateless token mechanism (DRF token / SimpleJWT — choice deferred to implementation phase) for client apps.
- Credentials: usernames + passwords per staff account; ADMIN-only staff creation (BR-TRN-01).
- All authentication successes/failures emit audit events (BR-AUD-01).

### 4.2 Permission classes (planner view)

| Layer | Class (proposed) | Effect |
|---|---|---|
| Base | `IsAuthenticated` | All business endpoints require a staff identity |
| Role gate | `HasRole("ADMIN")` family | Coarse-grained per-viewset/module gate; roles read from the user's role FK (JWT/session injected) |
| Module gate example | `FinancialDataPermission` | Denies TRAINER for financial endpoints; grants ACCOUNTANT/ADMIN; limits to report reads |
| Object-level | `IsAssignedTrainerOfMember` | Trainer endpoints resolve object member → verify an open/related active assignment exists |
| Approval rule | `NotSameRecorderApprover` | Expense decisions reject `decided_by == Expense.recorded_by` (even for ADMIN-performed recording) |
| Read scoping | ViewSet querysets filtered by role | e.g., trainer viewsets only expose assigned members; receptionist plan endpoints exclude price mutation |

- Enforcement order: authentication → role gate → object-level check → service-layer business validation → DB constraint. Failures: 401/403 with audited access denials where meaningful.
- Django model permissions/Groups: role →Group mapping seeds model-wide permissions (documented recommendation; seeded in the accounts app migration-phase).

### 4.3 Administrative overrides & their boundaries

| Override | Who | Boundary | Audit |
|---|---|---|---|
| Manual attendance (check-in/out in lieu of biometric) | RECEPTIONIST/ADMIN | Only for identities staff can attest; reason mandatory | `AttendanceSession` method=MANUAL + `AuditLog` event |
| Door access override (grant despite objectionable eligibility) | ADMIN (recommended; RECEPTIONIST only if explicitly granted later) | Must record reason; cannot override biometric identity failure (identity must be known) | `AccessAttempt(decision=OVERRIDE, override_by, reason)` + `AuditLog` |
| Payment reversal | ADMIN only | New reversal payment record linked to original; original immutable | `AuditLog` |
| Expense correction/re-approval | ADMIN | Original decision retained as history; corrections add new records | `AuditLog` |
| Membership manual start-date correction | ADMIN ([UNRESOLVED policy — BR-MEM-13]) | Never used to skip validation for paid services | `AuditLog` |

No role obtains a "silent" path: every override writes to the audit log with actor, reason, and target object references.

## 5. Boundaries of Administrative Power (guardrails)

- ADMIN actions that alter financial truth (reversal, price change, expense decisions) all generate audit events — no exception path exists.
- Even ADMIN cannot grant themself the ACCOUNTANT-approval of a self-recorded expense (object rule 3).
- Audit log is append-only from the API layer; DB-level protections (e.g., trigger-based immutability) are a documented recommendation for the database phase.
- Staff deactivation (not deletion) preserves all history and audit attribution.
