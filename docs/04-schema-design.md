# Schema Design — MVP Canonical Persistence Spec

**Status:** Field-level design in progress — **Business** complete; other entities not yet designed  
**Depends on:**  
- [`01-product-architecture.md`](./01-product-architecture.md)  
- [`02-api-contract.md`](./02-api-contract.md)  
- [`03-data-model.md`](./03-data-model.md)  

**Done in this doc so far:** conventions, enum inventory, entity index, review order, architecture OPENs, **Business field specification**.  
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
| User | § User (planned) | Concrete |
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

These remain unresolved at the **schema-architecture** layer (carried from Data Model). Do **not** invent new ones. Do **not** prematurely decide them in this skeleton:

1. **AuthToken physical model** — generic purpose-scoped table vs purpose-specific tables for customer password reset, customer email-change, business password reset (and optionally registration verify secret if not embedded on `CustomerVerificationIntent`). `CustomerVerificationIntent` remains concrete either way.  
2. **Exact DB indexes** beyond known uniques / exclusion constraint — deferred until indexes are chosen during/after entity design and implementation planning.  
3. **Exact FK on-delete behavior** (RESTRICT vs CASCADE vs SET NULL) — deferred; must preserve soft-delete and history rules when decided.

**Architecture-level Schema OPEN count:** **3**

### 6.2 Field-level decisions (not architecture OPENs)

Ordinary per-entity column details are **intentionally deferred** to the entity-by-entity schema pass. They are **not** tracked as architecture-level OPENs. Examples:

- nullability, lengths, and exact column names for `NotificationDelivery` fields such as `recipient`, `skipReason`, `lastError`  
  (`NotificationDelivery.businessId` is already **FINAL** and **required**)  
- other entities’ field matrices, check constraints on individual columns, Prisma attribute spelling  

Those are resolved when each entity section is designed—not held as separate OPEN items. **Business** field-level decisions are recorded in §9; remaining entities still deferred.

---

## 7. Out of scope (still)

- Prisma models / `@db` attributes as source files  
- Migration SQL / exclusion constraint DDL text  
- Seed data  
- Premature resolution of the three architecture-level OPENs in §6.1  
- Field-level design of entities other than **Business**  

---

## 8. Document consistency report

| Metric | Value |
| --- | --- |
| Enum inventory rows | **13** |
| Concrete entities | **19** |
| Logical abstractions | **1** (AuthToken) |
| Architecture-level Schema OPEN count | **3** (unchanged) |
| Entities with field-level design | **1** (Business) |
| Missing entity vs Data Model | **None** |
| Contradiction vs PA / API / Data Model | **None** |

Next field-level pass: **User** (§5 review order #2).

---

## 9. Entity: Business — field-level design

**Kind:** Tenant root (concrete).  
**Does not** carry a `businessId` column (it *is* the tenant).  
**Does not** use soft delete / `deletedAt`.

### 9.1 Business — Field Specification

| Field | Type (logical) | Required | Default | Constraints / Semantics |
| --- | --- | --- | --- | --- |
| `id` | UUID v7 | yes | app-generated on insert | PK; never client-supplied as authorization; PostgreSQL native `uuid` eventually |
| `name` | string | yes | — | Trimmed; length **1–80**; display name / branding |
| `slug` | string | yes | — | Global unique; **2–48** chars; `^[a-z0-9]+(?:-[a-z0-9]+)*$`; reserved-word list (PA); never reused; mutable **only** while `publishedAt IS NULL` |
| `timezone` | string (IANA) | yes | — | Validated + canonical IANA name; sole TZ for local dates/hours/display; **immutable after first publish** (`publishedAt` set) |
| `currency` | string (ISO 4217) | yes | — | Active ISO 4217; 3-letter **uppercase**; unknown/invalid rejected; no fixed allow-list; **immutable after first publish** |
| `phone` | string (E.164) \| null | no | `null` | When set: E.164; parse default country **`TR`** if no calling code; empty → `null` |
| `email` | string \| null | no | `null` | When set: trim + **lowercase** store (contact email; not a separate `normalizedEmail` column on Business); empty → `null` |
| `address` | string \| null | no | `null` | Plain text; max **300**; empty → `null`; no structured address |
| `logoUrl` | string \| null | no | `null` | Absolute **HTTPS** URL; max **2048**; no userinfo; reject localhost/private best-effort at validation layer; empty → `null` |
| `primaryColor` | string \| null | no | `null` | `#` + 6 hex; store **lowercase**; invalid reject; null → platform default at render |
| `slotIntervalMinutes` | int | yes | **15** | One of **5 \| 10 \| 15 \| 20 \| 30 \| 60** |
| `minBookingNoticeMinutes` | int | yes | **60** | ≥ 0; **elapsed** duration; public booking + customer reschedule |
| `maxBookingWindowDays` | int | yes | **60** | **1–365**; **local calendar days** in business TZ |
| `cancellationNoticeMinutes` | int | yes | **1440** | ≥ 0; **elapsed** duration; customer cancel + customer reschedule eligibility |
| `status` | enum `BusinessStatus` | yes | **`INACTIVE`** | `INACTIVE` \| `ACTIVE` only; no other lifecycle enums |
| `publishedAt` | timestamptz(3) \| null | no | `null` | Set **once** on first successful activate to UTC now; **never cleared**; null ⇒ never published |
| `createdAt` | timestamptz(3) | yes | set on create (UTC) | Authoritative creation time |
| `updatedAt` | timestamptz(3) | yes | set on create/update (UTC) | Last row mutation; settings are last-write-wins (**no** `version` on Business) |

**Not persisted on Business (explicit non-fields):** `businessId`, `deletedAt`, `version`, `locale`, `rescheduleNotice`, `deactivatedAt`, browser timezone, payment fields.

### 9.2 Business — Invariants

1. `id` is unique globally (PK).  
2. `slug` is unique globally and never reassigned to another business after use.  
3. `status ∈ {INACTIVE, ACTIVE}`.  
4. If `publishedAt IS NULL`, business has never been activated; public slug → 404.  
5. If `publishedAt IS NOT NULL`, first publish has occurred; `publishedAt` must not change.  
6. After `publishedAt` is set: `slug`, `timezone`, and `currency` are immutable at the product layer.  
7. Booking-rule integers always present (defaults at create); contact/branding may be null.  
8. Activate/reactivate requires go-live checklist (hours, service, staff, staff–service, staff hours) — checklist is **derived**, not stored columns.  
9. Deactivate sets `status = INACTIVE` and **keeps** `publishedAt`.  
10. No soft delete; operator offboarding uses INACTIVE + retain rows (PA).  
11. Money for the tenant is defined by `currency`; prices live on Service/Appointment snapshots, not on Business beyond the currency code.

### 9.3 Business — Uniqueness / Lookup Rules

| Rule | Scope | FINAL? |
| --- | --- | --- |
| PK `id` | Global | Yes |
| Unique `slug` | Global | Yes (reserved words + format at app validation) |
| Public tenant resolution | Lookup by `slug` | Yes (API public surface) |
| Admin tenant resolution | Session `activeBusinessId` → `id` | Yes (never trust client `businessId`) |

**Candidate** supporting indexes (not a finalized index strategy — **Depends on existing architecture OPEN #2**): unique constraint/index on `slug` is implied by uniqueness; additional lookup indexes TBD later.

### 9.4 Business — Lifecycle

```text
Create (CLI)     → status=INACTIVE, publishedAt=null, booking-rule defaults applied
First activate   → checklist OK → status=ACTIVE, publishedAt=now()  (once)
Deactivate       → status=INACTIVE, publishedAt unchanged
Reactivate       → checklist OK → status=ACTIVE, publishedAt unchanged
```

- Public profile: unpublished (`publishedAt` null) → 404; published+INACTIVE → limited inactive message; ACTIVE → full booking profile.  
- Settings concurrency: last-write-wins; activate/deactivate via conditional status transition (0 rows → 409).  
- No `Business.version` column.

### 9.5 Business — Relationships

Business is the **parent/root** of tenant-owned children (designed later). Conceptual children only:

`BusinessMember`, `BusinessSession` (via `activeBusinessId`), `Staff`, `Service`, `StaffService`, `BusinessWorkingHour`, `BusinessClosedDate`, `StaffTimeOff` (via Staff), `Customer`, `CustomerAccount`, `CustomerSession`, `CustomerVerificationIntent`, `Appointment`, `AppointmentManageToken`, `BusinessInvitation`, `NotificationDelivery`.

**FK on-delete for children:** not decided here — **Depends on existing architecture OPEN #3**. Safe statement at Business level only: Business rows are retained for MVP offboarding (no hard-delete cascade of tenant history as a product action).

### 9.6 Business — Schema Notes

- Field-level design for Business does **not** close architecture OPENs #1–#3.  
- AuthToken (#1) has no dependency from Business fields.  
- Exact secondary indexes (#2): only uniqueness of `slug` is locked as a rule; physical index catalog remains OPEN.  
- Child FK `ON DELETE` (#3): deferred until child entities are designed.  
- Implementation mapping (Prisma `@db.Uuid`, `@db.Timestamptz(3)`, enum native vs text) is out of this section’s scope.  
- Reserved slug list is application/config enforcement, not a Business column.  
- Ambiguity noted without deciding: whether `Business.email` max length needs an explicit numeric cap beyond “valid email” — PA specifies lowercase nullable email but no max length for business email (unlike Customer flows); **leave unbound except email-format validation** until a later field pass if product requires a cap — do not invent a length here.
