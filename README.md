# White-Label Booking Platform

Reusable multi-tenant white-label booking infrastructure for appointment-based service businesses (primary: 3–15 employee clinics/studios) and agencies that need the same stack for custom implementations.

## Stack

- Frontend: React + TypeScript + Vite
- Backend: Node.js + Express + TypeScript
- Database: PostgreSQL + Prisma
- Jobs: Redis + BullMQ
- Email: Resend
- Deployment: Docker (single VM)
- CI/CD: GitHub Actions

## Documentation (source of truth)

- [`docs/01-product-architecture.md`](docs/01-product-architecture.md) — product and architecture decisions (FINAL)
- [`docs/02-api-contract.md`](docs/02-api-contract.md) — API contract (placeholder; next phase)

## Layout

```text
apps/api   — backend (not scaffolded yet)
apps/web   — frontend (not scaffolded yet)
packages/  — shared packages (none yet)
docs/      — authoritative product/engineering docs
```

Implementation work starts after this repository bootstrap; do not treat empty `apps/` directories as runnable apps yet.
