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
| `businessId` | Denormalized tenant for isolation/queries (tenant-aware) |
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

| Concept | Notes |
| --- | --- |
| `businessId`, `staffId` | Tenant-aware |
| Range | Stored as UTC instants (`startsAt`, `endsAt`) half-open |
| All-day | Local `YYYY-MM-DD` → `[local 00:00, next local 00:00)` (not 24 elapsed hours) |
| Effect | Blocks future availability only; never auto-cancels appointments |

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

## 7. Customer / business security intents (tokens)

### Domain requirements (FINAL capabilities)

| Purpose | Realm | Needs |
| --- | --- | --- |
| Account verification | Customer | Pending payload may include password hash + name/phone; no Customer/Account until verify |
| Customer password reset | Customer | Purpose-scoped; revoke sessions on success |
| Customer email change | Customer | New email pending; notice to old on success |
| Business password reset | Business user | Platform-level email; may have `businessId` null on related delivery |

### Physical modeling — OPEN

Whether these are:

- **A)** one `AuthToken` / `SecurityToken` table with `purpose` enum + polymorphic subject, or  
- **B)** separate tables per purpose  

is **OPEN** (implementation/schema choice). Domain requirements above are fixed either way.

Same OPEN covers business password-reset token storage shape.

---

## 8. Notification / email domain

### 8.1 NotificationDelivery

| Concept | Notes |
| --- | --- |
| `id` | UUID v7 |
| `businessId?` | Null only for platform-level (e.g. business password reset) |
| `type` | Email type enum (PA notification types) |
| `dedupeKey` | Unique idempotency key |
| `appointmentId?` | When appointment-related; tenant-aware FK |
| Recipient email | As sent |
| `status` | `PENDING` \| `SENDING` \| `SENT` \| `SKIPPED` \| `FAILED` |
| `skipReason?`, `attempts`, `claimedAt?` | Claim/retry |
| `providerMessageId?`, `sentAt?`, `lastError?` | Provider outcome |
| Timestamps | `createdAt`, `updatedAt` |

**Jobs:** BullMQ email job payloads are **not** rows in this table; payload stays identifier-only (PA). Delivery row is the idempotency/claim ledger.

---

## 9. Tenant isolation (conceptual)

| Entity | Tenant key | Notes |
| --- | --- | --- |
| Business | self | Root |
| User | — | Global |
| BusinessMember | `businessId` | |
| BusinessSession | via `activeBusinessId` / user | Not a “row owned by business” in the same sense; still scoped in queries |
| Staff, Service, StaffService, hours, TimeOff, ClosedDate | `businessId` | Composite FKs where child points at parent+business |
| Customer, CustomerAccount, CustomerSession | `businessId` | |
| Appointment, AppointmentManageToken | `businessId` | |
| BusinessInvitation | `businessId` | |
| NotificationDelivery | `businessId` nullable | Platform-only null path |
| Auth tokens (logical) | purpose-dependent | Must not cross tenants |

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
| Appointment | Status machine; `CANCELLED` terminal; not soft-deleted |
| Invitation | Pending → consumed/revoked |
| Sessions | Deleted/revoked on logout, reset, membership disable, etc. |
| Manage tokens | Revoked/expired; not soft-deleted rows required |

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
```

Logical **AuthToken** intents (verification / resets / email-change) attach to User or CustomerAccount/Customer subjects — physical tables OPEN (§7).

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
| StaffTimeOff | Business | Staff exclusion range | Staff | CRUD | Yes |
| BusinessClosedDate | Business | Full-day closure | Business | CRUD | Yes |
| Customer | Business | Customer record | Account, Appointment | Soft delete | Yes |
| CustomerAccount | Business | Customer login | Customer, CustomerSession | Active/disabled | Yes |
| CustomerSession | CustomerAccount | Customer realm session | CustomerAccount | Expiry/revoke | Yes |
| Appointment | Business | Booking aggregate | Customer, Staff, Service, manage tokens | Status machine | Yes |
| AppointmentManageToken | Business | Guest manage auth | Appointment | Expiry/revoke | Yes |
| BusinessInvitation | Business | Owner/Staff invite | Business, optional Staff | Pending/consumed/revoked | Yes |
| NotificationDelivery | Business or platform | Email idempotency ledger | optional Appointment | PENDING→… | Usually yes |
| AuthToken (logical) | Subject-dependent | Verify/reset/email-change | User or Customer* | Single-use/rotate | Purpose-dependent |

**Persisted entity count (concrete):** **18**  
**Logical security-token model:** **+1** (physical split OPEN) → treat inventory as **18 + AuthToken(logical)**.

---

## 13. API ↔ entity coverage (sanity)

| API area | Primary entities |
| --- | --- |
| Business auth / session | User, BusinessMember, BusinessSession |
| Invitations | BusinessInvitation, User, BusinessMember, Staff |
| Customer auth / account | Customer, CustomerAccount, CustomerSession, AuthToken(logical) |
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

1. **Auth/security token physical model:** single purpose-scoped `AuthToken` table vs separate tables for customer verification, customer password reset, customer email-change, and business password reset (domain purposes FINAL; table shape OPEN).
2. **Exact DB indexes** beyond known uniques/exclusion (e.g. calendar `(businessId, startsAt)`, CM search) — deferred to schema/implementation.
3. **Exact FK on-delete behavior** (RESTRICT vs CASCADE vs SET NULL) per relation — deferred; must not violate soft-delete/history rules.
4. **NotificationDelivery.recipient / skipReason / lastError** precise column constraints — deferred to schema.
5. **Whether `CustomerAccount.businessId` is stored vs derived-only** — recommendation store for isolation; confirm at schema time if not already treated as required above (listed as denormalized tenant key; treat as **preferred FINAL**, escalate only if implementers disagree).

Non-OPEN (already FINAL in PA / API): UUID v7, exclusion overlap rule, snapshot set, manage-token hashing + multi-active manage tokens, invitation model, session realm split, closed-date/all-day TimeOff semantics, booking reference format, StaffService replacement API semantics.

---

## 15. Consistency check results

| Check | Result |
| --- | --- |
| PA entity missing from model? | **No** (AuthToken physical split remains OPEN) |
| API resource without entity? | **No** |
| Tenant boundary violation? | **None identified** |
| Appointment snapshots complete? | **Yes** (no business snapshot, by design) |
| Customer vs CustomerAccount split? | **Yes** |
| BusinessMember vs Staff? | **Yes** (optional link, bookable Staff independent) |
| Appointment concurrency model? | **Yes** (`blockedUntil` + exclusion where status ≠ CANCELLED) |
| Lifecycle distinctions? | **Yes** (§10) |
| UUID v7? | **Yes** |
| Unnecessary entities? | **No** hard-delete tables, outbox, audit log, RLS policies (out of MVP) |

### Missing entity

**None** for MVP domain persistence (pending AuthToken physical choice).

### Suspicious relation

**None.** Optional Staff↔BusinessMember must remain optional for staff-without-login.

### Contradiction

**None** vs Product Architecture or API Contract inventory.
