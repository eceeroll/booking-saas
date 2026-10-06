# API Contract — MVP Endpoint Inventory

**Status:** Inventory / boundary draft (not an implementation spec)  
**Depends on:** [`01-product-architecture.md`](./01-product-architecture.md) (FINAL, §23 = 0 OPEN)  
**This phase:** endpoint inventory and API surface boundaries only.  
**Not in this phase:** detailed request/response schemas, field validation tables, HTTP status catalogs, success/error envelopes.

Suggested naming below is a **convention proposal**. Items marked **OPEN** need an explicit naming/boundary decision before schema work.

---

## 0. Document rules

- Product Architecture remains the product/architecture source of truth.
- This inventory must not invent capabilities absent from Product Architecture.
- Operator CLI (business provision, Owner invite send, offboarding) is **out of HTTP API scope** unless noted.
- Exact guest manage-token transport (URL fragment vs body vs query) is **not finalized here**; Product Architecture requires high-entropy hashed tokens and fragment-oriented delivery for email links. Request binding details → later contract phase / Validation doc.

---

## 1. API surfaces

| Surface | Audience | Tenant resolution | Cookie / auth |
| --- | --- | --- | --- |
| **Public API** | Guests; optional Customer session for booking UX | Business **slug** | None required for guest book/manage; Customer cookie optional |
| **Customer API** | Tenant-scoped CustomerAccount | Slug + Customer session (`customerAccountId`, `businessId`) | Customer HTTP-only session cookie |
| **Business API** | Owner / Staff | Authenticated membership / `activeBusinessId` (never client `businessId`) | Business HTTP-only session cookie |

### Auth endpoint placement

| Concern | Surface |
| --- | --- |
| Business user login/logout/session/password-reset | **Auth (business realm)** under `/api/auth/business/*` |
| BusinessInvitation inspect/accept (Owner + Staff) | **Auth (invitation)** under `/api/auth/invitations/*` (unauthenticated token holder) |
| Staff invitation create/revoke | **Business API** Owner-only |
| Customer register/verify/login/logout/session/password-reset | **Customer auth** under `/api/public/:slug/auth/*` (tenant via slug) |
| Customer profile / own appointments | **Customer API** under `/api/public/:slug/account/*` (or `/api/public/:slug/me/*` — naming OPEN) |

Suggested top-level prefixes:

```text
/api/auth/*                 — business realm + invitation accept
/api/public/:slug/*         — public booking + customer realm (slug-tenant)
/api/business/*             — Owner/Staff panel
```

---

## 2. Global tenant isolation (API)

1. Business API tenant comes only from the authenticated membership/session. Client **must not** select tenant via `businessId`.
2. Public API tenant comes from `:slug`.
3. Customer API tenant comes from slug **and** customer session; session `businessId` must match slug-resolved business.
4. Cross-tenant or unauthorized resource id → **404** (no existence leak).
5. Wrong role on Owner-only surfaces (e.g. Customer Management) → **403** for authenticated Staff (Product Architecture).
6. Worker/job payloads’ tenant ids are not client-trustable; API handlers still resolve authorization from session/slug.
7. All list/detail/mutate queries are tenant-scoped.

---

## 3. Auth & sessions inventory

### 3.1 Business user (platform user + membership)

| Method | Path (proposed) | Purpose | Auth |
| --- | --- | --- | --- |
| `POST` | `/api/auth/business/login` | Login; rotate session | Public |
| `POST` | `/api/auth/business/logout` | Destroy business session | Business session |
| `GET` | `/api/auth/business/session` | Current user, role, `activeBusinessId`, basic business summary | Business session |
| `POST` | `/api/auth/business/password-reset/request` | Uniform response; enqueue reset email | Public (rate-limited) |
| `POST` | `/api/auth/business/password-reset/confirm` | Consume reset token; set password; revoke sessions | Public (token) |

**Count: 5**

### 3.2 BusinessInvitation (Owner + Staff; same model)

Owner invitations are **created by operator CLI**, not by Business API. Staff invitations are created by Owner via Business API. Both use `BusinessInvitation`.

| Method | Path (proposed) | Purpose | Auth |
| --- | --- | --- | --- |
| `GET` | `/api/auth/invitations/:invitationPublicId` | Inspect invite (business name, role, email hint, expiry) without leaking secrets | Token/knowledge of id — **OPEN:** path vs token-only lookup |
| `POST` | `/api/auth/invitations/accept` | Accept invite; create User/Member/(Staff link); set password; rotate session | Invitation token in body (**OPEN:** exact token field binding) |
| `POST` | `/api/business/invitations` | Create/re-issue **Staff** invitation (invalidates previous) | Owner |
| `POST` | `/api/business/invitations/:id/revoke` | Revoke pending invitation | Owner |
| `GET` | `/api/business/invitations` | List pending invitations (ops UX) | Owner |

**Count: 5** (CLI Owner-invite send is not an HTTP endpoint)

### 3.3 Customer (business-scoped)

All paths are slug-tenanted. Login identity = `(businessId, normalizedEmail)` among non-deleted customers.

| Method | Path (proposed) | Purpose | Auth |
| --- | --- | --- | --- |
| `POST` | `/api/public/:slug/auth/register` | Start registration (pending token; no Customer/Account yet); uniform response | Public |
| `POST` | `/api/public/:slug/auth/verify-email` | Complete verification; attach/create Customer + CustomerAccount; session | Token |
| `POST` | `/api/public/:slug/auth/verification/resend` | Resend verification (rotate token) | Public (rate-limited) |
| `POST` | `/api/public/:slug/auth/login` | Customer login; rotate session | Public |
| `POST` | `/api/public/:slug/auth/logout` | Destroy customer session | Customer session |
| `GET` | `/api/public/:slug/auth/session` | Current customer account summary | Customer session |
| `POST` | `/api/public/:slug/auth/password-reset/request` | Uniform response | Public |
| `POST` | `/api/public/:slug/auth/password-reset/confirm` | Consume token; revoke sessions | Token |

**Count: 8**

---

## 4. Public business & booking

Guest booking requires no session. Customer session may be present; booking still uses Public booking rules when created on this surface (`source = PUBLIC`).

| Method | Path (proposed) | Purpose | Auth |
| --- | --- | --- | --- |
| `GET` | `/api/public/:slug` | Public business profile (ACTIVE full; published+INACTIVE limited message; unpublished/unknown → 404) | Public |
| `GET` | `/api/public/:slug/availability` | Server-generated available slots only | Public |
| `POST` | `/api/public/:slug/appointments` | Create public booking (`CONFIRMED`); guest or logged-in customer | Public (+ optional Customer session) |
| `GET` | `/api/public/:slug/manage/appointments/:appointmentId` | Guest manage view for one appointment | Manage token (**transport OPEN**) |
| `POST` | `/api/public/:slug/manage/appointments/:appointmentId/cancel` | Guest cancel | Manage token |
| `POST` | `/api/public/:slug/manage/appointments/:appointmentId/reschedule` | Guest reschedule (grid + notices) | Manage token |
| `GET` | `/api/public/:slug/manage/appointments/:appointmentId/alternatives` | Nearby alternatives after conflict / for reschedule UX | Manage token |

**Count: 7**

### Availability contract responsibilities (no schema yet)

- Input: business slug, service, staff selection (specific staff **or** Fark etmez), business-local date(s) as needed.
- Server computes slots in business IANA timezone (Luxon module); respects hours ∩ staff hours − closed dates − time off − blocking appointments − buffer; DST rules per Product Architecture.
- Output includes UTC instants + local display labels + offsets; **frontend must not recompute timezone/availability math**.
- Inactive/unpublished: 409 `BUSINESS_INACTIVE` or 404 per Product Architecture (not empty slot lists).

### Public vs Customer-authenticated surfaces

| Capability | Guest | Customer session |
| --- | --- | --- |
| Public profile / availability / create booking | Yes | Yes (same public booking endpoints; account link via session when present) |
| Appointment history / profile edit / email change | No | Customer API |
| Guest manage cancel/reschedule | Manage token | Prefer account appointment mutations; manage token remains valid per PA |

---

## 5. Customer account & appointments

| Method | Path (proposed) | Purpose | Auth |
| --- | --- | --- | --- |
| `GET` | `/api/public/:slug/account` | Profile: name, phone, email, account status | Customer |
| `PATCH` | `/api/public/:slug/account` | Update name/phone only | Customer |
| `POST` | `/api/public/:slug/account/email-change/request` | Start email change (verify new; notice old) | Customer |
| `POST` | `/api/public/:slug/account/email-change/confirm` | Confirm new email; revoke other sessions; rotate | Token + Customer |
| `GET` | `/api/public/:slug/account/appointments` | Own appointments list | Customer |
| `GET` | `/api/public/:slug/account/appointments/:id` | Own appointment detail | Customer |
| `POST` | `/api/public/:slug/account/appointments/:id/cancel` | Cancel own (notice rules) | Customer |
| `POST` | `/api/public/:slug/account/appointments/:id/reschedule` | Reschedule own (grid + window/notice) | Customer |
| `GET` | `/api/public/:slug/account/appointments/:id/alternatives` | Alternatives for customer reschedule / conflict | Customer |

**Count: 9**

Customers cannot access another business’s customer data (slug + session mismatch → 404).

---

## 6. Business settings & lifecycle

References Product Architecture §6 (INACTIVE/ACTIVE, `publishedAt`, checklist, immutability after publish).

| Method | Path (proposed) | Purpose | Auth |
| --- | --- | --- | --- |
| `GET` | `/api/business/settings` | Full settings + status + checklist-derived readiness | Owner |
| `PATCH` | `/api/business/settings` | Update mutable settings (slug/timezone/currency rules enforced) | Owner |
| `POST` | `/api/business/activate` | Activate/publish or reactivate (checklist); conditional status | Owner |
| `POST` | `/api/business/deactivate` | Deactivate (public booking off) | Owner |
| `GET` | `/api/business/public-preview` | Authenticated Owner preview payload for public booking UI while inactive/unpublished | Owner |

**Count: 5**

---

## 7. Services

No hard-delete endpoint. Active flag via dedicated lifecycle actions (single convention; no parallel `mark-active` aliases).

| Method | Path (proposed) | Purpose | Auth |
| --- | --- | --- | --- |
| `GET` | `/api/business/services` | List | Owner (Staff read **OPEN** — PA implies Owner manages; Staff may need read for booking UX → prefer allow Staff read-only) |
| `GET` | `/api/business/services/:id` | Detail | Owner (+ Staff read recommended) |
| `POST` | `/api/business/services` | Create | Owner |
| `PATCH` | `/api/business/services/:id` | Update fields (not a substitute for activate/deactivate if those are explicit) | Owner |
| `POST` | `/api/business/services/:id/activate` | Set active | Owner |
| `POST` | `/api/business/services/:id/deactivate` | Set inactive | Owner |

**Count: 6** — Staff read access: **OPEN** (recommendation: Staff `GET` allowed; mutate Owner-only).

---

## 8. Staff

| Method | Path (proposed) | Purpose | Auth |
| --- | --- | --- | --- |
| `GET` | `/api/business/staff` | List staff | Owner; Staff may see self only — **OPEN** list shape for Staff |
| `GET` | `/api/business/staff/:id` | Detail | Owner; Staff own id only |
| `POST` | `/api/business/staff` | Create bookable Staff (login optional until invited) | Owner |
| `PATCH` | `/api/business/staff/:id` | Update display name / links / etc. | Owner |
| `POST` | `/api/business/staff/:id/activate` | Activate | Owner |
| `POST` | `/api/business/staff/:id/deactivate` | Deactivate (blocked if future CONFIRMED) | Owner |
| `PUT` | `/api/business/staff/:id/services` | Replace Staff–Service set (unlink preserves appointments) | Owner |

Invitations: see §3.2 (`POST/GET/revoke` under `/api/business/invitations`).

**Count: 7** (+ invitations already counted)

Authorization: Owner manages all; Staff cannot manage schedules/staff records (PA). Staff deactivate of “self” as staff profile follows PA (Owner action).

---

## 9. Working hours / TimeOff / closed dates

API must preserve business-local dates, all-day TimeOff = `[local 00:00, next local 00:00)`, closed dates, DST, and business timezone rules from Product Architecture. No browser-TZ inputs.

| Method | Path (proposed) | Purpose | Auth |
| --- | --- | --- | --- |
| `GET` | `/api/business/working-hours` | Business weekly hours | Owner |
| `PUT` | `/api/business/working-hours` | Replace full weekly schedule | Owner |
| `GET` | `/api/business/staff/:id/working-hours` | Staff weekly hours | Owner |
| `PUT` | `/api/business/staff/:id/working-hours` | Replace staff weekly hours | Owner |
| `GET` | `/api/business/staff/:id/time-off` | List TimeOff | Owner |
| `POST` | `/api/business/staff/:id/time-off` | Create (instant range or all-day local date) | Owner |
| `PATCH` | `/api/business/staff/:id/time-off/:timeOffId` | Update | Owner |
| `DELETE` | `/api/business/staff/:id/time-off/:timeOffId` | Delete | Owner |
| `GET` | `/api/business/closed-dates` | List business closed dates | Owner |
| `POST` | `/api/business/closed-dates` | Create full-day closure (`YYYY-MM-DD` local) | Owner |
| `DELETE` | `/api/business/closed-dates/:id` | Remove closure | Owner |

**Count: 11**

Staff cannot update own hours/TimeOff (Owner-only).

---

## 10. Business availability (manual booking)

Public availability is §4. Admin manual booking needs the same engine without relying on the public slug surface.

| Method | Path (proposed) | Purpose | Auth |
| --- | --- | --- | --- |
| `GET` | `/api/business/availability` | Slots for service/staff/date for manual booking; Owner may request override-aware previews **OPEN** whether override is query flag or only enforced at create | Owner, Staff (own staff only) |

**Count: 1**

---

## 11. Appointments / calendar

Single action convention for lifecycle (no `mark-completed` / `complete` duplicates):

```text
POST /api/business/appointments/:id/cancel
POST /api/business/appointments/:id/reschedule
POST /api/business/appointments/:id/complete
POST /api/business/appointments/:id/no-show
```

`complete` when status is `NO_SHOW` (and vice versa for `no-show`) performs the allowed COMPLETED ↔ NO_SHOW correction. When status is `CONFIRMED` and `now >= startsAt`, same endpoints perform the initial transition. No separate “correct” endpoints.

| Method | Path (proposed) | Purpose | Auth |
| --- | --- | --- | --- |
| `GET` | `/api/business/appointments` | Calendar/range list (`from`/`to` local dates, optional `staffId`); max 500 + `truncated` | Owner (all/filter); Staff forced to own staff |
| `GET` | `/api/business/appointments/:id` | Detail drawer payload | Owner any; Staff own |
| `POST` | `/api/business/appointments` | Manual create (`source` OWNER/STAFF; optional `overrideHours` Owner-only; customerId or create-inline) | Owner, Staff |
| `POST` | `/api/business/appointments/:id/cancel` | Cancel before start | Owner, Staff (own) |
| `POST` | `/api/business/appointments/:id/reschedule` | In-place reschedule | Owner, Staff (own; no staff change) |
| `POST` | `/api/business/appointments/:id/complete` | CONFIRMED→COMPLETED or NO_SHOW→COMPLETED | Owner, Staff (own) |
| `POST` | `/api/business/appointments/:id/no-show` | CONFIRMED→NO_SHOW or COMPLETED→NO_SHOW | Owner, Staff (own) |
| `GET` | `/api/business/appointments/:id/alternatives` | Same-staff 3 + other-staff ≤3 / Fark etmez 6; same day + 7 days; max window (PA §11.8) | Owner, Staff (own) |

**Count: 8**

Booking-context customer create/link: supported on `POST /api/business/appointments` (and optionally reuse Customer create). This is **not** Customer Management authorization; Staff may create/link here only.

---

## 12. Customer Management (Owner-only)

Staff → **403** on these routes.

| Method | Path (proposed) | Purpose | Auth |
| --- | --- | --- | --- |
| `GET` | `/api/business/customers` | List/search/filter/sort/paginate (`q`, `deleted`, …) | Owner |
| `POST` | `/api/business/customers` | Manual create | Owner |
| `GET` | `/api/business/customers/:id` | Detail | Owner |
| `PATCH` | `/api/business/customers/:id` | Edit name/phone; set email only if null | Owner |
| `DELETE` | `/api/business/customers/:id` | Soft-delete (PA guards) | Owner |
| `GET` | `/api/business/customers/:id/appointments` | Appointment history | Owner |

**Count: 6**

---

## 13. Dashboard

| Method | Path (proposed) | Purpose | Auth |
| --- | --- | --- | --- |
| `GET` | `/api/business/dashboard` | `todayCount`, `unmarkedCount`, `nextAppointment`, `today[]`, `unmarked[]`, `businessStatus`, checklist/warnings (PA §16) | Owner (business-wide); Staff (own staff scope **server-side only** — client cannot widen via `staffId`) |

**Count: 1**

---

## 14. Business user profile / account self-service

| Method | Path (proposed) | Purpose | Status |
| --- | --- | --- | --- |
| `GET` | `/api/auth/business/session` | Identity + membership (already §3.1) | Covered |
| `PATCH` | `/api/auth/business/me` | Change display name / password while logged in | **OPEN** — Product Architecture requires password reset flow but does not fully specify logged-in profile/password/email change for business users |
| Business-user email change | — | — | **OPEN** / likely out of MVP unless added to PA first |

**New endpoint count toward inventory:** 0 FINAL + 1 OPEN candidate (not counted in totals until decided).

---

## 15. Explicitly non-HTTP (CLI / workers)

| Capability | Surface |
| --- | --- |
| Create business + Owner invitation | Operator CLI |
| Operator offboarding | Operator CLI / runbook |
| Reminder / email workers | BullMQ workers (not public API) |
| Demo tenant provision/reset | Deferred technical/ops docs |

---

## 16. Authorization matrix (API)

| Surface | Guest | Customer | Staff | Owner |
| --- | --- | --- | --- | --- |
| Public business profile | ✅ | ✅ | ✅¹ | ✅¹ |
| Public availability | ✅ | ✅ | ✅¹ | ✅¹ |
| Public booking create | ✅ | ✅ | ❌² | ❌² |
| Guest manage (token) | ✅ | ✅³ | ❌ | ❌ |
| Own customer account / appointments | ❌ | ✅ | ❌ | ❌ |
| Business dashboard / calendar / manual booking | ❌ | ❌ | ✅ | ✅ |
| Business availability (admin) | ❌ | ❌ | ✅ (own) | ✅ |
| Customer Management | ❌ | ❌ | ❌ (403) | ✅ |
| Staff / services mutate / hours / closed dates | ❌ | ❌ | ❌ | ✅ |
| Services/staff **read** for booking UX | ❌ | ❌ | ✅ (recommended) | ✅ |
| Business settings / activate / deactivate / preview | ❌ | ❌ | ❌ | ✅ |
| Staff invitation create/revoke | ❌ | ❌ | ❌ | ✅ |

¹ May hit public URLs while browsing; not a substitute for Business API.  
² Manual booking uses Business API (`POST /api/business/appointments`), not public booking.  
³ Prefer account appointments; manage token still valid per PA.

---

## 17. Pagination / envelopes / errors (placeholders)

| Topic | Decision in this phase |
| --- | --- |
| Wire format | JSON |
| Pagination convention | **Next phase** |
| Success envelope | **Next phase** |
| Error envelope | Deferred to `04 Validation & Errors` |
| Validation rules | Deferred to `04 Validation & Errors` |
| HTTP status mapping | Deferred to `04 Validation & Errors` (PA already fixes some: 404 cross-tenant, 403 CM for Staff, 409 inactive/conflicts) |

---

## 18. Inventory review

### Counts by area

| Area | Endpoint count | Status | Notes |
| --- | ---: | --- | --- |
| Auth (business) | 5 | Draft inventory | |
| Auth (invitations HTTP) | 5 | Draft inventory | Owner invite **create** is CLI |
| Auth (customer) | 8 | Draft inventory | Includes resend verification |
| Public booking / manage | 7 | Draft inventory | Token transport OPEN |
| Customer account / appointments | 9 | Draft inventory | |
| Business settings / lifecycle | 5 | Draft inventory | Includes public-preview |
| Services | 6 | Draft inventory | Staff GET access OPEN |
| Staff | 7 | Draft inventory | + invitations in Auth/Business |
| Scheduling (hours/TimeOff/closed) | 11 | Draft inventory | |
| Availability (business) | 1 | Draft inventory | Public availability in Public |
| Appointments / calendar | 8 | Draft inventory | Single complete/no-show convention |
| Customers (CM) | 6 | Draft inventory | Owner-only |
| Dashboard | 1 | Draft inventory | |
| **Total (proposed inventory)** | **79** | | OPEN candidates not double-counted |

### OPEN

1. **Guest manage-token transport** for manage endpoints (fragment is for email links; how API receives the secret — header vs body — not FINAL).
2. **Invitation inspect path** (`:invitationPublicId` vs opaque token-only POST inspect).
3. **Invitation accept token binding** (body field naming only; product rules FINAL).
4. **Customer account path prefix**: `/account` vs `/me` vs `/customer`.
5. **Staff read access** to `GET /api/business/services` and staff list shape for Staff role.
6. **Business availability + override**: whether `overrideHours` preview is a query flag on `GET .../availability` or only applied at `POST .../appointments`.
7. **Business user logged-in profile/password (and email) change** endpoints — not fully specified in Product Architecture → do not treat as FINAL inventory items until PA or an explicit decision exists.

### Missing capabilities (vs Product Architecture)

| Capability | Notes |
| --- | --- |
| None identified as missing for MVP HTTP surface | CLI covers provision/Owner-invite/offboarding; workers cover email/reminders |
| Optional UX: slug availability check before save | Not required by PA; omit unless product asks |

### Duplicate / suspicious

| Item | Resolution |
| --- | --- |
| Separate `mark-completed` vs `complete` | **Rejected** — single `.../complete` and `.../no-show` |
| Public booking vs manual booking | **Intentional dual endpoints** (different auth, rules, `source`) |
| Customer cancel/reschedule vs guest manage cancel/reschedule | **Intentional dual** (session vs manage token) |
| `PATCH` service `active` + `POST .../activate` | Prefer **explicit activate/deactivate** only; avoid also toggling via ambiguous PATCH — implementers should not expose both styles |
| Customer CM `POST` vs booking-context customer create | Same resource create allowed under different auth paths is OK; avoid a third “quick-create” alias |

### Contradictions

**None** relative to Product Architecture FINAL decisions.

Known alignment checks:

- Tenant via session/slug; no client `businessId` trust.
- Staff CM → 403; cross-tenant → 404.
- No hard delete for services/staff/customers (soft/deactivate only).
- Dashboard Staff scope server-enforced.
- CLOSED dates / all-day TimeOff / Luxon TZ semantics preserved at contract responsibility level.
- Public vs INACTIVE response behavior referenced, not redefined.

---

## 19. Next phase (out of scope now)

- Per-endpoint request/response JSON Schema / TypeScript types  
- Idempotency keys, rate-limit headers  
- Exact cookie names and CSRF header names  
- OpenAPI file generation  
- Validation & error document (`04`)
