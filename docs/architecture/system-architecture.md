# System Architecture — Gym Management System (Backend)

**Status:** Proposed design (Phase 2). Nothing in this document describes implemented behavior. No code has been written for the modules below; the current repository contains only the verified Django project scaffold from Phase 1.
**Last updated:** 2026-11-06 — two amendments: role consolidation (AD-13) and confirmed-operational-decisions application (AD-14); full record in [`../progress/phase-3-decisions.md`](../progress/phase-3-decisions.md)

---

## 1. Overview

The Gym Management System is a single-location gym platform covering member registration, memberships and payments, access control at the physical entrance, attendance, trainer and workout management, equipment upkeep, expense control, notifications, financial reporting, and a complete audit trail.

The backend is a **Django monolith exposing a REST API via Django REST Framework (DRF)**. A single PostgreSQL database is the authoritative data store. Third-party hardware (biometric readers, door controllers) is integrated through an explicit **device integration boundary** (specification only in this phase) — Django never talks to raw hardware directly.

## 2. Objectives

1. Provide a single source of truth for members, memberships, payments, attendance, staff and financial records.
2. Enforce role-based operation of the gym: what each role sees and can do is a first-class architectural concern.
3. **Reliably control physical entrance access** based on verified identity and active, fully paid membership.
4. Preserve financial and membership **history immutably** — corrections are new audited records, not silent edits.
5. Keep every significant action **auditable** without ever storing secrets or raw biometric samples.
6. Remain monolithic and simple: correct core flows first, avoid distributed-systems complexity until a concrete need exists.

## 3. Scope

### In scope (single location)
One gym site; one centralized database; no branch-management module in the initial version.

### Out of scope (initial version)
Multi-branch functionality, member self-service portal login, automatic charging/renewals, biometric hardware integration code (Phase 2 documents only the boundary), external notification provider configuration.

## 4. Technology Stack

| Layer | Technology | Notes |
|---|---|---|
| Language | Python 3.14.4 | |
| Web framework | Django 6.1.3 | Monolith, ORM-first |
| API | Django REST Framework 3.18.3 | Role-based authorization |
| Database | PostgreSQL 18 | **Confirmed** (2026-11-06, AD-14); installation/connection still belongs to the database phase |
| Database access | Django ORM | No migrations or DB connections were made in Phases 1–2 |
| Hardware | Unselected biometric readers and door/turnstile controllers | Integration defined as a boundary contract, not code |
| Notifications | Channels defined (in-app, SMS, email); providers **not selected** | Provider choice requires approval |
| Version control | Git (+ GitHub origin) | Phase checkpoints commit per phase |

## 5. Architecture

### 5.1 Style: modular monolith

- One deployable Django process (WSGI/ASGI) serving the REST API and Django admin.
- Business domains are separated into Django apps (proposed layout only — see [`modules.md`](modules.md)); in-app module boundaries are enforced by import discipline, not network calls.
- PostgreSQL accessed only through Django ORM; raw SQL only for justified, reviewed optimizations.

### 5.2 Layered structure (per module)

```
HTTP request
  → urls.py routing (APIRouter)
    → DRF View/ViewSet layer   (thin: request/response, permission declarations)
      → service layer          (business rules, workflows, state transitions)
        → model layer / ORM    (persistence and integrity constraints)
          → PostgreSQL
```

Rules:
- Business rules live in services and model-level constraints, **never in views**.
- Serializers validate input shape; services validate business legality.
- Money math (outstanding balances, revenue) is **computed**, not stored redundantly (see [`database-design.md`](../database/database-design.md)).

### 5.3 Proposed module map

| Module (proposed future app) | Purpose |
|---|---|
| accounts | Custom `User` (`role` FK), `Role` |
| members | `Member` (registration, GMID, profile) |
| memberships | `MembershipPlan`, `Membership` lifecycle |
| billing | `Payment`, `Receipt` |
| attendance | `AttendanceSession` (biometric-assisted + manual receptionist ops) |
| access_control | `BiometricDevice`, `BiometricEnrollment`, `AccessDevice`, `AccessAttempt` — the device integration boundary |
| trainers | `Trainer` (business record, no login), `TrainerAssignment` (history) |
| workouts | `WorkoutPlan`, `Exercise`, `WorkoutPlanExercise`, `TrainingSchedule` |
| equipment | `EquipmentCategory`, `Equipment`, `MaintenanceRecord` |
| expenses | `ExpenseCategory`, `Expense`, `ExpenseApproval` |
| notifications | `Notification`, `NotificationDelivery` |
| reporting | Read-only queries/aggregations over other modules (no own domain tables) |
| audit | `AuditLog` (cross-cutting; other modules emit events to it) |

### 5.4 Cross-cutting services

- **AuditService** — every module emits audit events; never calls hardware, never logs secrets.
- **MembershipStatusService** — evaluates and applies `PENDING_PAYMENT → ACTIVE → EXPIRED` transitions (see §6.1).
- **AccessControlCoordinator** — the decision point for entrance access (see §6.5).
- **NotificationDispatcher** — fans out notifications per channel and records per-channel delivery outcomes (see §6.6).

## 6. Key Workflows (design, not implementation)

### 6.1 Membership lifecycle

```
REGISTRATION (receptionist creates Member)
      │ selects plan (existing plan, current price snapshot → Membership)
      ▼
MEMBERSHIP: PENDING_PAYMENT
      │ payment(s) recorded until outstanding = 0   (full or partial payments)
      ▼
ACTIVE  (confirmed 2026-11-06: start_date = date of full payment; end_date = start + plan duration)
      │ passage of time (status service)
      ▼
EXPIRED  (manual renewal creates a NEW membership; automatic charging excluded)
```

- Payments are recorded against the membership regardless of state; a partial payment never forces activation.
- Payment required in full before activation. All history is preserved: a member can hold many historical memberships.

### 6.2 Payment & receipt flow

1. Receptionist records a payment (`amount`, `method`, `paid_at`, `received_by`) linked to a `Membership`.
2. Validation: `0 < amount ≤ outstanding` (no overpayment in v1).
3. A `Receipt` (unique sequential number) is generated for each completed payment.
4. Any correction is performed as an auditable **reversal** record, never by mutating a completed payment.

### 6.3 Trainer management

ADMIN manages trainer business records and assigns/unassigns a member to a trainer; each change creates a new `TrainerAssignment` row (append-only history). **Trainers hold no login accounts and never authenticate** (AD-13); all trainer-record, workout-plan and training-schedule operations flow through ADMIN staff accounts, with RECEPTIONIST limited to read-only member-facing views (e.g., a member's current trainer). Audit events (`TRAINER_ASSIGNED`) continue to capture every assignment change.

### 6.4 Expense approval

```
ADMIN records Expense → PENDING_APPROVAL
      → a different ADMIN user reviews → APPROVED (with decision record) | REJECTED (reason required)
```

After the role consolidation (AD-13) there is no accountant role: recording, reviewing, approving and rejecting expenses are all ADMIN duties. Separation of duties is preserved by the object rule BR-EXP-04 — the recording user can never decide their own expense; a different ADMIN user grants every decision. Decisions are recorded in `ExpenseApproval` (decision, decider, timestamp, reason); approved expenses are locked against silent modification; re-opened items re-enter approval with new decision records.

### 6.5 Physical entrance access (core requirement)

```
Member presents fingerprint / face
  → Biometric device verifies identity (template held on device)
    → Backend AccessControlCoordinator
        1. identity resolution (enrollment → Member)
        2. membership eligibility (ACTIVE + outstanding = 0)
        3. decision → GRANT | DENY (with reason)
  → If granted: integration layer sends unlock command to door/turnstile controller
  → AttendanceSession check-in recorded
  → Exit recorded as check-out on a later verification
```

Every verification / access attempt generates an `AccessAttempt` record (identity, verification result, decision, deny reason, device, timestamp). Denied cases: pending payment, expired membership, unknown identity, device error — all logged. **Staff access events** — individually assigned access cards or PINs as the standard method; staff biometric enrollment is optional (confirmed 2026-11-06, amended 2026-11-06, BR-ACC-11) — are captured by the same `AccessAttempt` chain with `identity_kind = STAFF`, attributed to the individual staff account, and bypass member subscription/payment validation; when staff biometric access is configured, it must identify the staff account and is never treated as member access.

**Integration boundary (no code in this phase):** a dedicated **local access-control service** ("device gateway") — running in or near the gym premises — translates backend authorization into device actions:

- Backend exposes, e.g., `verify-and-authorize` endpoints (or message contract) to the gateway; the gateway implements the device's protocol/API (fingerprint templates, unlock commands, event pushes).
- Django never assumes direct control of devices; manufacturer/model/protocol are **unresolved** (decisions register).
- **Staff access (confirmed 2026-11-06, BR-ACC-11):** staff (ADMIN, RECEPTIONIST) use individually assigned access cards or PINs; issuance/revocation is an ADMIN-managed duty; staff access attempts are audited through the same event chain.
- **Safe failure behavior (CONFIRMED 2026-11-06 — fail-closed):**
  - Backend/network unreachable → the gateway denies scripted online authorization and serves a **secure offline access cache** of recent entitlement data whose entries **expire**; pending events are buffered and **synchronized** — replayed into `AccessAttempt`/`AttendanceSession` — when connectivity returns (audited local events; no entitlement cache in PostgreSQL). Cache expiry window: implementation parameter pending confirmation.
  - Backend error → "deny & manual help desk" — receptionist performs a **manually audited override** (`OVERRIDE` decision recorded with actor + reason).
  - **Emergency exit is a hardware/safety function**: emergency exits must NOT depend on membership validation, the backend, the network, or any software component. Fire-escape doors stay mechanically releasable at all times.

### 6.6 Attendance

- Biometric path: successful entry verification auto-checks the member in; exit is a separate verification auto-check-out.
- Manual path: receptionist performs manual check-in/out ("authorized manual attendance operation"), recorded with actor + reason, audited.
- Guards: no duplicate active session per member; no check-out without open session; no session without active fully-paid membership (strict rule; manual override is an auditable exception).

### 6.7 Notifications

- Event → notification → per-channel deliveries (`IN_APP`, `SMS`, `EMAIL`), each independently configurable.
- Delivery attempts and failures tracked per channel (`NotificationDelivery`).
- Use cases: member registration, payment confirmation, membership expiry reminders, membership renewal, attendance confirmation, trainer assignment, gym announcements.
- No provider is selected or configured (approval required). Email transport is currently Django's console backend (Phase 1 default), unsuitable for production.

### 6.8 Reporting & financial rules

- Revenue = sum of COMPLETED payments (cash-based recognition — **confirmed** 2026-11-06).
- Partial payments remain identifiable; outstanding balances are computed, never stored.
- Pending and rejected expenses are excluded from approved expense totals.
- Corrections flow through audited reversals/approval records (P&L, dashboards, KPIs read aggregated data — never recompute silently).
- Cash basis, KES currency, and payment methods (CASH, MOBILE_MONEY, CARD, BANK_TRANSFER) are **confirmed** (2026-11-06).

## 7. Security Design

- **Secrets:** all sensitive configuration via environment variables (Phase 1 pattern); nothing hardcoded; `.secret_key` is a dev-only git-ignored fallback.
- **Authorization:** DRF permission classes with a role-based model over exactly two staff login roles (see [`permissions.md`](permissions.md)); ADMIN-only gates protect staff management, plan CRUD, equipment, trainers/workouts, expense decisions, device configuration and audit; object-level scoping remains where duties must separate (e.g., an expense recorded by one ADMIN user cannot be approved by the same user).
- **Authentication:** Django session/token auth to be chosen at implementation time; out of scope in Phase 2.
- **PII & photos:** member data is personal data; uploads (photos) stored in server-side media storage with predictable paths and sanitized naming; exposure only through authorized endpoints.
- **Biometrics:** no raw fingerprint images or face images in ordinary records; device-managed templates referenced by ID (see [`business-rules.md`](business-rules.md) BR-BIO section). Secure mapping `Member ↔ Enrollment(template ref, device, modality)`.
- **Audit:** append-only `AuditLog`; logs changes and events; deliberately excludes passwords, biometric raw data, tokens and secrets (write-guard on values).
- **Transport & production hardening:** SSL/HSTS/secure-cookie settings exist as Phase 1 `DEBUG=False` hardening; TLS termination assumed at deployment (documented when deployment phase arrives).

## 8. Design Decisions (summary)

| # | Decision | Rationale |
|---|---|---|
| AD-01 | Monolith architecture | Single gym, small team; clear boundaries between apps provide modularity without operational overhead |
| AD-02 | Django `User` kept as staff/admin actor; `Member` is a separate non-login entity | Members do not manage the system; avoids entangling customer data with auth internals; a future member portal can link later without schema damage |
| AD-03 | Explicit `Role` lookup table + role FK (+ Django groups for granular permissions) | Stable, queryable RBAC for the two confirmed staff login roles (ADMIN, RECEPTIONIST — amended 2026-11-06); room for growth without schema changes |
| AD-04 | Membership unit = one membership record per (member, plan selection) | Enables clean history (endless separate memberships per member), pending-payment state, and renewal-by-new-record |
| AD-05 | Snapshot fields on Membership (`agreed_price`, `start_date`, `end_date`) | Historic memberships must not silently change when plans are edited |
| AD-06 | Outstanding balance and duration never stored — always computed | Eliminates redundant, drift-prone calculated fields (single source: payments vs agreed price) |
| AD-07 | Payment append-only; corrections via reversal records + audit | Financial integrity requirement, auditable corrections |
| AD-08 | `AccessAttempt` unifies biometric verification outcome + door decision | The door flow is one event; avoids near-duplicate verification tables |
| AD-09 | `AttendanceEvent` candidate dropped | Entry/exit fully represented by `AttendanceSession` timestamps + methods; denied attempts belong to `AccessAttempt`; plain duplication otherwise |
| AD-10 | Device integration behind a **gateway service** contract | Hardware variability isolated from core; Django remains device-agnostic |
| AD-11 | Status transitions applied by a maintenance job + evaluated on read at decision points | Guarantees gate-time correctness even if a job lapses |
| AD-12 | AuditLog = generic append-only event log (actor, type, target ref, redacted context) | Covers all §O audit targets with one pattern |
| AD-13 | **Exactly two staff login roles: ADMIN and RECEPTIONIST.** Trainers and accountants hold **no login accounts**; trainer and accounting functionality is preserved as business functionality exercised by ADMIN. Trainer identity moved to a standalone `Trainer` business record (no User FK); expense recorder≠approver object rule replaces the accountant self-approval rule. No third-party user hierarchy or additional roles may be introduced. | Confirmed business requirement (2026-11-06); preserves all 13 modules and trainer/accounting capabilities while shrinking the authentication surface |
| AD-14 | Confirmed operational decisions (2026-11-06): PostgreSQL 18; single currency **KES**; payment methods **CASH, MOBILE_MONEY, CARD, BANK_TRANSFER**; **cash-basis** accounting; membership starts on the **date of full payment**; offline access fails **closed** via a **secure entitlement cache with expiry + event synchronization**; staff physical access via **individual cards or PINs** | User-confirmed decisions; resolve former open items 1, 2, 3, 6, 7 of the Phase 2 unresolved register |

## 9. Terminology

| Term | Meaning |
|---|---|
| **GMID** | Gym Member ID — unique, auto-generated internal member identifier (the physical key for memberships, attendance, biometric enrollment) |
| **Membership** | One member's enrollment in one plan with its own lifecycle and payments |
| **Outstanding balance** | `Membership.agreed_price − sum(COMPLETED payment amounts)` (≥ 0) |
| **Activation** | Membership transition to ACTIVE once outstanding balance = 0; start_date = date of full payment (BR-MEM-13 confirmed 2026-11-06), end_date = start + plan duration |
| **Manual attendance** | Receptionist-recorded entry/exit as an authorized, audited exception |
| **Subscription** | Synonym used in the confirmed requirements; modeled in this design as the `Membership` lifecycle (one member's enrollment in one plan with payment-driven activation) |
| **Template reference** | Device-side ID of a stored biometric template (never raw sample data) |
| **Gateway** | The future local service translating backend decisions into device commands; also hosts the secure offline entitlement cache (expiring entries) and event buffering/synchronization when the backend is unreachable |
| **Offline access cache** | Secure device-gateway-local store of recent entitlement data with entry expiry, used when the backend/network is unavailable (fail-closed — BR-ACC-07); no entitlement cache lives in PostgreSQL |

## 10. Related Documents

- Modules: [`modules.md`](modules.md)
- Rules: [`business-rules.md`](business-rules.md)
- Permissions: [`permissions.md`](permissions.md)
- Database: [`database-design.md`](../database/database-design.md), ER diagram: [`erd.md`](../database/erd.md)
- Phase record: [`../progress/phase-2-architecture.md`](../progress/phase-2-architecture.md)
- Role-consolidation decision record: [`../progress/phase-3-decisions.md`](../progress/phase-3-decisions.md)
