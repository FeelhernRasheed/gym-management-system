# Phase 2 — System Architecture, Database Design and Git Checkpoint

- **Date:** 2026-11-03
- **Amended:** 2026-11-06 — role-consolidation corrections applied (D-03, D-14); full record in [`phase-3-decisions.md`](phase-3-decisions.md)
- **Type:** Documentation-only phase (no code, no models, no migrations, no DB access, no dependency changes).
- **Entry state:** branch `master`, commit `f87c6a8 chore: initialize Django gym management backend`, working tree clean. Remote `origin` (github.com/FeelhernRasheed/gym-management-system) was configured outside this session and points to the same commit; no fetch/push operations were performed.

---

## 1. Phase Summary

All confirmed architectural requirements were analyzed and translated into seven professional design documents: system architecture, module catalog, business rules, permission design, database design, an ER diagram, and this phase record. Candidates from the required entity list were evaluated — **27 entities finalized**, `AttendanceEvent` removed after review (entry/exit fully covered by `AttendanceSession`; denied flow covered by `AccessAttempt`). Terminology, state machines, and cross-document references were checked against each other.

**Nothing is implemented.** Every document explicitly marks design vs confirmed-requirement vs recommendation vs open decision.

## 2. The 16 Confirmed Decisions (register)

| # | Decision (confirmed) | Where documented |
|---|---|---|
| D-01 | Single gym location | system-architecture §3; business-rules header |
| D-02 | One centralized database; no branch-management module | system-architecture §3, modules §1/§4 |
| D-03 | Two staff login roles (ADMIN, RECEPTIONIST) with DRF role-based authorization — **amended 2026-11-06: originally recorded as four roles (ADMIN, RECEPTIONIST, TRAINER, ACCOUNTANT); the TRAINER and ACCOUNTANT login roles were retired while their functionality was preserved under ADMIN** | permissions §1–4; modules §2.1; phase-3-decisions.md |
| D-04 | Five fixed plans (DAILY 1d, WEEKLY 7d, MONTHLY 1mo, QUARTERLY 3mo, YEARLY 12mo); no custom durations; ADMIN controls pricing/availability | business-rules BR-MEM-01/02; database 2.3 |
| D-05 | Full + partial payments; outstanding balances tracked; individual payment records; receipts; partial never counts as full | business-rules BR-MEM-04..08; database 2.3 |
| D-06 | Full payment required before activation | business-rules BR-MEM-04; permissions §5 |
| D-07 | Lifecycle PENDING_PAYMENT → ACTIVE → EXPIRED | system-architecture §6.1; database 2.3 |
| D-08 | Manual renewal only; no automatic charging | business-rules BR-MEM-09 |
| D-09 | Previous membership & payment history preserved | database §4; system-architecture §6.1 |
| D-10 | Member registration fields (required/optional) with auto-generated unique GMID | business-rules BR-REG-*; database 2.2 |
| D-11 | Attendance modes (biometric + manual receptionist ops) and guards (duplicate sessions, orphan check-out, eligibility) | business-rules BR-ATT-*; database 2.4 |
| D-12 | Biometrics: fingerprint + facial; no raw images; device-managed templates; secure mapping; method recorded; attempts audited | business-rules BR-BIO-*; database 2.4 |
| D-13 | Physical entrance control core workflow + deny rules + audited override + emergency-exit independence + safe-failure documentation — **amended 2026-11-06: offline fail-closed with a secure access cache (expiry + event synchronization) and staff card/PIN access confirmed** | business-rules BR-ACC-*; system-architecture §6.5 |
| D-14 | Trainer management: admin-driven trainer records and assignments with history (trainer records hold no login accounts — amended 2026-11-06; former "assigned-member-only trainer visibility" retired with the trainer login role) | business-rules BR-TRN-*; database 2.5 |
| D-15 | Workout management (plans, exercises, schedules, progress) | business-rules §6.1; database 2.5 |
| D-16 | Equipment full lifecycle (catalog, identification, purchases, condition, maintenance, repairs, costs, statuses) | business-rules BR-EQP-*; database 2.6 |

Expense workflow, notifications, finance-reporting rules, and audit logging are part of the confirmed set as well; they are fully specified in business-rules BR-EXP-*, BR-NTF-*, BR-FIN-*, BR-AUD-* and mirrored in permissions §2 and system-architecture §6 — the register above groups them under the brief's section map without dropping any confirmed requirement.

## 3. Documents Created (7)

| File | Content |
|---|---|
| `docs/architecture/system-architecture.md` | Overview, objectives, scope, stack, layered monolith, module map, workflows (membership, payment, access door flow, expense, notifications, reporting), security, integration boundary, 12 numbered architecture decisions |
| `docs/architecture/modules.md` | 13 proposed modules, responsibilities, dependencies, role access, 11-step implementation order, exclusions |
| `docs/architecture/business-rules.md` | 80+ rules with IDs and status tags [CONFIRMED]/[REC]/[UNRESOLVED] across 11 domains |
| `docs/architecture/permissions.md` | Role matrix (25+ capability rows), explicit restrictions, DRF authorization architecture, override boundaries & guardrails |
| `docs/database/database-design.md` | Conventions, 27-entity catalog with fields/types/keys/FK policy/counts, candidate evaluation, integrity & history rules, seed data plan, PostgreSQL notes |
| `docs/database/erd.md` | Full Mermaid ER diagram (27 entities), FK dictionary (every edge mapped to a field + on_delete), cardinality justifications |
| `docs/progress/phase-2-architecture.md` | This file |

## 4. Unresolved Decisions Register (status updated 2026-11-06 — second amendment)

**Resolved as CONFIRMED (2026-11-06 — decisions P3-07…P3-11; full record in [`phase-3-decisions.md`](phase-3-decisions.md) §7):**

| # | Decision (now confirmed) | Where documented |
|---|---|---|
| 1 | PostgreSQL 18 (was 17-vs-18 open) | P3-07; system-architecture §4; database §6 |
| 2 | Membership starts on the date full payment is received | P3-11; BR-MEM-13; database 2.3 |
| 3 | Single currency KES; payment methods CASH, MOBILE_MONEY, CARD, BANK_TRANSFER | P3-08/09; BR-MEM-14/15 |
| 6 | Offline access fails closed; secure offline access cache with expiry; buffered event synchronization | P3-12; BR-ACC-07; system-architecture §6.5 |
| 7 | Cash-basis accounting | P3-10; BR-FIN-05 |

**Still open (unchanged by the amendments):**

| # | Open decision | Recommendation recorded |
|---|---|---|
| 4 | Biometric device manufacturer & model | Generic registry fields only; no invented specs |
| 5 | Door controller protocol / integration method | Abstract gateway contract; hardware decides protocol |
| 8 | Multiple active trainer assignments per member | Primary + co-trainers recommended (BR-TRN-05); affects unique-constraint design |
| 9 | Same vs separate entry/exit terminals | Both supported by the domain model |

Additional minor open items (flagged inside documents, not blocking): member phone uniqueness (BR-REG-06), minor/age policy (BR-REG-07), overpayment/credit handling (BR-MEM-06), notification provider selection (approval required), audit retention policy (BR-AUD-05), member portal future linkage (AD-02), offline-cache expiry window + refresh policy (implementation parameter — P3-12).

## 5. Risks & Assumptions

**Assumptions (documented, pending confirmation):** adult-only registration; latest `ExpenseApproval` row governs mutable fields; recurring weekly training schedules (no exceptions calendar in v1); serialized rows = one unit for equipment. (Single currency KES was confirmed 2026-11-06 — now documented as confirmed, no longer an assumption.)

**Risks:** hardware dependency uncertainty (device protocol could constrain the gateway contract); offline trade-off policy could conflict with venue expectations (revenue vs security); biometric PII obligations under local data-protection regulation; single-staff-held `MEDIA` photos require access control when endpoints materialize; timezone/PII regulations demand retention policy for audit data.

## 6. Quality Checks (this phase)

- Cross-document consistency: the 27 entity names are identical in `database-design.md` and `erd.md` (both the diagram blocks and FK dictionary); the same terms are used across rules, permissions and modules; state values (PENDING_PAYMENT/ACTIVE/EXPIRED, PENDING_APPROVAL/APPROVED/REJECTED, COMPLETED/REVERSED, channel and device enums) are consistent everywhere.
- Mermaid syntax reviewed block-by-block (entity names, cardinality symbols, quoted labels); fence balance and diagram/greeting correctness confirmed.
- No code, no settings changes, no dependency changes, no DB access — boundaries respected.

## 7. Git Checkpoint Status

- Working tree after documentation: only the seven new documentation files are present (verified via `git status`; reviewed before this report).
- Nothing staged, **no commit created** — awaiting explicit user review/approval.
- Proposed commit message: `docs: add gym management system architecture and database design`
- Previous commit `f87c6a8` remains intact; nothing amended or rewritten; no push authorized or performed.

## 8. Approval Status

Phase 2 documentation **awaiting user review**. On approval: stage only the seven files, re-check `git diff --cached`, commit with the proposed message, report hash and clean status. Phase 3 does not start without explicit authorization.

## 9. Amendment Log

- **2026-11-06 — Role consolidation (documentation correction, no code):** the confirmed requirement "exactly two staff login roles (ADMIN, RECEPTIONIST)" was applied across all Phase 2 documents. Register entries D-03 and D-14 were updated above; application details, rationale, and affected-file inventory live in [`phase-3-decisions.md`](phase-3-decisions.md). Trainer and accountant functionality remains available as ADMIN-managed business modules; no implementation artifacts, migrations, or databases were touched.
- **2026-11-06 (second amendment — confirmed operational decisions):** PostgreSQL 18; KES; payment-method codes CASH/MOBILE_MONEY/CARD/BANK_TRANSFER; cash-basis accounting; membership start = full-payment date; offline fail-closed with secure access cache (expiry + event synchronization); staff card/PIN physical access. Former open items 1, 2, 3, 6, 7 resolved (register §4). All edits documentation-only.
