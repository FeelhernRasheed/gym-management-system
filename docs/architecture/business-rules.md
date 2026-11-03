# Business Rules — Gym Management System

**Status:** Rule catalog derived from the confirmed requirements (Phase 2). Rules marked **[CONFIRMED]** come directly from the user's decisions; **[REC]** are architect recommendations proposed for confirmation; **[UNRESOLVED]** flag open policy questions (tracked in [`../progress/phase-2-architecture.md`](../progress/phase-2-architecture.md)). No rule is implemented yet.

Rule ID convention: `BR-<DOMAIN>-<NN>` with domains MEM (memberships/payments), REG (registration), ATT (attendance), BIO (biometrics), ACC (physical access), TRN (trainers), EXP (expenses), NTF (notifications), EQP (equipment), FIN (finance/reporting), AUD (audit).

---

## 1. Membership & Payment

| ID | Rule | Status |
|---|---|---|
| BR-MEM-01 | Exactly five predefined plans exist, identified by fixed codes: `DAILY` (1 day), `WEEKLY` (7 days), `MONTHLY` (1 month), `QUARTERLY` (3 months), `YEARLY` (12 months). No custom-duration plans are supported. | [CONFIRMED] |
| BR-MEM-02 | ADMIN manages plan availability (`is_active`) and pricing. RECEPTIONIST selects existing plans only and cannot alter prices. | [CONFIRMED] |
| BR-MEM-03 | Plan durations are calendar-accurate: monthly → add 1 calendar month; quarterly → add 3; yearly → add 12. Durations are never approximated as fixed day counts (leap-year safe). | [REC — derives from confirmed duration semantics] |
| BR-MEM-04 | A membership is created with `PENDING_PAYMENT` status; it becomes `ACTIVE` only once its outstanding balance reaches zero via completed payments. | [CONFIRMED] |
| BR-MEM-05 | Partial payments are supported and recorded individually; a partial payment never changes state to ACTIVE, and a provisional state never exports "activation by partial payment" to any downstream consumer. | [CONFIRMED] |
| BR-MEM-06 | Payment amount is restrained to `0 < amount ≤ outstanding balance`. Overpayment (credit) is not supported in v1 — attempting to pay beyond outstanding is rejected. | [CONFIRMED core] with [UNRESOLVED] overpayment handling |
| BR-MEM-07 | Outstanding balance = membership `agreed_price − Σ COMPLETED payment amounts`. It is always derived at read time, never stored. | [CONFIRMED concept; REC implementation] |
| BR-MEM-08 | A member may hold multiple memberships over time (history preserved). At most one membership may be ACTIVE at a given moment; renewals create new memberships rather than extending old ones. | [CONFIRMED multi; REC single-active] |
| BR-MEM-09 | Manual renewal only. No automatic charging, no recurring billing. | [CONFIRMED] |
| BR-MEM-10 | Inactive/EXPIRED memberships keep all payment records and receipts; the price, start date and end date snapshotted on the membership never change if plans become invalid or re-priced later. | [CONFIRMED] |
| BR-MEM-11 | Membership price is captured as `agreed_price` at enrollment time (price at plan selection), independent of later plan price changes. | [REC] |
| BR-MEM-12 | Expiry: an ACTIVE membership whose `end_date` has passed becomes EXPIRED via the membership status service (planned daily maintenance job + on-read evaluation at access/decision points). | [REC] |
| BR-MEM-13 | **Membership start-date policy is UNRESOLVED**: two candidate policies exist — (a) membership starts on the registration/enrollment date, (b) membership starts on the moment of full payment. Impact analysis: earlier start under policy (a) shortens paid-up utilization if payment comes late. **Recommendation: policy (b) — the paid window should start when payment completes; not silently finalized.** | [UNRESOLVED] |
| BR-MEM-14 | Payment method set is not finalized. Candidate set (recommendation only): CASH, MOBILE MONEY, CARD, BANK TRANSFER. Methods recorded per payment; currency-related terms are tracked in the unresolved register. | [UNRESOLVED] |
| BR-MEM-15 | Currency is not finalized (recommendation: single-currency gym, e.g., KES given Africa/Nairobi context); amounts stored as decimal money values; multi-currency support deferred until confirmed. | [UNRESOLVED] |
| BR-MEM-16 | Receipt: every COMPLETED payment yields a receipt with a unique sequential number; a payment has at most one receipt. | [CONFIRMED receipts; REC numbering] |

## 2. Member Registration

| ID | Rule | Status |
|---|---|---|
| BR-REG-01 | Each member receives a unique, automatically generated **Gym Member ID (GMID)** at registration; GMID is the linkage key for memberships, attendance and biometric enrollment. | [CONFIRMED] |
| BR-REG-02 | GMID format (recommendation): sequential `GM-000001 …`; format must remain stable; GMID is immutable after issuance. | [REC] |
| BR-REG-03 | Required fields: full name, gender, date of birth, phone number, physical address, emergency contact name, emergency contact phone, registration date. Registration date is system-generated. | [CONFIRMED] |
| BR-REG-04 | Optional fields: email, profile photograph. Email must be unique when present (null email allowed — PostgreSQL multi-null uniqueness). | [CONFIRMED with REC uniqueness] |
| BR-REG-05 | Emergency contact phone is free-form (family members may share numbers); uniqueness not enforced on it. | [REC] |
| BR-REG-06 | Member phone number uniqueness recommended (business SMS/reminder key), flagged for confirmation. | [REC/UNRESOLVED] |
| BR-REG-07 | Minimum-age/minor policy is not specified. Assumption for v1: adult (18+) registration; a minor policy is required before launch. | [UNRESOLVED] |
| BR-REG-08 | Photographs are personal data: stored outside the database (server media storage), path-scoped per member, served only through authorized endpoints; no claims about current implementation — storage design is documentation-stage. | [REC] |

## 3. Attendance

| ID | Rule | Status |
|---|---|---|
| BR-ATT-01 | A member may hold at most ONE open (not yet checked-out) attendance session; a second check-in for the same member is rejected. | [CONFIRMED — "prevent duplicate active sessions"] |
| BR-ATT-02 | Check-out impossible without an open session for that member. | [CONFIRMED] |
| BR-ATT-03 | Attendance requires an ACTIVE membership with outstanding balance zero (fully paid). Pending-payment and expired members cannot start sessions; a manual receptionist override is an authorized, audited exception. | [CONFIRMED; override mechanics REC] |
| BR-ATT-04 | Both biometric and manual operations record: entry timestamp, entry method (fingerprint / facial / manual), exit timestamp, exit method. Session elapsed time = exit − entry (derived, not stored). | [CONFIRMED support; REC derivation] |
| BR-ATT-05 | Manual attendance records carry the actor (receptionist) and a mandatory reason; durations are computed identically, and the record is explicitly labeled `MANUAL` by its method field. | [REC] |
| BR-ATT-06 | Attendance history per member is permanent (never deleted; corrections via audited adjustments only). | [REC] |

## 4. Biometric Verification

| ID | Rule | Status |
|---|---|---|
| BR-BIO-01 | Fingerprint and facial verification are supported modalities. | [CONFIRMED] |
| BR-BIO-02 | Raw fingerprint images and raw facial images are never stored in attendance records — or anywhere in this application's data store. The system stores only device-side template references. | [CONFIRMED] |
| BR-BIO-03 | Biometric templates are managed and matched by the biometric devices themselves (edge matching); Django handles mapping and authorization, never template matching. | [CONFIRMED architectural stance] |
| BR-BIO-04 | Secure mapping: `Member ↔ (device, modality, template reference, status)`. The mapping row is created at enrollment and revoked (never re-used) on removal; a member may be enrolled on multiple devices. | [CONFIRMED] |
| BR-BIO-05 | Verification method is recorded per event; successful AND failed verification attempts are audited (see BR-ACC-04, BR-AUD rules). | [CONFIRMED] |
| BR-BIO-06 | Hardware (brand, model, matching SDK, template storage) is UNRESOLVED; no device specifications may be invented; the device registry keeps generic manufacturer/model text only. | [UNRESOLVED] |

## 5. Physical Entrance Access Control

| ID | Rule | Status |
|---|---|---|
| BR-ACC-01 | Entry decision requires: verified identity (biometric match) + membership ACTIVE + outstanding balance zero. | [CONFIRMED] |
| BR-ACC-02 | Expired membership → deny entry. Pending payment → deny entry. | [CONFIRMED] |
| BR-ACC-03 | Unknown identity (no match / no enrollment) → deny with `UNKNOWN_IDENTITY` reason. | [CONFIRMED deny; REC reason code] |
| BR-ACC-04 | All access attempts — granted, denied, and overridden — are logged with identity, verification result, decision, reason, device, and timestamp. | [CONFIRMED] |
| BR-ACC-05 | Receptionist override is possible and only IF specifically authorized (permission matrix) and always produces an audited override record (actor + reason). | [CONFIRMED audited override permissioning] |
| BR-ACC-06 | **Emergency exit must never depend on membership validation, the backend, or internet connectivity**; emergency exits are mechanical safety functions outside the authorization path. | [CONFIRMED] |
| BR-ACC-07 | When the backend, network, or hardware is unavailable the system must fail safe. Recommendation: automated authorization defaults to deny; the gateway buffers events and replays them on recovery; receptionist manual override is the documented human fallback. Final policy (fail-open vs fail-closed trade-offs) is UNRESOLVED. | [UNRESOLVED — REC fail-closed + replay] |
| BR-ACC-08 | Door/turnstile unlock is executed by the device gateway service, not by Django directly; the backend emits authorization events and does not talk to raw hardware. | [CONFIRMED boundary] |
| BR-ACC-09 | Whether entry and exit use the same biometric terminal or separate entry/exit readers is UNRESOLVED. The domain model records entry and exit as separate events so both topologies are supported. | [UNRESOLVED] |
| BR-ACC-10 | Door controller protocol/API is UNRESOLVED — no hardware assumptions; integration is an abstract contract (authorize/deny/unlock command + event callback) until a controller is selected. | [UNRESOLVED] |

## 6. Trainer Management & Workouts

| ID | Rule | Status |
|---|---|---|
| BR-TRN-01 | ADMIN creates trainer staff accounts (no self-service trainer registration). | [CONFIRMED] |
| BR-TRN-02 | ADMIN assigns members to trainers, changes assignments, and removes them; every change appends to assignment history (never destructive editing). | [CONFIRMED] |
| BR-TRN-03 | Trainers access ONLY their assigned members' training information (workout plans, schedules, attendance history). Trainer access is enforced at the API object level. | [CONFIRMED] |
| BR-TRN-04 | Trainers cannot access financial management or system administration in any form. | [CONFIRMED] |
| BR-TRN-05 | Whether a member may be assigned to multiple trainers simultaneously is pending — recommendation: primary trainer + optional co-trainers. Affects the uniqueness constraint design. | [UNRESOLVED] |
| BR-TRN-06 | Assignment history is immutable: rows close with an end date/cause (e.g., reassigned, removed); past assignments are never deleted. | [REC] |

### 6.1 Workout content

| ID | Rule | Status |
|---|---|---|
| BR-WKO-01 | Workout plans are created by trainers for their own assigned members; ADMIN may view/manage all. | [CONFIRMED scope; REC ownership check] |
| BR-WKO-02 | A plan contains exercises with per-line guidance (e.g., sets/reps/notes) and ordering; the exercise library is shared across the gym. | [REC] |
| BR-WKO-03 | Training schedules bind trainer + member + time slot; collisions (same member double-booked) are validated at save time. | [REC] |

## 7. Equipment

| ID | Rule | Status |
|---|---|---|
| BR-EQP-01 | Equipment lifecycle state set: `AVAILABLE`, `UNDER_MAINTENANCE`, `DAMAGED`, `RETIRED`. | [CONFIRMED] |
| BR-EQP-02 | Individual machines carry an identity (asset tag / serial); bulk/consumable items may be tracked as quantity lines (see database design). | [CONFIRMED tracking individuals; REC quantity] |
| BR-EQP-03 | Purchase information (date, cost), condition and service history are preserved per item. | [CONFIRMED] |
| BR-EQP-04 | Maintenance schedules generate records; repair records, maintenance costs, and service history are kept for reporting (financial module consumes them). | [CONFIRMED] |

## 8. Expenses

| ID | Rule | Status |
|---|---|---|
| BR-EXP-01 | ACCOUNTANT records expenses from confirmed categories: RENT, UTILITIES (electricity & water), STAFF_SALARIES, EQUIPMENT_MAINTENANCE, CLEANING, INTERNET, OTHER. | [CONFIRMED] |
| BR-EXP-02 | Workflow: `PENDING_APPROVAL → APPROVED | REJECTED`. Only ADMIN decides; decisions (who, when, decision, reason) are stored as immutable `ExpenseApproval` records. | [CONFIRMED] |
| BR-EXP-03 | Rejection requires a documented reason. | [CONFIRMED] |
| BR-EXP-04 | An accountant can NEVER approve their own expense (object-level rule separate from role permissions). | [CONFIRMED] |
| BR-EXP-05 | Approved expenses are protected from silent modification; corrections go through reversal/adjustment records with approval trail. Pending and rejected expenses are excluded from approved expense totals. | [CONFIRMED] |
| BR-EXP-06 | Approved expense decisions appear in financial reports. | [CONFIRMED] |

## 9. Notifications

| ID | Rule | Status |
|---|---|---|
| BR-NTF-01 | Channels are independently configurable: in-app, SMS, email — each can be enabled/disabled per deployment. | [CONFIRMED] |
| BR-NTF-02 | Trigger events: member registration, payment confirmation, membership expiry reminder, membership renewal, attendance confirmation, trainer assignment, gym announcements. | [CONFIRMED] |
| BR-NTF-03 | Every delivery attempt per channel is tracked (queued → sent/failed) including failure reasons and retry counts. | [CONFIRMED] |
| BR-NTF-04 | External providers (SMS gateway, email service) are NOT selected or configured — approval required first. In-app delivery is renderable from the database alone. | [CONFIRMED constraint] |
| BR-NTF-05 | Members have no login in v1: member-directed in-app content is not rendered to members; in-app notifications target staff roles; member channels are SMS/email. | [REC deriving from AD-02] |

## 10. Finance & Reporting

| ID | Rule | Status |
|---|---|---|
| BR-FIN-01 | Only COMPLETED payments count as received revenue. | [CONFIRMED] |
| BR-FIN-02 | Partial payments remain individually identifiable in reports (never averaged away or bundled). | [CONFIRMED] |
| BR-FIN-03 | Approved expense totals exclude pending and rejected expenses. | [CONFIRMED] |
| BR-FIN-04 | Reports include daily payment summaries, weekly/monthly revenue, membership payment reports, outstanding balances, expense reports, equipment maintenance costs, P&L, and KPI dashboards. | [CONFIRMED] |
| BR-FIN-05 | Accounting basis (cash vs accrual) is UNRESOLVED. Recommendation: cash basis (revenue = received payments; expense = approved expense decisions), simple and reconcilable for a single gym. | [UNRESOLVED] |
| BR-FIN-06 | Financial corrections must be auditable: payments can only be reversed (reversal records linked to the original), expenses corrected through approval trail — never blanked. | [CONFIRMED] |
| BR-FIN-07 | P&L = Σ COMPLETED payments − Σ approved (current) expenses within the period; both legs exclude pending/void/rejected items. | [REC] |

## 11. Audit

| ID | Rule | Status |
|---|---|---|
| BR-AUD-01 | Audit (never optional) all: authentication events, user/role changes, member changes, membership changes, payment actions, expense decisions, trainer assignments, attendance events, biometric verification, door access attempts, administrative overrides. | [CONFIRMED] |
| BR-AUD-02 | The audit log NEVER contains: passwords, authentication tokens, raw biometric data, or secrets. Values are guarded at the emitting layer; context payloads are redacted. | [CONFIRMED] |
| BR-AUD-03 | Audit entries are append-only: no update/delete API; corrections are new entries. | [REC] |
| BR-AUD-04 | Actor attribution: staff action → acting user; system/automatic action → SYSTEM actor; device events → device reference + code path context. | [REC] |
| BR-AUD-05 | Audit retention/partitioning policy: undecided; PII retention must follow local regulation at deployment (noted as risk). | [UNRESOLVED] |
