# Permissions & Role-Based Access Control Design

**Status:** Authorization design (Phase 2). No implementation exists; this defines the model that DRF authorization will implement in a later phase, in line with the confirmed requirement to implement permissions through DRF authorization + role-based access control.
**Last updated:** 2026-11-06 — two amendments: (1) role consolidation: exactly two staff login roles (ADMIN, RECEPTIONIST); trainer and accountant functionality preserved as ADMIN-managed functionality (AD-13); (2) confirmed operational decisions: PostgreSQL 18, KES, payment-method codes, cash basis, membership start = full-payment date, offline fail-closed with secure cache, staff card/PIN physical access (AD-14). Full record: [`../progress/phase-3-decisions.md`](../progress/phase-3-decisions.md).

---

## 1. Roles

Exactly **two** staff login roles exist in v1 (single gym location):

| Role | Core mandate | Financial access | Administration |
|---|---|---|---|
| ADMIN | Full system authority: staff, members, memberships, payments, expenses, equipment, trainers, workouts, biometrics/access control, reports, settings, audit | All (records, manages, approves/rejects expenses; reversals; full reports) | Full (staff accounts, roles/permissions, plans, configuration, audit) |
| RECEPTIONIST | Authorized front-desk operations: member registration/profiles, membership purchases & renewals, full and partial payments, receipts, membership status/outstanding balances, attendance monitoring, assisted biometric enrollment, relevant member information | Records payments in the payment flow; issues receipts; views membership status and outstanding balances; no financial approvals and no sensitive system-wide financial reports | None |

**Role-consolidation rule (confirmed 2026-11-06):**

- There is **no TRAINER login role** and **no ACCOUNTANT login role**; neither exists as a staff account.
- Trainers exist as **business records** (`Trainer`, ADMIN-managed) — no credentials, no login, no API role. Trainer assignment and workout/training management are performed by ADMIN.
- Accounting functionality (expense recording, review, approval/rejection, financial records) is exercised by **ADMIN**, subject to the same recorder≠approver object rule (BR-EXP-04).
- No third-party user hierarchy or additional login role may be introduced.

## 2. Role–Permission Matrix

Legend: ✔ allowed · ✖ denied · ◇ partial/conditional (scope noted) · ○ read-only

| Capability area | ADMIN | RECEPTIONIST |
|---|---|---|
| Manage staff accounts (create/update/activate/deactivate) | ✔ | ✖ |
| Manage receptionist accounts & assign role-based permissions | ✔ | ✖ |
| System configuration & settings | ✔ | ✖ |
| Member registration | ✔ | ✔ |
| Member profile management (update, photo, contact data) | ✔ | ✔ |
| Member info read (relevant member information) | ✔ | ✔ |
| Membership plan CRUD (availability & pricing) | ✔ | ✖ (cannot change prices) |
| Plan catalog read | ✔ | ✔ |
| Create membership / purchase & renewal (plan selection) | ✔ | ✔ |
| Membership lifecycle actions (activation via payment, expiry handling) | ✔ | ◇ (payments drive activation) |
| Membership status & outstanding balance read | ✔ | ✔ |
| Record full & partial membership payment | ✔ | ✔ |
| Reverse/adjust payment | ✔ (audited reversal) | ✖ |
| Receipt generation & issue | ✔ | ✔ |
| Manual attendance operation (authorized, audited) | ✔ | ✔ |
| Attendance & check-in/check-out monitoring, attendance history read | ✔ | ✔ |
| Biometric enrollment (member ↔ template mapping), assisted workflows | ✔ | ◇ (assisted enrollment, audited) |
| Biometric/device registry & access-control configuration | ✔ | ✖ |
| Staff access card/PIN assignment & revocation (individual issuance) | ✔ | ✖ |
| Door access decision & unlock command path | ✔ (via gateway) | ◇ (audited manual override only) |
| Door access attempt log (operational view) | ✔ | ✔ |
| Trainer record creation/update/activation (no login accounts) | ✔ | ✖ |
| Trainer assignment (assign/change/remove) | ✔ | ✖ |
| View member's current trainer assignment (member-facing context) | ✔ | ○ (read-only) |
| Workout plan & exercise library create/manage | ✔ | ✖ |
| Training schedule manage | ✔ | ✖ |
| Equipment inventory CRUD | ✔ | ✖ |
| Maintenance schedule / repair records | ✔ | ✖ |
| Equipment read | ✔ | ○ |
| Expense recording | ✔ (ADMIN duty since consolidation; recorder may never self-approve — see note) | ✖ |
| Expense review / approval / rejection | ✔ (with reason for rejection) | ✖ |
| Financial reports (revenue, outstanding, P&L, KPIs) | ✔ | ✖ (basic daily operational summary only) |
| Send/configure notifications | ✔ (configuration) | ◇ (receives in-app operational items) |
| Audit log read | ✔ | ✖ |

> Note: since the 2026-11-06 role consolidation, expense recording is an ADMIN duty. The self-approval prohibition still applies to *whoever recorded the expense*: an expense recorded by one ADMIN user must be decided (approved/rejected) by a **different** ADMIN user (BR-EXP-04).

## 3. Explicit Restriction List (from confirmed requirements)

1. RECEPTIONIST cannot create/modify membership plan prices; can only select plans. → BR-MEM-02
2. RECEPTIONIST cannot create, update, activate/deactivate staff accounts, manage receptionist accounts, or assign role-based permissions. → role matrix
3. RECEPTIONIST cannot configure system settings/system-wide configuration. → role matrix
4. RECEPTIONIST cannot record, review, approve or reject expenses, and has no access to sensitive financial reports (P&L, revenue, KPIs); only a basic daily operational summary. → BR-EXP-02, role matrix
5. RECEPTIONIST cannot manage biometric devices, access-control configuration, or enrollment permissions; may only assist member enrollment through the authorized, audited workflow. → BR-BIO-*, role matrix
6. The user who records an expense can NEVER approve it; a different ADMIN user must decide (object-level rule separate from role permissions). → BR-EXP-04
7. Only ADMIN can approve/reject expenses; no other role obtains any approval path. → BR-EXP-02
8. Full payment required before activation (no activation shortcut for any role without payment). Partial payment never counts as full payment. → BR-MEM-04/05
9. Non-ADMIN cannot read the audit log (recommended; see §5 boundaries). → modules.md §2.2
10. Trainers and accountants hold no staff accounts: no login identity exists for them, so no permission can ever be granted to a "trainer" or "accountant" principal. → §1, AD-13

## 4. API Authorization Architecture

### 4.1 Authentication layer

- DRF authentication will combine: (recommended, deferred decision) session authentication for Django admin/browsable API usage and a stateless token mechanism (DRF token / SimpleJWT — choice deferred to implementation phase) for client apps.
- Credentials: usernames + passwords per staff account; staff creation/activation/deactivation is ADMIN-only (role matrix §2). Exactly two login roles are seeded (ADMIN, RECEPTIONIST); trainers and accountants hold no accounts (§1, AD-13).
- All authentication successes/failures emit audit events (BR-AUD-01).

### 4.2 Permission classes (planner view)

| Layer | Class (proposed) | Effect |
|---|---|---|
| Base | `IsAuthenticated` | All business endpoints require a staff identity |
| Role gate | `HasRole("ADMIN")` family | Coarse-grained per-viewset/module gate; roles read from the user's role FK (JWT/session injected). ADMIN-only gates protect staff management, plan CRUD, expense decisions, equipment, trainers/workouts, device configuration, audit |
| Financial gate | `FinancialDataPermission` | Restricts financial endpoints (payments beyond recording, expense decisions, financial reports) to ADMIN; the receptionist path exposes only front-desk recording + receipt issuing + basic daily summary |
| Approval rule | `NotSameRecorderApprover` | Expense decisions reject `decided_by == Expense.recorded_by` (a different ADMIN user must decide), preserving separation of duties after consolidation |
| Read scoping | ViewSet querysets filtered by role | e.g., receptionist plan endpoints exclude price mutation; receptionist reporting endpoints exclude sensitive financial aggregates |

- Enforcement order: authentication → role gate → object-level check → service-layer business validation → DB constraint. Failures: 401/403 with audited access denials where meaningful.
- Django model permissions/Groups: role →Group mapping seeds model-wide permissions only for the two seeded roles (documented recommendation; seeded in the accounts app migration-phase).
- **Guarantee:** the backend authorization design prevents every receptionist path to ADMIN-only operations — no permission group, django-admin route, or API endpoint grants the RECEPTIONIST role staff management, financial approval, device configuration, or system-settings capabilities.

### 4.3 Administrative overrides & their boundaries

| Override | Who | Boundary | Audit |
|---|---|---|---|
| Manual attendance (check-in/out in lieu of biometric) | RECEPTIONIST/ADMIN | Only for identities staff can attest; reason mandatory | `AttendanceSession` method=MANUAL + `AuditLog` event |
| Door access override (grant despite objectionable eligibility) | ADMIN (recommended; RECEPTIONIST only if explicitly granted later) | Must record reason; cannot override biometric identity failure (identity must be known) | `AccessAttempt(decision=OVERRIDE, override_by, reason)` + `AuditLog` |
| Payment reversal | ADMIN only | New reversal payment record linked to original; original immutable | `AuditLog` |
| Expense correction/re-approval | ADMIN | Original decision retained as history; corrections add new records | `AuditLog` |
| Membership manual start-date correction | ADMIN (policy CONFIRMED 2026-11-06 — start_date = date of full payment, BR-MEM-13) | Never used to skip validation for paid services | `AuditLog` |

No role obtains a "silent" path: every override writes to the audit log with actor, reason, and target object references.

## 5. Boundaries of Administrative Power (guardrails)

- ADMIN actions that alter financial truth (reversal, price change, expense decisions) all generate audit events — no exception path exists.
- Even ADMIN cannot approve an expense they recorded themself; approval must come from a **different** ADMIN user (BR-EXP-04 object rule).
- Audit log is append-only from the API layer; DB-level protections (e.g., trigger-based immutability) are a documented recommendation for the database phase.
- Staff deactivation (not deletion) preserves all history and audit attribution.
