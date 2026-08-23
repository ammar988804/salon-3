# Handoff — salon 2

Written 2026-08-12 at the end of a long session, for whoever picks this up next.

## The project

Salon management SaaS. Next.js 16 (App Router, Turbopack) + TypeScript + Tailwind v4,
Supabase (Postgres + RLS + auth) as the backend, deployed on Netlify. Currency PKR,
timezone Asia/Karachi, WhatsApp for client messaging. Roles: `admin`, `receptionist`, `staff`.

- Back office lives in `src/app/(app)/` — dashboard, bookings, clients, invoices, finance
  (with inventory + reports as tabs), reminders, settings, my-schedule (staff view).
- Public booking site lives in `src/app/book/`.
- Schema is `supabase/migrations/0001..0014`, with `supabase/merged_migrations.sql` as a
  single re-runnable concatenation of all of them.
- **Not a git repository.** No version control, so there's no diff to inspect — be careful.

## ⚠️ Do this first: migrations are written but NOT applied

**Paste `supabase/merged_migrations.sql` into the Supabase SQL Editor and run it.** That one
file is the whole schema and it is idempotent — verified statement by statement as safe
against the LIVE database, not just a fresh project. Re-running it applies whatever is new
and leaves the rest alone. This is the only way schema changes ship; there is no separate
"pending" file.

What's outstanding inside it, and what breaks until it runs:

1. `supabase/migrations/0014_record_payment_void_guard.sql` — payments **fail in production**
   until this runs. It **supersedes 0013**: both are `create or replace function record_payment`
   with the same signature `(uuid, text, numeric, uuid)`, and 0014 already carries 0013's two
   enum casts plus the void guard. Running 0013 as well is harmless, just redundant.
2. `supabase/migrations/0015_sequential_invoice_numbers.sql` — **invoice creation fails until
   this runs.** The app code already calls `next_invoice_no()`, so every new bill errors with
   "function next_invoice_no does not exist" until the function exists. This is a deliberate
   trade: the alternative was leaving a random suffix that fails a sale at the counter.

Only the user can do this. `.env` holds just `NEXT_PUBLIC_SUPABASE_URL`,
`NEXT_PUBLIC_SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY`, `NEXT_PUBLIC_SITE_URL` — no
connection string and no access token, so **DDL cannot be executed from this environment**.
Don't offer to run it; hand them the file.

**Migration workflow, for anything new (user's instruction, 2026-08-12): append the SQL to
`supabase/merged_migrations.sql` and update its header list. Do NOT create new SQL files** —
no `supabase/migrations/00NN_*.sql`, no separate "apply this" file. Then tell the user to
paste the merged file. It's a last-wins concatenation, so a fix must be appended *after* the
thing it fixes. The numbered files under `supabase/migrations/` are history; leave them.

### Why those two exist (the bug class to remember)

Postgres will coerce a lone quoted literal into an enum column, but **not** a `text`
expression. Two instances bit this app, one after the other, in `record_payment`:

- `p_method` (declared `text`) inserted into `payments.method` (`payment_method`) → fixed in
  0012 with `p_method::payment_method`.
- `status = case when … then 'paid' else 'partial' end` — a CASE whose branches are all
  untyped literals resolves to `text` → fixed in 0013 with `(case … end)::invoice_status`.

I audited every other enum write in the schema; they're all safe (enum-typed params, an
enum-typed variable with an explicit cast, or single bare literals). `expenses.category`
stopped being an enum in 0007, so the CASE in `adjust_stock` is text-into-text.

## What changed this session

**Payment UI rework** — `src/app/(app)/invoices/new/NewBillForm.tsx`
- Split payment removed entirely: one method, one amount per bill (state is `method`,
  `cashReceived`, `partAmount` — no payment array).
- Three modes: **Pay now** (read-only full amount + cash received + change to give; nothing
  can be left owed), **Half payment** (renamed from "Part pay"; a single amount field, and
  whatever it leaves behind is recorded as owed), **Pay later** (whole bill owed).
- Half payment opens prefilled at half of what's due and is capped at the amount due.
- Client "Payment pending" tagging needed no work — `clients/page.tsx:102` already derives
  it from invoice balance.

**Migrations** — `0013` (status enum cast), `0014` (reject payments on voided invoices;
`addInvoiceLine` and `finaliseInvoice` already refused, this closed the third door). Both
appended to `merged_migrations.sql`.

**Phase 0 fixes from the review**
- `my-schedule/page.tsx` — staff "Revenue generated" was **always 0**: it filtered
  `invoice_items.created_at`, a column that doesn't exist, so PostgREST errored and returned
  null. Now dated through `invoices!inner(created_at, status)` with voids excluded.
- Salon closures were ignored — `bookings/actions.ts` and `book/checkout/actions.ts` both
  hardcoded `holidays: []` while the engine (`availability.ts:221`) had always checked them.
  Both now load the date's row. Note the public one uses the **service-role** client, so it
  needs an explicit `.eq("salon_id", …)`; the back-office one is RLS-scoped.
- New **Closures** settings tab (`settings/closures/`) managing one-off holiday dates *and*
  `salons.weekly_off`, which had no editor either. Actions: `updateWeeklyOff`, `addHoliday`,
  `deleteHoliday` in `settings/actions.ts`.
- Currency was decorative in the back office (every call took `formatCurrency`'s hardcoded
  "PKR" default; only `src/app/book/` passed the setting). Now: `src/lib/currency-context.tsx`
  provides `useMoney()` for client components, seeded by `CurrencyProvider` in
  `src/app/(app)/layout.tsx`; `src/lib/currency.ts` exports a `cache()`d `getSalonCurrency()`
  for server components. ~20 files call a local `money(…)` instead of `formatCurrency(…)`.

## /book performance + the blue (2026-08-13)

`/book` took ~8s on every load. Cause: `page.tsx` called `getNextAvailableForStaff` once per
staff member, and that helper calls `loadContext()` for every day it scans — which re-reads
the salon row plus the **entire** `staff_shifts` and `staff_time_off` tables each time. 5 staff
x up to 14 days ≈ 210 sequential round trips before first paint. Replaced with
`getNextAvailableForStaffList` (same file): fetches everything once (~2 round trips) and runs
the 14-day scan in memory against the same engine. **8.0s → 1.8s, with byte-identical labels.**
`getPublicAvailableSlots` was deliberately left alone — it's on the checkout path.
`getNextAvailableForStaff` is now unused but kept.

**Services UI reworked to match a Fresha reference the owner supplied (2026-08-13):**
`ServicesSection.tsx` and step 1 of `BookingWizard.tsx` both used collapsed `<details>`
accordions, so finding one service among ~54 meant opening boxes one at a time. Both now use a
horizontally-scrolling category chip row with one category listed below (chips in the wizard
also carry a selected-count badge). `ServicesSection` became a client component for this. Side
effect: `/book` HTML halved, 255KB -> 127KB, because only the active category renders. The
wizard's 5-chip chevron stepper became a progress bar + "Step N of 5" — at `text-xs` the labels
were unreadable and overflowed a phone. Fixed while there: `ServiceCard` called
`formatCurrency(price)` with no currency, so the wizard showed PKR regardless of the salon
setting.

**Still open there:** the call passes `services.slice(0, 1)` — the first of 54 services, the
same one for every staff member. A stylist who doesn't perform it never gets a label and burns
the full 14-day scan. Needs a product decision on what "next available" should mean.

The blue: the accent was already `#1d4ed8`, but `--color-surface` was `#f7f9fc`, so the page
read as white. Moved the blue into the surfaces in `:root` (surface `#e9f0fc`, ink `#0f1b2e`,
border `#cbdaf3`, accent-soft `#d8e6fd`); cards stay pure white. All 9 text/background pairs
pass WCAG AA (tightest 6.46:1). `.app-shell` redefines every one of these, so the software is
untouched — only `/book` and `/login` changed.

## Verification

```
npx tsc --noEmit -p tsconfig.json     # currently clean
npx vitest run                        # 15/15 pass (availability engine)
npx eslint src                        # 11 errors — ALL PRE-EXISTING, see below
```

The 11 lint errors are the baseline, not regressions: 6 `react-hooks/set-state-in-effect`
(debounced client-search effects, AppShell theme restore), 4 `react/no-unescaped-entities`,
1 `react-hooks/purity`. Don't "fix" them as part of unrelated work; confirm the count is
still 11 after changes.

Files are **LF** line endings throughout — preserve that if scripting edits.

## Dev server gotchas

- `npm run dev` on port 3000. A background instance was started at the end of this session
  (never confirmed serving — verify before trusting it).
- Turbopack's persistent cache in `.next/dev` had been corrupted by **two dev servers
  writing it at once** ("Unable to write SST file", "Compaction failed: Another write batch
  … is already active", then a missing `middleware-manifest.json`). Fix: stop all instances,
  `rm -rf .next`, start one. `.next` is a regenerable cache.
- This machine runs dev servers for **other** projects (`Downloads/p web`,
  `Downloads/pub/juicebar-pos`, both Vite). Never kill node processes indiscriminately —
  match on the project path first.

## What's next

The full review (evidence, effort sizes, sequencing) is at
`C:\Users\Ammar Shafi\.claude\plans\i-want-you-to-vectorized-music.md`. Condensed:

**Still broken / half-finished**
- ~~**A5 invoice numbering**~~ — **DONE** (migration 0015, pending application). `generateInvoiceNo()`
  is deleted; `createInvoiceRecord` now calls the `next_invoice_no()` RPC, which bumps a
  per-salon per-day counter under a row lock. Existing numbers were deliberately **not**
  backfilled — renumbering bills clients hold receipts for is a worse audit position than a
  mixed series. Two things found while doing it: the unique constraint on `invoice_no` was
  **global, not per-salon** (so naive sequential numbering would have collided across salons
  every day — 0015 moves it to `(salon_id, invoice_no)`), and the old date stamp came from a
  UTC Node process, so bills between midnight and 05:00 PKT got the previous day's date.
  Known trade-off: the number is allocated in a separate round trip from the INSERT, so a
  failed insert burns a number and leaves a real gap.
- **`appointments.reference_no` has the same collision defect** (`generateReferenceNo()`,
  `reference_no text unique`, 4 random base36) — but **do not fix it the same way.** That
  reference doubles as the access token on the public `/book/manage` page, so a sequence would
  let anyone enumerate other people's bookings. It wants more entropy plus a retry, not a
  counter.
- **A6 refunds** — `invoice_status` includes `refunded` and three screens label it, but
  there's no way to issue one (only void). **Open decision:** what a refund does to stock
  and to the payment record.

**Biggest genuine gaps (in priority order)**
1. **Payroll / commission.** `staff.commission_rate` and `staff.salary_type` are editable in
   `settings/staff/StaffCard.tsx:119-124` and appear *nowhere else in `src/`* — nothing ever
   multiplies anything, so the owner does payroll on paper. All inputs exist
   (`invoice_items.staff_id` + `amount`). **Open decisions:** commission on gross or after
   discount; whether retail lines earn a different rate.
2. **Tips can't be attributed** — `invoices.tip` is invoice-level, `invoice_items` has no tip
   column, so on a two-stylist bill nobody owns the tip. Schema decision; settle it *before*
   building payroll.
3. **Attendance / clock in–out** — no table at all. `staff_shifts` is the roster (intent),
   nothing records hours actually worked. The user explicitly wants this ("staff track").
4. **Reports are locked to the current month** (`finance/page.tsx:31-32` hardcodes it) — no
   range picker, no comparison, and every screen reads near-zero on the 1st.
5. Staff performance stops at revenue — no average ticket, utilisation, retail attach rate,
   rebooking or no-show rate, all derivable from existing data.

**Dead schema worth activating:** `waitlist`, `activity_log` (no audit trail of who voided or
discounted), `notifications`. Also absent entirely: day-end cash-up, discount caps, and
packages/memberships/gift vouchers/client credit.

**Already strong — don't rebuild:** billing and part payments, back-bar consumption
auto-expensing, the atomic `adjust_stock` / `record_payment` functions, rule-driven client
categories, client formulas and before/after photos, and the unit-tested availability engine.

## Working style notes

The user communicates by voice, so transcripts arrive garbled — read for intent and confirm
the interpretation rather than the literal words ("part pair" = part pay, "mini option" was
never resolved). They want suggestions before implementation when they ask for a review, and
they say so explicitly. `/ultrareview` needs a git repo and cannot be launched by the agent.
