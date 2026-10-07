# Schema Design — MVP Canonical Persistence Spec (Skeleton)

**Status:** Schema-design skeleton + inventory (no table columns yet)  
**Depends on:**  
- [`01-product-architecture.md`](./01-product-architecture.md)  
- [`02-api-contract.md`](./02-api-contract.md)  
- [`03-data-model.md`](./03-data-model.md)  

**This phase:** conventions, enum inventory, entity index, review order, schema OPENs.  
**Not in this phase:** per-table field designs, Prisma schema, SQL/migrations, indexes DDL, implementation.

---

## 1. Purpose

This document is the **canonical persistence-model source of truth** for the MVP schema, independent of:

- Prisma schema files  
- Migration history / SQL dialects  
- Application repository layout  

It records **what must be persisted and under which global rules**, so later Prisma/SQL work implements decisions already locked here (and in Product Architecture / Data Model), rather than inventing them during coding.

Field-level table designs will be added in later sections of this document (or subsequent revisions). This revision is intentionally a **skeleton + inventory only**.

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
| Business | § Business | Concrete |
| User | § User | Concrete |
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

Carried forward from [`03-data-model.md`](./03-data-model.md) §14 — **no new OPENs invented**:

1. **AuthToken physical model** — generic purpose-scoped table vs purpose-specific tables for customer password reset, customer email-change, business password reset (and optionally registration verify secret if not embedded on `CustomerVerificationIntent`). `CustomerVerificationIntent` remains concrete either way.  
2. **Exact DB indexes** beyond known uniques / exclusion constraint — deferred to detailed schema / implementation.  
3. **Exact FK on-delete behavior** (RESTRICT vs CASCADE vs SET NULL) — deferred; must preserve soft-delete and history rules.  
4. **NotificationDelivery** precise column constraints (`recipient`, `skipReason`, `lastError`, lengths, etc.) — deferred to detailed schema.

**Schema OPEN count:** **4**

---

## 7. Out of scope for this skeleton

- Per-entity column lists and nullability matrices  
- Prisma models / `@db` attributes  
- Migration SQL / exclusion constraint DDL text  
- Seed data  

Those belong in subsequent schema-design passes using the review order above.

---

## 8. Skeleton consistency report

| Metric | Value |
| --- | --- |
| Enum inventory rows | **13** |
| Concrete entities | **19** |
| Logical abstractions | **1** (AuthToken) |
| Schema OPEN count | **4** |
| Missing entity vs Data Model | **None** |
| Contradiction vs PA / API / Data Model | **None** |

This document is ready for the next pass: field-level design starting with **Business**.
