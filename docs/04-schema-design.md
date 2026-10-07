# Schema Design — MVP Canonical Persistence Spec

**Status:** Field-level design **COMPLETE** for MVP concrete entities + AuthToken logical note — ready for Prisma/migration implementation.  
**Depends on:**  
- [`01-product-architecture.md`](./01-product-architecture.md)  
- [`02-api-contract.md`](./02-api-contract.md)  
- [`03-data-model.md`](./03-data-model.md)  

**Done in this doc:** conventions, enum inventory, entity index, review order, architecture OPENs (still **3**), field-level design for all **19** concrete entities, AuthToken logical abstraction (§30), cross-entity consistency (§31).  
**Never in this doc as executable artifacts:** Prisma schema files, SQL migrations, index DDL scripts, application code.

---

## 1. Purpose

This document is the **canonical persistence-model source of truth** for the MVP schema, independent of:

- Prisma schema files  
- Migration history / SQL dialects  
- Application repository layout  

It records **what must be persisted and under which global rules**, so later Prisma/SQL work implements decisions already locked here (and in Product Architecture / Data Model), rather than inventing them during coding.

Field-level table designs are added entity-by-entity using the review order in §5. This keeps persistence decisions implementation-independent and locked before Prisma/SQL work.

---

## 2. Global conventions

Collected from Product Architecture and Data Model (FINAL unless noted).

| Convention | Rule |
| --- | --- |
| Primary keys | UUID v7; PostgreSQL native `uuid`; **application-generated** (not PG `uuidv7()`) |
| Instant storage | `timestamptz` with millisecond precision aligned to JS (`timestamptz(3)` intent) |
| Clock / containers | Store and compute checks in **UTC**; DB/containers `TZ=UTC` |
| Business timezone | IANA on `Business`; all local-date / wall-clock / display conversion via shared Luxon module — never browser TZ for business logic |
| `createdAt` | Present on persisted entities that need audit of creation; authoritative vs ID order |
| `updatedAt` | Present where mutable business rows are updated |
| `deletedAt` | Used for **Customer soft delete** only in MVP (not a generic soft-delete on every table) |
| Soft delete | Customer: irreversible soft delete. Do **not** soft-delete Appointment (use status), Service/Staff/Business (use active/INACTIVE) |
| Active / inactive | Service & Staff: boolean (or equivalent) active flag. Business: `INACTIVE` \| `ACTIVE` status enum + `publishedAt` |
| Email normalization | Trim + lowercase → `normalizedEmail`; no dot/`+tag` stripping |
| Phone normalization | Store E.164 when set; default parse country **`TR`** if no calling code |
| Booking reference | `BK-` + 8 uppercase Crockford Base32; business-scoped unique; **not** PK |
| Money | Integer **minor units** + ISO 4217 currency code |
| Enum naming | PascalCase type names in docs (`AppointmentStatus`); DB/Prisma mapping deferred to schema implementation |
| Secrets | Tokens hashed at rest; raw only in memory/email |
| Tenant columns | Tenant-owned rows carry `businessId`; `CustomerAccount.businessId` stored; `NotificationDelivery.businessId` required |
| RLS | Not MVP; tables remain RLS-compatible |

Working-hours wall-clock: local minutes on ISO `dayOfWeek` 1–7 (no `@db.Time` overnight intervals). StaffTimeOff / BusinessClosedDate: **business-local full-day** semantics (local midnight → next local midnight; not fixed 24h elapsed).

---

## 3. Enum inventory

Domain enums implied by Product Architecture / Data Model. Values listed where already FINAL; notes where representation may be boolean vs enum at schema time.

| Enum | Values (FINAL unless noted) | Used by |
| --- | --- | --- |
| `BusinessStatus` | `INACTIVE`, `ACTIVE` | Business |
| `BusinessMemberRole` | `OWNER`, `STAFF` | BusinessMember, BusinessInvitation |
| `BusinessMemberStatus` | `ACTIVE`, `DISABLED` — **FINAL** (field-level §11; dual-state membership lifecycle from PA/DM) | BusinessMember |
| `AppointmentStatus` | `CONFIRMED`, `COMPLETED`, `CANCELLED`, `NO_SHOW` (no `PENDING`) | Appointment |
| `AppointmentSource` | `PUBLIC`, `OWNER`, `STAFF` | Appointment |
| `CancelledBy` | `CUSTOMER`, `STAFF`, `OWNER` | Appointment |
| `InvitationStatus` | `PENDING`, `CONSUMED`, `REVOKED` — **FINAL** (§28) | BusinessInvitation |
| `CustomerVerificationIntentStatus` | `PENDING`, `CONSUMED`, `REVOKED`, `SUPERSEDED` — **FINAL** (§22) | CustomerVerificationIntent |
| `CustomerAccountStatus` | `ACTIVE`, `DISABLED` — **FINAL** (§23) | CustomerAccount |
| `NotificationDeliveryStatus` | `PENDING`, `SENDING`, `SENT`, `SKIPPED`, `FAILED` — **FINAL** (§29) | NotificationDelivery |
| `NotificationType` | `BOOKING_CONFIRMED`, `BOOKING_RESCHEDULED`, `BOOKING_CANCELLED`, `BOOKING_REMINDER`, `ACCOUNT_VERIFICATION`, `ACCOUNT_ALREADY_EXISTS`, `CUSTOMER_PASSWORD_RESET`, `EMAIL_CHANGE_VERIFICATION`, `EMAIL_CHANGED_NOTICE`, `OWNER_INVITATION`, `STAFF_INVITATION`, `BUSINESS_PASSWORD_RESET`, `BUSINESS_APPOINTMENT_NOTICE` — **FINAL** (§29); B4 uses `noticeVariant` | NotificationDelivery |
| `BusinessAppointmentNoticeVariant` | `NEW`, `CANCELLED`, `RESCHEDULED` — **FINAL** (§29); only when `type = BUSINESS_APPOINTMENT_NOTICE` | NotificationDelivery |
| `AuthTokenPurpose` | `CUSTOMER_PASSWORD_RESET`, `CUSTOMER_EMAIL_CHANGE`, `BUSINESS_PASSWORD_RESET` — logical only (§30); registration verify is **not** an AuthToken purpose (FINAL on `CustomerVerificationIntent.verificationTokenHash`, §22) | AuthToken (logical) |
| `ManageTokenStatus` | `ACTIVE`, `REVOKED`, `CONSUMED` — **FINAL** (§27); all three are first-class; expiry is via `expiresAt` (not a fourth status) | AppointmentManageToken |

**Enum count (named inventory rows):** **14** (added `BusinessAppointmentNoticeVariant` for B4 notice variants)

Boolean flags that are **not** enums by product decision: Service/Staff `active`, Appointment `overrideHours`. Slot interval is a constrained integer set (5/10/15/20/30/60), not necessarily a DB enum.

---

## 4. Entity catalog

Index only — no columns in this revision.

| Entity | Section (planned) | Kind |
| --- | --- | --- |
| Business | [§9 Business](#9-entity-business--field-level-design) | Concrete — **fields designed** |
| User | [§10 User](#10-entity-user--field-level-design) | Concrete — **fields designed** |
| BusinessMember | [§11 BusinessMember](#11-entity-businessmember--field-level-design) | Concrete — **fields designed** |
| BusinessSession | [§12 BusinessSession](#12-entity-businesssession--field-level-design) | Concrete — **fields designed** |
| Staff | [§13 Staff](#13-entity-staff--field-level-design) | Concrete — **fields designed** |
| Service | [§14 Service](#14-entity-service--field-level-design) | Concrete — **fields designed** |
| StaffService | [§15 StaffService](#15-entity-staffservice--field-level-design) | Concrete — **fields designed** |
| BusinessWorkingHour | [§16 BusinessWorkingHour](#16-entity-businessworkinghour--field-level-design) | Concrete — **fields designed** |
| StaffWorkingHour | [§17 StaffWorkingHour](#17-entity-staffworkinghour--field-level-design) | Concrete — **fields designed** |
| BusinessClosedDate | [§18 BusinessClosedDate](#18-entity-businesscloseddate--field-level-design) | Concrete — **fields designed** |
| StaffTimeOff | [§19 StaffTimeOff](#19-entity-stafftimeoff--field-level-design) | Concrete — **fields designed** |
| Customer | [§21 Customer](#21-entity-customer--field-level-design) | Concrete — **fields designed** |
| CustomerVerificationIntent | [§22 CustomerVerificationIntent](#22-entity-customerverificationintent--field-level-design) | Concrete — **fields designed** |
| CustomerAccount | [§23 CustomerAccount](#23-entity-customeraccount--field-level-design) | Concrete — **fields designed** |
| CustomerSession | [§24 CustomerSession](#24-entity-customersession--field-level-design) | Concrete — **fields designed** |
| Appointment | [§26 Appointment](#26-entity-appointment--field-level-design) | Concrete — **fields designed** |
| AppointmentManageToken | [§27 AppointmentManageToken](#27-entity-appointmentmanagetoken--field-level-design) | Concrete — **fields designed** |
| BusinessInvitation | [§28 BusinessInvitation](#28-entity-businessinvitation--field-level-design) | Concrete — **fields designed** |
| NotificationDelivery | [§29 NotificationDelivery](#29-entity-notificationdelivery--field-level-design) | Concrete — **fields designed** |
| AuthToken | [§30 AuthToken](#30-authtoken--logical-abstraction) | **Logical** (physical shape **OPEN #1**) |

**Concrete entity count:** **19**  
**Logical abstraction count:** **1** (`AuthToken`)

---

## 5. Schema review order

Entities will be designed in this order (field-level work in later revisions):

1. **Business** — tenant root; everything hangs off `businessId`  
2. **User** — global identity before membership  
3. **BusinessMember** — links User↔Business + role  
4. **BusinessSession** — business auth realm  
5. **Staff** — bookable profile (optional member link)  
6. **Service** — offerable catalog  
7. **StaffService** — M:N after both parents exist  
8. **WorkingHours** — `BusinessWorkingHour` then `StaffWorkingHour`  
9. **BusinessClosedDate** — business-level closures  
10. **StaffTimeOff** — staff full-day exclusions (after Staff)  
11. **Customer** — before account/verification consumers  
12. **CustomerVerificationIntent** — registration pending (no Customer FK)  
13. **CustomerAccount** — 1:1 with Customer; stores `businessId`  
14. **CustomerSession** — customer auth realm  
15. **Appointment** — aggregate needing Business, Customer, Staff, Service  
16. **AppointmentManageToken** — depends on Appointment  
17. **BusinessInvitation** — depends on Business (+ optional Staff)  
18. **NotificationDelivery** — depends on Business (+ optional Appointment)  
19. **AuthToken abstraction** — last; physical choice OPEN; subjects already defined  

**Rationale:** dependency order (parents before children), auth realms after identities, appointment subgraph after parties/catalog, tokens/deliveries after the rows they reference. Working hours / closures before appointment engine assumptions are encoded in constraints.

---

## 6. Open schema decisions

### 6.1 Architecture-level OPENs

These remain unresolved at the **schema-architecture** layer (carried from Data Model). Do **not** invent new ones. Do **not** prematurely decide them while designing Business or later entities:

1. **AuthToken physical model** — generic purpose-scoped table vs purpose-specific tables for **customer password reset**, **customer email-change** (incl. pending new email), and **business password reset**. **Not** in this OPEN: registration email verification (FINAL as `CustomerVerificationIntent.verificationTokenHash`, §22). Invitation and manage tokens remain dedicated entities.  
2. **Exact secondary / performance / query indexes** — deferred. **Does not** include correctness constraints (PKs, product UNIQUEs, partial uniques required by product invariants, Appointment exclusion). Those are **FINAL** where stated on each entity.  
3. **Exact FK on-delete behavior** (RESTRICT vs CASCADE vs SET NULL) — deferred; must preserve soft-delete and history rules when decided.

**Architecture-level Schema OPEN count:** **3**

**OPEN #2 scope (explicit):** correctness constraints = **FINAL**; performance/query indexes only = **OPEN #2**.

### 6.2 Field-level decisions (not architecture OPENs)

Ordinary per-entity column details are resolved in entity sections (§9–§29), not as architecture OPENs. Prisma attribute spelling remains implementation. **All 19 concrete entities** designed; AuthToken logical §30; consistency §31. Max-length TBDs where sources omit limits are noted per entity (not architecture OPENs).

---

## 7. Out of scope (still)

- Prisma models / `@db` attributes as source files  
- Migration SQL / exclusion constraint DDL text  
- Seed data  
- Premature resolution of the three architecture-level OPENs in §6.1  

---

## 8. Document consistency report

| Metric | Value |
| --- | --- |
| Enum inventory rows | **14** |
| Concrete entities | **19** (all field-designed) |
| Logical abstractions | **1** (AuthToken) |
| Architecture-level Schema OPEN count | **3** (unchanged) |
| Entities with field-level design | **19** + AuthToken logical §30 |
| Missing entity vs Data Model | **None** |
| Contradiction vs PA / API / Data Model | **None** after §31 pass |

**Next phase (outside this doc):** Prisma schema + migrations implementing §9–§30 under OPENs #1–#3.

---

## 9. Entity: Business — field-level design

**Kind:** Tenant root (concrete).  
**Sources:** PA §6 / §5; Data Model §2.1; API settings surface maps to these columns (behavior lives in API Contract, not here).  
**Does not** carry a `businessId` column (it *is* the tenant).  
**Does not** use soft delete / `deletedAt`.

### 9.1 Business — Field Specification

| Field | Type (logical) | Required | Default | Constraints / Semantics (source-backed) |
| --- | --- | --- | --- | --- |
| `id` | UUID v7 | yes | app-generated | PK; app-generated UUID v7; eventual PG native `uuid` (PA ID strategy) |
| `name` | string | yes | — | Trimmed; length **1–80** (PA §6.2) |
| `slug` | string | yes | — | Global unique; **2–48** chars; `^[a-z0-9]+(?:-[a-z0-9]+)*$`; reserved-word list (PA §6.4); **never reused** (PA §6.4 / §6.9); mutable only while `publishedAt IS NULL` |
| `timezone` | string (IANA) | yes | — | Validated + canonical IANA; immutable after first publish |
| `currency` | string (ISO 4217) | yes | — | Active ISO 4217; 3-letter uppercase; unknown rejected; no fixed allow-list; immutable after first publish |
| `phone` | string (E.164) \| null | no | `null` | When set: E.164; default parse country `TR` if no calling code (PA); empty → `null` |
| `email` | string \| null | no | `null` | When set: stored **lowercase** (PA §6.2); empty → `null`. **No** separate `normalizedEmail` column on Business. **Max length:** not specified in source docs → field-level length TBD (not invented here) |
| `address` | string \| null | no | `null` | Plain text; max **300** (PA §6.2); empty → `null`; not structured |
| `logoUrl` | string \| null | no | `null` | Absolute HTTPS; max **2048** (PA §6.2); no userinfo; reject localhost/private best-effort (PA §6.7); empty → `null` |
| `primaryColor` | string \| null | no | `null` | `#` + 6 hex; store lowercase (PA §6.2); invalid reject; null means platform default at render time (not a DB default color) |
| `slotIntervalMinutes` | int | yes | **15** | ∈ {5, 10, 15, 20, 30, 60} (PA) |
| `minBookingNoticeMinutes` | int | yes | **60** | ≥ 0; elapsed duration (PA) |
| `maxBookingWindowDays` | int | yes | **60** | 1–365; local calendar days (PA) |
| `cancellationNoticeMinutes` | int | yes | **1440** | ≥ 0; elapsed duration (PA) |
| `status` | enum `BusinessStatus` | yes | **`INACTIVE`** | `INACTIVE` \| `ACTIVE` only (PA) |
| `publishedAt` | timestamptz(3) \| null | no | `null` | Null = never published; set once on first successful activation; never cleared/changed afterward (PA) |
| `createdAt` | timestamptz(3) | yes | set on create (UTC) | Authoritative creation time |
| `updatedAt` | timestamptz(3) | yes | set on create/update (UTC) | Last mutation; **no** `version` column (PA: settings last-write-wins) |

**Explicit non-fields (PA):** `rescheduleNotice`, `deactivatedAt`, `locale`, soft-delete column, self-`businessId`, payment fields.

### 9.2 Business — Invariants

Persistence / product invariants (not HTTP codes):

1. `id` is the global primary key.  
2. `slug` is globally unique and **never reused** across businesses (PA).  
3. `status ∈ {INACTIVE, ACTIVE}` only.  
4. `publishedAt IS NULL` ⇔ never successfully activated/published.  
5. Once `publishedAt` is set, it must not be cleared or overwritten.  
6. After `publishedAt` is set: `slug`, `timezone`, and `currency` are immutable.  
7. Booking-rule fields are always non-null (defaults at create); contact/branding may be null.  
8. Go-live checklist is **derived** from related rows (hours/services/staff/…), not stored Business columns.  
9. Deactivation sets `status = INACTIVE` and retains `publishedAt`.  
10. Business is not soft-deleted; MVP offboarding retains the row as `INACTIVE` (PA).  
11. Tenant currency code is `currency`; monetary amounts live on Service / Appointment snapshots.

### 9.3 Business — Uniqueness / Lookup Rules

| Rule | Scope | Kind |
| --- | --- | --- |
| Unique `id` (PK) | Global | FINAL persistence |
| Unique `slug` | Global | FINAL product/data invariant (PA) |
| Resolve public tenant by `slug` | Global | FINAL product rule (lookup key) |
| Resolve admin tenant by session → `id` | Session | FINAL product rule; client `businessId` not trusted |

**Physical index strategy** (how uniqueness is enforced in DDL, extra secondary indexes, etc.) remains **architecture OPEN #2**. Uniqueness of `slug` itself is not OPEN.

### 9.4 Business — Lifecycle

```text
Create          → status=INACTIVE, publishedAt=null, booking-rule defaults applied
First activate  → checklist satisfied → status=ACTIVE, publishedAt=now() (once)
Deactivate      → status=INACTIVE, publishedAt unchanged
Reactivate      → checklist satisfied → status=ACTIVE, publishedAt unchanged
```

Persistence notes:

- `publishedAt` null vs non-null distinguishes never-published vs previously published.  
- Public visibility / booking eligibility by `status` + `publishedAt` is defined in Product Architecture; **HTTP status codes and response payloads are API Contract concerns**, not schema columns.  
- Settings updates: last-write-wins; no optimistic `version` on Business (PA). Activate/deactivate are conditional status transitions at the application layer.

### 9.5 Business — Relationships

Parent/root of tenant-owned children (child field designs pending):

`BusinessMember`, `BusinessSession` (via `activeBusinessId`), `Staff`, `Service`, `StaffService`, `BusinessWorkingHour`, `BusinessClosedDate`, `StaffTimeOff` (via Staff), `Customer`, `CustomerAccount`, `CustomerSession`, `CustomerVerificationIntent`, `Appointment`, `AppointmentManageToken`, `BusinessInvitation`, `NotificationDelivery`.

Child FK **`ON DELETE`** behavior: **architecture OPEN #3** — not decided here. Safe product statement only: MVP does not hard-delete a business and cascade-erase history as an offboarding action (retain `INACTIVE` row).

### 9.6 Business — Schema Notes

- Does **not** close architecture OPENs #1–#3.  
- #1 AuthToken: no Business-field dependency.  
- #2 Indexes: `slug` uniqueness is FINAL; exact index DDL / secondary indexes remain OPEN.  
- #3 FK on-delete: deferred until children are designed.  
- Length/format limits in §9.1 for `name` / `slug` / `address` / `logoUrl` / booking-rule bounds are taken from **PA §6**, not invented in this schema pass.  
- `Business.email` max length is **not** in source docs → left TBD (not invented).  
- Prisma/`@db` mapping is implementation, not specified here.

---

## 10. Entity: User — field-level design

**Kind:** Platform-global business-auth identity (concrete).  
**Sources:** PA §5.1–§5.2 / §8.1 / §8.4; Data Model §2.2 / §7 / §10 / §12; API Contract §3.1 / §3.2 / §14.  
**Does not** carry `businessId` (not tenant-scoped).  
**Does not** use soft delete / `deletedAt`.  
**Auth realm:** business users only (`/api/auth/business/*`); not customer realm.

### 10.1 User — Field Specification

| Field | PostgreSQL type | Nullable | Default | Constraints | Notes |
| --- | --- | --- | --- | --- | --- |
| `id` | `uuid` | no | app-generated UUID v7 | PK | PostgreSQL native `uuid`; **application-generated** UUID v7 (PA §21 / global conventions); not PG `uuidv7()` |
| `email` | `text` | no | — | **UNIQUE** (global); stored **normalized** | Trim + lowercase at write time (PA email normalization / §2 conventions). **Single column** — no separate `normalizedEmail`. Case-insensitive uniqueness = normalize-then-store + unique constraint (not `citext`). Max length **not** in source docs → TBD (not invented) |
| `passwordHash` | `text` | no | — | Argon2id encoded hash only | Plaintext never persisted. User is created on invitation accept with password set in the same transaction (PA §5 / API §3.2) → hash always present at insert |
| `name` | `text` | no | — | Trimmed non-empty | Data Model §2.2 “Profile name” for invites/UI; **not** appointment staff snapshot source (`Staff.displayName` is). Max length **not** in source docs → TBD (not invented). No MVP `/me` profile PATCH (API §14); still required on create for session/UI |
| `createdAt` | `timestamptz(3)` | no | set on create (UTC) | — | Authoritative creation time |
| `updatedAt` | `timestamptz(3)` | no | set on create/update (UTC) | — | Updated when mutable User fields change (e.g. password hash, name) |

**Explicit non-fields:**

| Candidate | Decision | Why |
| --- | --- | --- |
| `businessId` | **Omit** | User is global; tenant link is only via `BusinessMember` (PA §5.1, DM §2.2) |
| `normalizedEmail` (extra column) | **Omit** | Normalization applied into `email` (same pattern as `Business.email`; Customer keeps a separate column for soft-delete partial unique — N/A here) |
| Account / lifecycle status (`status`, `disabledAt`, …) | **Omit** | Access disable is **`BusinessMember`** active/disabled (PA staff deactivate; DM §2.3 / §10). No PA User status machine |
| `deletedAt` / soft delete | **Omit** | Soft delete is Customer-only in MVP (global conventions / DM §10) |
| Session token / cookie secret columns | **Omit** | Live on `BusinessSession` (hashed at rest) — PA §8.1 |
| Password-reset / invite raw secrets | **Omit** | Purpose-scoped **AuthToken** (logical) / `BusinessInvitation`; hashed at rest — PA §8.4, DM §7 |
| Customer-auth fields | **Omit** | Separate realm (`CustomerAccount` / `CustomerSession`) |

### 10.2 User — Identity rules

1. User is **global**; **no** `businessId` column.  
2. Business-user `email` is **globally unique** (PA §5.1; DM §2.2).  
3. Email normalization: **trim + lowercase** before persist; no dot/`+tag` stripping (§2 conventions).  
4. Case-insensitive email behavior in DB: **store only the normalized form** + **UNIQUE(`email`)** — equivalent uniqueness without `citext` or a functional unique index.  
5. Primary key: UUID v7, PostgreSQL native `uuid`, **generated in the application layer**.  
6. Password: never plaintext; only Argon2id hash in `passwordHash` (PA §8.1).  
7. Password reset (business realm): confirm consumes purpose-scoped reset token, sets new `passwordHash`, and **revokes all `BusinessSession` rows for that `userId`** (API §3.1 `password-reset/confirm`; aligned with PA session-revocation patterns on credential change). Reset secrets are **not** User columns.

### 10.3 User — Relations

| Relation | Cardinality | Child / peer notes |
| --- | --- | --- |
| User → `BusinessMember` | **1 : N** | Membership is the only User↔Business link; unique `(userId, businessId)` on member (DM §2.3). Schema may allow multiple memberships; MVP practice: one active membership (PA §5.1) |
| User → `BusinessSession` | **1 : N** | Business-realm sessions; each session carries `userId` + `activeBusinessId` (DM §2.4). Tokens hashed on session row, not on User |

**Logical (not User columns):** business password-reset AuthToken subject = User (DM §7.1). Physical AuthToken shape remains **architecture OPEN #1**.

Child FK **`ON DELETE`**: **architecture OPEN #3** — not decided on this entity pass.

#### Tenant isolation (User → BusinessMember)

`User → BusinessMember` does **not** weaken tenant isolation:

- User has no tenant column and is never a substitute for `businessId` filters.  
- `BusinessMember` **is** tenant-scoped (`businessId` required).  
- Authorization loads membership/role from DB every request; session holds identity + `activeBusinessId`, not authz (PA §8.1 / §7).  
- Tenant-owned data access must still scope by `BusinessMember.businessId` / session `activeBusinessId` (revalidated); client `businessId` is never trusted (API §2).  
- Listing a User’s memberships is identity→membership navigation; it must not be used to read another tenant’s Staff/Customer/Appointment rows without an authorized membership for that business.

`BusinessSession.activeBusinessId` is business context for the realm cookie; it does not make User tenant-scoped.

### 10.4 User — Uniqueness / indexes

| Rule | Scope | Kind |
| --- | --- | --- |
| Unique `id` (PK) | Global | FINAL persistence |
| Unique `email` (normalized) | Global | FINAL product/data invariant (PA / DM) |
| Email lookup (login / password-reset request) | Global by `email` | Served by the unique constraint’s supporting index — **no** extra secondary index required for MVP |

**Not added:** partial uniques, expression indexes, or `citext` — unnecessary once `email` is always normalized.

**Physical index DDL / extra secondary indexes** beyond known uniques remain **architecture OPEN #2**. Uniqueness of normalized `email` itself is not OPEN.

### 10.5 User — Lifecycle

```text
Invitation accept (new email)  → create User (email, name, passwordHash) + BusinessMember (+ Staff link if staff invite) + create authenticated BusinessSession
Invitation accept (existing User email, later multi-membership)
                               → no second User row; new BusinessMember only (schema allows; MVP practice usually one active membership)
Staff / membership deactivate  → BusinessMember disabled; BusinessSessions for that context revoked (PA §5.2)
                               → User row UNCHANGED (still global identity; reactivation = new login via membership)
Business password reset confirm → update passwordHash; revoke all BusinessSessions for userId
Logout / idle / absolute expiry → BusinessSession revoke/delete only
Operator business offboarding  → revoke business-user sessions (PA §6.9); does not delete User
```

**Do not conflate:**

- `BusinessMember` disabled ≠ User deleted or User “inactive” column.  
- User has **no** soft-delete.  
- MVP **hard-delete of User** is **not** a product offboarding action (domain: retain the identity row; no cascade-erase-user workflow in PA/DM). Implementation/cascade details deferred (OPEN #3).

### 10.6 User — Security alignment

| Requirement | User schema posture |
| --- | --- |
| Session tokens not plaintext on User | **Compliant** — sessions are `BusinessSession` (hashed at rest) |
| No credential secrets besides password hash on User | **Compliant** — only `passwordHash` |
| Reset tokens in separate security-token model | **Compliant** — AuthToken logical / not User columns (OPEN #1 for physical table) |
| Auth realm = business users | **Compliant** — User is business-realm identity only; customers use `CustomerAccount` |

### 10.7 User — Tenant / FK consistency

| Check | Result |
| --- | --- |
| User tenant-scoped? | **No** |
| BusinessMember tenant-scoped? | **Yes** (`businessId`) |
| BusinessSession carries business context? | **Yes** (`activeBusinessId`) |
| User → BusinessMember / BusinessSession open cross-tenant data access? | **No** — relations identify who may act; every tenant read/write still requires authorized membership + trusted server tenant context |

### 10.8 User — Review summary

**Final fields:** `id`, `email`, `passwordHash`, `name`, `createdAt`, `updatedAt`

**Unique constraints:**

- `PRIMARY KEY (id)`  
- `UNIQUE (email)` on normalized email  

**Indexes:**

- PK index on `id`  
- Unique index on `email` (covers login / reset lookup)  
- No additional User indexes in this pass  

**Relations:**

- `User` 1 — N `BusinessMember`  
- `User` 1 — N `BusinessSession`  
- Logical subject of business password-reset AuthToken  

**Remaining OPEN decisions (User-scoped only):**

| # | Topic | Notes |
| --- | --- | --- |
| — | *(none)* | No User-only schema OPENs. Email/name max lengths are **TBD from missing source limits** (same treatment as `Business.email`), not architecture OPENs. AuthToken physical model, exact secondary-index DDL, and FK `ON DELETE` remain **architecture OPENs #1–#3** (§6.1) and are not re-opened here. |

### 10.9 User — Consistency check

| Check | Result |
| --- | --- |
| Contradiction vs Product Architecture? | **None.** Global unique email; Argon2id; invite-creates-User; membership disable ≠ User delete; sessions/reset tokens off-row; UUID v7 app-generated. |
| Contradiction vs Data Model? | **None.** Fields match §2.2 (email, Argon2id hash, profile name, timestamps); relations match §11/§12; not tenant-scoped. |
| Contradiction vs API Contract? | **None.** Business login/session/password-reset + invitation accept create User; no `/me` profile API → no extra profile columns required; reset confirm revokes sessions (session entity, not User flags). |
| Missing field? | **No** required source field missing. `name` included per DM (not optional invent). |
| Unnecessary field? | **No** status/soft-delete/`businessId`/`normalizedEmail`/token columns added. |

### 10.10 User — Schema Notes

- Does **not** close architecture OPENs #1–#3.  
- #1 AuthToken: business password reset subject is User; physical table deferred.  
- #2 Indexes: `email` uniqueness FINAL; extra secondary indexes remain OPEN.  
- #3 FK on-delete for `BusinessMember` / `BusinessSession` → User deferred.  
- Prisma/`@db` mapping is implementation, not specified here.

---

## 11. Entity: BusinessMember — field-level design

**Kind:** Tenant-scoped User↔Business membership with role (concrete).  
**Sources:** PA §5.1–§5.2 / §7 / §8.1 / §10.2; Data Model §2.3 / §2.5 / §9 / §10 / §12; API Contract §2 / §3.1–§3.2 / §8.  
**Carries** `businessId` (tenant-owned).  
**Does not** use soft delete / `deletedAt`.  
**Does not** store session secrets or invitation tokens.

### 11.1 BusinessMember — Field Specification

| Field | PostgreSQL type | Nullable | Default | Constraints | Notes |
| --- | --- | --- | --- | --- | --- |
| `id` | `uuid` | no | app-generated UUID v7 | PK | PostgreSQL native `uuid`; application-generated UUID v7 (PA §21) |
| `businessId` | `uuid` | no | — | FK → `Business.id` | Tenant key; required on every membership row (DM §2.3 / §9) |
| `userId` | `uuid` | no | — | FK → `User.id` | Global User identity; membership is the only User↔Business link (PA §5.1) |
| `role` | enum `BusinessMemberRole` | no | — | ∈ {`OWNER`, `STAFF`} | PA roles; authorization reloaded from this row every request (not from session alone) |
| `status` | enum `BusinessMemberStatus` | no | **`ACTIVE`** | ∈ {`ACTIVE`, `DISABLED`} | Active / disabled dual-state from PA/DM (staff deactivate disables membership). Representation **FINAL** as named enum (resolves prior inventory “enum vs boolean” note for this entity) |
| `createdAt` | `timestamptz(3)` | no | set on create (UTC) | — | Authoritative creation time (DM §2.3) |
| `updatedAt` | `timestamptz(3)` | no | set on create/update (UTC) | — | Updated when role/status (or other mutable membership fields) change |

**Explicit non-fields:**

| Candidate | Decision | Why |
| --- | --- | --- |
| `staffId` on BusinessMember | **Omit** | Staff↔Member link is optional composite tenant-aware FK **on Staff** including `businessId` (DM §2.5 / PA §10.2). Cardinality rules live in relations (§11.3), not a Staff FK column here |
| `invitationId` | **Omit** | Invitation is consumed on accept; row is source of truth until then (API §3.2). No persistent FK required on membership |
| Session / token hash columns | **Omit** | Belong to `BusinessSession` / `BusinessInvitation` / AuthToken |
| Soft-delete `deletedAt` | **Omit** | Disable via `status = DISABLED`; history retained (PA §5.2) |
| Per-member permission bitset | **Omit** | Role enum only in MVP (PA §4 matrix) |

### 11.2 BusinessMember — Identity / invariants

1. PK `id` is UUID v7, app-generated, PG native `uuid`.  
2. Every row is tenant-scoped by non-null `businessId`.  
3. Unique membership per user per business: **`UNIQUE (userId, businessId)`** (DM §2.3).  
4. `role ∈ {OWNER, STAFF}` only.  
5. `status ∈ {ACTIVE, DISABLED}` only.  
6. MVP practice: exactly one Owner (or one pending owner invite) per business; schema **may** allow multiple Owners later — **no** DB unique forcing single Owner in MVP (PA §5.1).  
7. MVP practice: one active membership per user; schema **may** allow more (PA §5.1) — enforced by product practice, not an extra unique beyond `(userId, businessId)`.  
8. **STAFF** membership ↔ **exactly one** Staff profile; **OWNER** membership ↔ **zero or one** Staff profile (DM §2.5 / PA §5.2). Enforced when Staff link is designed (Staff-side composite FK); not a BusinessMember column.  
9. Disabled membership does **not** delete or disable the global `User` (User §10 / PA §5.2).  
10. Authorization: session supplies `userId` + `activeBusinessId`; **membership + role loaded from DB every request** (PA §8.1). Disabled membership → not authorized for that business.

### 11.3 BusinessMember — Relations

| Relation | Cardinality | Notes |
| --- | --- | --- |
| `Business` → BusinessMember | **1 : N** | Tenant parent |
| `User` → BusinessMember | **1 : N** | Global identity; unique pair `(userId, businessId)` |
| BusinessMember ↔ `Staff` | **0..1 : 0..1** (role-constrained) | Optional login link; FK composite including `businessId` on **Staff** (DM §2.5). STAFF ⇒ exactly one Staff; OWNER ⇒ 0 or 1 |
| `BusinessInvitation` → BusinessMember | **create-on-accept** | Accept transaction: consume invitation → create User (if needed) + BusinessMember (+ bind Staff if staff invite) + create authenticated BusinessSession (PA §5 / API §3.2). No lasting invitation FK on member |
| BusinessMember → `BusinessSession` | **indirect** | Sessions reference `userId` + `activeBusinessId`; membership disable **deletes** sessions for that user/business context (PA §5.2) — not a FK from Member → Session |

Child / peer FK **`ON DELETE`**: **architecture OPEN #3**.

### 11.4 BusinessMember — Uniqueness / indexes

| Rule | Scope | Kind |
| --- | --- | --- |
| Unique `id` (PK) | Global | FINAL persistence |
| Unique `(userId, businessId)` | Global pair | FINAL product/data invariant (DM §2.3) |
| Lookup membership for authz `(userId, businessId)` | Tenant + user | Served by the unique pair constraint — **no** extra secondary index required for that lookup |

**Not added:** partial unique “one OWNER per business” (explicitly not locked at schema — PA allows multiple Owners later); indexes beyond this unique/PK (architecture OPEN #2).

### 11.5 BusinessMember — Lifecycle

```text
Owner invite accept     → status=ACTIVE, role=OWNER (+ User create if new email)
Staff invite accept     → status=ACTIVE, role=STAFF + Staff↔Member bind
Staff deactivate        → membership status=DISABLED; sessions for that user/business deleted;
                          pending invites revoked; Staff bookable profile inactivated per PA;
                          history retained; User unchanged
Reactivation            → membership may return to ACTIVE; new login allowed (PA §5.2)
Owner’s own Staff profile deactivate
                        → does NOT touch OWNER membership (PA §5.2)
Business INACTIVE (public off)
                        → membership rows unchanged; admin panel still uses active memberships
Operator offboarding    → revoke sessions; memberships not hard-deleted as offboarding action (PA §6.9)
```

**Disabled semantics:** `DISABLED` membership cannot authorize Business API for that tenant; does not remove User; does not erase appointment/staff history.

### 11.6 BusinessMember — Security alignment

| Requirement | Posture |
| --- | --- |
| Role not trusted from client | **Compliant** — role column server-owned; API derives OWNER/STAFF from membership |
| Session alone not authorization | **Compliant** — every request reloads membership/status/role (PA §8.1) |
| No secrets on membership row | **Compliant** |
| Invite proves email ownership | **Compliant** — invite flow; no separate business-user email-verify column (PA out-of-scope list) |

### 11.7 BusinessMember — Tenant consistency

| Check | Result |
| --- | --- |
| Tenant-scoped? | **Yes** (`businessId`) |
| User remains global? | **Yes** — `userId` FK only |
| Cross-tenant leak via membership list? | **No** — queries must filter `businessId` / authorized membership; other tenants → API 404 |
| Staff link tenant-safe? | **Yes** — composite FK including `businessId` when present (DM §2.5) |

### 11.8 BusinessMember — Review summary

**Final fields:** `id`, `businessId`, `userId`, `role`, `status`, `createdAt`, `updatedAt`

**Unique constraints:**

- `PRIMARY KEY (id)`  
- `UNIQUE (userId, businessId)`  

**Indexes:**

- PK on `id`  
- Unique on `(userId, businessId)` (authz lookup)  
- No additional BusinessMember indexes in this pass  

**Relations:**

- N:1 `Business`, N:1 `User`  
- Optional Staff bind (Staff-side composite FK; role cardinalities above)  
- Created from `BusinessInvitation` accept; sessions revoked on disable  

**Remaining OPEN decisions (BusinessMember-scoped only):**

| # | Topic | Notes |
| --- | --- | --- |
| — | *(none)* | Status representation finalized as `BusinessMemberStatus`. Architecture OPENs #1–#3 unchanged. |

### 11.9 BusinessMember — Consistency check

| Check | Result |
| --- | --- |
| Contradiction vs Product Architecture? | **None.** OWNER/STAFF; disable on staff deactivate; sessions deleted; User untouched; invite accept creates member; MVP one-Owner practice without schema lock. |
| Contradiction vs Data Model? | **None.** Fields/uniques match §2.3; Staff link on Staff side §2.5; tenant-scoped §9. |
| Contradiction vs API Contract? | **None.** Session returns role + `activeBusinessId`; invite accept creates BusinessMember; Staff deactivate endpoint drives membership disable rules from PA. |
| Missing field? | **No.** Optional Staff association documented as relation, not invented `staffId` column. |
| Unnecessary field? | **No.** |

### 11.10 BusinessMember — Schema Notes

- Does **not** close architecture OPENs #1–#3.  
- #2: `(userId, businessId)` uniqueness FINAL; extra secondary indexes remain OPEN.  
- #3: FK on-delete for User/Business/Staff peers deferred.  
- Enum inventory `BusinessMemberStatus` values locked here to `ACTIVE` \| `DISABLED`.  
- Prisma/`@db` mapping is implementation, not specified here.

---

## 12. Entity: BusinessSession — field-level design

**Kind:** Server-side **business-realm** session (concrete).  
**Sources:** PA §7 / §8.1 / §5.2 / §6.9; Data Model §2.4 / §9 / §10 / §12; API Contract §3.1 / §3.2; User §10 distinction preserved.  
**Not** a normal tenant-owned business row: no `businessId` ownership column; **`activeBusinessId` is session context** (DM §9).  
**User** remains global; session is not authorization by itself.

### 12.1 BusinessSession — Field Specification

| Field | PostgreSQL type | Nullable | Default | Constraints | Notes |
| --- | --- | --- | --- | --- | --- |
| `id` | `uuid` | no | app-generated UUID v7 | PK | Session row id; **rotated** on login (new row / new id) per PA §8.1 — “session ID rotated on login” |
| `userId` | `uuid` | no | — | FK → `User.id` | Business-realm identity |
| `activeBusinessId` | `uuid` | no | — | FK → `Business.id` | Active tenant **context** only — not client-trusted tenant selector; revalidated with membership every request (PA §7 / §8.1) |
| `tokenHash` | `text` | no | — | **UNIQUE**; hash at rest | High-entropy session secret hashed at rest; raw token only in HTTP-only cookie / memory (PA §8.1 / secrets convention). Never plaintext on this row |
| `absoluteExpiresAt` | `timestamptz(3)` | no | — | must be > `createdAt` | Absolute session deadline. **Exact duration** deferred to technical docs (PA); store the computed deadline here |
| `lastUsedAt` | `timestamptz(3)` | no | set on create (UTC) | — | Activity watermark for **idle** expiry (PA idle+absolute). Idle duration itself deferred; compare `now()` vs `lastUsedAt` + configured idle window at request time |
| `createdAt` | `timestamptz(3)` | no | set on create (UTC) | — | Session mint time (rotation creates a new row) |

**Explicit non-fields:**

| Candidate | Decision | Why |
| --- | --- | --- |
| `businessId` (owner column) | **Omit** | Not a tenant-owned catalog row; context is `activeBusinessId` (DM §9) |
| `updatedAt` | **Omit** | Not listed in DM §2.4; mutations reflected by `lastUsedAt` / row replace on rotation |
| `role` / authz cache columns | **Omit** | Membership/role loaded from `BusinessMember` every request (PA §8.1) |
| `revokedAt` / status enum | **Omit** | Revocation = **destroy/delete** session row (PA “deletes sessions”; API logout/reset “destroy/revoke sessions”) |
| Raw session token | **Omit** | Hash only at rest |
| Customer-session fields | **Omit** | Separate `CustomerSession` realm |

### 12.2 BusinessSession — Identity / invariants

1. PK `id` is UUID v7, app-generated, PG native `uuid`.  
2. Holds **identity** (`userId`) + **context** (`activeBusinessId`) — **not** authorization.  
3. Every authenticated Business API request: resolve session by cookie → verify hash/expiry → load **ACTIVE** `BusinessMember` for `(userId, activeBusinessId)` and role from DB (PA §8.1; API §2).  
4. `tokenHash` uniquely identifies at most one live session.  
5. Idle + absolute expiry both enforced; durations not fixed in this doc (PA deferred).  
6. Separate cookie/table from customer realm (PA §8.1).  
7. User remains global; session does not make User tenant-scoped (User §10).

### 12.3 BusinessSession — Relations

| Relation | Cardinality | Notes |
| --- | --- | --- |
| `User` → BusinessSession | **1 : N** | All business-realm sessions for that user |
| `Business` ← `activeBusinessId` | **N : 1** | Context FK; session is not “owned” as a business child catalog row |
| `BusinessMember` | **no direct FK** | Authz join on `(userId, activeBusinessId)` + `status = ACTIVE` at request time |

FK **`ON DELETE`**: **architecture OPEN #3**.

### 12.4 BusinessSession — Uniqueness / indexes

| Rule | Scope | Kind |
| --- | --- | --- |
| Unique `id` (PK) | Global | FINAL persistence |
| Unique `tokenHash` | Global | FINAL — session cookie lookup by hash |
| Lookup by `userId` (bulk revoke on password reset) | Per user | Product need; **extra secondary index DDL** remains under architecture OPEN #2 (uniqueness of revoke behavior is FINAL; physical secondary index not locked here) |
| Lookup by `activeBusinessId` (offboarding / membership-disable session delete) | Per business context | Same — behavior FINAL; secondary index not locked (OPEN #2) |

**Not added:** indexes beyond PK + unique `tokenHash` as finalized invariants in this pass.

### 12.5 BusinessSession — Lifecycle

```text
Login                         → create authenticated BusinessSession (new id + tokenHash);
                                prior session id for that login rotation path invalidated (PA rotate on login)
Invitation accept             → create authenticated BusinessSession (same TX as User/Member)
Logout                        → destroy this session row (API §3.1)
Password-reset confirm        → revoke/destroy all BusinessSessions for that userId (API §3.1; User §10)
Membership disable            → delete sessions for that user + business context (PA §5.2)
Operator business offboarding → revoke business-user sessions for that business (PA §6.9)
Idle expiry                   → reject + destroy/ignore session when lastUsedAt older than idle window
Absolute expiry               → reject when now >= absoluteExpiresAt
Successful authenticated use  → advance lastUsedAt (idle watermark)
```

### 12.6 BusinessSession — Security alignment

| Requirement | Posture |
| --- | --- |
| Token hashed at rest | **Compliant** (`tokenHash`) |
| HTTP-only cookie transport | **Compliant** (API/PA; not a DB column) |
| Authz from membership DB | **Compliant** — no role on session |
| Instant revocation | **Compliant** — row delete; PG session store rationale (PA) |
| Realm split vs customer | **Compliant** — dedicated entity/cookie |

### 12.7 BusinessSession — Tenant-context behavior

| Check | Result |
| --- | --- |
| Normal tenant-owned row? | **No** |
| `activeBusinessId` meaning | Session’s active business **context**; revalidated every request |
| Client may set tenant via body/`businessId`? | **No** — trusted context from session only (API §2) |
| Cross-tenant access if membership missing/disabled? | **Denied** (404/unauthorized per API/PA); session alone insufficient |
| Alignment with User §10 | **Preserved** — User global; Member tenant-scoped; Session contextual |

### 12.8 BusinessSession — Review summary

**Final fields:** `id`, `userId`, `activeBusinessId`, `tokenHash`, `absoluteExpiresAt`, `lastUsedAt`, `createdAt`

**Unique constraints:**

- `PRIMARY KEY (id)`  
- `UNIQUE (tokenHash)`  

**Indexes:**

- PK on `id`  
- Unique on `tokenHash`  
- No additional secondary indexes locked in this pass (revoke-by-user / by-business remain OPEN #2 for DDL)  

**Relations:**

- N:1 `User`  
- N:1 `Business` via `activeBusinessId` (context)  
- Logical join to `BusinessMember` for authz (no FK)  

**Remaining OPEN decisions (BusinessSession-scoped only):**

| # | Topic | Notes |
| --- | --- | --- |
| — | *(none)* | Idle/absolute **durations** remain deferred technical config (PA), not schema OPENs. Architecture OPENs #1–#3 unchanged. |

### 12.9 BusinessSession — Consistency check

| Check | Result |
| --- | --- |
| Contradiction vs Product Architecture? | **None.** Hashed token; idle+absolute; rotate on login; membership from DB; deletes on disable/offboarding/reset. |
| Contradiction vs Data Model? | **None.** §2.4 fields/concepts covered; contextual tenancy §9; not conflated with User lifecycle. |
| Contradiction vs API Contract? | **None.** Login/logout/session/password-reset/invite-accept behaviors map to create/destroy/revoke session rows. |
| Missing field? | **No** required source concept missing. `lastUsedAt` included only to support FINAL idle expiry. `updatedAt` omitted (not in DM). |
| Unnecessary field? | **No** role/raw token/`businessId` owner column/status soft-revoke. |

### 12.10 BusinessSession — Schema Notes

- Does **not** close architecture OPENs #1–#3.  
- #1 AuthToken: unrelated to session cookie hash (sessions are this entity).  
- #2: unique `tokenHash` FINAL; secondary indexes for bulk revoke remain OPEN.  
- #3: FK on-delete toward User/Business deferred.  
- Exact idle/absolute durations: technical documents (PA), not architecture schema OPENs.  
- Prisma/`@db` mapping is implementation, not specified here.

---

## 13. Entity: Staff — field-level design

**Kind:** Business-owned **bookable** profile (concrete).  
**Sources:** PA §5.2 / §10.2 / §6.5 / §11; Data Model §2.5 / §10 / §11 / §12; API Contract §8; BusinessMember §11.  
**Tenant-scoped** via `businessId`.  
**May exist without login** (no required `BusinessMember`).  
**Does not** soft-delete; uses `active` boolean.

### 13.1 Staff — Field Specification

| Field | PostgreSQL type | Nullable | Default | Constraints | Notes |
| --- | --- | --- | --- | --- | --- |
| `id` | `uuid` | no | app-generated UUID v7 | PK | PG native `uuid`; app-generated (PA §21). Also used as stable input for UI staff-color token hashing (PA §14.2) — **color is not a stored column** |
| `businessId` | `uuid` | no | — | FK → `Business.id` | Tenant key |
| `displayName` | `text` | no | — | Trimmed non-empty | Public UI + appointment **staff-name snapshot** source (PA §10.2 / DM §2.5). Max length **not** in source docs → TBD (not invented) |
| `active` | `boolean` | no | **`true`** | — | Inactive → no new bookings; existing CONFIRMED continue; history retained (PA §10.2 / DM §10). Boolean per global conventions (not an enum) |
| `businessMemberId` | `uuid` | yes | `null` | FK composite with `businessId` → `BusinessMember(businessId, id)` when set; **UNIQUE** among non-null | Optional login link (PA §10.2 / DM §2.5). `null` = staff without panel login |
| `createdAt` | `timestamptz(3)` | no | set on create (UTC) | — | |
| `updatedAt` | `timestamptz(3)` | no | set on create/update (UTC) | — | |

**Explicit non-fields:** profile photo, bio, stored calendar color, permissions bitset, phone/email on Staff (login identity is User; bookable label is `displayName` only).

### 13.2 Staff — Identity / invariants

1. PK UUID v7, app-generated, PG `uuid`.  
2. Always tenant-scoped (`businessId` required).  
3. Staff **can exist without** `businessMemberId` (create bookable staff; invite later — API §8 / PA §5.2).  
4. When `businessMemberId` is set: composite tenant-aware FK including `businessId` (same business as member).  
5. At most one Staff per `BusinessMember` (`businessMemberId` unique when non-null).  
6. **STAFF** membership ↔ **exactly one** Staff; **OWNER** membership ↔ **zero or one** Staff (DM §2.5 / PA §5.2) — application enforces role cardinalities with this optional link.  
7. Deactivate **blocked** if future `CONFIRMED` appointments exist for this staff (PA §5.2 / API §8).  
8. On successful Staff deactivate (`Staff.active = false`), side effects depend on the optional membership link (PA §5.2):  
   - **Linked to `STAFF` `BusinessMember`:** set that member `status = DISABLED`; revoke/delete `BusinessSession`s for that user in this business context; revoke pending business invitations for that Staff/member context; appointment history retained; Staff may later be reactivated (and membership re-enabled for new login).  
   - **Linked to `OWNER` `BusinessMember`:** **only** `Staff.active = false`; OWNER member `status` **unchanged**; Owner `BusinessSession`s **not** revoked/deleted; OWNER membership stays active; appointment history retained; Staff may later be reactivated.  
   - **No login link** (`businessMemberId` null): **only** `Staff.active = false`; no membership/session/invitation side effects; appointment history retained; Staff may later be reactivated.  
9. Appointment history / snapshots retained; staff rename does not rewrite past snapshots (updated only on reschedule staff change per PA).  
10. Public profile lists bookable staff `displayName`s among eligible active staff (PA §6.5) — inactive staff excluded from new public booking/assignment.

### 13.3 Staff — Relations

| Relation | Cardinality | Notes |
| --- | --- | --- |
| `Business` → Staff | **1 : N** | Tenant parent |
| Staff → `BusinessMember` | **0..1 : 0..1** | Optional; Staff-side `businessMemberId` + `businessId` composite FK |
| Staff → `StaffService` | **1 : N** | Offerings; replace set via API |
| Staff → `StaffWorkingHour` | **1 : N** | Weekly hours — [§17](#17-entity-staffworkinghour--field-level-design) |
| Staff → `StaffTimeOff` | **1 : N** | Full-day exclusions — [§19](#19-entity-stafftimeoff--field-level-design) |
| Staff → `Appointment` | **1 : N** | `staffId` party; exclusion constraint per staff (PA) |
| `BusinessInvitation` → Staff | **optional `staffId`** | Staff invites bound to staff row (DM / API) |

FK **`ON DELETE`**: architecture OPEN #3.

### 13.4 Staff — Uniqueness / indexes

| Rule | Scope | Kind |
| --- | --- | --- |
| Unique `id` (PK) | Global | FINAL |
| Unique `businessMemberId` (when not null) | Global among linked staff | FINAL — one Staff per membership link |
| Tenant list by `businessId` | Tenant | Lookup need; **secondary index DDL** under architecture OPEN #2 |

No other Staff uniques required by sources (no unique `displayName`).

### 13.5 Staff — Lifecycle

```text
Create (Owner)           → active=true; businessMemberId null unless bound
Invite + accept          → set businessMemberId (Staff role) or Owner optional bind
Activate                 → active=true
Deactivate               → blocked if future CONFIRMED; else Staff.active=false, then:
                           STAFF-linked  → BusinessMember.status=DISABLED;
                                           revoke/delete sessions for that user+business context;
                                           revoke pending invites for that Staff/member context;
                                           history retained
                           OWNER-linked  → Staff.active=false only;
                                           OWNER BusinessMember unchanged (stays ACTIVE);
                                           Owner sessions NOT revoked;
                                           history retained
                           Unlinked      → Staff.active=false only;
                                           no membership/session/invite side effects;
                                           history retained
Reactivate               → Staff.active=true;
                           STAFF-linked: membership may be re-enabled for new login;
                           OWNER-linked / unlinked: bookable again (Owner membership was never disabled)
Unlink services / hours  → never auto-cancel appointments (PA §10.2)
```

### 13.6 Staff — Security / tenant consistency

| Check | Result |
| --- | --- |
| Tenant-scoped? | **Yes** (`businessId`) |
| Cross-tenant Staff id? | API **404** |
| STAFF role data scope | Own staff profile only (API/PA) — enforced via membership→Staff link, not extra Staff columns |
| Login optional? | **Yes** — `businessMemberId` null allowed |
| Colors/permissions stored? | **No** |

### 13.7 Staff — Review summary

**Final fields:** `id`, `businessId`, `displayName`, `active`, `businessMemberId`, `createdAt`, `updatedAt`

**Unique constraints:** PK(`id`); UNIQUE(`businessMemberId`) where non-null (partial unique at implementation)

**Indexes:** PK; unique on link; other tenant secondary indexes → OPEN #2

**Relations:** Business; optional BusinessMember; StaffService; StaffWorkingHour; StaffTimeOff; Appointment; BusinessInvitation

**Remaining OPEN decisions (Staff-scoped only):** none (displayName max length TBD from missing source limit — not an architecture OPEN)

### 13.8 Staff — Consistency check

| Check | Result |
| --- | --- |
| vs Product Architecture? | **None.** Optional member link; owner-as-staff 0..1; deactivate rules; no invented profile fields. |
| vs Data Model? | **None.** §2.5 concepts covered; boolean active. |
| vs API Contract? | **None.** Create without login; activate/deactivate; services/hours/time-off children. |
| Missing field? | **No.** |
| Unnecessary field? | **No** photo/bio/color/permissions. |

### 13.9 Staff — Schema Notes

- Does **not** close architecture OPENs #1–#3.  
- Composite FK spelling / partial-unique DDL are implementation; uniqueness of member link is FINAL.  
- Calendar color tokens are derived at UI layer from `id`, not persisted (PA §14.2).

---

## 14. Entity: Service — field-level design

**Kind:** Business-owned offerable catalog item (concrete).  
**Sources:** PA §10.1 / §10.3 / §6.5; Data Model §3.1 / §4.2 / §10; API Contract §7; money conventions §2.  
**Tenant-scoped.** No hard delete; `active` boolean. **No payments/deposits** in MVP.

### 14.1 Service — Field Specification

| Field | PostgreSQL type | Nullable | Default | Constraints | Notes |
| --- | --- | --- | --- | --- | --- |
| `id` | `uuid` | no | app-generated UUID v7 | PK | PG native `uuid`; app-generated |
| `businessId` | `uuid` | no | — | FK → `Business.id` | Tenant key |
| `name` | `text` | no | — | Trimmed non-empty | Public + admin catalog. Max length **not** in source docs → TBD |
| `durationMinutes` | `int` | no | — | **> 0** | Independent of business `slotIntervalMinutes` (PA §10.1). Exact upper bound not in sources → not invented |
| `bufferMinutes` | `int` | no | — | **≥ 0** | **Buffer after** appointment only (PA §11.4). Column name per PA/DM (`bufferMinutes`); not a before-buffer |
| `priceMinor` | `int` | no | — | Integer minor units | No floats (PA §10.3). Free/`0` allowed unless later product rule says otherwise — sources do not forbid 0 |
| `currency` | `text` | no | — | Active ISO 4217; 3-letter uppercase | **Aligned with `Business.currency` at write** (PA §10.1 / DM §3.1). Unknown codes rejected. Not a fixed allow-list |
| `active` | `boolean` | no | **`true`** | — | Inactive → no new bookings; existing CONFIRMED continue (PA §10.1) |
| `createdAt` | `timestamptz(3)` | no | set on create (UTC) | — | |
| `updatedAt` | `timestamptz(3)` | no | set on create/update (UTC) | — | |

**Explicit non-fields:** category, tax, inventory, SKU, images, deposit/payment fields.

### 14.2 Service — Identity / invariants

1. PK UUID v7, app-generated.  
2. Tenant-scoped via `businessId`.  
3. `durationMinutes` independent of slot interval; scheduling uses duration for `[startsAt, endsAt)` fit.  
4. `bufferMinutes` is after-only; contributes to `blockedUntil` at booking time from **snapshot**, not live Service edits (PA §11.4 / §10.1).  
5. `currency` must match business currency policy at write; business currency immutable after first publish (PA §6.3) — services written afterward use that currency.  
6. Money: integer minor units + ISO currency on Service; Appointment stores **snapshot** `priceMinor` + `currency` at booking (PA §10.3). Live Service mutations **never** alter historical snapshots or stored `endsAt` / `blockedUntil`.  
7. No payments, checkout, or deposits in MVP.  
8. Public ACTIVE business profile shows **active** services (name, duration, price) (PA §6.5).  
9. Override / booking paths never bypass inactive service (PA §11.7 / API §10).

### 14.3 Service — Relations

| Relation | Cardinality | Notes |
| --- | --- | --- |
| `Business` → Service | **1 : N** | Tenant parent; currency source of alignment |
| Service → `StaffService` | **1 : N** | Which staff offer this service |
| Service → `Appointment` | **1 : N** | `serviceId` + service field snapshots on appointment |

FK **`ON DELETE`**: architecture OPEN #3.

### 14.4 Service — Uniqueness / indexes

| Rule | Scope | Kind |
| --- | --- | --- |
| Unique `id` (PK) | Global | FINAL |
| Unique service name per business? | — | **Not** required by sources |
| Tenant list by `businessId` | Tenant | Secondary index → OPEN #2 |

### 14.5 Service — Lifecycle

```text
Create     → active=true; currency aligned to Business.currency; duration/buffer/price set
Update     → may change name/duration/buffer/priceMinor (and currency only while business rules allow);
             does not rewrite appointment snapshots / endsAt / blockedUntil
Activate   → active=true
Deactivate → active=false; no new bookings; existing CONFIRMED continue; no hard delete
```

### 14.6 Service — Security / tenant consistency

| Check | Result |
| --- | --- |
| Tenant-scoped? | **Yes** |
| Staff may mutate Service? | **No** — Owner only; Staff read-only catalog (API §7) |
| Cross-tenant? | **404** |
| Payments columns? | **None** |

### 14.7 Service — Review summary

**Final fields:** `id`, `businessId`, `name`, `durationMinutes`, `bufferMinutes`, `priceMinor`, `currency`, `active`, `createdAt`, `updatedAt`

**Unique constraints:** PK(`id`) only (product)

**Indexes:** PK; tenant secondary → OPEN #2

**Relations:** Business; StaffService; Appointment (snapshots)

**Remaining OPEN decisions (Service-scoped only):** none (`name` max length / duration upper bound TBD from missing source limits)

### 14.8 Service — Consistency check

| Check | Result |
| --- | --- |
| vs Product Architecture? | **None.** Fields match §10.1; buffer-after; money rules; no payments. |
| vs Data Model? | **None.** §3.1; boolean active. Column named `bufferMinutes` (not invented `bufferAfterMinutes`). |
| vs API Contract? | **None.** Explicit activate/deactivate; Owner mutate; Staff read. |
| Missing field? | **No.** |
| Unnecessary field? | **No** category/tax/SKU/image/deposit. |

### 14.9 Service — Schema Notes

- Does **not** close architecture OPENs #1–#3.  
- Appointment snapshot columns are specified on Appointment (later), not duplicated here.  
- `bufferMinutes` ≡ after-buffer semantics from PA §11.4.

---

## 15. Entity: StaffService — field-level design

**Kind:** Tenant-scoped Staff↔Service **M:N** link (concrete).  
**Sources:** PA §10.1–§10.2 / §11.7; Data Model §3.2; API Contract §8 (GET/PUT replace).  
**Not** a soft-delete entity; rows inserted/deleted by replacement.

### 15.1 StaffService — Field Specification

| Field | PostgreSQL type | Nullable | Default | Constraints | Notes |
| --- | --- | --- | --- | --- | --- |
| `id` | `uuid` | no | app-generated UUID v7 | PK | Global convention: all concrete PKs UUID v7. Sources do not require a composite-only PK; surrogate `id` kept for consistency with other entities |
| `businessId` | `uuid` | no | — | FK → `Business.id`; part of composite FKs | Tenant key denormalized for tenant-safe composites (DM §3.2 / §9) |
| `staffId` | `uuid` | no | — | Composite FK `(businessId, staffId)` → `Staff(businessId, id)` | Same-business staff only |
| `serviceId` | `uuid` | no | — | Composite FK `(businessId, serviceId)` → `Service(businessId, id)` | Same-business service only |
| `createdAt` | `timestamptz(3)` | no | set on create (UTC) | — | Row creation time; set is replaced wholesale (no in-place field updates in API) |

**Explicit non-fields:** `updatedAt` (replacement deletes/inserts; API has no PATCH on link rows); per-link “priority” / custom duration overrides; active flag on the link (inactive Staff/Service handled on parent rows; inactive services **may remain linked** — API §8).

### 15.2 StaffService — Identity / invariants

1. M:N between Staff and Service within one business.  
2. **UNIQUE `(businessId, staffId, serviceId)`** — duplicate prevention (DM §3.2).  
3. Tenant-safe composites: staff and service must share `businessId` (enforced by composite FKs).  
4. API **replacement only**: `PUT /api/business/staff/:id/services` with full `serviceIds[]`; no attach/detach endpoints (API §8 FINAL).  
5. Unlink (row removed from set) **does not** mutate appointments; future bookings cannot use the pair (PA §10.2 / API §8).  
6. Inactive Service or Staff: link rows may still exist; scheduling/booking must still reject inactive staff/service and missing StaffService pair (PA §11.7 — override never bypasses inactive or missing relationship).  
7. Go-live checklist requires active staff offering ≥1 **active** service (PA §5.3) — derived from Staff + Service `active` plus StaffService presence, not a column here.

### 15.3 StaffService — Relations

| Relation | Cardinality | Notes |
| --- | --- | --- |
| `Staff` → StaffService | **1 : N** | |
| `Service` → StaffService | **1 : N** | |
| `Business` → StaffService | **1 : N** | Via `businessId` |
| Appointment | **none direct** | Appointments reference `staffId`/`serviceId`; do not FK to StaffService row |

FK **`ON DELETE`**: architecture OPEN #3 (must not cascade-erase appointment history).

### 15.4 StaffService — Uniqueness / indexes

| Rule | Scope | Kind |
| --- | --- | --- |
| Unique `id` (PK) | Global | FINAL persistence convention |
| Unique `(businessId, staffId, serviceId)` | Tenant triple | FINAL product/data invariant (DM §3.2) |
| List by staff / by service | — | Covered by unique triple leading columns / OPEN #2 for extra DDL |

### 15.5 StaffService — Lifecycle

```text
PUT replace set     → diff desired serviceIds vs current rows; insert missing; delete extras (same business)
Unlink              → delete StaffService row; appointments unchanged; pair unavailable for new bookings
Staff/Service deactivate → links may remain; engine excludes inactive parents + requires link for new bookable pairs
Hard delete parents → not an MVP product action; FK ON DELETE OPEN #3
```

### 15.6 StaffService — Scheduling engine implications

- Eligible bookable pair = StaffService row **and** `Staff.active` **and** `Service.active` (plus hours/time-off/closed-date/overlap rules).  
- Owner `overrideHours` never bypasses missing StaffService or inactive staff/service (PA §11.7 / API §10).  
- Fark etmez / alternatives only among staff linked to the service and available for the slot (PA §11.8).  
- Buffer/duration come from Service (snapshot at booking), not from StaffService.

### 15.7 StaffService — Security / tenant consistency

| Check | Result |
| --- | --- |
| Tenant-scoped? | **Yes** (`businessId` + composite FKs) |
| Who mutates? | **Owner only** (`PUT`); Staff may `GET` own staff’s links (API §8) |
| Cross-tenant serviceId in PUT? | Reject — same-business ids only (API §8) |
| Duplicate links? | Prevented by unique triple |

### 15.8 StaffService — Review summary

**Final fields:** `id`, `businessId`, `staffId`, `serviceId`, `createdAt`

**Unique constraints:** PK(`id`); UNIQUE(`businessId`, `staffId`, `serviceId`)

**Indexes:** PK + unique triple; further secondary → OPEN #2

**Relations:** Business, Staff, Service (M:N); no FK from Appointment to this row

**Remaining OPEN decisions (StaffService-scoped only):** none

### 15.9 StaffService — Consistency check

| Check | Result |
| --- | --- |
| vs Product Architecture? | **None.** M:N; unlink preserves appointments; inactive parents block booking. |
| vs Data Model? | **None.** §3.2 fields/unique/replace semantics. Surrogate UUID PK added per global PK convention (sources silent on composite-only PK). |
| vs API Contract? | **None.** Replacement PUT; inactive services may remain linked; Owner-only mutate. |
| Missing field? | **No.** |
| Unnecessary field? | **No** link-level active/priority/overrides. |

### 15.10 StaffService — Schema Notes

- Does **not** close architecture OPENs #1–#3.  
- #2: unique triple FINAL; extra indexes OPEN.  
- #3: ON DELETE must preserve appointments when links/parents change.  
- Prisma/`@db` mapping is implementation, not specified here.

---

## 16. Entity: BusinessWorkingHour — field-level design

**Kind:** Business weekly default open intervals (concrete).  
**Sources:** PA §5.3 / §11.2–§11.3; Data Model §3.3; API Contract §9; global conventions §2.  
**Tenant-scoped.** Local wall-clock minutes — **not** UTC timestamps, **not** `@db.Time`.  
**No overnight** intervals (PA FINAL).

### 16.1 BusinessWorkingHour — Field Specification

| Field | PostgreSQL type | Nullable | Default | Constraints | Notes |
| --- | --- | --- | --- | --- | --- |
| `id` | `uuid` | no | app-generated UUID v7 | PK | PG native `uuid`; app-generated |
| `businessId` | `uuid` | no | — | FK → `Business.id` | Tenant key |
| `dayOfWeek` | `int` | no | — | ∈ {1…7} ISO | `1 = Monday` … `7 = Sunday` (PA §11.2) |
| `startMinute` | `int` | no | — | multiple of 5; `0 ≤ startMinute < endMinute ≤ 1440` | Business-local minute-of-day; end exclusive (PA §11.2) |
| `endMinute` | `int` | no | — | multiple of 5; same bounds | Local wall-clock; not a timestamptz |
| `createdAt` | `timestamptz(3)` | no | set on create (UTC) | — | Rows inserted by full-schedule replace |

**Explicit non-fields:** `updatedAt` (API replaces full weekly set — API §9); overnight intervals; `@db.Time` columns; timezone column (use `Business.timezone`); name/label.

### 16.2 BusinessWorkingHour — Identity / invariants

1. PK UUID v7, app-generated.  
2. Tenant-scoped via `businessId`.  
3. **Multiple intervals per weekday allowed** if they **do not overlap**; they **may touch** (PA §11.2 / DM §3.3). Overlap validation is application/engine (fetch → validate → sort → merge in memory — merge not persisted).  
4. No overnight intervals: always `startMinute < endMinute` within one local day (`≤ 1440`).  
5. Minutes are multiples of 5.  
6. Not UTC instants; interpreted with `Business.timezone` (IANA) at engine time.  
7. Go-live checklist requires ≥1 business working-hours interval (PA §5.3).  
8. Schedule changes never auto-cancel appointments; only affect future availability (PA §10.2).  
9. Inactive `Business` (`status = INACTIVE`): **no new public booking** (PA §6.5); weekly hour rows remain; admin/manual paths still use engine rules with Owner override where allowed.

### 16.3 BusinessWorkingHour — Relations

| Relation | Cardinality | Notes |
| --- | --- | --- |
| `Business` → BusinessWorkingHour | **1 : N** | Weekly schedule rows |
| Effective availability | with `StaffWorkingHour` | **Intersection** `business hours ∩ staff hours` (PA §11.2) — not replaced by staff-only hours |

FK **`ON DELETE`**: architecture OPEN #3.

### 16.4 BusinessWorkingHour — Uniqueness / indexes

| Rule | Scope | Kind |
| --- | --- | --- |
| Unique `id` (PK) | Global | FINAL |
| Unique `(businessId, dayOfWeek, startMinute)`? | — | **Not** required; multiple intervals/day allowed |
| Non-overlap per `(businessId, dayOfWeek)` | Same day | App invariant (may touch); not a DB exclusion in sources |
| List by business | Tenant | Secondary index → OPEN #2 |

### 16.5 BusinessWorkingHour — Lifecycle

```text
PUT /api/business/working-hours  → replace full weekly set (delete+insert); Owner only
GET                              → list current rows
No per-row PATCH/DELETE surface in API inventory (replace only)
```

### 16.6 BusinessWorkingHour — Scheduling engine implications

- Supplies **business** side of `business hours ∩ staff hours − closed − time off − blockers − buffer` (PA §11.2).  
- Slot grid uses business `slotIntervalMinutes`; appointment must fit in a single business∩staff interval (PA §11.3).  
- Browser TZ never used; Luxon + `Business.timezone`.

### 16.7 BusinessWorkingHour — Security / tenant consistency

| Check | Result |
| --- | --- |
| Tenant-scoped? | **Yes** |
| Mutate | **Owner only** (API §9) |
| Cross-tenant | **404** |

### 16.8 BusinessWorkingHour — Review summary

**Final fields:** `id`, `businessId`, `dayOfWeek`, `startMinute`, `endMinute`, `createdAt`

**Uniques:** PK(`id`) only  

**Indexes:** PK; extras → OPEN #2  

**Relations:** N:1 Business; intersects with StaffWorkingHour at engine  

**Entity-level OPENs:** none  

### 16.9 BusinessWorkingHour — Consistency check

| Check | Result |
| --- | --- |
| vs PA / DM / API? | **None.** Multi-interval/day; no overnight; replace PUT; local minutes. |
| Missing / extra? | **No** invented labels/overnight/partial recurrence. |

### 16.10 BusinessWorkingHour — Schema Notes

- Does **not** close architecture OPENs #1–#3.  
- Overlap checks are application invariants, not new architecture OPENs.

---

## 17. Entity: StaffWorkingHour — field-level design

**Kind:** Staff weekly open intervals (concrete).  
**Sources:** PA §5.3 / §11.2–§11.3; Data Model §3.4; API Contract §9; Staff §13.  
**Same shape as BusinessWorkingHour**, scoped by `businessId` + `staffId`.

### 17.1 StaffWorkingHour — Field Specification

| Field | PostgreSQL type | Nullable | Default | Constraints | Notes |
| --- | --- | --- | --- | --- | --- |
| `id` | `uuid` | no | app-generated UUID v7 | PK | |
| `businessId` | `uuid` | no | — | FK → `Business.id`; composite with `staffId` | Tenant key |
| `staffId` | `uuid` | no | — | Composite FK `(businessId, staffId)` → `Staff(businessId, id)` | Same-business staff |
| `dayOfWeek` | `int` | no | — | ∈ {1…7} ISO | Same as business hours |
| `startMinute` | `int` | no | — | multiple of 5; `0 ≤ start < end ≤ 1440` | Local wall-clock; no overnight |
| `endMinute` | `int` | no | — | multiple of 5; end exclusive | |
| `createdAt` | `timestamptz(3)` | no | set on create (UTC) | — | Full-schedule replace |

**Explicit non-fields:** `updatedAt`; overnight; stored “inherits business hours” flag; `@db.Time`.

### 17.2 StaffWorkingHour — Identity / invariants

1. Same minute/dayOfWeek/no-overnight/multi-interval-touch rules as BusinessWorkingHour.  
2. Tenant-safe: `staffId` must belong to `businessId`.  
3. **Effective weekly open time for a staff/day** = **intersection** of that day’s BusinessWorkingHour intervals with that staff’s StaffWorkingHour intervals (PA conceptual formula).  
4. **No fallback:** empty StaffWorkingHour set ⇒ intersection empty for that staff (staff not bookable on weekly hours alone). Sources do **not** say “use business hours when staff hours missing.”  
5. Go-live requires that staff **has working hours** (PA §5.3 item 5) — StaffWorkingHour rows must exist for activation checklist.  
6. `Staff.active = false` ⇒ staff excluded from new booking/assignment regardless of hour rows (Staff §13 / PA). Hour rows retained.  
7. Schedule changes never auto-cancel appointments (PA §10.2).

### 17.3 StaffWorkingHour — Relations

| Relation | Cardinality | Notes |
| --- | --- | --- |
| `Staff` → StaffWorkingHour | **1 : N** | |
| `Business` → StaffWorkingHour | **1 : N** | via `businessId` |
| vs BusinessWorkingHour | **intersect** | Not parent/child FK; engine ∩ |

FK **`ON DELETE`**: architecture OPEN #3.

### 17.4 StaffWorkingHour — Uniqueness / indexes

| Rule | Scope | Kind |
| --- | --- | --- |
| Unique `id` (PK) | Global | FINAL |
| Non-overlap per `(staffId, dayOfWeek)` | Same staff/day | App invariant (may touch) |
| List by staff | — | Secondary → OPEN #2 |

### 17.5 StaffWorkingHour — Lifecycle

```text
PUT /api/business/staff/:id/working-hours → replace full weekly set; Owner only
GET                                         → list
Staff deactivate                            → hour rows retained; staff unavailable for new bookings
```

### 17.6 StaffWorkingHour — Scheduling engine implications

- Staff side of `business hours ∩ staff hours`.  
- Appointment must fit in one intersected interval (PA §11.3).  
- Owner `overrideHours` may relax out-of-hours; Staff path cannot (PA §11.7).

### 17.7 StaffWorkingHour — Security / tenant consistency

| Check | Result |
| --- | --- |
| Tenant-scoped? | **Yes** + composite FK |
| Mutate | **Owner only**; Staff cannot edit own hours (API §9 / PA §4.2) |

### 17.8 StaffWorkingHour — Review summary

**Final fields:** `id`, `businessId`, `staffId`, `dayOfWeek`, `startMinute`, `endMinute`, `createdAt`

**Uniques:** PK(`id`)  

**Entity-level OPENs:** none  

### 17.9 StaffWorkingHour — Consistency check

| Check | Result |
| --- | --- |
| vs sources? | **None.** Same shape as business hours; ∩ not fallback. |
| Invented fallback? | **No** — explicitly rejected as unsupported by PA formula + checklist. |

### 17.10 StaffWorkingHour — Schema Notes

- Does **not** close architecture OPENs #1–#3.

---

## 18. Entity: BusinessClosedDate — field-level design

**Kind:** Business-level **full-day** closure on a business-local calendar date (concrete).  
**Sources:** PA §11.2; Data Model §3.6; API Contract §9; conventions §2.

### 18.1 BusinessClosedDate — Field Specification

| Field | PostgreSQL type | Nullable | Default | Constraints | Notes |
| --- | --- | --- | --- | --- | --- |
| `id` | `uuid` | no | app-generated UUID v7 | PK | |
| `businessId` | `uuid` | no | — | FK → `Business.id` | Tenant key |
| `localDate` | `date` | no | — | **UNIQUE (`businessId`, `localDate`)** | Business-TZ calendar date (`YYYY-MM-DD`); **not** a UTC instant; **not** fixed 24h elapsed storage |
| `createdAt` | `timestamptz(3)` | no | set on create (UTC) | — | |

**Explicit non-fields:** name/reason/note (not in sources); `updatedAt` (API create/delete only — no PATCH); recurrence/holiday rules; stored UTC `[start,end)` pair (engine derives from `localDate` + `Business.timezone`).

### 18.2 BusinessClosedDate — Identity / invariants

1. Unique per `(businessId, localDate)` (PA / DM FINAL).  
2. Closure = **full local day** in business timezone: `[local 00:00 that date, local 00:00 next date)` — DST may yield 23h/25h elapsed (same mental model as TimeOff; not “24 hours”).  
3. Public availability: **no slots** that day; new public bookings blocked (PA §11.2).  
4. New manual bookings blocked under normal rules; Owner may use `overrideHours` (PA §11.2 / §11.7).  
5. Existing appointments **not** auto-cancelled or mutated; reminders continue (PA §11.2).  
6. Past `localDate` values may exist (history/audit); no forced purge semantics in sources.  
7. Does not rewrite appointment snapshots.

### 18.3 BusinessClosedDate — Relations

| Relation | Cardinality | Notes |
| --- | --- | --- |
| `Business` → BusinessClosedDate | **1 : N** | |
| vs StaffTimeOff | independent subtract | Both subtract from availability; closed date is business-wide |

FK **`ON DELETE`**: architecture OPEN #3.

### 18.4 BusinessClosedDate — Uniqueness / indexes

| Rule | Scope | Kind |
| --- | --- | --- |
| Unique `id` (PK) | Global | FINAL |
| Unique `(businessId, localDate)` | Tenant | FINAL product invariant |
| Extras | — | OPEN #2 |

### 18.5 BusinessClosedDate — Lifecycle

```text
POST /api/business/closed-dates     → create full-day closure (YYYY-MM-DD local); Owner
DELETE /api/business/closed-dates/:id → remove closure; Owner
GET list                            → Owner
No recurrence / no auto-delete of past rows required by sources
```

### 18.6 BusinessClosedDate — Scheduling engine implications

- Applied after TZ → local date resolution; removes that local day from availability before/with hours (PA conversion order: hours − closed − time off).  
- Overrides weekly open hours for that calendar date (business closed even if weekday hours exist).

### 18.7 BusinessClosedDate — Security / tenant consistency

| Check | Result |
| --- | --- |
| Tenant-scoped? | **Yes** |
| Mutate | **Owner only** |

### 18.8 BusinessClosedDate — Review summary

**Final fields:** `id`, `businessId`, `localDate`, `createdAt`

**Uniques:** PK(`id`); UNIQUE(`businessId`, `localDate`)  

**Entity-level OPENs:** none  

### 18.9 BusinessClosedDate — Consistency check

| Check | Result |
| --- | --- |
| vs sources? | **None.** Full-day local date; unique pair; no invented reason field. |

### 18.10 BusinessClosedDate — Schema Notes

- Does **not** close architecture OPENs #1–#3.  
- UTC bounds computed at engine time from `localDate` + IANA TZ — not stored columns.

---

## 19. Entity: StaffTimeOff — field-level design

**Kind:** Staff **full-day** availability exclusion on a business-local calendar date (concrete).  
**Sources:** PA §11.2; Data Model §3.5; API Contract §9; Staff §13.  
**Product model:** full-day local date — **not** arbitrary partial-day ranges (DM §3.5).

### 19.1 StaffTimeOff — Field Specification

| Field | PostgreSQL type | Nullable | Default | Constraints | Notes |
| --- | --- | --- | --- | --- | --- |
| `id` | `uuid` | no | app-generated UUID v7 | PK | |
| `businessId` | `uuid` | no | — | FK → `Business.id`; composite with `staffId` | Tenant key |
| `staffId` | `uuid` | no | — | Composite FK `(businessId, staffId)` → `Staff(businessId, id)` | |
| `localDate` | `date` | no | — | **UNIQUE (`businessId`, `staffId`, `localDate`)** | Business-TZ `YYYY-MM-DD`. Canonical persistence (DM allowed local-date-only **or** materialized UTC bounds — **local date chosen** here) |
| `createdAt` | `timestamptz(3)` | no | set on create (UTC) | — | |
| `updatedAt` | `timestamptz(3)` | no | set on create/update (UTC) | — | API supports PATCH |

**Explicit non-fields:** reason/note; partial-day start/end minutes; recurrence; stored UTC bound columns (derived at engine); overnight multi-day spans as a single product feature.

### 19.2 StaffTimeOff — Identity / invariants

1. All-day semantics: `[local 00:00 on localDate, local 00:00 on next calendar day)` in `Business.timezone` — **not** fixed 24 elapsed hours (PA / DM).  
2. Engine converts those local bounds to a UTC half-open interval for availability checks.  
3. Closes that staff’s **entire** availability for that local day (even if weekly hours exist).  
4. Unique one full-day row per staff per local date (duplicate prevention for the same exclusion day; aligns with closed-date uniqueness pattern).  
5. Adding/changing/deleting TimeOff **never** auto-cancels or mutates existing appointments (PA §11.2).  
6. Does not rewrite appointment snapshots.  
7. API create may accept all-day `YYYY-MM-DD` **or** equivalent UTC instant bounds representing that full local day (API §9 wording); schema persists **`localDate`**. **Partial-day** free-form ranges are **out of product model** (DM §3.5) — not invented here.

### 19.3 StaffTimeOff — Relations

| Relation | Cardinality | Notes |
| --- | --- | --- |
| `Staff` → StaffTimeOff | **1 : N** | |
| `Business` → StaffTimeOff | **1 : N** | |
| vs BusinessClosedDate | both subtract | Closed date closes **all** staff; TimeOff closes **one** staff |
| vs StaffWorkingHour | subtract after ∩ | Hours ∩ then − time off (PA formula) |

FK **`ON DELETE`**: architecture OPEN #3.

### 19.4 StaffTimeOff — Uniqueness / indexes

| Rule | Scope | Kind |
| --- | --- | --- |
| Unique `id` (PK) | Global | FINAL |
| Unique `(businessId, staffId, localDate)` | Tenant+staff | FINAL field-level (full-day duplicate prevention) |
| Extras | — | OPEN #2 |

### 19.5 StaffTimeOff — Lifecycle

```text
POST   → create all-day exclusion for localDate; Owner
PATCH  → update (e.g. localDate); Owner
DELETE → remove; Owner
GET    → list; Owner
Past dates may remain; no source-mandated purge
```

### 19.6 StaffTimeOff — Scheduling engine implications

- Subtracted in `… − staff time off` after hours/closed-date resolution for that local day.  
- Staff path must honor TimeOff; Owner may `overrideHours` (PA §11.7).  
- Buffer may spill into time off and still block other appointments (PA §11.4) — about appointment buffer, not TimeOff storage.

### 19.7 StaffTimeOff — Security / tenant consistency

| Check | Result |
| --- | --- |
| Tenant-scoped? | **Yes** + composite FK |
| Mutate | **Owner only** (Staff cannot manage own TimeOff) |

### 19.8 StaffTimeOff — Review summary

**Final fields:** `id`, `businessId`, `staffId`, `localDate`, `createdAt`, `updatedAt`

**Uniques:** PK(`id`); UNIQUE(`businessId`, `staffId`, `localDate`)  

**Entity-level OPENs:** none  

### 19.9 StaffTimeOff — Consistency check

| Check | Result |
| --- | --- |
| vs PA / DM / API? | **None.** Full-day local date; DST-safe midnight→next midnight; no reason field. |
| Partial-day invented? | **No.** |
| Representation choice | `localDate` only (DM deferred options resolved at field-level; not architecture OPEN). |

### 19.10 StaffTimeOff — Schema Notes

- Does **not** close architecture OPENs #1–#3.  
- Materialized UTC bounds are engine-computed, not columns.

---

## 20. Group B — Cross-entity scheduling invariants

Answers locked to PA §11 / DM §3 / API §9 (no new architecture decisions).

| # | Question | Answer |
| --- | --- | --- |
| 1 | Effective staff working hours? | For a local weekday: **intersect** that day’s `BusinessWorkingHour` intervals with that staff’s `StaffWorkingHour` intervals (touch-ok, no overlap within each set; merge in memory). **No** “fallback to business hours if staff hours empty.” |
| 2 | Business closed date vs staff availability? | Full local day closed for the **business**: no public slots; normal manual blocked; subtracts for all staff that day (Owner override only). |
| 3 | Staff time off vs business hours? | After hours ∩ (and closed-date handling), **subtract** that staff’s full-day TimeOff for the local date. |
| 4 | Inactive Staff? | `Staff.active = false` → excluded from new booking/assignment; hour/TimeOff rows retained; history kept. |
| 5 | Inactive Business? | `Business.status = INACTIVE` → new **public** booking closed; hour/closed/TimeOff rows retained; admin/manual per PA (Owner override rules unchanged). |
| 6 | Existing CONFIRMED vs schedule edits? | **Not** auto-cancelled or status-mutated (PA). |
| 7 | Snapshots vs schedule edits? | Schedule entities do **not** rewrite appointment snapshots / `endsAt` / `blockedUntil`. |
| 8 | Booking engine timezone? | **`Business.timezone` (IANA)** via shared Luxon module; never browser/server local. |
| 9 | Date boundaries? | Business-local midnights for the calendar date (DST ⇒ 23h/25h elapsed possible); closed dates & all-day TimeOff use `[local 00:00, next local 00:00)`. |

**Architecture OPEN count after Group B:** **3** (unchanged: AuthToken physical model; exact secondary indexes; FK ON DELETE).

---

## 21. Entity: Customer — field-level design

**Kind:** Business-scoped customer **record** (guest or account-backed) (concrete).  
**Sources:** PA §9 / §8.2; Data Model §2.6 / §10; API Contract §3.3 / §5 / §12; conventions §2.  
**Not** a login identity — that is `CustomerAccount`.  
**May exist without** any account or session.  
**MVP soft-delete:** only entity using `deletedAt` (global conventions).

### 21.1 Customer — Field Specification

| Field | PostgreSQL type | Nullable | Default | Constraints | Notes |
| --- | --- | --- | --- | --- | --- |
| `id` | `uuid` | no | app-generated UUID v7 | PK | |
| `businessId` | `uuid` | no | — | FK → `Business.id` | Tenant key |
| `name` | `text` | no | — | Trimmed non-empty | Required on Owner create; set on guest book / verify (PA). Max length **not** in sources → TBD |
| `email` | `text` | yes | `null` | — | Optional contact; empty → `null`. When set, paired with `normalizedEmail`. Max length TBD |
| `normalizedEmail` | `text` | yes | `null` | Partial **UNIQUE** with `businessId` among non-deleted, non-null | Trim + lowercase; **no** dot/`+tag` stripping (PA §9.1). Matching key; null iff `email` null |
| `phone` | `text` | yes | `null` | E.164 when set | Default parse country **`TR`** if no calling code (PA). **Not** unique; phone never auto-links |
| `deletedAt` | `timestamptz(3)` | yes | `null` | — | Soft-delete marker; irreversible; never revived (PA §9.1 / §9.4) |
| `createdAt` | `timestamptz(3)` | no | set on create (UTC) | — | |
| `updatedAt` | `timestamptz(3)` | no | set on create/update (UTC) | — | |

**Explicit non-fields:** `password` / `passwordHash`, session/verification tokens, role/permissions, login status enum, global unique email/phone, `userId`.

### 21.2 Customer — Identity / invariants

1. Tenant-scoped; same person may be **independent** Customers (and accounts) on different businesses (API §5).  
2. Login identity for accounts is `(businessId, Customer.normalizedEmail)` among **non-deleted** (PA §8.2) — email uniqueness is **business-scoped**, not global.  
3. Partial unique `(businessId, normalizedEmail)` excluding soft-deleted and null emails (PA / DM FINAL).  
4. Guest/unauthenticated bookings never overwrite existing customer fields (PA §9.1).  
5. Soft-deleted never revived; same email later → **new** Customer row.  
6. Appointment ownership: `Appointment.customerId` → Customer (not Account). Chain: Appointment → Customer → optional CustomerAccount (PA §9.3).  
7. Profile/email edits and soft-delete **do not** rewrite appointment contact snapshots (PA §9.2 / §9.4).  
8. Soft-delete blocked if future `CONFIRMED` exist; same TX disables account, deletes sessions/tokens; history via snapshots (PA §9.4). **No hard delete** in MVP. **No restore.**

### 21.3 Customer — Relations

| Relation | Cardinality | Notes |
| --- | --- | --- |
| `Business` → Customer | **1 : N** | |
| Customer → `CustomerAccount` | **0..1 : 1** | Optional; 1:1 when account exists |
| Customer → `Appointment` | **1 : N** | History identity |
| `CustomerVerificationIntent` | **none** | Intent has **no** Customer FK; resolves into Customer on verify |

FK **`ON DELETE`**: architecture OPEN #3 (must preserve appointment history).

### 21.4 Customer — Uniqueness / indexes

| Rule | Scope | Kind |
| --- | --- | --- |
| Unique `id` (PK) | Global | FINAL |
| Partial unique `(businessId, normalizedEmail)` where `deletedAt IS NULL` and `normalizedEmail IS NOT NULL` | Tenant | **FINAL correctness constraint** (PA/DM) — prevents duplicate **active** customers under concurrency. Soft-deleted rows excluded so the same email may create a **new** Customer later. DDL is migration SQL (Prisma cannot express); **not** OPEN #2 |
| Phone unique? | — | **No** |
| Tenant list / search secondaries | Tenant | Performance only → **OPEN #2** |

### 21.5 Customer — Lifecycle

```text
Guest book / admin create     → Customer row (often no account)
Register verify success       → find-or-create non-deleted by normalizedEmail (existing fields win) + CustomerAccount
Profile PATCH                 → name/phone; email only via email-change flow (customer) or Owner set-once if null
Soft-delete (Owner)           → deletedAt=now; account DISABLED; sessions/tokens deleted; appointments retained
```

### 21.6 Customer — Security / tenant consistency

| Check | Result |
| --- | --- |
| Tenant-scoped? | **Yes** |
| Cross-tenant | **404** |
| Auth realm | Customer is **not** the session subject; Account/Session are |

### 21.7 Customer — Review summary

**Final fields:** `id`, `businessId`, `name`, `email`, `normalizedEmail`, `phone`, `deletedAt`, `createdAt`, `updatedAt`

**Uniques / correctness:** PK; **FINAL** partial unique `(businessId, normalizedEmail)` among non-deleted, non-null emails  

**Performance indexes:** tenant list/search → OPEN #2 only  

**Entity-level OPENs:** none  

### 21.8 Customer — Consistency check

| Check | Result |
| --- | --- |
| vs PA/DM/API? | **None.** Soft-delete-only; business-scoped email; no password on Customer. |
| Missing/extra? | **No** invented status/role/global unique. |

### 21.9 Customer — Schema Notes

- Does **not** close architecture OPENs #1–#3.  
- Active-email partial unique is a **correctness** constraint, not OPEN #2.  
- Manage tokens revoked on customer delete per PA — see §27.

---

## 22. Entity: CustomerVerificationIntent — field-level design

**Kind:** Pending **customer registration** before any Customer/CustomerAccount exists (concrete).  
**Sources:** PA §8.4 / §9.3; Data Model §2.9 / §7; API Contract §3.3.  
**Independent** of AuthToken physical OPEN (#1): intent row **must** exist either way (DM).  
**Not** used as the generic password-reset / email-change table (those remain AuthToken logical subjects).

### 22.1 CustomerVerificationIntent — Field Specification

| Field | PostgreSQL type | Nullable | Default | Constraints | Notes |
| --- | --- | --- | --- | --- | --- |
| `id` | `uuid` | no | app-generated UUID v7 | PK | |
| `businessId` | `uuid` | no | — | FK → `Business.id` | Tenant |
| `normalizedEmail` | `text` | no | — | Trim + lowercase | Pending login identity within business (DM §2.9) |
| `pendingName` | `text` | no | — | Trimmed non-empty | From register form; applied on verify if creating Customer |
| `pendingPhone` | `text` | yes | `null` | E.164 when set | Pending phone; TR default parse when applicable |
| `passwordHash` | `text` | no | — | Argon2id only | Chosen password hashed at register; **never** raw (PA §9.3 / DM) |
| `verificationTokenHash` | `text` | no | — | Hash at rest; lookup by hash of presented raw token | **FINAL MVP/default:** registration email-verification secret persisted **on this row**. Raw secret only in email/worker memory (PA §8.4 / §17.6). **Not** AuthToken OPEN #1 |
| `status` | enum `CustomerVerificationIntentStatus` | no | **`PENDING`** | ∈ {`PENDING`,`CONSUMED`,`REVOKED`,`SUPERSEDED`} | DM/PA lifecycle |
| `expiresAt` | `timestamptz(3)` | no | — | — | Exact TTL deferred to technical docs (DM); store deadline |
| `createdAt` | `timestamptz(3)` | no | set on create (UTC) | — | |
| `updatedAt` | `timestamptz(3)` | no | set on create/update (UTC) | — | |

**Explicit non-fields:** `customerId` / `customerAccountId` FKs; raw verification secret; password-reset / email-change token tables (those remain **OPEN #1**); session creation columns.

**Uniqueness of `verificationTokenHash`:** global **UNIQUE** (**FINAL correctness**) so presented tokens resolve to at most one intent (same posture as invitation/manage `tokenHash`).

### 22.2 CustomerVerificationIntent — Identity / invariants

1. Exists **before** Customer/Account; expiry/failure leaves **no** Customer/Account (PA §9.3).  
2. Re-registration / resend with same email **invalidates** previous pending → prior intent `SUPERSEDED` (or equivalent invalidate); new `PENDING` (PA / API resend rotates).  
3. At most one **active `PENDING`** intent per `(businessId, normalizedEmail)` (PA one active intent per purpose/subject).  
4. On successful verify: mark `CONSUMED`; one TX find-or-create Customer by normalized email (**existing Customer field values win**) then create `CustomerAccount`; API then creates customer session (API §3.3) — session is **not** a column on intent.  
5. `REVOKED` for explicit invalidation paths as needed (e.g. policy); not inventing extra revoke products.  
6. No FK to Customer/Account (DM §2.9).

### 22.3 CustomerVerificationIntent — Relations

| Relation | Cardinality | Notes |
| --- | --- | --- |
| `Business` → Intent | **1 : N** | |
| → Customer / Account | **resolve on consume** | No persistent FK |
| AuthToken (logical) | **none for registration verify** | Password-reset / email-change remain OPEN #1 — not this entity |

### 22.4 CustomerVerificationIntent — Uniqueness / indexes

| Rule | Scope | Kind |
| --- | --- | --- |
| Unique `id` (PK) | Global | FINAL |
| Unique `verificationTokenHash` | Global | **FINAL correctness** (token lookup) |
| One `PENDING` per `(businessId, normalizedEmail)` | Tenant | **FINAL correctness** partial unique — prevents concurrent duplicate pending registrations; migration SQL (not OPEN #2) |
| Lookup by email for register/resend | Tenant | Performance only → **OPEN #2** |

### 22.5 CustomerVerificationIntent — Lifecycle

```text
POST register     → create PENDING + `verificationTokenHash` (hash written when secret minted per PA token-email lifecycle); enqueue ACCOUNT_VERIFICATION; no Customer/Account
POST resend       → supersede prior PENDING; new PENDING; rotate secret → new `verificationTokenHash`
POST verify-email → validate hash+expiry; CONSUMED; create/link Customer + CustomerAccount; create CustomerSession
Expiry            → unusable; no orphan Customer/Account
Already registered email → uniform API response + ACCOUNT_ALREADY_EXISTS email (no enumeration)
```

### 22.6 CustomerVerificationIntent — Security / tenant consistency

| Check | Result |
| --- | --- |
| Tenant-scoped? | **Yes** |
| Raw secret at rest? | **No** — only `verificationTokenHash` |
| Closes AuthToken OPEN #1? | **No** — OPEN #1 remains for password-reset / email-change / business-reset |

### 22.7 CustomerVerificationIntent — Review summary

**Final fields:** `id`, `businessId`, `normalizedEmail`, `pendingName`, `pendingPhone`, `passwordHash`, `verificationTokenHash`, `status`, `expiresAt`, `createdAt`, `updatedAt`

**Correctness constraints:** PK; UNIQUE(`verificationTokenHash`); partial unique one `PENDING` per `(businessId, normalizedEmail)`

**Entity-level OPENs:** none  

### 22.8 CustomerVerificationIntent — Consistency check

| Check | Result |
| --- | --- |
| vs sources? | **None.** Registration-only pending; verify secret on-row; reset/email-change stay OPEN #1. |

### 22.9 CustomerVerificationIntent — Schema Notes

- Registration verify secret = **`verificationTokenHash` on this entity** (FINAL for MVP).  
- Architecture OPEN #1 still covers password-reset / email-change (/ pending new email) physical tables — **not** registration verify.  
- Does **not** close OPENs #2–#3.

---

## 23. Entity: CustomerAccount — field-level design

**Kind:** Tenant-scoped customer **login** identity, 1:1 with Customer (concrete).  
**Sources:** PA §8.2 / §9.3–§9.4; Data Model §2.7; API Contract §3.3 / §5.  
**Not** global `User`. **Not** the appointment party (Customer is).

### 23.1 CustomerAccount — Field Specification

| Field | PostgreSQL type | Nullable | Default | Constraints | Notes |
| --- | --- | --- | --- | --- | --- |
| `id` | `uuid` | no | app-generated UUID v7 | PK | |
| `businessId` | `uuid` | no | — | FK → `Business.id`; composite with `customerId` | **Stored FINAL** (DM §2.7) |
| `customerId` | `uuid` | no | — | **UNIQUE**; composite FK `(businessId, customerId)` → `Customer(businessId, id)` | 1:1 with Customer |
| `passwordHash` | `text` | no | — | Argon2id only | Never plaintext |
| `verifiedAt` | `timestamptz(3)` | no | set on account create (UTC) | — | Set when verification completes / account created (PA §8.2) |
| `status` | enum `CustomerAccountStatus` | no | **`ACTIVE`** | ∈ {`ACTIVE`,`DISABLED`} | Disabled on customer soft-delete / policy (DM §10) |
| `createdAt` | `timestamptz(3)` | no | set on create (UTC) | — | |
| `updatedAt` | `timestamptz(3)` | no | set on create/update (UTC) | — | |

**Explicit non-fields:** email (lives on Customer); soft-delete column (Customer.`deletedAt`); raw reset tokens (AuthToken logical / OPEN #1); roles/permissions.

### 23.2 CustomerAccount — Identity / invariants

1. Exactly one account per Customer when registered; Customer may exist with **zero** accounts.  
2. Login: `(businessId, Customer.normalizedEmail)` among non-deleted + account usable (`ACTIVE`, customer not deleted) (PA §8.2 / API §3.3).  
3. Same email on different businesses → **independent** accounts (API §5).  
4. Password reset: purpose-scoped AuthToken (OPEN #1 physical); on confirm update `passwordHash` and **revoke all CustomerSessions** for the account (PA §9.4 / API).  
5. Soft-delete Customer → `status = DISABLED` in same TX; sessions deleted (PA §9.4).  
6. Appointment history remains on Customer; account is not history owner.

### 23.3 CustomerAccount — Relations

| Relation | Cardinality | Notes |
| --- | --- | --- |
| `Customer` → CustomerAccount | **0..1 : 1** | |
| `Business` → CustomerAccount | **1 : N** | via stored `businessId` |
| CustomerAccount → `CustomerSession` | **1 : N** | |

### 23.4 CustomerAccount — Uniqueness / indexes

| Rule | Scope | Kind |
| --- | --- | --- |
| Unique `id` (PK) | Global | FINAL |
| Unique `customerId` | Global | FINAL 1:1 |
| Unique `(businessId, customerId)` | Tenant pair | FINAL for composite FK / tenant safety |
| Login lookup | via Customer email + business | Not a column here |

### 23.5 CustomerAccount — Lifecycle

```text
Verify registration success → create ACTIVE account + verifiedAt + passwordHash from intent
Login                        → rotate CustomerSession (not stored on account)
Password-reset confirm       → new passwordHash; revoke all sessions
Customer soft-delete         → DISABLED; sessions/tokens deleted
No MVP self-service account deletion
```

### 23.6 CustomerAccount — Security / tenant consistency

| Check | Result |
| --- | --- |
| Tenant-scoped? | **Yes** (`businessId` stored) |
| Global User conflation? | **No** |
| AuthToken OPEN #1 closed? | **No** |

### 23.7 CustomerAccount — Review summary

**Final fields:** `id`, `businessId`, `customerId`, `passwordHash`, `verifiedAt`, `status`, `createdAt`, `updatedAt`

**Entity-level OPENs:** none  

### 23.8 CustomerAccount — Consistency check

| Check | Result |
| --- | --- |
| vs sources? | **None.** 1:1; businessId stored; Argon2id; status ACTIVE/DISABLED. |

### 23.9 CustomerAccount — Schema Notes

- Does **not** close architecture OPENs #1–#3.

---

## 24. Entity: CustomerSession — field-level design

**Kind:** Server-side **customer-realm** session (concrete).  
**Sources:** PA §8.2; Data Model §2.8; API Contract §2 / §3.3; mirrors BusinessSession security principles (§12) without merging realms.  
**Separate cookie/table from `BusinessSession`.**

### 24.1 CustomerSession — Field Specification

| Field | PostgreSQL type | Nullable | Default | Constraints | Notes |
| --- | --- | --- | --- | --- | --- |
| `id` | `uuid` | no | app-generated UUID v7 | PK | Rotated on login (new row/id) |
| `customerAccountId` | `uuid` | no | — | FK → `CustomerAccount.id` | Session subject |
| `businessId` | `uuid` | no | — | FK → `Business.id` | **Required** — must match slug-resolved business (PA §8.2 / DM §2.8); not optional denormalization |
| `tokenHash` | `text` | no | — | **UNIQUE**; hash at rest | Raw token only in HTTP-only cookie / memory |
| `absoluteExpiresAt` | `timestamptz(3)` | no | — | — | Absolute deadline; exact duration deferred (PA) |
| `lastUsedAt` | `timestamptz(3)` | no | set on create (UTC) | — | Idle watermark; idle duration deferred |
| `createdAt` | `timestamptz(3)` | no | set on create (UTC) | — | |

**Explicit non-fields:** `updatedAt`; raw token; role/authz cache; merge with BusinessSession; soft `revokedAt` (destroy row on revoke — same posture as BusinessSession §12).

### 24.2 CustomerSession — Identity / invariants

1. Holds `customerAccountId` + `businessId` — **not** full authorization payload.  
2. Every Customer API request: cookie → hash/expiry → load account + Customer; require `ACTIVE` account, non-deleted Customer; **session `businessId` must equal slug-resolved business** else **404** (API §2 / §3.3).  
3. Single customer cookie; logging into another business **replaces** it (PA §8.2).  
4. Logout / password-reset / account disable / soft-delete → destroy sessions.  
5. Idle + absolute expiry (durations technical-deferred).  
6. Verify-email success **does** create a session (API §3.3) — after Account exists; not automatic from Intent alone without verify handler.

### 24.3 CustomerSession — Relations

| Relation | Cardinality | Notes |
| --- | --- | --- |
| `CustomerAccount` → CustomerSession | **1 : N** | |
| `Business` ← `businessId` | **N : 1** | Tenant context check vs slug |
| `BusinessSession` | **none** | Separate realm |

### 24.4 CustomerSession — Uniqueness / indexes

| Rule | Scope | Kind |
| --- | --- | --- |
| Unique `id` (PK) | Global | FINAL |
| Unique `tokenHash` | Global | FINAL lookup |
| Bulk revoke by account / business | — | Behavior FINAL; secondary DDL OPEN #2 |

### 24.5 CustomerSession — Lifecycle

```text
Login / verify-email success → create authenticated CustomerSession (rotate id/token)
Logout                       → destroy this session
Password-reset confirm       → destroy all sessions for account
Account DISABLED / Customer soft-delete → destroy sessions
Idle / absolute expiry       → reject (+ destroy)
Cross-business login         → replace single customer cookie/session
```

### 24.6 CustomerSession — Security / tenant consistency

| Check | Result |
| --- | --- |
| Realm | **Customer only** — never BusinessSession |
| Per-request checks | token hash, expiry, account status, customer not deleted, businessId↔slug |
| Raw secret at rest? | **No** |

### 24.7 CustomerSession — Review summary

**Final fields:** `id`, `customerAccountId`, `businessId`, `tokenHash`, `absoluteExpiresAt`, `lastUsedAt`, `createdAt`

**Entity-level OPENs:** none  

### 24.8 CustomerSession — Consistency check

| Check | Result |
| --- | --- |
| vs sources? | **None.** businessId on session required; realm split preserved. |

### 24.9 CustomerSession — Schema Notes

- Does **not** close architecture OPENs #1–#3.  
- Guest manage-token flow is `AppointmentManageToken` (later) — not this entity.

---

## 25. Group C — Cross-entity auth lifecycle

| # | Question | Answer |
| --- | --- | --- |
| 1 | How is Customer created? | Guest/admin booking create/link; Owner CM create; or registration **verify** find-or-create by normalized email. |
| 2 | Appointment without login? | **Yes** — Appointment → Customer; account optional. |
| 3 | When is VerificationIntent created? | `POST .../customer/register` (pending; no Customer/Account yet). |
| 4 | Consume / revoke / supersede? | Verify → `CONSUMED` + create/link Customer+Account; resend/re-register → prior `SUPERSEDED`; revoke/expiry → no Account; raw secret never stored. |
| 5 | When is CustomerAccount created? | Only on successful email verification (same TX as Customer link/create). |
| 6 | Account ↔ Customer? | **1:1** when account exists; Customer **0..1** account. |
| 7 | Session authenticates via? | `CustomerSession` → `customerAccountId` (+ `businessId` vs slug); then Customer for profile/email. |
| 8 | Account disable → sessions? | **Destroyed/revoked**. |
| 9 | Password reset → sessions? | **All revoked** for that account. |
| 10 | Customer soft-delete → account/session? | Account `DISABLED`; sessions/tokens deleted; Customer `deletedAt` set. |
| 11 | Appointment history? | **Retained** (snapshots + customerId); admin uses snapshot fallback if deleted. |
| 12 | Email/phone change vs snapshots? | **Snapshots unchanged** (PA §9.4). |
| 13 | Tenant isolation? | All four entities carry `businessId`; session businessId must match slug; accounts not global; cross-tenant → 404. |
| 14 | Customer vs Business auth? | Separate tables/cookies/realms (`CustomerSession` ≠ `BusinessSession`; `CustomerAccount` ≠ `User`). |

**Distinctions (FINAL):**

| Entity | Role |
| --- | --- |
| Customer | Contact/history record; soft-deletable; no password |
| CustomerVerificationIntent | Pre-account registration pending only |
| CustomerAccount | Login credentials + status; 1:1 Customer |
| CustomerSession | Customer-realm server session |
| AuthToken (logical) | Password-reset / email-change / business-reset physical model — OPEN #1 only; registration verify is on `CustomerVerificationIntent` |
| AppointmentManageToken | Guest manage — **not** Group C |

**Architecture OPEN count after Group C:** **3** (unchanged).

---

## 26. Entity: Appointment — field-level design

**Kind:** Business-scoped booking aggregate / historical booking record (concrete).  
**Sources:** PA §11.4 / §11.7 / §12–§13 / §21; Data Model §4 / §5.1; API Contract §4 / §10–§11; Groups A–C.  
**Not** soft-deleted (status machine). **Not** dependent on `CustomerAccount` / `CustomerSession`.  
**No** direct FK to `StaffService` (eligibility checked at booking time).  
**No** business branding snapshot (current Business at send/view).

### 26.1 Appointment — Field Specification

| Field | PostgreSQL type | Nullable | Default | Constraints | Notes |
| --- | --- | --- | --- | --- | --- |
| `id` | `uuid` | no | app-generated UUID v7 | PK | |
| `businessId` | `uuid` | no | — | FK → `Business.id` | Tenant key |
| `customerId` | `uuid` | no | — | Composite FK `(businessId, customerId)` → `Customer(businessId, id)` | Party identity — **Customer**, not Account (PA §9.3) |
| `staffId` | `uuid` | no | — | Composite FK `(businessId, staffId)` → `Staff(businessId, id)` | Booked staff |
| `serviceId` | `uuid` | no | — | Composite FK `(businessId, serviceId)` → `Service(businessId, id)` | Booked service |
| `startsAt` | `timestamptz(3)` | no | — | — | UTC instant; business logic via `Business.timezone` + Luxon |
| `endsAt` | `timestamptz(3)` | no | — | `endsAt > startsAt` | From booking-time **snapshot duration**; not live Service |
| `blockedUntil` | `timestamptz(3)` | no | — | `blockedUntil >= endsAt` | After-buffer boundary from booking-time **snapshot buffer**; protected half-open `[startsAt, blockedUntil)` |
| `status` | enum `AppointmentStatus` | no | **`CONFIRMED`** | ∈ {`CONFIRMED`,`COMPLETED`,`CANCELLED`,`NO_SHOW`} | No `PENDING` (PA §13.3) |
| `source` | enum `AppointmentSource` | no | — | ∈ {`PUBLIC`,`OWNER`,`STAFF`} | Public book → `PUBLIC`; manual → `OWNER`/`STAFF` (PA §12) |
| `overrideHours` | `boolean` | no | **`false`** | — | Owner-only true on create/admin reschedule (API §10 / PA §11.7) |
| `bookingReference` | `text` | no | app-generated | **UNIQUE (`businessId`, `bookingReference`)** | `BK-` + 8 uppercase Crockford Base32; **business-scoped**; **not** PK (PA §21.1) |
| `version` | `int` | no | **1** (on create) | Monotonic; increment every mutation | Optimistic/conditional concurrency (PA §13.3) |
| `customerNote` | `text` | yes | `null` | Immutable after create | Optional booking note; Customer cannot edit later (PA §9.5 / DM §4.1) |
| `snapshotCustomerName` | `text` | no | — | — | Booking-time customer name |
| `snapshotCustomerEmail` | `text` | yes | `null` | — | Booking-time email (may be null) |
| `snapshotCustomerPhone` | `text` | yes | `null` | — | Booking-time phone (may be null) |
| `snapshotStaffDisplayName` | `text` | no | — | — | From `Staff.displayName` at book; **updated only if staff changes on reschedule** (PA §13.4) |
| `snapshotServiceName` | `text` | no | — | — | Immutable after create (including reschedule) |
| `snapshotDurationMinutes` | `int` | no | — | `> 0` | Booking-time service duration; drives `endsAt` |
| `snapshotBufferMinutes` | `int` | no | — | `≥ 0` | Booking-time **after-only** buffer; drives `blockedUntil` |
| `snapshotPriceMinor` | `int` | no | — | Integer minor units | Booking-time price |
| `snapshotCurrency` | `text` | no | — | ISO 4217 | Booking-time currency |
| `cancelledAt` | `timestamptz(3)` | yes | `null` | Set with `cancelledBy` when `CANCELLED` | DB `now()` at cancel (PA §13.1); null otherwise |
| `cancelledBy` | enum `CancelledBy` | yes | `null` | ∈ {`CUSTOMER`,`STAFF`,`OWNER`} when cancelled | **No** acting user id; **no** cancellation reason field (PA §13.3) |
| `createdAt` | `timestamptz(3)` | no | set on create (UTC) | — | |
| `updatedAt` | `timestamptz(3)` | no | set on create/update (UTC) | — | |

**Explicit non-fields:** `userId`; `customerAccountId` / session FKs; `staffServiceId`; business logo/color snapshot; cancellation reason; soft-delete `deletedAt`; payments/deposit.

### 26.2 Appointment — Identity / invariants

1. Tenant-scoped; all party FKs composite with `businessId`.  
2. New bookings start **`CONFIRMED`** (no approval / no `PENDING`) (PA §12.1).  
3. Timing is historical: `endsAt` / `blockedUntil` from **snapshots**, never re-derived from live Service after book (PA §10.1 / §13.2).  
4. Buffer is **after-only**; working hours constrain `[startsAt, endsAt)` only; buffer may spill past close/time-off and still block (PA §11.4).  
5. Double-booking: PostgreSQL **exclusion** per staff on `tstzrange(startsAt, blockedUntil, '[)')` where `status <> 'CANCELLED'` (`btree_gist`) — DB-enforced, not app-only (PA §21.2 / DM §4.3). Half-open ⇒ **touching** intervals allowed.  
6. Blocking statuses: `CONFIRMED`, `COMPLETED`, `NO_SHOW` (everything except `CANCELLED`) (DM §4.3).  
7. `overrideHours` may relax hours / closed dates / time off for **OWNER** only; **never** bypasses inactive Staff/Service, missing StaffService, past time, overlap/exclusion, or tenant isolation (PA §11.7 / API §10).  
8. `bookingReference` unique per business; not a global namespace.  
9. Customer soft-delete / profile edits do **not** rewrite snapshots; admin UI falls back to snapshots when deleted (PA §9.2).  
10. No `userId` on Appointment.

### 26.3 Appointment — Relations

| Relation | Cardinality | Notes |
| --- | --- | --- |
| `Business` → Appointment | **1 : N** | |
| `Customer` → Appointment | **1 : N** | History owner |
| `Staff` → Appointment | **1 : N** | Exclusion key |
| `Service` → Appointment | **1 : N** | Snapshot source at book |
| Appointment → `AppointmentManageToken` | **1 : N** | Guest manage (entity later) |
| Appointment → `NotificationDelivery` | **0..N** | Optional appointment-related deliveries (entity later) |
| `StaffService` | **no FK** | Validated at book/reschedule time |
| `CustomerAccount` / `CustomerSession` | **none** | Auth realms only |

FK **`ON DELETE`**: architecture OPEN #3 (history retention must be preserved).

### 26.4 Appointment — Uniqueness / indexes

| Rule | Scope | Kind |
| --- | --- | --- |
| Unique `id` (PK) | Global | FINAL |
| Unique `(businessId, bookingReference)` | Tenant | FINAL product invariant |
| Exclusion `(staffId, tstzrange(startsAt, blockedUntil, '[)'))` where `status <> 'CANCELLED'` | Per staff | **FINAL** integrity mechanism (PA/DM) — not OPEN #2 |
| Secondary indexes e.g. `(businessId, startsAt)`, `(businessId, staffId, startsAt)` | Calendar queries | Mentioned in PA calendar notes as complements; **exact secondary inventory** remains architecture OPEN #2 |

### 26.5 Appointment — Lifecycle

```text
Create (public/manual)     → status=CONFIRMED; snapshots + endsAt/blockedUntil from booking-time values;
                             source PUBLIC|OWNER|STAFF; version=1; bookingReference minted
Reschedule                 → in-place UPDATE same row; status stays CONFIRMED; version++;
                             recompute startsAt/endsAt/blockedUntil from NEW start + snapshot duration/buffer;
                             service snapshots unchanged; staff name snapshot updates iff staffId changes;
                             manage tokens revoked+reissued per PA; reminder job renewed
Cancel (before startsAt)   → CANCELLED; cancelledAt/cancelledBy set; frees exclusion slot; terminal
Complete / No-show         → only if now ≥ startsAt (from CONFIRMED); COMPLETED ↔ NO_SHOW allowed; Customer never these
Past CONFIRMED             → no auto-transition (UI “İşaretlenmedi”)
```

**External mutations do NOT auto-cancel/rewrite appointments** (unless sources say otherwise — they do not):

| Change | Effect on existing appointments |
| --- | --- |
| Staff deactivate | Blocked if future CONFIRMED for that staff; else staff inactive for **new** books; history retained |
| Service deactivate | Existing CONFIRMED continue; no new books with that service |
| StaffService unlink | Appointments preserved; pair unavailable for **new** books |
| Customer soft-delete | Blocked if future CONFIRMED; else account/sessions/tokens cleared; appointments + snapshots retained |
| Business INACTIVE | New **public** booking closed; existing appointments/reminders/manage continue per PA |
| Hours / TimeOff / ClosedDate change | Future availability only; no auto-cancel |

### 26.6 Appointment — Scheduling / concurrency implications

- Engine, exclusion constraint, and tests share the same range + `status <> CANCELLED` predicate (PA §21.2).  
- Reschedule: exclude the row being moved from conflict checks; success before old slot freed (same-row move).  
- Public/customer paths: slot grid + notice + max window; admin manual: any minute (seconds = 0) (PA §11.3 / §13.4).  
- Browser TZ never used.

### 26.7 Appointment — Security / tenant consistency

| Check | Result |
| --- | --- |
| Tenant-scoped? | **Yes** (`businessId` + composite FKs) |
| Public book | Resolve Business by slug first |
| Business API | Membership/`activeBusinessId`; Staff scoped to own staff |
| Customer API | Own Customer via session; not Account as party FK |
| Guest manage | `AppointmentManageToken` (separate entity) — not CustomerSession |
| Cross-tenant id | **404** |

### 26.8 Appointment — Review summary

**Final fields:**  
`id`, `businessId`, `customerId`, `staffId`, `serviceId`, `startsAt`, `endsAt`, `blockedUntil`, `status`, `source`, `overrideHours`, `bookingReference`, `version`, `customerNote`, `snapshotCustomerName`, `snapshotCustomerEmail`, `snapshotCustomerPhone`, `snapshotStaffDisplayName`, `snapshotServiceName`, `snapshotDurationMinutes`, `snapshotBufferMinutes`, `snapshotPriceMinor`, `snapshotCurrency`, `cancelledAt`, `cancelledBy`, `createdAt`, `updatedAt`

**Uniques:** PK(`id`); UNIQUE(`businessId`, `bookingReference`)

**Integrity:** per-staff exclusion on `[startsAt, blockedUntil)` excluding `CANCELLED`

**Relations:** Business, Customer, Staff, Service; → ManageToken / NotificationDelivery (later); no StaffService/Account/Session FKs

**Entity-level OPENs:** none  

### 26.9 Appointment — Consistency check

| Check | Result |
| --- | --- |
| vs Product Architecture? | **None.** Fields/snapshots/status machine/exclusion/override bounds match §13 / §11 / §21. |
| vs Data Model? | **None.** §4.1–§4.3 covered; no invented reason field. |
| vs API Contract? | **None.** PUBLIC vs OWNER/STAFF sources; Owner-only overrideHours; lifecycle POSTs. |
| vs Groups A–C? | **None.** Customer≠Account; no StaffService FK; buffer after-only historical. |
| Missing/extra? | **No.** |

### 26.10 Appointment — Schema Notes

- Does **not** close architecture OPENs #1–#3.  
- #2: exclusion + bookingReference unique are FINAL; other secondary indexes remain OPEN.  
- #3: FK ON DELETE deferred; product requires retaining historical appointment rows.  
- Snapshot column names are schema-doc labels for the PA/DM snapshot group; Prisma spelling is implementation.  
- Prisma/`@db` mapping is implementation, not specified here.

**Architecture OPEN count after Appointment:** **3** (unchanged).

---

## 27. Entity: AppointmentManageToken — field-level design

**Kind:** Guest appointment manage authorization (concrete).  
**Sources:** PA §8.3 / §13.4 / §17.6; Data Model §5.2 / §7.1; API Contract manage-token transport; Appointment §26.  
**Not** a session. **Not** CustomerSession / BusinessSession. **Not** AuthToken (dedicated entity outside OPEN #1).

### 27.1 AppointmentManageToken — Field Specification

| Field | PostgreSQL type | Nullable | Default | Constraints | Notes |
| --- | --- | --- | --- | --- | --- |
| `id` | `uuid` | no | app-generated UUID v7 | PK | |
| `businessId` | `uuid` | no | — | FK → `Business.id`; composite with `appointmentId` | Tenant key |
| `appointmentId` | `uuid` | no | — | Composite FK `(businessId, appointmentId)` → `Appointment(businessId, id)` | Appointment-specific |
| `tokenHash` | `text` | no | — | **UNIQUE**; hash at rest | Raw token never stored (PA/API) |
| `status` | enum `ManageTokenStatus` | no | **`ACTIVE`** | ∈ {`ACTIVE`,`REVOKED`,`CONSUMED`} | All three values are **FINAL** and first-class — see §27.2 / §27.5 |
| `expiresAt` | `timestamptz(3)` | no | — | — | Authoritative expiration boundary; exact TTL deferred (PA) |
| `createdAt` | `timestamptz(3)` | no | set on create (UTC) | — | |
| `updatedAt` | `timestamptz(3)` | no | set on create/update (UTC) | — | |

**Explicit non-fields:** raw token; `customerId` / account / session FKs; multi-purpose flags; soft-delete.

### 27.2 AppointmentManageToken — Identity / invariants

1. High-entropy, single-purpose **manage**, hashed at rest, appointment-bound (PA §8.3).  
2. **Multiple `ACTIVE` tokens** may exist for one appointment (PA §17.6 / DM §5.2). Do **not** invent blanket single-use on every successful manage POST.  
3. Transport: email URL **fragment** → frontend → POST body `token`; never query string; **GET must not consume** (API FINAL / PA §8.3).  
4. Valid until expiry, cancel, reschedule (revoke + new token emailed), appointment no longer manageable, or customer deletion (PA §8.3).  
5. Reschedule / cancel / customer soft-delete → **`REVOKED`** for the appointment’s manage tokens; reschedule issues a new `ACTIVE` token (PA §13.4 / §17.6).  
6. **Usable token:** `status = ACTIVE` **and** `now < expiresAt` **and** appointment still manageable (CONFIRMED, before start, etc. per PA). An expired `ACTIVE` row must be **rejected** even if status has not been rewritten.  
7. Lookup by hash of presented raw token.

#### Status semantics (FINAL — non-ambiguous)

| Status | Meaning |
| --- | --- |
| **`ACTIVE`** | Token is currently usable **provided** `expiresAt` has not passed (and appointment rules still allow manage). |
| **`REVOKED`** | Token was **explicitly invalidated** before successful one-time consumption (or as part of invalidation flows). Established examples: appointment **reschedule**, appointment **cancellation**, **customer deletion**, and other explicit invalidation paths defined by PA/API. |
| **`CONSUMED`** | Token was **successfully used** in a flow that requires **one-time consumption**. This is a **required, first-class** terminal status in `ManageTokenStatus` (DM §5.2: active / revoked / **consumed-as-needed**) — **not** optional, **not** unused, and **not** a synonym for expiry. When a presentation path requires one-time consumption, the row transitions to `CONSUMED`. Multi-active guest manage cancel/reschedule **authorization** remains reusable across multiple `ACTIVE` tokens until revoke/expiry as established above — do not invent one-time consumption for those multi-active flows. |

#### Expiry (FINAL)

- `expiresAt` is the **authoritative** expiration boundary.  
- Reject when `now >= expiresAt`, regardless of whether `status` is still `ACTIVE`.  
- **No** background job is required merely to flip `ACTIVE` → “expired”; expiry is evaluated at use time. Do **not** overload `CONSUMED` or `REVOKED` to mean “expired.”

### 27.3 AppointmentManageToken — Relations

| Relation | Cardinality | Notes |
| --- | --- | --- |
| `Appointment` → ManageToken | **1 : N** | |
| `Business` → ManageToken | **1 : N** | |
| CustomerSession / BusinessSession | **none** | Separate auth mechanisms |

FK **`ON DELETE`**: architecture OPEN #3.

### 27.4 AppointmentManageToken — Uniqueness / indexes

| Rule | Scope | Kind |
| --- | --- | --- |
| Unique `id` (PK) | Global | FINAL correctness |
| Unique `tokenHash` | Global | FINAL correctness |
| List by appointment | — | Performance → OPEN #2 |

### 27.5 AppointmentManageToken — Lifecycle

```text
Booking confirmed (email path) → create ACTIVE + expiresAt; email fragment link
Manage POST (cancel/reschedule/etc.)
  → validate tokenHash + status=ACTIVE + now < expiresAt + appointment rules
  → GET never consumes; token arrives in POST body only
Reschedule success             → REVOKED all prior tokens for appointment; mint new ACTIVE; email
Cancel success                 → REVOKED relevant tokens for appointment
Customer soft-delete           → REVOKED relevant tokens
One-time consumption path      → CONSUMED (when a flow requires single use — DM “consumed-as-needed”)
Expiry at use time             → reject while status may still be ACTIVE; no job required to rewrite status
```

### 27.6 AppointmentManageToken — Security / tenant consistency

| Check | Result |
| --- | --- |
| Tenant-scoped? | **Yes** |
| Raw secret at rest? | **No** |
| New auth realm? | **No** — purpose-scoped manage only; **≠** CustomerSession |
| Multi-active preserved? | **Yes** (PA/DM) |
| Fragment → POST body? | **Yes** (API FINAL) |

### 27.7 AppointmentManageToken — Review summary

**Final fields:** `id`, `businessId`, `appointmentId`, `tokenHash`, `status`, `expiresAt`, `createdAt`, `updatedAt`  

**Status:** `ACTIVE` \| `REVOKED` \| `CONSUMED` — all first-class; expiry via `expiresAt` at use time  

**Entity-level OPENs:** none  

### 27.8–27.9 Consistency / Schema Notes

- Matches PA/DM/API; does **not** close OPENs #1–#3.  
- Outside AuthToken OPEN #1 by design (DM §7.2).  
- `CONSUMED` is part of the FINAL enum and means successful one-time consumption — never treat as optional/unused; never use it for expiry.

---

## 28. Entity: BusinessInvitation — field-level design

**Kind:** Owner/Staff invite with hashed single-use token (concrete).  
**Sources:** PA §5.1–§5.2 / §8.4; Data Model §6.1; API Contract §3.2; BusinessMember/Staff §11/§13.  
**Not** BusinessMember. **Not** AuthToken (dedicated; outside OPEN #1).

### 28.1 BusinessInvitation — Field Specification

| Field | PostgreSQL type | Nullable | Default | Constraints | Notes |
| --- | --- | --- | --- | --- | --- |
| `id` | `uuid` | no | app-generated UUID v7 | PK | |
| `businessId` | `uuid` | no | — | FK → `Business.id` | Tenant |
| `email` | `text` | no | — | Stored **normalized** (trim+lowercase) | Invitee; binding trust is invitation row (API) |
| `role` | enum `BusinessMemberRole` | no | — | ∈ {`OWNER`,`STAFF`} | |
| `staffId` | `uuid` | yes | `null` | Composite FK `(businessId, staffId)` → `Staff` when set | **Required for STAFF** invites; **null for OWNER** (DM §6.1) |
| `tokenHash` | `text` | no | — | **UNIQUE**; hash at rest | High-entropy; single-use |
| `status` | enum `InvitationStatus` | no | **`PENDING`** | ∈ {`PENDING`,`CONSUMED`,`REVOKED`} | |
| `expiresAt` | `timestamptz(3)` | no | — | — | Exact TTL deferred |
| `createdAt` | `timestamptz(3)` | no | set on create (UTC) | — | |
| `updatedAt` | `timestamptz(3)` | no | set on create/update (UTC) | — | |

**Explicit non-fields:** raw token; permissions bitset; `acceptedUserId` (accept creates User/Member in TX; no required post-accept FK); soft-delete.

### 28.2 BusinessInvitation — Identity / invariants

1. Shared model for Owner (CLI) and Staff (Owner API) invites (API §3.2).  
2. Token hashed; GET inspect does not consume; accept re-validates and consumes (API).  
3. Client `businessId`/email/role **not** trusted — row is source of truth.  
4. Re-invite invalidates previous pending for the same binding; Owner may revoke (PA §5.2).  
5. Accept TX: consume → create User (if needed) + BusinessMember + Staff↔Member bind if Staff + set password + create authenticated BusinessSession (PA/API).  
6. MVP: exactly one Owner or one pending owner invite practice (PA §5.1) — not a DB unique forcing single Owner later.  
7. At most one **`PENDING`** invitation per active binding: Owner → `(businessId, role=OWNER)`; Staff → `(businessId, staffId)` (re-invite invalidates prior).

### 28.3 BusinessInvitation — Relations

| Relation | Cardinality | Notes |
| --- | --- | --- |
| `Business` → Invitation | **1 : N** | |
| Invitation → `Staff` | **0..1** | Staff invites |
| → User / BusinessMember | **create on accept** | No lasting required FK |

FK **`ON DELETE`**: architecture OPEN #3.

### 28.4 BusinessInvitation — Uniqueness / indexes

| Rule | Scope | Kind |
| --- | --- | --- |
| Unique `id` (PK) | Global | FINAL |
| Unique `tokenHash` | Global | FINAL |
| One **`PENDING`** owner invite per `businessId` (role=`OWNER`); one **`PENDING`** staff invite per `(businessId, staffId)` | Tenant | **FINAL correctness** partial uniques — re-invite must revoke prior PENDING then insert; migration SQL (not OPEN #2) |
| List pending invitations | Tenant | Performance → **OPEN #2** |

### 28.5 BusinessInvitation — Lifecycle

```text
CLI Owner invite / Owner staff invite → PENDING (+ enqueue invitation email)
Re-issue                               → prior PENDING → REVOKED; new PENDING
Revoke                                 → REVOKED
Accept                                 → CONSUMED; User+Member(+Staff link)+session in one TX
Expiry                                 → unusable
```

### 28.6–28.9 Review / Consistency / Notes

**Final fields:** `id`, `businessId`, `email`, `role`, `staffId`, `tokenHash`, `status`, `expiresAt`, `createdAt`, `updatedAt`  
**Entity-level OPENs:** none  
Outside AuthToken OPEN #1; does not close #1–#3.

---

## 29. Entity: NotificationDelivery — field-level design

**Kind:** Email idempotency / claim ledger (concrete).  
**Sources:** PA §17; Data Model §8.1; conventions (`businessId` required FINAL).  
**Not** an auth entity. **Not** BullMQ job storage (payload identifier-only in Redis).

### 29.1 NotificationDelivery — Field Specification

| Field | PostgreSQL type | Nullable | Default | Constraints | Notes |
| --- | --- | --- | --- | --- | --- |
| `id` | `uuid` | no | app-generated UUID v7 | PK | |
| `businessId` | `uuid` | no | — | FK → `Business.id` | **Required NOT NULL (DM FINAL)**. PA §7 null path for “platform-only” emails is **out of MVP** per DM (“future extension”) — MVP types including `BUSINESS_PASSWORD_RESET` remain business-contextual |
| `type` | enum `NotificationType` | no | — | See §3 inventory | |
| `noticeVariant` | enum `BusinessAppointmentNoticeVariant` | yes | `null` | ∈ {`NEW`,`CANCELLED`,`RESCHEDULED`} when `type = BUSINESS_APPOINTMENT_NOTICE`; else null | B4 variants (PA §17.2) |
| `dedupeKey` | `text` | no | — | **UNIQUE** (global) | Idempotency key (PA §17.3) |
| `appointmentId` | `uuid` | yes | `null` | Composite FK `(businessId, appointmentId)` when set | Appointment-related types |
| `recipient` | `text` | no | — | Email as sent | Length TBD (source gap) |
| `status` | enum `NotificationDeliveryStatus` | no | **`PENDING`** | ∈ {`PENDING`,`SENDING`,`SENT`,`SKIPPED`,`FAILED`} | |
| `skipReason` | `text` | yes | `null` | — | When `SKIPPED`; length TBD |
| `attempts` | `int` | no | **0** | `≥ 0` | Retry count (PA: up to 8 attempts policy is worker config) |
| `claimedAt` | `timestamptz(3)` | yes | `null` | — | Claim timestamp; stale `SENDING` reclaim after **5 minutes** (PA) |
| `providerMessageId` | `text` | yes | `null` | — | Resend (etc.) id when sent |
| `sentAt` | `timestamptz(3)` | yes | `null` | — | Set on successful send |
| `lastError` | `text` | yes | `null` | — | Last failure detail; length TBD |
| `createdAt` | `timestamptz(3)` | no | set on create (UTC) | — | |
| `updatedAt` | `timestamptz(3)` | no | set on create/update (UTC) | — | |

**Explicit non-fields:** rendered HTML/PII secrets in this table as job payload; SMS/WhatsApp; webhook bus; payment notifications; outbox table (explicitly not MVP).

### 29.2 NotificationDelivery — Identity / invariants

1. Unique `dedupeKey` ⇒ at-least-once with DB guard (PA §17.3).  
2. Claim → validate → send; fail validation → `SKIPPED`; provider/network retries per PA policy.  
3. Worker re-reads DB; Redis payload identifier-only (no secrets/PII/HTML).  
4. Appointment facts from **Appointment snapshots**; branding from **current** Business (PA §17.4).  
5. `businessId` always required in MVP (DM over PA null note for schema).

### 29.3 NotificationDelivery — Relations

| Relation | Cardinality | Notes |
| --- | --- | --- |
| `Business` → Delivery | **1 : N** | Required |
| `Appointment` → Delivery | **0..1 : N** | Optional |

FK **`ON DELETE`**: architecture OPEN #3.

### 29.4 NotificationDelivery — Uniqueness / indexes

| Rule | Scope | Kind |
| --- | --- | --- |
| Unique `id` (PK) | Global | FINAL |
| Unique `dedupeKey` | Global | FINAL |
| Claim/list secondaries | — | OPEN #2 |

### 29.5 NotificationDelivery — Lifecycle

```text
Enqueue after commit → insert PENDING (+ unique dedupeKey) → BullMQ job
Worker claim         → PENDING/stale SENDING → SENDING (claimedAt)
Validate OK → send   → SENT (providerMessageId, sentAt)
Validate fail        → SKIPPED (skipReason)
Retryable fail       → FAILED/back to retry until attempts exhausted
```

### 29.6–29.9 Review / Consistency / Notes

**Final fields:** `id`, `businessId`, `type`, `noticeVariant`, `dedupeKey`, `appointmentId`, `recipient`, `status`, `skipReason`, `attempts`, `claimedAt`, `providerMessageId`, `sentAt`, `lastError`, `createdAt`, `updatedAt`  

**TBD (not arch OPENs):** max lengths for `recipient` / `skipReason` / `lastError` / `dedupeKey` / `providerMessageId`.  

**PA§7 vs DM§8.1:** resolved for MVP schema as **`businessId` required** (DM FINAL).  

Does not close OPENs #1–#3.

---

## 30. AuthToken — logical abstraction

**Kind:** Logical only — **not** a concrete Prisma model in this pass.  
**Sources:** Data Model §7; PA §8.4 / §9.4; CustomerVerificationIntent §22; User §10.

### 30.1 Scope (FINAL domain mapping)

| Flow | Persistence home |
| --- | --- |
| Customer registration verify | **`CustomerVerificationIntent.verificationTokenHash`** (concrete, **FINAL** MVP/default) — **outside OPEN #1** |
| Customer password reset | Purpose-scoped secret on **CustomerAccount** subject → **OPEN #1** physical |
| Customer email change | Purpose-scoped secret (+ pending new email) → **OPEN #1** physical |
| Business password reset | Purpose-scoped secret on **User** subject → **OPEN #1** physical |
| Invitation | **`BusinessInvitation.tokenHash`** — outside OPEN #1 |
| Guest manage | **`AppointmentManageToken.tokenHash`** — outside OPEN #1 |

**Distinction (explicit):** registration email verification → concrete `verificationTokenHash` on `CustomerVerificationIntent` (§22). Other AuthToken purposes (password reset, email change / pending new email, business password reset) → still **OPEN #1**.

### 30.2 Physical model — Architecture OPEN #1 (preserved)

OPEN #1 covers **only** the unresolved physical model for password-reset / email-change (/ pending new email) / business-reset token purposes. It does **not** include registration verification.

Sources **do not** pick A vs B for those remaining purposes:

- **A)** one generic purpose-scoped auth/security token table  
- **B)** purpose-specific tables  

Either must support: hashed secret at rest; purpose; subject binding; expiry; single-use; rotate/invalidate previous active intent per purpose/subject; raw secret only in memory/email worker.

**Do not** invent a concrete AuthToken entity/table in this doc to force-close the OPEN. **Do not** close OPEN #1.

### 30.3 Compatibility note for Prisma handoff

Implementors may choose A or B when writing Prisma for the remaining OPEN #1 purposes. `CustomerVerificationIntent` (incl. `verificationTokenHash`), `BusinessInvitation`, and `AppointmentManageToken` remain concrete regardless. Password-reset / email-change / business-reset require **some** physical token store under OPEN #1 before those flows ship.

**Architecture OPEN count:** still **3**.

---

## 31. Cross-entity consistency pass (holistic)

Performed against PA / API / DM and §9–§30. **No contradictory rewrites** of Groups A–C or Appointment were required.

| Area | Result |
| --- | --- |
| Tenant ownership | Tenant-owned entities carry `businessId`; User global; BusinessSession contextual via `activeBusinessId`; composites where required. `NotificationDelivery.businessId` required (DM). |
| Identity separation | User ≠ CustomerAccount ≠ Customer; BusinessSession ≠ CustomerSession ≠ AppointmentManageToken ≠ BusinessInvitation ≠ CustomerVerificationIntent ≠ AuthToken(logical). |
| Historical data | Appointment snapshots authoritative; Staff/Service/Customer mutations do not rewrite them (staff name snapshot only on reschedule staff change). |
| Authentication | Three guest/business/customer paths distinct; secrets hashed; registration verify on Intent; OPEN #1 only for password-reset / email-change / business-reset physical model. |
| Correctness vs OPEN #2 | Product UNIQUEs / partial uniques / exclusion / bookingReference = **FINAL**; secondary/performance indexes only = **OPEN #2**. |
| ManageToken statuses | `ACTIVE` / `REVOKED` / `CONSUMED` / `expiresAt` semantics FINAL in §27 — `CONSUMED` first-class, not expiry. |
| Scheduling | Hours ∩; closed/TimeOff subtract; no staff-hours fallback; overrideHours bounds; inactive Staff/Service block new books; exclusion FINAL. |
| Appointment | Status/source/snapshots/buffer/blockedUntil/exclusion/bookingReference/version/cancel/reschedule consistent with §26. |
| Money | Minor units + ISO4217; Service + Appointment snapshots; Business currency policy. |
| Time | timestamptz(3) UTC; Business IANA + Luxon; local-date closed/TimeOff; no browser TZ. |
| Soft delete | **Only** Customer.`deletedAt`. |
| Enums | Inventory §3 aligned with entity FINAL values (+ B4 variant enum). |
| Indexes | PK/UNIQUE/exclusion FINAL where stated; secondary inventory **OPEN #2**. |
| FK ON DELETE | **OPEN #3** throughout. |

### 31.1 Contradiction handled

| Topic | Handling |
| --- | --- |
| PA §7 `NotificationDelivery.businessId` nullable vs DM §8.1 required | **MVP schema follows DM FINAL: required.** Documented in §29. |

### 31.2 Readiness for Prisma

`docs/04-schema-design.md` is sufficient to implement Prisma models for all 19 concrete entities. OPEN #1 deferred for password-reset / email-change / business-reset physical tables only (registration verify already FINAL on `CustomerVerificationIntent`). OPEN #2 = performance/query indexes only. OPEN #3 = FK ON DELETE.

**Architecture OPEN count at completion:** **3**.
