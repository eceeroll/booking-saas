Status: Living document · Last updated: 2026-10-06

Decision states:

- **FINAL**: locked. Change only by updating this document first.
- **OPEN**: not decided. Must not be assumed or silently resolved during implementation.

---

## 1. Document purpose

- Single source of truth for product and architecture decisions finalized before implementation.
- Audience: Cursor Agent (implementation), human reviewers, portfolio readers.
- Usage rules:
  - Sections 3–22 are **FINAL**.
  - Section 23 is **OPEN**. If a task depends on an open decision, raise it, decide it explicitly, and record the outcome here before implementing.
  - Any change to a FINAL decision must be made in this document before it is made in code.
- This document describes **what the product does** and **which architecture principles are locked**. It is not an API reference, Prisma schema, endpoint catalog, test suite, or deployment runbook.

---

## 2. Product summary

- **Working name:** White-Label Booking Platform
- **Positioning:** Reusable multi-tenant white-label booking infrastructure for freelance/custom implementations. Not positioned initially as a cheap generic SaaS subscription.
- **Primary target:** Appointment-based service businesses with 3–15 employees (psychology/consulting, dietitian, physiotherapy, beauty, Pilates/yoga, etc.).
- **Secondary target:** Web agencies that need reusable white-label booking infrastructure.
- **Core problem:** Manual appointment booking and management.
- **Core value proposition:** "İşletmeler, kendi markaları altında müşterilerinin 7/24 online randevu almasını sağlayabilir ve tüm randevu sürecini tek bir panelden yönetebilir."

---

## 3. Product scope and positioning — FINAL

### 3.1 MVP includes

- Multi-tenancy
- Owner and Staff roles
- Optional Customer accounts
- Guest booking
- Business, Services, Staff, Staff–Service
- Business working hours (multiple intervals per day)
- Staff working hours
- Owner-managed time off
- Availability engine
- Public booking (`/book/{business-slug}`)
- Database-level double-booking protection
- Concurrency tests
- Signed guest cancel/reschedule (manage) links
- Admin calendar (Day + Week)
- Manual booking (including Owner override)
- Appointment statuses and lifecycle
- Customer Management
- Dashboard
- Transactional emails + 24h reminders (BullMQ)
- Rate limiting
- Demo tenant
- CI (GitHub Actions)
- Production deployment (Docker, single VM)
- White-label branding (public page + emails)

### 3.2 Explicitly out of scope

See §22. Do not implement those items in the MVP sprint.

---

## 4. Roles and actors — FINAL

### 4.1 Owner

- Sees all business data and all appointments.
- Manages business settings, branding, activation/deactivation, services, staff, schedules, time off, **business closed dates**, customers.
- Manual booking for any staff; may override hours/time-off/closed-day (including business closed dates) rules with `overrideHours`.
- Cancel/reschedule any CONFIRMED appointment before start; Complete / No-show at or after start; correct COMPLETED ↔ NO_SHOW.
- Customer Management full access.
- Dashboard: business-wide.

### 4.2 Staff

- Sees and manages **only own** appointments.
- Manual booking for **own** staff profile only; **cannot** override hours.
- Cancel/reschedule own CONFIRMED before start; Complete / No-show at or after start; correct COMPLETED ↔ NO_SHOW for own appointments.
- In booking context: search/create/link customers.
- **Cannot** access Customer Management, business settings, activation, or other staff’s data.
- Dashboard: own-staff scope only.
- Cannot manage own working hours or time off (Owner manages).

### 4.3 Customer (account or guest)

- Guest booking without session.
- Optional tenant-scoped CustomerAccount.
- Sees/manages only own appointments.
- Cancel/reschedule before start subject to customer notice / max window / grid rules.
- Never sets COMPLETED or NO_SHOW; no actions after start.
- Profile: edit name and phone; email change via verification flow only.
- Guest manage token grants cancel/reschedule for that appointment without login.

### 4.4 Operator

- Provisions businesses via server-side CLI only.
- No public business signup; no web super-admin in MVP.

### 4.5 Authorization summary

| Action | OWNER | STAFF | CUSTOMER |
| --- | --- | --- | --- |
| Dashboard (scoped) | business-wide | own staff | — |
| Calendar | all / filter staff | own | — |
| Manual booking | any staff + override | own staff, no override | — |
| Customer Management | yes | no (403) | — |
| Create/link customer in booking | yes | yes | — |
| Business settings / activate | yes | no | — |
| Own appointments cancel/reschedule | any | own | own (+ notice) |
| Complete / No-show | any | own | no |

Cross-tenant or unauthorized resource access returns **404** (no existence leak), except wrong-role access to Owner-only surfaces (e.g. Customer Management) which returns **403** for authenticated Staff.

---

## 5. Business and onboarding — FINAL

### 5.1 Creation

- Operator-only CLI: `name`, `slug`, `timezone`, `currency`, `ownerEmail`.
- Business creation is an application service callable by the CLI (reusable for future signup).
- Business starts **`INACTIVE`**, `publishedAt = null`.
- Booking-rule fields are written with defaults at creation (§6.3).
- Owner receives `BusinessInvitation` (role OWNER). No temporary password.
- On accept: one transaction creates User + OWNER `BusinessMember`, Owner sets password, session rotated.
- MVP: exactly one Owner (or one pending owner invite). Schema may allow multiple Owners later; no ownership transfer in MVP.
- User is platform-global (globally unique email). `BusinessMember` links User↔Business with role `OWNER` | `STAFF`. Schema supports multiple memberships; MVP practice: one active membership.

### 5.2 Staff invitation and bookable Owner

- Generalized `BusinessInvitation` (role + optional `staffId`).
- Staff entity can exist without login; panel access is optional via invite.
- Invite: high-entropy, hashed, single-use, time-limited; bound to business, staff, email, role. Re-invite invalidates previous; Owner can revoke.
- Acceptance: one transaction creates User + BusinessMember + Staff link; session rotated.
- Owner may also have a bookable Staff profile (0 or 1). STAFF membership has exactly one Staff profile.
- Staff deactivation: blocked if future CONFIRMED appointments exist; removes from booking/assignment; disables membership; deletes sessions; revokes pending invites in same transaction; history retained; reactivation allowed (new login); deactivating Owner’s own staff profile does not touch owner membership.

### 5.3 Go-live checklist

Activation requires all of:

1. ≥1 business working-hours interval  
2. ≥1 active service  
3. ≥1 active Staff  
4. That staff offers ≥1 active service  
5. That staff has working hours  

Contact fields and branding are **not** required for activation.

Owner authenticated **preview** of the public booking UI is allowed while inactive/unpublished; public slug behavior is unchanged (§6.5).

---

## 6. Business lifecycle and settings — FINAL

### 6.1 Status model

- `status`: `INACTIVE` | `ACTIVE`
- `publishedAt`: null until first successful activation; set once; **never cleared**
- Create → INACTIVE (`publishedAt` null)
- First activate → ACTIVE (`publishedAt = now`) after checklist
- Deactivate → INACTIVE (`publishedAt` kept)
- Reactivate → ACTIVE; checklist re-run; `publishedAt` unchanged

“Never published” = `publishedAt IS NULL`. No separate UNPUBLISHED/DEACTIVATED enums.

### 6.2 Settings fields

| Field | Notes |
| --- | --- |
| `name` | required, 1–80, trimmed |
| `slug` | required, global unique |
| `timezone` | IANA, validated/canonical |
| `currency` | ISO 4217, uppercase |
| `phone` | nullable, E.164 when set; parse with default country `TR` if no calling code (§9.1) |
| `email` | nullable, lowercase |
| `address` | nullable, plain text ≤300 |
| `logoUrl` | nullable, absolute HTTPS ≤2048 |
| `primaryColor` | nullable, `#` + 6 hex, stored lowercase |
| `slotIntervalMinutes` | 5 \| 10 \| 15 \| 20 \| 30 \| 60; default **15** |
| `minBookingNoticeMinutes` | ≥0; default **60**; elapsed duration |
| `maxBookingWindowDays` | 1–365; default **60**; local calendar days |
| `cancellationNoticeMinutes` | ≥0; default **1440**; elapsed duration; also gates customer reschedule eligibility |
| `status` | INACTIVE \| ACTIVE |
| `publishedAt` | nullable timestamptz(3) |
| `createdAt` / `updatedAt` | timestamptz(3) |

No `rescheduleNotice` field. No `deactivatedAt`. No business locale in MVP.

### 6.3 Mutability after first publish

| Freely editable | Immutable after `publishedAt` set |
| --- | --- |
| name, phone, email, address, logo, primaryColor | timezone |
| booking rules, hours, services, staff | currency |
| | slug |

Before first publish, timezone, currency, and slug remain Owner-editable.

### 6.4 Slug

- Global unique; never reused; reserved on create (inactive holds slug).
- 2–48 chars; `^[a-z0-9]+(?:-[a-z0-9]+)*$`
- Turkish transliteration on suggest: ç→c, ğ→g, ı/İ→i, ö→o, ş→s, ü→u
- Collision → reject (no silent `-2` suffix); alternatives are UX-only suggestions
- Reserved words: central config list (e.g. admin, api, login, book, app, settings, static, assets, health, internal, support, help, …). Full list OPEN (§23).
- Mutable only while `publishedAt IS NULL`; immutable after. No redirect table in MVP.
- Manage/token links do not depend on slug.

### 6.5 Public profile visibility

| State | Public `/book/{slug}` |
| --- | --- |
| Unknown slug | 404 |
| `publishedAt` null | 404 |
| Published + INACTIVE | 200 limited DTO + message: “Bu işletme şu anda online randevu kabul etmiyor.” No booking UI |
| ACTIVE | Full booking profile |

Public ACTIVE profile shows: name, logo?, primaryColor application, address/phone/email if set, active services (name, duration, price), bookable staff `displayName`s, timezone label for display. No internal IDs as authorization; opaque ids for selection only. No slotInterval or admin-only settings dump.

Availability / create booking when inactive or unpublished: **409 `BUSINESS_INACTIVE`** (if published inactive) or **404** (unknown/unpublished). Do not return empty slots.

### 6.6 Deactivation behavior

Deactivation closes **new public booking only**. Remains available: admin dashboard, calendar, Owner/Staff manual booking, customer manage cancel/reschedule, reminders and appointment emails, historical data. Persistent admin banner: “Online randevu kapalı. Yalnızca panelden manuel randevu oluşturabilirsiniz.” Owner gets activation/settings CTA; Staff sees banner without CTA.

### 6.7 Branding (business)

- Logo: HTTPS URL only; no upload/object storage; no userinfo; reject localhost/private hosts best-effort; display via `<img>` (no inline SVG).
- Primary color: `#RRGGBB` only; invalid → 400; null → platform default. Used for CTA, selected/active states, focus accent, links — not full CSS theme override.
- Public/admin/email must escape business name and user-controlled text.

### 6.8 Settings concurrency

- Settings updates: last-write-wins; no Business `version` in MVP.
- Activate/deactivate: conditional UPDATE on expected `status`; 0 rows → 409.
- Public booking checks `status = ACTIVE` inside the booking transaction.

---

## 7. Multi-tenancy — FINAL

- One shared PostgreSQL database; shared tables; `businessId` on tenant-owned rows.
- Not used: database-per-tenant, schema-per-tenant.
- Isolation: application-level via tenant-scoped Prisma access.
- Tenant ID from trusted server context only:
  - Admin: authenticated membership/session (`activeBusinessId` revalidated every request).
  - Public: business slug.
  - Client-provided `businessId` never trusted.
- RLS not in the sprint; deferred until before first real client. Tables must remain RLS-compatible.
- Tenant-aware composite FKs where appropriate; tenant-scoped unique constraints.
- No casual `$queryRaw` that bypasses isolation.
- Cross-tenant integration tests required (business + customer realms).
- Other tenants’ records → **404**.
- Scoped client must cover findUnique/First/Many, update, delete, upsert, count, and other tenant-sensitive ops.
- Workers, cache keys, and rate-limit keys carry tenant context where relevant.
- `NotificationDelivery.businessId` may be null only for platform-level emails (e.g. business password reset); system worker only.

---

## 8. Authentication and sessions — FINAL

### 8.1 Business realm

- HTTP-only cookie; server-side sessions in PostgreSQL.
- Separate session table/cookie from customers.
- Secure cookie; `SameSite=Lax`; Origin check / CSRF on mutating requests.
- Session holds identity + `activeBusinessId`, not authorization; membership and role loaded from DB every request.
- Argon2id passwords; session tokens hashed at rest; idle + absolute expiry; session ID rotated on login.
- Login and password reset rate-limited.
- Business password reset supported (email via queue).

### 8.2 Customer realm

- Tenant-scoped `CustomerAccount` 1:1 with `Customer` (password hash, verifiedAt, status); login identity `(businessId, Customer.normalizedEmail)` among non-deleted.
- Separate customer session table/cookie; session holds `customerAccountId` + `businessId`, checked against slug.
- Single customer cookie; logging into another business replaces it.
- Same security principles as business realm; revalidate account/business/deleted state every request.

### 8.3 Guest manage tokens

- High-entropy, purpose-scoped, hashed at rest; appointment-specific.
- Short-lived (exact durations deferred to technical documents).
- Valid after account creation until expiry, cancel, reschedule (new token emailed), appointment no longer manageable, or customer deletion.
- Token in URL fragment `#token=...`; GET does not consume; frontend POST consumes.

### 8.4 Invitations and other tokens

- Verification, password reset, email-change, invitation intents: hashed, single-use/purpose-scoped; one active intent per purpose/subject; new send rotates and invalidates previous.
- Raw secret minted in email worker memory only — never Redis/DB/logs (see §17).

---

## 9. Customers and customer accounts — FINAL

### 9.1 Customer record

- Business-scoped: name, email / `normalizedEmail`, phone (E.164 searchable), soft-delete.
- Matching: **normalized email only** (trim + lowercase; no dot or `+tag` stripping), within business, non-deleted only. Deterministic. Phone never auto-links. Email wins over phone; no merge.
- Phone parsing (`libphonenumber-js`): if the input has no country calling code, default country is **`TR`**. If a country code is present (`+49`, `+44`, `+1`, etc.), that code is authoritative. Store searchable E.164 when set.
- Partial unique index on `(businessId, normalizedEmail)` excluding soft-deleted and nulls (migration SQL; Prisma cannot express; upsert cannot target it).
- Unauthenticated/guest bookings never overwrite existing customer fields.
- Soft-deleted customers never revived; same email later creates a **new** Customer.
- Owner manual create: name required; email/phone optional; duplicate email blocked; matching phone → non-blocking warning. No account/verification on create.
- Race on first bookings: unique index + re-read and link; never 409 for uniqueness race on guest book. Guest get-or-create + appointment insert in one transaction (slot conflict rolls back; no orphan customers).

### 9.2 Snapshots vs current

- Appointment stores booking-time customer contact snapshot.
- Admin appointment detail shows **current** Customer name/phone/email; if soft-deleted → snapshot fallback + “Silinmiş müşteri”.
- No dual “booking contact” UI block.

### 9.3 Registration and account linking

- Form: email, password, name, phone.
- Pending data in verification token (incl. password hash); **no** Customer/Account before verification.
- On verify: one transaction attaches to existing non-deleted Customer by email (existing field values win) or creates Customer, then creates `CustomerAccount`.
- Re-registration invalidates pending; response always uniform; already-registered email gets `ACCOUNT_ALREADY_EXISTS` email.
- Guest history appears automatically once account linked (appointments already reference Customer).
- No `userId` on Appointment; chain appointment → customerId → Customer → CustomerAccount.

### 9.4 Profile, email change, password reset, deletion

- Customer edits name and phone.
- Email change: customer-only; verify new address; notice to old; reject if another active customer in business has email; update `Customer.email`; revoke other sessions; rotate current; snapshots unchanged.
- Owner cannot change a non-null customer email; may **set email once** when null (no verification/account). Owner cannot change email of account holders.
- Password reset required; purpose-scoped tokens; uniform responses; revoke all sessions after reset.
- Soft-delete: blocked if future CONFIRMED exist; same transaction disables account, deletes sessions/tokens; Owner still sees past appointments via snapshots; **no restore**; no self-service account deletion in MVP.

### 9.5 Customer note

- Optional on booking; stored on Appointment; **immutable** after create.
- Owner/Staff read-only; Customer cannot edit later or on reschedule.
- No internal note in MVP.

---

## 10. Services and staff — FINAL

### 10.1 Service

- Fields: name, `durationMinutes`, `priceMinor` (integer minor units), currency (ISO 4217; aligned with business currency at write time), `bufferMinutes`, active/inactive.
- Duration independent of slot interval.
- Multiple staff may offer a service via StaffService.
- Inactive: no new bookings; existing CONFIRMED continue.
- Mutations never alter historical appointment snapshots or stored `endsAt` / `blockedUntil`.

### 10.2 Staff

- Belongs to business; optional link to `BusinessMember` via composite FK including `businessId`.
- Active/inactive; working hours; time off including all-day time off (Owner-managed; semantics §11.2).
- Display name used for public UI and booking snapshots.
- Staff–service unlink: allowed even with future CONFIRMED for that pair; appointments preserved; new bookings cannot use the pair.
- Working-hours / time-off / closed-date changes: never auto-cancel appointments; only affect future availability.

### 10.3 Money

- Integer minor units everywhere; no floats.
- Format with `Intl.NumberFormat('tr-TR', { style: 'currency', currency })`.
- Appointment snapshot stores `priceMinor` + `currency` at booking time.
- No payments in MVP. Emails do not show price in MVP notification content rules (§17) unless later revised in this document.

---

## 11. Scheduling, availability, and timezone — FINAL

### 11.1 Instants and timezone

- Store instants as `timestamptz` `@db.Timestamptz(3)` (ms precision matching JS).
- Containers and DB: `TZ=UTC`.
- Business timezone: IANA; Node/ICU single tzdata source; no SQL `AT TIME ZONE` for product logic.
- Display (public, admin, email): always business timezone with label; never browser/server local.
- One shared time module (library choice OPEN: Luxon vs Temporal).

### 11.2 Working hours, closed dates, and time off

- Working hours: local wall-clock; ISO `dayOfWeek` 1–7; integer start/end minutes, end exclusive; multiples of 5; `0 ≤ start < end ≤ 1440`. No `@db.Time`; no overnight intervals.
- Same-day intervals may touch, not overlap; engine fetch → validate → sort → merge → calculate (merge is in-memory only).
- **Business closed dates (full-day):**
  - Owner-managed; keyed by business + local calendar date in the business timezone; unique per `(businessId, localDate)`.
  - Closure is the full local day.
  - Public availability produces **no slots** that day; new public bookings are blocked.
  - New manual bookings are blocked under normal rules (Owner may still use `overrideHours` as with other closed-day overrides — §11.7).
  - Existing appointments are **not** auto-cancelled or mutated; their statuses are unchanged; reminders continue.
- **Staff time off:** stored as instants (half-open ranges in UTC after conversion).
  - **All-day TimeOff:** a business-local `YYYY-MM-DD` means `[local 00:00 that day, local 00:00 next day)` in the business timezone — **not** “24 elapsed hours”.
  - All-day TimeOff closes that staff’s entire availability for that local day.
  - Adding/changing TimeOff never auto-cancels or mutates existing appointments.
- Conversion order: TZ → local date → that day’s local rules (hours − closed date − time off) → local candidates → UTC → all checks in UTC.
- Durations: real elapsed minutes.
- API dates: `YYYY-MM-DD` business-local; booking sends UTC instant; server verifies instant is a generated slot (public path).

Conceptual availability:

```text
business hours
∩ staff hours
− business closed dates
− staff time off
− existing blocking appointments
− required buffer
```

### 11.3 Slot grid

- Business-level interval: 5/10/15/20/30/60; default 15.
- Aligned to local midnight; one grid per day. Example: hours 09:10–17:00 @ 15 → starts 09:15, 09:30, …
- Interval defines start candidates only; rule: start + duration ≤ end of containing working interval.
- Appointment must fit in a single business∩staff interval.
- Grid applies to **public bookings and customer reschedules** only.
- Manual/business bookings and admin reschedules: any minute, seconds = 0.
- Two validation paths (slot membership vs direct rules) share the same rule functions.
- Availability response: `date`, `timezone`, `slots[]` with `startsAt` (UTC), `localTime`, `utcOffset`. Only available slots; no staff info except alternatives. Frontend does no TZ math.

### 11.4 Buffer and protected interval

- Buffer **after** appointment only; from snapshot at booking time.
- Protected interval: half-open `[startsAt, blockedUntil)`.
- `blockedUntil` stored column computed by app (`timestamptz + interval` is STABLE — not a generated column/index expression).
- Exclusion constraint per staff on `tstzrange(startsAt, blockedUntil, '[)')` where `status <> 'CANCELLED'`.
- Working hours constrain `[startsAt, endsAt)` only; buffer may spill past closing or into time off and still blocks other appointments.

### 11.5 Notices and windows

- `minBookingNoticeMinutes`, `maxBookingWindowDays`: public booking + customer reschedule only; business-side exempt.
- `cancellationNoticeMinutes`: customer cancel and customer reschedule eligibility only; Owner/Staff exempt before start.
- Reminder lead uses real duration (DST-safe).

### 11.6 DST

- Nonexistent local times skipped; interval boundaries in gap clamped to transition instant.
- Ambiguous times: two slots with offset labels; ambiguous start → earlier occurrence; ambiguous end → later.
- Day boundaries from local midnights (23h/25h days possible).
- Tests: Berlin 29 Mar & 25 Oct 2026; New York 8 Mar & 1 Nov 2026.

### 11.7 Manual / Owner override

- Staff path: must follow availability rules (including business closed dates and time off).
- Owner: may override weekly closed days, **business closed dates**, out-of-hours, and time off with explicit flag; honored only for OWNER.
- Never override: past time, overlaps, inactive staff/service, staff not offering service.
- Persisted as `Appointment.overrideHours` boolean (default false).

### 11.8 Staff selection and alternatives

- Specific staff: that staff’s availability.
- **“Fark etmez / İlk uygun çalışan”** — among staff who are eligible for the service and available for the chosen slot, select by:
  1. Lowest total **CONFIRMED** appointment **duration** (sum of snapshot durations / `endsAt − startsAt` of CONFIRMED rows) over the next **7 business-local calendar days**. `CANCELLED` appointments are excluded from workload; other non-CONFIRMED statuses are excluded.
  2. If tied: staff whose next upcoming **CONFIRMED** appointment **ends earliest** (smallest next `endsAt`). Staff with no upcoming CONFIRMED appointment win this step over staff who have one.
  3. If still tied: Staff primary-key UUID ascending (stable, deterministic).
- Alternatives on conflict/nearby:
  - Specific staff: 3 nearest same staff + up to 3 other staff (separate group).
  - Fark etmez: 6 nearest without staff names.
  - Search: same day + 7 days, capped by max window; reuse availability engine.

---

## 12. Booking — FINAL

### 12.1 Public flow

Service → Staff (or Fark etmez) → Date → Time → Contact (name, email, phone, optional note) → Confirmation.

- No manual approval; new bookings start `CONFIRMED`.
- `source = PUBLIC`.
- Response: confirmation info only; identical for new vs existing customer (anti-enumeration; no welcome-back / prefill).

### 12.2 Manual admin flow

Service → Staff → Date → Time → Customer → Confirm.

- Staff mandatory; no Fark etmez.
- `source = OWNER` or `STAFF`.
- Customer: search existing or create (same matching rules as §9).
- Confirm copy: if email present, “Onay e-postası gönderilir”; else no email. No send toggle.
- Email: enqueue `BOOKING_CONFIRMED` if email exists; **no** business appointment notice for admin-created bookings.
- Allowed when business INACTIVE.
- Race: exclusion constraint; loser 409; form preserved; nearby alternatives offered.

### 12.3 Public booking concurrency

- Same DB exclusion + conditional insert semantics.
- Loser keeps form data and receives nearby alternatives (incl. other staff when service allows).

---

## 13. Appointment model and lifecycle — FINAL

### 13.1 Core fields (conceptual)

- Scheduling: `startsAt`, `endsAt`, `blockedUntil`, `staffId`, `serviceId`, `customerId`, `status`, `version`
- Cancellation: `cancelledAt` (DB `now()`), `cancelledBy` ∈ {CUSTOMER, STAFF, OWNER} (no user id)
- Meta: `source`, `overrideHours`, customer note (immutable), snapshots (§13.2), booking reference `BK-XXXXXXXX` (§21.1)
- No business snapshot on appointment; emails/UI branding use current Business at send/view time.

### 13.2 Snapshots (immutable)

- Service name, duration, buffer, priceMinor, currency
- Staff display name
- Customer booking contact/name as applicable

Later service/staff/hours/business edits do not mutate these or `endsAt`/`blockedUntil`.

### 13.3 Statuses and transitions

Statuses: `CONFIRMED` | `COMPLETED` | `CANCELLED` | `NO_SHOW`. No `PENDING`.

```
CONFIRMED → CANCELLED          (only if now < startsAt)
CONFIRMED → COMPLETED|NO_SHOW  (only if now ≥ startsAt; do not wait for endsAt)
COMPLETED ↔ NO_SHOW            (no time limit)
CANCELLED terminal
```

- Past CONFIRMED: no auto-transition; UI badge **“İşaretlenmedi”**; excluded from no-show rate.
- No-show rate: `NO_SHOW / (COMPLETED + NO_SHOW)` (dashboard does not show this widget in MVP).
- Permissions: Cancel — Customer (own), Staff (own), Owner (any). Complete/No-show/corrections — Owner or appointment’s current staff. Customer never Complete/No-show.
- Customer notice: `startsAt − now ≥ cancellationNotice`.
- Blocking for engine + DB: `status <> CANCELLED` (identical predicate).
- Concurrency: single conditional UPDATE on id, businessId, expected status, time condition via DB `now()`; role checked before UPDATE; 0 rows → re-read: already target → idempotent success without side effects; else 409. `version` increments every mutation.
- Cancelled detail: show `cancelledAt`, `cancelledBy`; no rebook action; immutable. No cancellation reason field.

### 13.4 Reschedule

- In-place conditional UPDATE (no new row); status CONFIRMED; version or old startsAt match.
- Service unchanged; staff name snapshot updated when staff changes.
- Owner may change staff (+ override rules); Customer may change staff or Fark etmez; Staff only within own schedule (cannot change staff).
- Engine excludes appointment being moved from conflicts.
- Public/customer: grid + notice + max window; admin: arbitrary minute.
- Reminder renewed; manage token revoked and new one emailed.
- New slot success before old slot effectively free (same-row move under exclusion).

### 13.5 Reminder worker

- Job: businessId, appointmentId, scheduled startsAt; deterministic job id.
- Send only if CONFIRMED, startsAt matches, `now < startsAt`; else no-op.
- Claim via NotificationDelivery conditional update before send; clear claim on failure.
- Enqueue after commit; enqueue failure logged, does not fail request; no outbox in MVP.
- Timing: if `lead ≥ 24h`, delay = startsAt − 24h − now (exactly 24h → immediate); if `lead < 24h`, no reminder.
- Reschedule: remove old job best-effort; schedule new; cancel removes job; guards cover leftovers.
- No email for COMPLETED/NO_SHOW transitions.

---

## 14. Calendar — FINAL

### 14.1 Views and navigation

- Day + Week only (no Month). Default: **Week**. Same for Owner and Staff.
- Prev/next, Today, date picker. Week starts **Monday**.
- All boundaries and “now” indicator: **Business timezone only**.

### 14.2 Staff filter and layout

- Owner: All staff | single staff. Single calendar (no resource columns). All view: staff name on card + deterministic staff color.
- Staff: own calendar only; no filter.
- No calendar search; no service/status filters on calendar.

### 14.3 Cards and detail

- Card: local time–end, customer current name (or snapshot fallback), service snapshot name, staff name only in All view, status **text** badge. No phone/email/price/note.
- Detail: **right drawer**. Shows time, duration, status (+ İşaretlenmedi), service snapshot, price/currency snapshot, staff snapshot, customer (current/fallback), note, source, overrideHours, createdAt, cancellation metadata.
- Actions follow §13 transitions; past CONFIRMED hides Cancel/Reschedule; shows Complete/No-show.
- COMPLETED/NO_SHOW: only correction between the two. CANCELLED: view only.

### 14.4 API semantics (conceptual)

- `from`/`to` as business-local `YYYY-MM-DD`, `to` exclusive; server converts via business TZ to UTC range.
- Max **500** appointments; `truncated: true` if needed.
- Return all statuses in range (cancelled muted).
- Indexes: `(businessId, startsAt)`, `(businessId, staffId, startsAt)` complement exclusion gist.
- No SSE; after mutation refetch range; 409 → toast + refetch; form preserved where applicable.
- Manual booking entry from calendar uses §12.2; inactive banner when business INACTIVE.

---

## 15. Customer Management — FINAL

### 15.1 Purpose and access

- Owner finds customers, edits basic fields, inspects history. **Not a CRM.**
- Owner only; Staff → **403**.

### 15.2 List

- Columns: name, phone, email, next upcoming (future CONFIRMED), last appointment (latest startsAt any status).
- No total-count column, account badge column, or tags.
- Search `q` (min 2 chars): name `ILIKE` partial; `normalizedEmail` partial; phone normalized/trailing match. Server-side; ~300ms debounce UI. No unaccent; no Elasticsearch.
- Filter: `deleted=active|deleted|all` (default active).
- Sort: default `name asc`; alternative `createdAt desc`.
- Pagination: offset, page 1-based, pageSize default 25 max 50, include `total`.

### 15.3 Detail and history

- **Full page** detail: name, phone, email, account state (`NONE`|`ACTIVE`|`DISABLED`), createdAt, deletedAt?, history.
- No compact CRM stats strip.
- History: upcoming first, then past descending; all statuses; İşaretlenmedi for past CONFIRMED; service/staff snapshots; click opens shared appointment drawer; paginate 25/max 50.
- Soft-deleted: hidden by default; Owner can open detail; **no restore**.
- “Yeni randevu” → existing manual booking flow with `customerId` preselected (Owner).

### 15.4 Create / edit / delete

- Create: same as booking-context create (§9).
- Edit: name; phone (E.164 or null); email only when currently null (set once). Empty string → null for optional fields.
- Soft-delete: existing rules (§9.4).

---

## 16. Dashboard — FINAL

### 16.1 Purpose

Answer “Şimdi / bugün ne yapmalıyım?” — not analytics.

### 16.2 Scope and metrics

- Same layout; Owner business-wide; Staff own staff.
- Metrics only: `todayCount`, `unmarkedCount`, `nextAppointment`.
- No revenue, customer KPIs, charts, week selector, no-show-rate widget, Redis cache, SSE.

### 16.3 Today / next / unmarked

- Today: business-local day; all statuses chronological; cancelled muted; past CONFIRMED = İşaretlenmedi; max 100 + `truncated`.
- Next: nearest future CONFIRMED only.
- Unmarked: **all** past CONFIRMED (`unmarkedCount` has no age cap in MVP), oldest first, max 20 listed; optional calendar deep-link by local date; not Customer Management.
- Click rows → appointment drawer.

### 16.4 Quick actions and empty states

- Owner: New appointment, Calendar, Customers, Settings.
- Staff: New appointment, Calendar.
- INACTIVE: persistent banner; Owner activation CTA; Staff no CTA.
- Empty: reuse go-live checklist summary (no new wizard); CTAs for first appointment / calendar.

### 16.5 Read model

- Conceptual `GET .../dashboard`: 3–4 tenant-scoped queries, app-side assembly, always fresh.
- Refetch on mount and window focus; after returning from mutations.

---

## 17. Notifications and email — FINAL

### 17.1 Transport and architecture

- **All** emails via BullMQ after DB commit; request does not await send.
- Never enqueue inside `$transaction` callback.
- No event bus; application services call a notifications module (`enqueue` per type).
- Single `email` queue; reminders are delayed jobs on same queue.
- Worker: same image, separate container/command; concurrency configurable (default 5); queue limiter under Resend plan limit.
- Redis: persistent volume; AOF `appendfsync everysec`; `maxmemory-policy noeviction`.
- Known limitation: commit→enqueue not transactional; enqueue failure → log, request still succeeds. No outbox in MVP (post-MVP: reminder sweeper, then outbox).

### 17.2 Email types

**Customer:** `BOOKING_CONFIRMED`, `BOOKING_RESCHEDULED`, `BOOKING_CANCELLED` (copy by `cancelledBy`), `BOOKING_REMINDER`, `ACCOUNT_VERIFICATION`, `ACCOUNT_ALREADY_EXISTS`, `CUSTOMER_PASSWORD_RESET`, `EMAIL_CHANGE_VERIFICATION`, `EMAIL_CHANGED_NOTICE`.

**Business:** `OWNER_INVITATION`, `STAFF_INVITATION`, `BUSINESS_PASSWORD_RESET`, `BUSINESS_APPOINTMENT_NOTICE` (variants NEW / CANCELLED / RESCHEDULED).

- B4 only for **customer-initiated** public events; recipients: active Owner + appointment staff (+ previous staff on reschedule); dedupe by userId; no preference toggles.
- Manual/admin booking: customer C1 if email; no B4.

### 17.3 Payload, validation, idempotency

- Payload: type, businessId, target ids, state assertions, previous values if needed, schema `v`. No PII/secrets/rendered HTML in Redis.
- Identifier-only; worker re-reads DB and validates (fingerprint `(startsAt, staffId)` for confirm/reschedule notices; CANCELLED for cancel; reminder per existing rules). Fail → `SKIPPED`.
- `NotificationDelivery` table with unique `dedupeKey`; statuses PENDING | SENDING | SENT | SKIPPED | FAILED; claim then validate then send; stale SENDING reclaim after 5 minutes.
- At-least-once with DB guard; provider timeout duplicate window accepted and documented.
- Retries: 8 attempts, exponential backoff base 30s, provider timeout 10s; retry network/5xx/429/DB; permanent 4xx validation, 401/403 (alert), render/payload bugs.

### 17.4 Content and templates

- Appointment facts from snapshots; branding/contact/timezone from **current** Business; recipient current email (C9 uses token old email).
- Timezone formatting in shared time module; templates receive ready strings; locale `tr-TR` only in MVP (copy objects separated for future).
- React Email; typed props + registry; HTML+text rendered in-app; preview via email dev; no Resend `react` param coupling.
- Branding: name, logo, primaryColor (CTA + accent + contrast text), contact, booking block. No powered-by, custom CSS/fonts/HTML.
- From: `noreply@mail.{platform}`; display name = sanitized business name (≤64) for customer/invite emails; platform name for B3/B4. Reply-To: business email when set (customer + staff invite); operator support for owner invite; none for B3/B4.

### 17.5 Provider and tests

- `EmailProvider` abstraction: Resend / Fake / Log; `EMAIL_PROVIDER` env; production without key fails startup; tests must not use real Resend.
- Fake provider records `sentEmails`; processor is DI-friendly pure function for unit tests.

### 17.6 Token email lifecycle

- Intent row in request transaction; secret minted in worker; hash written if still active; raw secret only in memory/email.
- Resend rotates secret and renews expiry; automatic retries rotate secret but do not extend expiry beyond the configured policy (exact durations deferred to technical documents).
- Manage tokens: multiple may be active; reschedule/cancel/delete revoke appointment’s manage tokens.

---

## 18. White-label branding — FINAL

- Public booking and emails use business name, logoUrl, primaryColor, contact as configured.
- Platform brand is not dominant on the public booking experience.
- Boundaries: validated hex only; HTTPS logo URL; escaped text; no arbitrary CSS; public DTO omits sensitive settings (§6.5, §6.7, §17.4).

---

## 19. Security architecture — FINAL

### 19.1 Principles

1. Security-sensitive invariants enforced in the DB where practical (exclusion constraint, uniques, conditional updates).
2. Tenant isolation centralized; never left to ad-hoc filters.
3. Client tenant IDs never trusted.
4. IDs are never authorization.
5. Snapshots keep history stable.
6. Prefer simple robust MVP; avoid premature distributed complexity.
7. External providers only when they reduce complexity.

### 19.2 AuthN/AuthZ checklist

- Secure HTTP-only cookies; SameSite=Lax; Origin/CSRF; Argon2id; hashed tokens/sessions; idle+absolute expiry; rotation on login; rate limits on login/reset.
- Membership/role from DB; separate business/customer realms.
- Tokens: high-entropy, hashed, purpose-scoped; fragment delivery; POST consume.
- Public branding inputs sanitized/validated (§6.7).

### 19.3 Tenant isolation checklist

- Composite tenant FKs; tenant uniques; scoped Prisma; no casual raw bypass; cross-tenant tests; 404 on cross-tenant; workers carry tenant context.

---

## 20. Technical architecture and stack — FINAL

| Layer | Choice |
| --- | --- |
| Frontend | React, TypeScript, Vite |
| Backend | Node.js, Express, TypeScript |
| Database | PostgreSQL |
| ORM | Prisma |
| Jobs | Redis + BullMQ |
| Email | Resend |
| Deployment | Docker, single production VM |
| CI/CD | GitHub Actions |

Not in sprint: Next.js, Kubernetes, Terraform, DB/schema-per-tenant, RLS, PG-generated `uuidv7()`.

UX consistency across Calendar, Customer Management, Dashboard: shared time formatting, Business TZ, status badges (text + style, not color-only), shared appointment drawer, 403/404 rules, loading/empty patterns, refetch-after-mutation.

---

## 21. IDs and concurrency — FINAL

### 21.1 IDs

- All PKs UUID v7; PG native `uuid`; Prisma `@db.Uuid`.
- Generated in Prisma/application layer (not PG `uuidv7()`).
- `createdAt` authoritative; ID order never business logic.
- Public identifiers separate from primary keys:
  - Business slug
  - **Booking reference** (Appointment): business-scoped human-readable public reference `BK-XXXXXXXX`
    - Literal prefix `BK-`
    - 8 characters after the prefix
    - Uppercase **Crockford Base32** (ambiguous characters `I`, `L`, `O`, `U` excluded)
    - Unique per business (`businessId` + reference); **not** the primary key
    - Safe for customer communication / support
    - Generation helper details deferred to Data Model / API docs
  - High-entropy secret tokens (manage/guest, etc.)
- Non-Prisma inserts (seeds, raw SQL) must supply UUIDs via the same application helper (helper mechanics deferred to technical documents).

### 21.2 Double-booking

- PostgreSQL exclusion on staff + `tstzrange(startsAt, blockedUntil, '[)')` where status ≠ CANCELLED (`btree_gist`).
- Engine, constraint, and tests share the same range definition and status predicate.
- Required race tests: book vs book; reschedule vs book; reschedule vs cancel; buffer boundary; cancel frees slot.
- Exactly one overlapping success; losers 409 + alternatives; form preserved.

---

## 22. Explicitly out of scope — FINAL

### 22.1 Product features

- Online payments/deposits  
- SMS/WhatsApp  
- Google Calendar  
- Recurring appointments  
- Group/capacity appointments  
- Multi-location  
- Waitlist  
- POS/inventory/packages  
- Custom domains  
- Detailed audit log  
- Health/medical records  
- Drag-and-drop calendar  
- SSE / realtime sync  
- Customer tags/segments, CRM analytics, marketing consent/preferences  
- Customer import/export, CSV, bulk actions, merge, duplicate-resolution UI  
- Customer-level or internal notes beyond immutable booking note  
- Restore soft-deleted customers  
- Revenue/conversion/staff-performance analytics, charts, custom dashboard widgets, saved filters, advanced reports  
- Dashboard Redis cache  
- ICS/calendar attachments; “password changed” notice emails; business-user email verification (invite proves ownership)  
- Multiple Owners in MVP practice; ownership transfer; multi-business switcher UI  
- Transactional outbox (known enqueue limitation accepted for MVP; see §17.1)

### 22.2 Infrastructure

- Kubernetes, Terraform, Next.js (this sprint)  
- Database-per-tenant, schema-per-tenant  
- PostgreSQL RLS (deferred until before the first real client; not part of the sprint)  
- PostgreSQL-generated UUIDs  
- Elasticsearch / search appliances  
- CQRS / read replicas / analytics warehouse / materialized views / event sourcing for MVP reads  

---

## 23. Remaining open product / architecture decisions

Nothing in this section may be assumed during implementation. Decide explicitly and record the outcome in this document before implementing behavior that depends on it.

1. Date/time library choice (Luxon vs Temporal) for the shared time module.  
2. Complete reserved-slug list (central list required; exhaustive entries not yet frozen).  
3. Currency allow-list vs any ISO 4217 code.  
4. Staff calendar color palette.  
5. Business offboarding / data retention policy (slug remains reserved until that policy exists).  
6. Exact email template wording/copy (template architecture and branding rules are FINAL; copy text is not).

Implementation-specific decisions such as exact token/session durations, rate limits (values, store, endpoint list), deployment/VM provider, observability, test framework, repository structure, PostgreSQL/Prisma versions, non-Prisma UUID helper mechanics, frontend/API origin topology, frontend component/state architecture, demo-tenant provisioning/reset mechanics, and email deployment configuration (`EMAIL_FROM_ADDRESS`, platform sending domain, `EMAIL_PLATFORM_NAME`, operator support inbox, provider limiter numbers) are intentionally deferred to the corresponding technical documents. They are not Product Architecture OPEN decisions.

**Do not resurrect** items already finalized elsewhere in this document (including: customer matching, registration/verification, email change, soft-delete, phone default country `TR`, onboarding/CLI, Owner/Staff model, Owner-as-bookable-staff, slot intervals, buffer semantics, business closed dates, all-day TimeOff, Fark etmez workload algorithm, booking reference `BK-XXXXXXXX`, override flag, status transitions, cancellation/reschedule, snapshots, notification/BullMQ architecture, EmailProvider, token architecture shape, calendar/CM/dashboard behavior including unmarked = all past CONFIRMED, branding boundaries, tenant isolation, authentication model, UUID strategy, double-booking constraint, timezone/DST rules, business lifecycle, logo URL-only, i18n = `tr-TR` only for MVP).

---

## 24. Decision history and rationale

| Decision | Alternatives considered | Rationale |
| --- | --- | --- |
| Build a booking platform | Field Service SaaS, embeddable pricing calculator | Highest share of well-defined agent-parallel tasks in a 6-day sprint; demoable mid-sprint; reusable for freelance clients. |
| Not cheap generic SaaS | Low-price subscription SaaS | TR market ~99–499 TL/mo with free options; value is custom white-label for multi-staff businesses and agencies. |
| Server-side sessions + HTTP-only cookie | JWT; Clerk/Auth0/Supabase | Instant revocation; authz from DB; providers costly/awkward for per-tenant customers and white-label. |
| PostgreSQL session store | Redis sessions | Revoke in same TX as membership changes; survives Redis restarts. |
| Shared DB + `businessId` | DB/schema per tenant | Native Prisma; one migration path; fits many small tenants. |
| RLS deferred | RLS in MVP | Prisma+RLS tax competes with features; add later if `businessId` + central access ready. |
| UUID v7 in Prisma app layer | v4, CUID, identity, PG18 `uuidv7()` | Order-friendly indexes without enumerable IDs; avoid PG18 dependency. |
| DB exclusion for overlaps | App-only checks | App checks race under concurrency. |
| No PENDING status | Approval workflow | Instant confirm product. |
| Booking snapshots | Live service/staff reads | History must not drift (principle). |
| Email-only customer matching | Phone / both / fuzzy merge | Deterministic, anti-enumeration friendly, unique index enforceable. |
| Operator CLI onboarding | Public signup | Control demo quality; service still reusable later. |
| Buffer after + stored `blockedUntil` | Before/both; generated column | Matches product rule; STABLE interval math cannot index as generated expr. |
| INACTIVE + `publishedAt` | Multi-state enum | Distinguishes never-live (404) vs temporarily closed (message) with minimal states. |
| TZ/currency/slug immutable after publish | Free edit / deactivate-to-edit | Wall-clock hours + minor-unit prices + public URLs must not silently reinterpret. |
| All email via BullMQ + NotificationDelivery | Sync send; appointment sentAt columns | Request latency; multi-type idempotency; token emails; retries. |
| Calendar Day+Week, no Month/DnD/SSE | Full calendar suite | Ops need day/week; complexity budget. |
| CM Owner-only; Dashboard thin ops | Staff CRM; analytics widgets | Matches roles; avoids SaaS dashboard clutter. |
| React + Vite + Express | Next.js | Team velocity and reviewability in sprint. |
| Fark etmez: 7-day CONFIRMED duration workload + earliest next end + UUID | Random / round-robin only | Deterministic, fair load, easy to test. |
| Booking ref `BK-` + Crockford Base32 | UUID / numeric sequence | Human-readable, non-ambiguous, business-scoped, not a PK. |
| Phone default country `TR` | Require always-E.164 input | Matches primary market; explicit `+` country codes still win. |
| Business full-day closed dates | Weekly hours only | Holidays/one-off closures without mutating history. |
| All-day TimeOff = local midnight→next midnight | Fixed 24h elapsed | Correct under DST; matches calendar-day mental model. |

---

*End of authoritative Product & Architecture Decision Document.*
