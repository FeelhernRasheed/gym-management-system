# Database Design — Gym Management System (PostgreSQL / Django ORM)

**Status:** Logical design (Phase 2, documentation only). No apps, models, or migrations exist. Names are model-class names for future Django apps; Django will generate physical tables as `<app_name>_<model_name_lower>` (e.g., `members_member`) when apps are created in a later phase. Nothing here has touched a database.

---

## 1. Design Conventions

| Convention | Decision |
|---|---|
| Primary keys | `id BIGINT` auto-increment (Django `BigAutoField`; Postgres BIGSERIAL equivalent) — set as default in Phase 1 settings |
| Django ORM | All access through the ORM; no raw SQL in application code |
| Datetimes | `TIMESTAMPTZ` (Django `DateTimeField` with `USE_TZ=True`; display timezone Africa/Nairobi) |
| Dates | `DATE` (Django `DateField`) |
| Money | `NUMERIC(12,2)` (Django `DecimalField(max_digits=12, decimal_places=2)`). Balances/durations are never stored (§4) |
| Booleans | `BOOLEAN NOT NULL DEFAULT` explicit |
| Enumerated sets | Django `TextChoices` (CharField) + recommended Postgres CHECK constraints for must-not-drift values (§ per entity) |
| FK policy | Explicit `on_delete` per FK; deletes of master data are prevented (`PROTECT`) wherever history/audit depend on it; staff records are deactivated, never deleted |
| Naming | Model class names PascalCase, singular; DB tables Django-standard lowercase `<app>_<modelpascal>`; indexes `idx`-prefixed Django default |
| Normalization | 3NF everywhere except two justified snapshot denormalizations on `Membership` (§4.3) and event log context fields |

## 2. Entity Catalog (27 finalized entities)

Candidate list was 28 tables; after evaluation **`AttendanceEvent` was removed** (§3) and none were added beyond the candidates — additions were judged unnecessary duplication.

### 2.1 accounts / access domain

#### User (staff)
| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | BIGINT | PK | |
| username | VARCHAR(150) | UNIQUE, NOT NULL | login ID; recommendation: extend `AbstractUser` at implementation |
| password | VARCHAR(128) | NOT NULL | Django-hashed; NEVER written to logs |
| email | VARCHAR(254) | NULL | staff email |
| first_name / last_name | VARCHAR(150) | optional-ish | display identity |
| role_id | BIGINT | FK -> Role, PROTECT, NOT NULL | role assignment |
| is_active | BOOLEAN | NOT NULL DEFAULT TRUE | deactivation instead of delete (audit preserves attribution) |
| date_joined / last_login | TIMESTAMPTZ | NOT NULL / NULL | standard |

Rows: 4 system roles seeded later (data migration, not executed in this phase). Users are NOT members (AD-02).

#### Role
| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | BIGINT | PK | |
| code | VARCHAR(32) | UNIQUE, NOT NULL | `ADMIN`, `RECEPTIONIST`, `TRAINER`, `ACCOUNTANT` |
| name | VARCHAR(64) | NOT NULL | display |
| description | TEXT | NULL | |

### 2.2 Member

#### Member

| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | BIGINT | PK | internal key |
| gym_member_id | VARCHAR(12) | UNIQUE, NOT NULL | **GMID** — business master key; format rec: `GM-000001` (BR-REG-02 [REC]) |
| full_name | VARCHAR(255) | NOT NULL | |
| gender | VARCHAR(16) | NOT NULL | choices: MALE / FEMALE / OTHER (sensitive attr; backend never exposes beyond need) |
| date_of_birth | DATE | NOT NULL, CHECK <= CURRENT_DATE | minor policy unresolved (BR-REG-07) |
| phone | VARCHAR(32) | NOT NULL, UNIQUE [REC] | SMS/notification key |
| address | TEXT | NOT NULL | physical address |
| email | VARCHAR(254) | NULL, UNIQUE | nulls allowed (multi-null unique in PG) |
| photo | VARCHAR (PATH) | NULL | media path; not in DB as blob (BR-REG-08) |
| emergency_contact_name | VARCHAR(255) | NOT NULL | no uniqueness (family share) |
| emergency_contact_phone | VARCHAR(32) | NOT NULL | |
| registration_date | DATE | NOT NULL | set automatically at creation (BR-REG-01) |

Indexes: UNIQUE(gym_member_id), UNIQUE(phone) [REC], UNIQUE(email) NULL-safe; suggested `idx_member_fullname` for later search UX.

### 2.3 Memberships, Plans, Payments

#### MembershipPlan
| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | BIGINT | PK | |
| code | VARCHAR(16) | UNIQUE, NOT NULL | DAILY / WEEKLY / MONTHLY / QUARTERLY / YEARLY |
| name | VARCHAR(64) | NOT NULL | |
| duration_count | INT | CHECK > 0, NOT NULL | e.g., 1, 3, 12 |
| duration_unit | VARCHAR(8) | NOT NULL | DAY / MONTH (see note) |
| price | NUMERIC(12,2) | NOT NULL, CHECK >= 0 | current price |
| is_active | BOOLEAN | NOT NULL DEFAULT TRUE | ADMIN availability control |

Note (BR-MEM-03): calendar-accurate durations. To express all five plans: DAILY(1, DAY), WEEKLY(7, DAY), MONTHLY(1, MONTH), QUARTERLY(3, MONTH), YEARLY(12, MONTH). Duration is used to compute `Membership.end_date` at creation; the plan row remains editable for pricing going forward — historical memberships snapshot it (AD-05).

#### Membership (one member + one plan instance = one lifecycle)
| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | BIGINT | PK | |
| member_id | BIGINT | FK -> Member, PROTECT, NOT NULL | many historical memberships per member |
| plan_id | BIGINT | FK -> MembershipPlan, PROTECT, NOT NULL | |
| agreed_price | NUMERIC(12,2) | NOT NULL, CHECK >= 0 | snapshot of plan price at selection (BR-MEM-11) |
| start_date | DATE | NOT NULL | **policy unresolved (BR-MEM-13)** |
| end_date | DATE | NOT NULL, CHECK (end_date > start_date) | computed at creation from plan duration; stored as snapshot |
| status | VARCHAR(16) | NOT NULL | PENDING_PAYMENT / ACTIVE / EXPIRED; CHECK constraint [REC] |
| created_at | TIMESTAMPTZ | NOT NULL | |

State machine lives in the membership service (BR-MEM-04/09/12); exactly-one-ACTIVE guard is a recommended partial unique index `(member_id) WHERE status = 'ACTIVE'` — pending confirmation of the single-active recommendation (BR-MEM-08 [REC]).

#### Payment
| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | BIGINT | PK | |
| membership_id | BIGINT | FK -> Membership, PROTECT, NOT NULL | many payments per membership |
| amount | NUMERIC(12,2) | NOT NULL, CHECK (amount > 0) | |
| method | VARCHAR(24) | NOT NULL | set unresolved (BR-MEM-14); recorded per payment |
| status | VARCHAR(16) | NOT NULL | COMPLETED / REVERSED (append-only; corrections make reversal rows, BR-FIN-06) |
| paid_at | TIMESTAMPTZ | NOT NULL | |
| received_by_id | BIGINT | FK -> User, PROTECT, NOT NULL | accountability |
| note | TEXT | NULL | |

Indexes: (membership_id), (paid_at) for daily/weekly/monthly revenue queries, (status).

Outstanding balance (never stored, BR-MEM-07):
```
outstanding = Membership.agreed_price - COALESCE(SUM(Payment.amount WHERE status='COMPLETED'), 0)
```

#### Receipt
| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | BIGINT | PK | |
| payment_id | BIGINT | FK -> Payment, **OneToOne**, CASCADE, UNIQUE | a payment has at most one receipt |
| receipt_number | VARCHAR(24) | UNIQUE, NOT NULL | rec sequential `RCP-YYYY-NNNNNN` |
| issued_at | TIMESTAMPTZ | NOT NULL | |
| issued_by_id | BIGINT | FK -> User, PROTECT, NOT NULL | |

### 2.4 Attendance & Physical Access

#### AttendanceSession (one visit)
| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | BIGINT | PK | |
| member_id | BIGINT | FK -> Member, PROTECT, NOT NULL | |
| check_in_at | TIMESTAMPTZ | NOT NULL | |
| check_in_method | VARCHAR(24) | NOT NULL | FINGERPRINT / FACIAL / MANUAL |
| check_out_at | TIMESTAMPTZ | NULL | exit recorded separately (BR-ATT-04) |
| check_out_method | VARCHAR(24) | NULL (required once check_out set) | |
| recorded_by_id | BIGINT | FK -> User, SET_NULL, NULL | manual operations actor (BR-ATT-05) |
| manual_reason | TEXT | NULL | mandatory when recorded_by used (BR-ATT-05) |

Constraints: CHECK (check_out_at IS NULL OR check_out_at > check_in_at); **partial UNIQUE (member_id) WHERE check_out_at IS NULL** (BR-ATT-01). Index (member_id, check_in_at). Elapsed time always derived = check_out - check_in.

#### BiometricDevice (registry; no hardware assumptions)
| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | BIGINT | PK | |
| label | VARCHAR(64) | NOT NULL | |
| vendor_name | VARCHAR(64) | NULL | raw text; UNRESOLVED hardware (BR-BIO-06) |
| model_name | VARCHAR(64) | NULL | |
| modality | VARCHAR(24) | NOT NULL | FINGERPRINT / FACIAL |
| location_note | VARCHAR(128) | NULL | e.g., entrance |
| is_active | BOOLEAN | NOT NULL DEFAULT TRUE | |

#### AccessDevice (doors / turnstiles)
| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | BIGINT | PK | |
| label | VARCHAR(64) | NOT NULL | |
| kind | VARCHAR(24) | NOT NULL | DOOR / TURNSTILE |
| location_note | VARCHAR(128) | NULL | |
| is_active | BOOLEAN | NOT NULL DEFAULT TRUE | |
| biometric_device_id | BIGINT | FK -> BiometricDevice, SET_NULL, NULL | optional pairing; topologies undefined (BR-ACC-09) |

#### BiometricEnrollment (member <-> template mapping; never raw data)
| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | BIGINT | PK | |
| member_id | BIGINT | FK -> Member, PROTECT, NOT NULL | |
| device_id | BIGINT | FK -> BiometricDevice, PROTECT, NOT NULL | templates live on device (BR-BIO-03) |
| modality | VARCHAR(24) | NOT NULL | FINGERPRINT / FACIAL |
| template_ref | VARCHAR(128) | NOT NULL | device-side handle/ID only (BR-BIO-02) |
| status | VARCHAR(16) | NOT NULL | ACTIVE / REVOKED (revoked on removal, never reused) |
| enrolled_by_id | BIGINT | FK -> User, PROTECT, NOT NULL | |
| enrolled_at | TIMESTAMPTZ | NOT NULL | |
| revoked_at | TIMESTAMPTZ | NULL | |

Partial UNIQUE (member_id, device_id, modality) WHERE status = 'ACTIVE'.

#### AccessAttempt (biometric verification result + door decision, one event)
| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | BIGINT | PK | |
| occurred_at | TIMESTAMPTZ | NOT NULL | |
| member_id | BIGINT | FK -> Member, SET_NULL, NULL | NULL = unrecognized identity (BR-ACC-03) |
| biometric_device_id | BIGINT | FK -> BiometricDevice, PROTECT, NOT NULL | verifying device |
| access_device_id | BIGINT | FK -> AccessDevice, SET_NULL, NULL | NULL when verification ran without a door decision |
| modality | VARCHAR(24) | NOT NULL | |
| verification_result | VARCHAR(24) | NOT NULL | MATCHED / NO_MATCH / TEMPLATE_UNREADABLE |
| decision | VARCHAR(24) | NOT NULL | GRANTED / DENIED / OVERRIDE_GRANT |
| deny_reason | VARCHAR(32) | NULL (required if DENIED) | MEMBERSHIP_PENDING / MEMBERSHIP_EXPIRED / UNKNOWN_IDENTITY / DEVICE_ERROR |
| override_by_id | BIGINT | FK -> User, SET_NULL, NULL | receptionist/admin override accountability (BR-ACC-05) |
| override_reason | TEXT | NULL | mandatory when OVERRIDE_GRANT |

Indexes: (occurred_at), (member_id), (decision).

### 2.5 Trainers & Workouts

#### TrainerProfile
| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | BIGINT | PK | |
| user_id | BIGINT | FK -> User, OneToOne, CASCADE, NOT NULL | role=TRAINER enforced in app layer |
| specialization | VARCHAR(128) | NULL | |
| bio | TEXT | NULL | |
| is_accepting_members | BOOLEAN | NOT NULL DEFAULT TRUE |ADMIN assignment aid |

#### TrainerAssignment (append-only history)
| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | BIGINT | PK | |
| member_id | BIGINT | FK -> Member, PROTECT, NOT NULL | |
| trainer_id | BIGINT | FK -> User, PROTECT, NOT NULL (role TRAINER) | |
| assigned_by_id | BIGINT | FK -> User, PROTECT, NOT NULL | ADMIN (BR-TRN-02) |
| assigned_at | TIMESTAMPTZ | NOT NULL | |
| unassigned_at | TIMESTAMPTZ | NULL | closes the row (BR-TRN-06) |
| end_reason | VARCHAR(32) | NULL | REASSIGNED / REMOVED / TRAINER_LEFT |

Immutability rule: never edited destructively; new states = new rows. Multiple-active-per-member allowed or not = BR-TRN-05 [UNRESOLVED]; if single-active confirmed, add partial UNIQUE (member_id) WHERE unassigned_at IS NULL.

#### WorkoutPlan
| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | BIGINT | PK | |
| member_id | BIGINT | FK -> Member, PROTECT, NOT NULL | |
| created_by_id | BIGINT | FK -> User, PROTECT, NOT NULL | trainer (BR-WKO-01) — object-level ownership check |
| title | VARCHAR(128) | NOT NULL | |
| description | TEXT | NULL | |
| start_date | DATE | NULL | |
| end_date | DATE | NULL | |
| is_active | BOOLEAN | NOT NULL DEFAULT TRUE | |

#### Exercise (shared library)
| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | BIGINT | PK | |
| name | VARCHAR(128) | UNIQUE, NOT NULL | |
| description | TEXT | NULL | |

#### WorkoutPlanExercise (plan composition through-table)
| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | BIGINT | PK | |
| workout_plan_id | BIGINT | FK -> WorkoutPlan, CASCADE, NOT NULL | |
| exercise_id | BIGINT | FK -> Exercise, PROTECT, NOT NULL | |
| day_of_week | VARCHAR(9) | NULL | MON..SUN (rec) |
| sets / reps | VARCHAR(32) | NULL | free guidance (e.g., "3 x 12") |
| guidance | TEXT | NULL | |
| order | INT | NOT NULL | ordering inside plan |
| | | UNIQUE (workout_plan_id, order) | |

#### TrainingSchedule (trainer + member + slot)
| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | BIGINT | PK | |
| trainer_id | BIGINT | FK -> User, PROTECT, NOT NULL | |
| member_id | BIGINT | FK -> Member, PROTECT, NOT NULL | |
| workout_plan_id | BIGINT | FK -> WorkoutPlan, SET_NULL, NULL | optional context |
| day_of_week | VARCHAR(9) | NOT NULL | |
| start_time / end_time | TIME | NOT NULL, CHECK start<end | |
| is_active | BOOLEAN | NOT NULL DEFAULT TRUE | |

Recurring weekly slots in v1 [REC]; date exceptions deferred.

### 2.6 Equipment

#### EquipmentCategory
| id BIGINT PK | name VARCHAR(64) UNIQUE NOT NULL | description TEXT NULL |

#### Equipment
| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | BIGINT | PK | |
| category_id | BIGINT | FK -> EquipmentCategory, PROTECT, NOT NULL | |
| name | VARCHAR(128) | NOT NULL | |
| asset_tag | VARCHAR(32) | UNIQUE, NULL | individual identification where applicable (BR-EQP-02) |
| serial_number | VARCHAR(64) | UNIQUE, NULL | manufacturer serial |
| quantity | INT | NULL | only for non-serialized consumables [REC] |
| purchase_date | DATE | NULL | |
| purchase_cost | NUMERIC(12,2) | NULL | |
| condition_note | TEXT | NULL | |
| status | VARCHAR(24) | NOT NULL | AVAILABLE / UNDER_MAINTENANCE / DAMAGED / RETIRED |
| is_serialized | BOOLEAN | NOT NULL DEFAULT TRUE | row = one item when TRUE |

Rule: rows with serial/asset tag represent exactly one unit; bulk rows carry quantity [REC]; no mixed cardinality beyond this.

#### MaintenanceRecord (schedule + repair + service history)
| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | BIGINT | PK | |
| equipment_id | BIGINT | FK -> Equipment, PROTECT, NOT NULL | |
| type | VARCHAR(24) | NOT NULL | PREVENTIVE / REPAIR |
| status | VARCHAR(24) | NOT NULL | SCHEDULED / COMPLETED / CANCELLED |
| scheduled_date | DATE | NULL (required when SCHEDULED) | |
| performed_date | DATE | NULL (required when COMPLETED) | |
| cost | NUMERIC(12,2) | NOT NULL DEFAULT 0, CHECK >= 0 | maintenance cost exposure for finance (module dependency) |
| performed_by | VARCHAR(128) | NULL | free text: staff or external vendor |
| description | TEXT | NULL | |

### 2.7 Expenses

#### ExpenseCategory
| id BIGINT PK | code VARCHAR(32) UNIQUE NOT NULL | name VARCHAR(64) NOT NULL | description TEXT NULL |

Seven seeded categories: RENT, UTILITIES, STAFF_SALARIES, EQUIPMENT_MAINTENANCE, CLEANING, INTERNET, OTHER.

#### Expense
| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | BIGINT | PK | |
| category_id | BIGINT | FK -> ExpenseCategory, PROTECT, NOT NULL | |
| recorded_by_id | BIGINT | FK -> User, PROTECT, NOT NULL | accountant (never self-approves, BR-EXP-04) |
| amount | NUMERIC(12,2) | NOT NULL, CHECK > 0 | |
| expense_date | DATE | NOT NULL | |
| description | TEXT | NOT NULL | what was purchased |
| attachment | VARCHAR (PATH) | NULL | receipt file |
| status | VARCHAR(16) | NOT NULL | PENDING_APPROVAL / APPROVED / REJECTED |
| created_at | TIMESTAMPTZ | NOT NULL | |

Approved expenses are locked at the service layer (BR-EXP-05); corrections happen via audit + new expenses (financial corrections auditable).

#### ExpenseApproval (immutable decision history)
| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | BIGINT | PK | |
| expense_id | BIGINT | FK -> Expense, PROTECT, NOT NULL | many decisions over an expense's life |
| decided_by_id | BIGINT | FK -> User, PROTECT, NOT NULL | must differ from Expense.recorded_by (object rule) |
| decision | VARCHAR(16) | NOT NULL | APPROVED / REJECTED |
| reason | TEXT | NOT NULL when REJECTED | BR-EXP-03 |
| decided_at | TIMESTAMPTZ | NOT NULL | |

Note: rows never updated; a re-opened cycle appends NEW approval rows. Latest decision per expense = max(decided_at) with final state matching Expense.status.

### 2.8 Notifications

#### Notification
| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | BIGINT | PK | |
| event_type | VARCHAR(32) | NOT NULL | MEMBER_REGISTERED / PAYMENT_CONFIRMED / MEMBERSHIP_EXPIRY_REMINDER / MEMBERSHIP_RENEWED / ATTENDANCE_CONFIRMED / TRAINER_ASSIGNED / GYM_ANNOUNCEMENT |
| recipient_user_id | BIGINT | FK -> User, SET_NULL, NULL | in-app target staff (BR-NTF-05) |
| recipient_member_id | BIGINT | FK -> Member, SET_NULL, NULL | contact target member |
| title | VARCHAR(128) | NOT NULL | |
| body | TEXT | NOT NULL | |
| created_at | TIMESTAMPTZ | NOT NULL | |

#### NotificationDelivery (1 row per channel attempt-stream)
| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | BIGINT | PK | |
| notification_id | BIGINT | FK -> Notification, CASCADE, NOT NULL | |
| channel | VARCHAR(8) | NOT NULL | IN_APP / SMS / EMAIL |
| status | VARCHAR(16) | NOT NULL | QUEUED / SENT / FAILED |
| attempts | INT | NOT NULL DEFAULT 0 | retry count |
| last_attempt_at | TIMESTAMPTZ | NULL | |
| sent_at | TIMESTAMPTZ | NULL | |
| error_message | TEXT | NULL | failure tracking (BR-NTF-03) |
| provider_ref | VARCHAR(128) | NULL | provider message id (post-approval) |

UNIQUE (notification_id, channel) — one current delivery row per channel; retry state fields update this row (attempt history summarized by counters; deep per-attempt history can be a future extension if required).

### 2.9 Audit

#### AuditLog
| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | BIGINT | PK | |
| occurred_at | TIMESTAMPTZ | NOT NULL | |
| actor_id | BIGINT | FK -> User, SET_NULL, NULL | NULL = SYSTEM (BR-AUD-04) |
| event_type | VARCHAR(48) | NOT NULL | AUTH_LOGIN / AUTH_FAILED / USER_CREATED / ROLE_CHANGED / MEMBER_CREATED / MEMBERSHIP_* / PAYMENT_* / EXPENSE_* / TRAINER_ASSIGNED / ATTENDANCE_* / BIOMETRIC_* / DOOR_ATTEMPT / OVERRIDE_* |
| object_type | VARCHAR(64) | NOT NULL | ContentType or string ref |
| object_id | VARCHAR(64) | NOT NULL | generic target reference |
| context | JSONB | NOT NULL DEFAULT '{}' | redacted payload; secrets/biometric raw data forbidden (BR-AUD-02, enforced at emitting layer) |
| ip_address | INET | NULL | for auth/access events |
| user_agent | VARCHAR(256) | NULL | |

Append-only (BR-AUD-03); indexes: (occurred_at), (object_type, object_id), (actor_id), (event_type).

## 3. Candidate Evaluation — Additions / Removals

| Candidate | Decision | Rationale |
|---|---|---|
| `AttendanceEvent` | **REMOVED** | Entry/exit are two timestamp columns on `AttendanceSession` plus method fields; denied gate attempts belong to `AccessAttempt`; a separate event table would duplicate the same facts twice |
| `Receipt` | KEPT separate from Payment | Distinct receipt numbering/issuance concerns; keeps Payment purely financial |
| `BiometricDevice` + `AccessDevice` | KEPT separate | Verification endpoint vs unlock endpoint are different roles in the access flow; a combined terminal can pair via `AccessDevice.biometric_device_id` |
| `BiometricVerification` (as a standalone table) | NOT ADDED | Merged into `AccessAttempt` (AD-08): one event chain = verify then authorize; avoids near-duplicate rows |
| `TrainerProfile` | KEPT | Clean extension of trainer staff without corrupting `User` with domain data |
| `Role` | KEPT as lookup | Stable FK integrity + auditable role assignment; Django Groups recommended for token-level permissions (mapped role->group) |
| New entity for plan price history | NOT ADDED | `Membership.agreed_price` + immutable payments already preserve pricing history; avoid table creep |
| New member-login entity | NOT ADDED | Member portal excluded from v1 scope |

## 4. Integrity & Historical Preservation

### 4.1 Delete policy
- Master/financial rows use `PROTECT`: Member, MembershipPlan, Payment/Membership links, ExpenseCategory, Equipment, Exercise, BiometricDevice (only soft-disable via is_active).
- Physical deletes are reserved for meaningless-without-parent rows (Receipt w/o Payment, NotificationDelivery w/o Notification) or profile extensions (TrainerProfile w/ User).
- Staff users DELETE ONLY via documented admin deactivation policy (is_active=FALSE) — the design never encourages user deletion.
- CASCADE only where a child truly cannot exist alone.

### 4.2 Append-only / immutable tables
- `Payment` (status transitions only add reversal rows, never edit)
- `TrainerAssignment` (close-out columns only)
- `ExpenseApproval` (decision history)
- `AuditLog` (no updates, no deletes)
- `BiometricEnrollment` revocation via status (no reuse)
- Recommended DB-level guard: before-update triggers guarding these tables in the migration phase (implementation-time decision; documented requirement).

### 4.3 Snapshot (denormalized) fields — justified
- `Membership.agreed_price`, `start_date`, `end_date`: membership history must not shift when plans change. These are immutable after creation (except the unresolved start-date policy decision).
- Everything else derived (outstanding balances, P&L, elapsed time) is COMPUTED — no stored counters (avoiding drift).

### 4.4 Referential uniqueness summaries
- One ACTIVE membership per member: partial index (pending BR-MEM-08 confirmation).
- One open attendance session per member: partial index (confirmed rule).
- One active enrollment per (member, device, modality): partial index.
- One current delivery per (notification, channel): confirmed.
- at most one receipt per payment: OneToOne.

## 5. Seeded Reference Data (later phase, listed for completeness)

Roles: ADMIN, RECEPTIONIST, TRAINER, ACCOUNTANT. Plans: five fixed codes with prices set by ADMIN first. Expense categories: seven codes above. Notification channels: IN_APP, SMS, EMAIL. Migration data only — nothing executed yet.

## 6. PostgreSQL-Specific Notes

- Version: PostgreSQL 17 vs 18 (installed) — UNRESOLVED; this design is version-agnostic; selection precedes the database phase.
- No database, schema, user, role, table, index or constraint has been created in this phase. Concurrency/locking and trigger-guard migrations belong to the DB implementation phase.
- Character encoding UTF-8 default; time zone storage UTC via TIMESTAMPTZ; Africa/Nairobi as display setting (already configured in project settings).
