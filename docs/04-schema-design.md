# Schema Design — MVP Canonical Persistence Spec

**Status:** Field-level design in progress — Business + User complete; remaining entities pending.  
**Depends on:**  
- [`01-product-architecture.md`](./01-product-architecture.md)  
- [`02-api-contract.md`](./02-api-contract.md)  
- [`03-data-model.md`](./03-data-model.md)  

**Done in this doc so far:** conventions, enum inventory, entity index, review order, architecture OPENs, **Business** and **User** field specifications.  
**Not in this phase for remaining entities:** per-table field designs.  
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
| `BusinessMemberStatus` | Active / disabled — **exact enum vs boolean OPEN at schema** if not already boolean in impl | BusinessMember |
| `AppointmentStatus` | `CONFIRMED`, `COMPLETED`, `CANCELLED`, `NO_SHOW` (no `PENDING`) | Appointment |
| `AppointmentSource` | `PUBLIC`, `OWNER`, `STAFF` | Appointment |
| `CancelledBy` | `CUSTOMER`, `STAFF`, `OWNER` | Appointment |
| `InvitationStatus` | Pending / consumed / revoked — names TBD at schema (`PENDING`, `CONSUMED`, `REVOKED` recommended) | BusinessInvitation |
| `CustomerVerificationIntentStatus` | Pending / consumed / revoked / superseded — names TBD at schema | CustomerVerificationIntent |
| `CustomerAccountStatus` | Active / disabled — enum vs boolean at schema | CustomerAccount |
| `NotificationDeliveryStatus` | `PENDING`, `SENDING`, `SENT`, `SKIPPED`, `FAILED` | NotificationDelivery |
| `NotificationType` | PA email types: `BOOKING_CONFIRMED`, `BOOKING_RESCHEDULED`, `BOOKING_CANCELLED`, `BOOKING_REMINDER`, `ACCOUNT_VERIFICATION`, `ACCOUNT_ALREADY_EXISTS`, `CUSTOMER_PASSWORD_RESET`, `EMAIL_CHANGE_VERIFICATION`, `EMAIL_CHANGED_NOTICE`, `OWNER_INVITATION`, `STAFF_INVITATION`, `BUSINESS_PASSWORD_RESET`, `BUSINESS_APPOINTMENT_NOTICE` (+ notice variants NEW/CANCELLED/RESCHEDULED — variant as enum field or composite type at schema) | NotificationDelivery |
| `AuthTokenPurpose` | At least: customer password reset, customer email-change, business password reset; optionally registration verify if secret not embedded on `CustomerVerificationIntent` | AuthToken (logical) |
| `ManageTokenStatus` | Active / revoked / expired-or-consumed as needed — exact labels at schema | AppointmentManageToken |

**Enum count (named inventory rows):** **13**

Boolean flags that are **not** enums by product decision: Service/Staff `active`, Appointment `overrideHours`. Slot interval is a constrained integer set (5/10/15/20/30/60), not necessarily a DB enum.

---

## 4. Entity catalog

Index only — no columns in this revision.

| Entity | Section (planned) | Kind |
| --- | --- | --- |
| Business | [§9 Business](#9-entity-business--field-level-design) | Concrete — **fields designed** |
| User | [§10 User](#10-entity-user--field-level-design) | Concrete — **fields designed** |
| BusinessMember | § BusinessMember | Concrete |
| BusinessSession | § BusinessSession | Concrete |
| Staff | § Staff | Concrete |
| Service | § Service | Concrete |
| StaffService | § StaffService | Concrete |
| BusinessWorkingHour | § WorkingHours | Concrete |
| StaffWorkingHour | § WorkingHours | Concrete |
| StaffTimeOff | § StaffTimeOff | Concrete |
| BusinessClosedDate | § ClosedDate | Concrete |
| Customer | § Customer | Concrete |
| CustomerVerificationIntent | § CustomerVerificationIntent | Concrete |
| CustomerAccount | § CustomerAccount | Concrete |
| CustomerSession | § CustomerSession | Concrete |
| Appointment | § Appointment | Concrete |
| AppointmentManageToken | § AppointmentManageToken | Concrete |
| BusinessInvitation | § BusinessInvitation | Concrete |
| NotificationDelivery | § NotificationDelivery | Concrete |
| AuthToken | § AuthToken abstraction | **Logical** (physical shape OPEN) |

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

1. **AuthToken physical model** — generic purpose-scoped table vs purpose-specific tables for customer password reset, customer email-change, business password reset (and optionally registration verify secret if not embedded on `CustomerVerificationIntent`). `CustomerVerificationIntent` remains concrete either way.  
2. **Exact DB indexes** beyond known uniques / exclusion constraint — deferred until indexes are chosen during/after entity design and implementation planning.  
3. **Exact FK on-delete behavior** (RESTRICT vs CASCADE vs SET NULL) — deferred; must preserve soft-delete and history rules when decided.

**Architecture-level Schema OPEN count:** **3**

### 6.2 Field-level decisions (not architecture OPENs)

Ordinary per-entity column details are **intentionally deferred** to the entity-by-entity schema pass. They are **not** tracked as architecture-level OPENs. Examples:

- nullability, lengths, and exact column names for `NotificationDelivery` fields such as `recipient`, `skipReason`, `lastError`  
  (`NotificationDelivery.businessId` is already **FINAL** and **required**)  
- other entities’ field matrices, check constraints on individual columns, Prisma attribute spelling  

Those are resolved when each entity section is designed—not held as separate OPEN items. **Business** field-level decisions are recorded in §9; **User** in §10; remaining entities still deferred.

---

## 7. Out of scope (still)

- Prisma models / `@db` attributes as source files  
- Migration SQL / exclusion constraint DDL text  
- Seed data  
- Premature resolution of the three architecture-level OPENs in §6.1  
- Field-level design of entities other than **Business** and **User**  

---

## 8. Document consistency report

| Metric | Value |
| --- | --- |
| Enum inventory rows | **13** |
| Concrete entities | **19** |
| Logical abstractions | **1** (AuthToken) |
| Architecture-level Schema OPEN count | **3** (unchanged) |
| Entities with field-level design | **2** (Business, User) |
| Missing entity vs Data Model | **None** |
| Contradiction vs PA / API / Data Model | **None** (User pass — see §10.9) |

Next field-level pass: **BusinessMember** (§5 review order #3).

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
Invitation accept (new email)  → create User (email, name, passwordHash) + BusinessMember (+ Staff link if staff invite) + rotated BusinessSession
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
