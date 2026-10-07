# Data Model — MVP Entity & Relation Inventory

**Status:** Conceptual data model inventory (not a Prisma/SQL schema)  
**Depends on:**  
- [`01-product-architecture.md`](./01-product-architecture.md) (FINAL)  
- [`02-api-contract.md`](./02-api-contract.md) (MVP API inventory, OPEN = 0)  

**This phase:** entities, ownership, relations, lifecycle, and DB-level concurrency requirements.  
**Not in this phase:** Prisma schema, migrations, exact indexes/SQL DDL, cascade syntax, column types beyond conceptual notes.

---

## 0. Document rules

- Product Architecture is the product/architecture source of truth.
- API Contract defines HTTP resources; every persistent resource here must map to those capabilities (or to CLI/workers).
- IDs: all primary keys **UUID v7**, PostgreSQL native `uuid`, application-generated (PA §21). No Prisma field syntax in this doc.
- RLS is **not** MVP (PA); tenant isolation is application-scoped access + tenant-aware FKs/uniques designed so RLS can be added later.
- Exact TTLs, index lists, and Prisma mapping remain implementation follow-ups unless marked domain-FINAL below.

---

## 1. ID & public identifier strategy — FINAL (from PA)

| Kind | Rule |
| --- | --- |
| Primary keys | UUID v7, app-generated, `@db.Uuid` when implemented |
| `createdAt` | Authoritative creation time; never use ID order as business logic |
| Public booking URL | `Business.slug` (not PK) |
| Human booking ref | `Appointment.bookingReference` = `BK-` + 8 Crockford Base32; **business-scoped unique**; not PK |
| Secrets | Manage/invite/reset/verify tokens: high-entropy; **hash at rest**; raw only in email/memory |

---

## 2. Core tenant / identity

### 2.1 Business (tenant root)

**Purpose:** Multi-tenant root for all business-owned data.

**Conceptual fields (PA §6):**

| Group | Fields |
| --- | --- |
| Identity | `name`, `slug` (global unique, reserved-word checked) |
| Locale/money | `timezone` (IANA), `currency` (active ISO 4217) |
| Contact | `phone?`, `email?`, `address?` |
| Branding | `logoUrl?`, `primaryColor?` |
| Booking rules | `slotIntervalMinutes`, `minBookingNoticeMinutes`, `maxBookingWindowDays`, `cancellationNoticeMinutes` |
| Lifecycle | `status` (`INACTIVE` \| `ACTIVE`), `publishedAt?` (set once on first activate) |
| Timestamps | `createdAt`, `updatedAt` |

**Notes:** No business snapshot on appointments. Offboarding → INACTIVE + retain rows; slug never reused.

### 2.2 User

**Purpose:** Platform-global business authentication identity (Owner/Staff login).

| Concept | Notes |
| --- | --- |
| Email | Globally unique |
| Password | Argon2id hash |
| Profile name | As needed for invites/UI (not appointment staff snapshot source) |
| Timestamps | `createdAt`, `updatedAt` |

Not tenant-scoped. Links to businesses only via `BusinessMember`.

### 2.3 BusinessMember

**Purpose:** User ↔ Business membership with role.

| Concept | Notes |
| --- | --- |
| `userId`, `businessId` | Unique `(userId, businessId)` |
| `role` | `OWNER` \| `STAFF` |
| Status | Active / disabled (Staff deactivate disables membership) |
| Timestamps | `createdAt`, `updatedAt` |

MVP practice: one active membership per user; schema may allow more. Session holds `userId` + `activeBusinessId`; role reloaded from DB each request.

### 2.4 BusinessSession

**Purpose:** Server-side session for **business realm** (separate cookie/table from customers).

| Concept | Notes |
| --- | --- |
| Session token | Hashed at rest |
| `userId`, `activeBusinessId` | Identity + active tenant context |
| Expiry | Idle + absolute (exact durations deferred) |
| Rotation | On login / invite accept |

### 2.5 Staff

**Purpose:** Business-owned **bookable** profile (may exist without login).

| Concept | Notes |
| --- | --- |
| `businessId` | Tenant |
| `displayName` | Public UI + appointment staff-name snapshot source |
| Active/inactive | Inactive → no new bookings |
| Optional `BusinessMember` link | Composite tenant-aware FK including `businessId` when login-linked |
| Timestamps | `createdAt`, `updatedAt` |

STAFF membership ↔ exactly one Staff; OWNER membership ↔ zero or one Staff.

### 2.6 Customer

**Purpose:** Business-scoped customer record (guest or account-backed).

| Concept | Notes |
| --- | --- |
| `businessId` | Tenant |
| `name` | Current |
| `email?`, `normalizedEmail?` | Matching key when present |
| `phone?` | E.164; default parse country `TR` |
| Soft delete | `deletedAt?`; irreversible in MVP |
| Timestamps | `createdAt`, `updatedAt` |

**Uniques:** Partial unique `(businessId, normalizedEmail)` among non-deleted, non-null emails (migration SQL; PA).

Matching: normalized email only; phone never auto-links.

### 2.7 CustomerAccount

**Purpose:** Tenant-scoped customer login identity (1:1 with Customer).

| Concept | Notes |
| --- | --- |
| `customerId` | 1:1 |
| `businessId` | **Stored (FINAL)** — tenant-scoped login/session lookups; enables tenant-aware FKs such as `(businessId, customerId)`; reduces cross-tenant relation risk |
| Password hash | Argon2id |
| `verifiedAt`, status | Active/disabled |
| No separate email | Login email lives on `Customer` |

### 2.8 CustomerSession

**Purpose:** Server-side session for **customer realm** (separate cookie/table).

| Concept | Notes |
| --- | --- |
| Token | Hashed |
| `customerAccountId`, `businessId` | Must match slug-resolved business |
| Expiry / rotation | Same security principles as business sessions |

**Realm split (FINAL):** `BusinessSession` ≠ `CustomerSession`; different cookies; logging into another business’s customer area replaces the single customer cookie (PA).

### 2.9 CustomerVerificationIntent

**Purpose:** Persistent pending **customer registration** before any `Customer` / `CustomerAccount` exists.

Product Architecture FINAL: register collects email + password + name + phone; nothing is created until verification succeeds; expiry/failure leaves no Customer/Account; re-registration with the same email invalidates the previous pending intent.

| Concept | Notes |
| --- | --- |
| `businessId` | Tenant |
| `normalizedEmail` | Pending login identity within business |
| Pending profile | Pending name, pending phone |
| Hashed password | Argon2id of chosen password (never raw) |
| Verification secret | Token/hash association (same-row or linked AuthToken — schema choice) |
| Expiry | Exact TTL deferred |
| State | Pending / consumed / revoked / superseded |
| Timestamps | `createdAt`, `updatedAt` |

**Not** FK to `CustomerAccount` (account is created only after successful verification). On verify: one transaction finds-or-creates active Customer by normalized email (existing Customer field values win) then creates `CustomerAccount`.

This entity is **independent** of the AuthToken physical-table OPEN (§7).

---

## 3. Business setup

### 3.1 Service

| Concept | Notes |
| --- | --- |
| `businessId` | Tenant |
| `name`, `durationMinutes`, `bufferMinutes` | |
| `priceMinor`, `currency` | Integer minor units; ISO 4217 aligned with business at write |
| Active/inactive | No hard delete |
| Timestamps | |

### 3.2 StaffService

| Concept | Notes |
| --- | --- |
| `businessId`, `staffId`, `serviceId` | M:N; tenant-aware composite FKs |
| Unique | `(businessId, staffId, serviceId)` |
| Semantics | API `PUT .../staff/:id/services` replaces set; unlink does not mutate appointments |

### 3.3 BusinessWorkingHour

| Concept | Notes |
| --- | --- |
| `businessId` | Tenant |
| `dayOfWeek` | ISO 1–7 |
| `startMinute`, `endMinute` | Local wall-clock; end exclusive; multiples of 5; no overnight |
| Multiple rows | Per day intervals allowed if non-overlapping (may touch) |

### 3.4 StaffWorkingHour

Same shape as business hours, scoped by `businessId` + `staffId`.

### 3.5 StaffTimeOff

**Purpose:** Business-local **full-day** staff availability exclusion (not an arbitrary free-form datetime range in the product model).

| Concept | Notes |
| --- | --- |
| `businessId`, `staffId` | Tenant-aware |
| Local date | Business-timezone calendar date (`YYYY-MM-DD`) |
| Interval meaning | Start = that date’s **local midnight**; end = **next** local midnight |
| Duration | Elapsed length is **not** required to be 24h (DST 23h/25h days) |
| Engine | Converts the local-date bounds to a UTC half-open interval for availability |
| Effect | Closes that staff’s entire availability for that local day; never auto-cancels appointments |

Exact DB column/type representation is deferred to schema (may store local date only, or materialized UTC bounds derived from it).

### 3.6 BusinessClosedDate

| Concept | Notes |
| --- | --- |
| `businessId` | Tenant |
| `localDate` | Business-TZ calendar date; unique `(businessId, localDate)` |
| Semantics | Full-day closure; no public slots; normal manual booking blocked; Owner override on mutation only; history unchanged |

---

## 4. Appointment domain

### 4.1 Appointment (aggregate root)

| Group | Concepts |
| --- | --- |
| Tenant / parties | `businessId`, `customerId`, `staffId`, `serviceId` |
| Schedule | `startsAt`, `endsAt`, `blockedUntil` (`timestamptz(3)`) |
| Status | `CONFIRMED` \| `COMPLETED` \| `CANCELLED` \| `NO_SHOW` |
| Meta | `source` (`PUBLIC` \| `OWNER` \| `STAFF`), `overrideHours` (bool), `version` (monotonic) |
| Public ref | `bookingReference` (`BK-XXXXXXXX`, business-scoped unique) |
| Cancel | `cancelledAt?`, `cancelledBy?` (`CUSTOMER` \| `STAFF` \| `OWNER`) |
| Note | Customer note optional; **immutable** after create |
| Timestamps | `createdAt`, `updatedAt` |

**Reschedule:** in-place UPDATE of the same row (no new appointment row). Staff-name snapshot updates when staff changes; service snapshots do not change on reschedule.

### 4.2 Booking-time snapshots (on Appointment)

Immutable columns (or equivalent snapshot group) capturing booking-time truth:

| Snapshot | Why |
| --- | --- |
| Customer name / email / phone | Soft-delete & profile edits must not rewrite history; admin detail falls back when customer deleted |
| Customer note | Create-time only; never edited later |
| Service name / duration / buffer / priceMinor / currency | Service edits must not alter past bookings; `endsAt`/`blockedUntil` derived from booking-time duration/buffer |
| Staff display name | Staff rename must not rewrite history (updated only on reschedule staff change per PA) |

**No business branding snapshot** on Appointment (emails/UI use current Business).

### 4.3 DB concurrency — FINAL requirements

| Rule | Requirement |
| --- | --- |
| Protected interval | Half-open `[startsAt, blockedUntil)` |
| Exclusion | Per-staff exclusion constraint on `tstzrange(startsAt, blockedUntil, '[)')` where `status <> 'CANCELLED'` (`btree_gist`) |
| Blocking statuses | Anything except `CANCELLED` (CONFIRMED, COMPLETED, NO_SHOW) |
| Double-booking | Enforced in DB, not only app checks |
| Status transitions | Conditional UPDATE + `version` increment |

Engine, constraint, and tests must share the same range + status predicate (PA).

---

## 5. Booking references & guest manage tokens

### 5.1 Booking reference

Field on `Appointment` (not a separate entity): business-scoped unique public reference `BK-XXXXXXXX`.

### 5.2 AppointmentManageToken

**Purpose:** Guest cancel/reschedule (and manage-context) authorization.

| Concept | Notes |
| --- | --- |
| `businessId`, `appointmentId` | Tenant-aware relation |
| Purpose | Manage (single-purpose) |
| `tokenHash` | Raw never stored |
| Expiry | Exact TTL deferred |
| State | Active / revoked / consumed-as-needed |
| Multiplicity | Multiple active manage tokens may exist; reschedule/cancel/customer delete revoke relevant tokens; new token issued on reschedule email |
| Timestamps | `createdAt`, etc. |

Transport (API): fragment in email URL → frontend → POST body `token` (API Contract FINAL).

---

## 6. Invitations

### 6.1 BusinessInvitation

Shared entity for **Owner** and **Staff** invites.

| Concept | Notes |
| --- | --- |
| `businessId` | Tenant |
| `email` | Invitee |
| `role` | `OWNER` \| `STAFF` |
| `staffId?` | Required/used for Staff invites; null for Owner invite |
| `tokenHash` | High-entropy; single-use |
| Expiry | Deferred exact TTL |
| Status | Pending / consumed / revoked |
| Rules | Re-invite invalidates previous; accept is one transaction: consume + User + BusinessMember + Staff link if needed + session |
| Timestamps | |

Token API binding: `/api/invitations/:token` (inspect/accept); invitation row is source of truth (not client businessId/email/role).

---

## 7. Security flows & AuthToken (logical)

### 7.1 Subject-separated flows (FINAL mapping)

| Flow | Persisted subject / entity | Notes |
| --- | --- | --- |
| Customer registration verification | **`CustomerVerificationIntent`** | Exists **before** Customer/CustomerAccount |
| Customer password reset | Existing **CustomerAccount** (+ Customer) | Purpose-scoped secret |
| Customer email change | Existing **Customer** / **CustomerAccount** | Pending new email + verify |
| Business password reset | Existing **User** | Purpose-scoped secret |
| Invitation | **`BusinessInvitation`** | Own token hash on invitation row |
| Guest appointment manage | **`AppointmentManageToken`** | Own token hash; multi-active allowed |

### 7.2 AuthToken — logical abstraction

`AuthToken` remains a **logical** name for purpose-scoped secrets used by password-reset / email-change (and optionally registration verify if not embedded on `CustomerVerificationIntent`).

**Physical modeling — OPEN:**

- **A)** one generic purpose-scoped auth/security token table, and/or  
- **B)** purpose-specific tables  

`CustomerVerificationIntent` **must exist** as a concrete entity either way; its presence does not depend on choosing A vs B. Invitation and manage tokens already have dedicated entities and are outside this OPEN.

---

## 8. Notification / email domain

### 8.1 NotificationDelivery

| Concept | Notes |
| --- | --- |
| `id` | UUID v7 |
| `businessId` | **Required / NOT NULL (FINAL)** — every MVP notification type is business-contextual; no platform-null path in MVP |
| `type` | Email type enum (PA notification types) |
| `dedupeKey` | Unique idempotency key |
| `appointmentId?` | When appointment-related; tenant-aware FK |
| Recipient email | As sent |
| `status` | `PENDING` \| `SENDING` \| `SENT` \| `SKIPPED` \| `FAILED` |
| `skipReason?`, `attempts`, `claimedAt?` | Claim/retry |
| `providerMessageId?`, `sentAt?`, `lastError?` | Provider outcome |
| Timestamps | `createdAt`, `updatedAt` |

**Jobs:** BullMQ email job payloads are **not** rows in this table; payload stays identifier-only (PA). Delivery row is the idempotency/claim ledger.

Future non-business/platform notifications would require an explicit model extension (out of MVP).

---

## 9. Tenant isolation (conceptual)

| Entity | Tenant key | Notes |
| --- | --- | --- |
| Business | self | Root |
| User | — | Global |
| BusinessMember | `businessId` | |
| BusinessSession | via `activeBusinessId` / user | Not a “row owned by business” in the same sense; still scoped in queries |
| Staff, Service, StaffService, hours, TimeOff, ClosedDate | `businessId` | Composite FKs where child points at parent+business |
| Customer, CustomerAccount, CustomerSession, CustomerVerificationIntent | `businessId` | |
| Appointment, AppointmentManageToken | `businessId` | |
| BusinessInvitation | `businessId` | |
| NotificationDelivery | `businessId` **required** | Always tenant-owned in MVP |
| AuthToken (logical) | purpose-dependent | Must not cross tenants; registration pending uses CustomerVerificationIntent |

Cross-tenant id mismatch → API **404**. Staff on Owner-only CM → **403**.

---

## 10. Soft delete / lifecycle (do not conflate)

| Entity | Lifecycle model |
| --- | --- |
| Business | `INACTIVE` / `ACTIVE` + `publishedAt`; no hard delete in MVP |
| Service | `active` boolean (deactivate); no hard delete |
| Staff | `active` boolean; deactivate blocked if future CONFIRMED; membership/sessions/invites handled per PA |
| Customer | **Soft delete** (`deletedAt`); account disabled; sessions/tokens revoked |
| CustomerAccount | Disabled with customer delete / policy; not independently “hard deleted” in MVP |
| CustomerVerificationIntent | Pending → consumed / revoked / superseded; never creates orphan Customer/Account |
| Appointment | Status machine; `CANCELLED` terminal; not soft-deleted |
| Invitation | Pending → consumed/revoked |
| Sessions | Deleted/revoked on logout, reset, membership disable, etc. |
| Manage tokens | Revoked/expired; not soft-deleted rows required |
| NotificationDelivery | Status machine PENDING→…; always tied to a Business |

---

## 11. Relationship inventory

```text
Business
 ├── BusinessMember
 ├── BusinessSession (via activeBusinessId)
 ├── Staff
 ├── Service
 ├── StaffService
 ├── BusinessWorkingHour
 ├── BusinessClosedDate
 ├── Customer
 ├── CustomerAccount
 ├── CustomerSession
 ├── CustomerVerificationIntent
 ├── Appointment
 ├── AppointmentManageToken
 ├── BusinessInvitation
 └── NotificationDelivery

User
 ├── BusinessMember
 └── BusinessSession

Staff
 ├── StaffService
 ├── StaffWorkingHour
 ├── StaffTimeOff
 ├── Appointment
 ├── optional BusinessMember (composite tenant-aware)
 └── BusinessInvitation (optional staffId)

Service
 └── StaffService

Customer
 ├── CustomerAccount (1:1)
 └── Appointment

CustomerAccount
 └── CustomerSession

Appointment
 └── AppointmentManageToken

CustomerVerificationIntent
 └── (no FK to Customer/Account; resolves into them on verify)
```

Logical **AuthToken** covers purpose-scoped secrets for password-reset / email-change (and optionally registration verify if not embedded on `CustomerVerificationIntent`). Invitation and manage tokens are dedicated entities. Physical AuthToken table shape OPEN (§14).

---

## 12. Entity review table

| Entity | Owner | Purpose | Key relations | Lifecycle | Tenant scoped? |
| --- | --- | --- | --- | --- | --- |
| Business | — | Tenant root | children below | INACTIVE/ACTIVE + publishedAt | Root |
| User | — | Business auth identity | BusinessMember, BusinessSession | Active user | No |
| BusinessMember | Business | Membership + role | User, Business, optional Staff | Active/disabled | Yes |
| BusinessSession | User (+ business context) | Business realm session | User | Expiry/revoke | Contextual |
| Staff | Business | Bookable profile | StaffService, hours, TimeOff, Appointment | Active/inactive | Yes |
| Service | Business | Offerable service | StaffService, Appointment | Active/inactive | Yes |
| StaffService | Business | Staff↔Service M:N | Staff, Service | Replace set | Yes |
| BusinessWorkingHour | Business | Weekly open intervals | Business | Replace schedule | Yes |
| StaffWorkingHour | Business | Staff weekly intervals | Staff | Replace schedule | Yes |
| StaffTimeOff | Business | Business-local full-day staff exclusion | Staff | CRUD | Yes |
| BusinessClosedDate | Business | Full-day business closure | Business | CRUD | Yes |
| Customer | Business | Customer record | Account, Appointment | Soft delete | Yes |
| CustomerAccount | Business | Customer login | Customer, CustomerSession; **stores businessId** | Active/disabled | Yes |
| CustomerSession | CustomerAccount | Customer realm session | CustomerAccount | Expiry/revoke | Yes |
| CustomerVerificationIntent | Business | Pending registration before Customer/Account | Business (no Customer FK) | Pending/consumed/revoked/superseded | Yes |
| Appointment | Business | Booking aggregate | Customer, Staff, Service, manage tokens | Status machine | Yes |
| AppointmentManageToken | Business | Guest manage auth | Appointment | Expiry/revoke | Yes |
| BusinessInvitation | Business | Owner/Staff invite | Business, optional Staff | Pending/consumed/revoked | Yes |
| NotificationDelivery | Business | Email idempotency ledger | Business (required), optional Appointment | PENDING→… | Yes |
| AuthToken (logical) | Subject-dependent | Password-reset / email-change secrets (optional shared verify secret) | User or CustomerAccount/Customer | Single-use/rotate | Purpose-dependent |

**Concrete persisted entities:** **19**  
**Logical abstraction:** **AuthToken (+1)** — physical split OPEN  

Inventory = **19 concrete + AuthToken (logical)**.

---

## 13. API ↔ entity coverage (sanity)

| API area | Primary entities |
| --- | --- |
| Business auth / session | User, BusinessMember, BusinessSession |
| Invitations | BusinessInvitation, User, BusinessMember, Staff |
| Customer auth / account | Customer, CustomerAccount, CustomerSession, CustomerVerificationIntent, AuthToken(logical) |
| Public profile / availability / book | Business, Service, Staff, hours, TimeOff, ClosedDate, Appointment |
| Guest manage | Appointment, AppointmentManageToken |
| Settings / activate | Business |
| Services / Staff / StaffService | Service, Staff, StaffService |
| Hours / TimeOff / ClosedDate | *WorkingHour, StaffTimeOff, BusinessClosedDate |
| Appointments / calendar / dashboard | Appointment (+ joins/snapshots) |
| Customer Management | Customer, Appointment |
| Email workers | NotificationDelivery (+ reads of Appointment/Business/…) |
| CLI provision | Business, BusinessInvitation |

---

## 14. OPEN decisions (data model only)

1. **AuthToken physical model** for remaining purpose-scoped secrets (customer password reset, customer email-change, business password reset; optionally registration verify secret if not embedded on `CustomerVerificationIntent`): generic purpose-scoped table vs purpose-specific tables. Domain flows FINAL; `CustomerVerificationIntent` concrete either way.
2. **Exact DB indexes** beyond known uniques/exclusion — deferred to schema/implementation.
3. **Exact FK on-delete behavior** (RESTRICT vs CASCADE vs SET NULL) per relation — deferred; must not violate soft-delete/history rules.
4. **NotificationDelivery** precise column constraints (`recipient`, `skipReason`, `lastError`, lengths) — deferred to schema.

Non-OPEN (already FINAL): UUID v7; exclusion overlap; snapshots; manage tokens; invitations; session realm split; closed dates; StaffTimeOff **full-day local-date** semantics; booking reference; StaffService replacement; **`CustomerAccount.businessId` stored**; **`NotificationDelivery.businessId` required**; **`CustomerVerificationIntent` concrete**.

---

## 15. Consistency check results

| Check | Result |
| --- | --- |
| PA entity missing from model? | **No** (registration pending covered by CustomerVerificationIntent) |
| API resource without entity? | **No** |
| Tenant boundary violation? | **None identified** |
| Appointment snapshots complete? | **Yes** |
| Customer vs CustomerAccount split? | **Yes** |
| Registration before account? | **Yes** — CustomerVerificationIntent |
| BusinessMember vs Staff? | **Yes** |
| Appointment concurrency model? | **Yes** |
| Lifecycle distinctions? | **Yes** |
| UUID v7? | **Yes** |
| Unnecessary entities? | **No** |

### Final review counts

| Metric | Value |
| --- | --- |
| Concrete persisted entities | **19** |
| Logical abstractions | **AuthToken (+1)** |
| OPEN count | **4** |
| Missing entity | **None** (after CustomerVerificationIntent) |
| Suspicious relation | **None** |
| Contradiction | **None** |
