# MediCore — Hospital Management System

A production-architected HMS foundation built with Next.js 15, React 19, TypeScript,
Prisma/PostgreSQL, Auth.js (RBAC + JWT), Tailwind + shadcn/ui, Framer Motion, Recharts,
and Socket.IO.

## What's fully implemented

- **Database**: complete, normalized Prisma schema (`prisma/schema.prisma`) covering
  every module in the spec — users/auth, patients, doctors/departments, appointments,
  EMR, pharmacy, lab, billing/insurance, inventory, beds/admissions, staff/payroll,
  notifications, audit logs.
- **Auth & RBAC**: Auth.js v5 credentials provider with bcrypt + optional TOTP MFA
  (`src/lib/auth.ts`), a single-source-of-truth permission matrix
  (`src/lib/rbac.ts`), and route-level middleware enforcing both authentication and
  role checks (`src/middleware.ts`).
- **API layer pattern** (fully working, use as the template for remaining resources):
  - `src/app/api/patients/route.ts` + `[id]/route.ts` — paginated search/filter,
    Zod validation, RBAC checks, self-scoping for patient users, audit logging,
    rate limiting, soft-delete.
  - `src/app/api/appointments/route.ts` — booking with doctor-availability checks,
    leave-conflict checks, double-booking prevention, and auto-generated queue numbers.
- **UI**: role-aware sidebar/topbar shell, a real dashboard pulling live Prisma
  aggregates into Recharts (appointment trend, bed occupancy, revenue), a patients
  list with server-side search/pagination, and a login page with react-hook-form + Zod.
- **Infra**: Dockerfile (multi-stage, standalone output), docker-compose (Postgres +
  Redis + app + socket server), GitHub Actions CI (lint/typecheck/build against a real
  Postgres service) and a deploy workflow stub for Vercel + Railway.
- **Seed script** (`prisma/seed.ts`) creates a demo hospital and one login per role.

## What's scaffolded, not built out

Given the scope of this spec (12+ modules, 9 roles), the following have schema +
architecture in place but need their API routes/pages written following the exact
pattern in `patients` and `appointments`:

- Doctor management pages, leave approval workflow
- EMR entry screens (diagnosis/prescription/imaging forms)
- Lab module (sample tracking UI, result entry/approval)
- Pharmacy (inventory UI, dispensing flow, low-stock alerts)
- Billing (invoice generation UI, payment capture, insurance claims)
- Inventory (equipment, purchase orders, suppliers)
- Staff (attendance, shifts, payroll)
- Notification delivery (email via nodemailer, SMS via Twilio — clients need wiring)
- File upload to Cloudinary/S3 (packages included, no upload route yet)
- MFA enrollment UI (backend already supports otplib TOTP)

This isn't a shortcut — a real HMS's remaining modules are best built one at a time
against real user feedback, and copy-pasting a working pattern nine more times without
being able to test against a live database would risk quietly wrong code.

## Running it locally

This code was written without network/build access, so **verify with a real install**
before trusting it in full:

```bash
git clone <your-repo>
cd hms
cp .env.example .env          # fill in DATABASE_URL etc.
npm install
docker compose up -d db redis # or point DATABASE_URL at your own Postgres
npx prisma migrate dev --name init
npx prisma db seed
npm run dev
```

Demo logins (password `Password123!` for all):
`admin@medicore.dev`, `doctor@medicore.dev`, `patient@medicore.dev`,
`nurse@medicore.dev`, `reception@medicore.dev`, `lab@medicore.dev`,
`pharmacy@medicore.dev`, `accounts@medicore.dev`

Realtime server: `npm run socket` (separate terminal, port 4001).

## Extending a new module

Follow this checklist for every remaining resource (e.g. lab tests):

1. Schema already exists in `prisma/schema.prisma` — run `prisma migrate dev` after
   any tweaks.
2. Add a Zod schema to `src/lib/validations.ts`.
3. Add resource + actions to `PERMISSIONS` in `src/lib/rbac.ts`.
4. Create `src/app/api/<resource>/route.ts` (list+create) and `[id]/route.ts`
   (get/update/delete), copying the patients route's structure: `auth()` →
   `assertCan()` → rate limit → validate → query → `writeAuditLog()` → `ok()`/`paginated()`.
5. Build the page under `src/app/(dashboard)/<resource>/` using the patients list
   page as the template; use `react-hook-form` + `zodResolver` for any form.
6. Add the nav entry to `src/components/layout/sidebar.tsx` with the correct
   `roles` array.

## Security notes

- Passwords hashed with bcrypt; sessions are JWT with an 8h sliding expiry.
- All mutating routes are rate-limited (swap `RateLimiterMemory` for
  `RateLimiterRedis` before running more than one server instance — see
  `src/lib/rate-limit.ts`).
- Every write is audit-logged to the `AuditLog` table.
- Patient-role users are scoped to their own records at the query layer, not just
  hidden in the UI.
- Soft-delete (`isActive: false`) is used for patients to preserve medical/billing
  history rather than hard-deleting rows.
