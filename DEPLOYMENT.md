# Deploying this app

## The most important thing: it is ONE deployment, not two

The public booking website and the staff software are **the same Next.js
application**, separated only by URL path:

| What | Path | Who sees it |
|---|---|---|
| Public booking website | `/book`, `/book/checkout`, `/book/manage` | Customers, no login |
| Staff software | `/dashboard`, `/bookings`, `/clients`, `/invoices`, `/finance`, `/settings` | Admin / receptionist / staff, login required |
| Login | `/login` | Everyone |

One repo, one build, one Netlify site, one domain. There is nothing to deploy
twice. If you want them on separate domains later, that is a DNS/redirect
concern, not a second deployment.

## Why this CANNOT be a drag-and-drop zip

Your gym app (`Downloads/app/gym-app-deploy.zip`) is 48 files: `index.html`,
an `assets/` folder, a service worker. That is a **static** single-page app.
The browser downloads it and runs everything client-side, so Netlify only has
to serve files. Dragging that zip in works.

This app is not that. The production build reports:

```
29 of 31 routes:  ƒ (Dynamic)  server-rendered on demand
 2 of 31 routes:  ○ (Static)   /login and /_not-found
                  ƒ Proxy (Middleware)
```

There is no `index.html`. Pages are built per request on a server because:

- **Server Actions** (`"use server"`) run booking, billing and payments on the server
- **Auth reads cookies** per request via `proxy.ts` middleware
- **`SUPABASE_SERVICE_ROLE_KEY`** must stay server-side; putting it in a static
  bundle would hand every visitor full database access
- Pages are marked `force-dynamic` because they read live salon data

A static zip of this would be a blank page. Netlify has to **build and run** it,
which is what `netlify.toml` + `@netlify/plugin-nextjs` configure.

## Deploy: the recommended path (Git)

1. **Commit the current state.** The project *is* already a git repo (one
   commit, `main`), but a lot of work sits uncommitted:
   ```bash
   git add .
   git commit -m "Billing, payroll, invoice printing, delete-invoice"
   ```
   Confirm `.env` is ignored — `.gitignore` already covers it. Never commit it.

2. **Push to GitHub**, then in Netlify: *Add new site > Import an existing
   project* and pick the repo. `netlify.toml` sets the build command, publish
   directory and the Next.js plugin, so leave those defaults alone.

3. **Set environment variables** in *Site configuration > Environment variables*:

   | Variable | Where it comes from | Secret? |
   |---|---|---|
   | `NEXT_PUBLIC_SUPABASE_URL` | Supabase > Project Settings > API | no |
   | `NEXT_PUBLIC_SUPABASE_ANON_KEY` | same page | no |
   | `SUPABASE_SERVICE_ROLE_KEY` | same page | **YES — bypasses all RLS** |
   | `NEXT_PUBLIC_SITE_URL` | your live URL, e.g. `https://salon.netlify.app` | no |

4. **Deploy.** Every push to the branch redeploys automatically.

## WhatsApp messaging

Click-to-chat only (wa.me links) — no paid Cloud API, no in-app chat dock, no
webhook. Clicking a WhatsApp button opens the message in WhatsApp itself, ready
to send; a bulk send (Clients > select several > Send WhatsApp) queues one
link per recipient since wa.me is single-recipient. Nothing to configure, no
env vars, no Meta app.

A Cloud API version (server-side sending, delivery receipts, inbound replies
in an app-side thread) was built and then deliberately removed on
2026-08-15 — not needed right now. It was never committed, so there's no
history to recover it from; rebuilding it means starting over (it lived in
`WhatsAppDock.tsx`, `clients/messaging.ts`, `lib/whatsapp-cloud.ts`, and
`api/whatsapp/webhook/route.ts`, for reference).

## Deploy: without Git (the CLI — not drag-and-drop)

**Confirmed 2026-08-15: do not drag the source zip onto Netlify's Deploys page.**
That box does not build anything — it publishes exactly what you drop. For a
static site that's fine; for this app it means no `index.html`, no functions,
100% of routes error. (`salon-netlify-v15.zip` hit this.)

`Downloads/salon-netlify-v<N>.zip` (rebuilt by `tools/build_netlify_zip.py`)
is still useful — as the **source** to unzip and build from locally, or to
upload to GitHub — but it is not something you drop directly.

The working no-Git path is the CLI, which actually runs the build:
```bash
npx netlify-cli login
npx netlify-cli link      # select the existing site, don't create a new one
npx netlify-cli deploy --build --prod
```
Full step-by-step walkthrough in `NETLIFY-SETUP.md`.

The zip deliberately excludes `node_modules`, `.next`, and **`.env`**. It ships
an `.env.example` instead — the service-role key must never travel in a file
you might email or upload.

## If the live site shows "Your project's URL and Key are required"

That is not a broken build. It means the site deployed **without its
environment variables**, so it has no database to talk to. It happened on the
first deploy of `lucky-parfait-a30f71`.

Fix it in this order — the last step is the one people miss:

1. **Netlify > Site configuration > Environment variables**, add all four from
   the table above. The values are in your local `.env` file; copy them exactly,
   with no quotes and no trailing spaces.
2. **Deploy again.** Git-connected site: *Deploys > Trigger deploy > Clear cache
   and deploy site*. No-Git site: `npx netlify-cli deploy --build --prod`.
3. Saving the variables **does nothing on its own**. Anything named
   `NEXT_PUBLIC_*` is compiled into the site at build time, so the already-built
   files still contain the old empty values until a new build runs.

The app now says this itself: instead of a black page with one line of library
text, a missing variable renders a setup screen naming exactly what is absent
(`src/app/error.tsx`). Middleware stands aside when configuration is missing so
that screen can actually render, rather than every request dying as a 500.

**Drag-and-drop onto Netlify's Deploys page does not work for this app at
all** — confirmed 2026-08-15, it produces a 100% error rate because nothing
gets built. Use the CLI (`npx netlify-cli deploy --build --prod`) for a
no-Git deploy. Connecting the GitHub repo takes five minutes once and makes
every later change a `git push`, with no local command to remember.

## After the first deploy

1. **Point `NEXT_PUBLIC_SITE_URL` at the real URL.** It is currently
   `http://localhost:3000`. Nothing reads it in code today, but anything that
   builds an absolute link later will be wrong until it is fixed.
2. **Check Supabase Auth redirect URLs.** Add your Netlify domain under
   *Authentication > URL Configuration*, or password login will bounce.
3. **Run the schema on the production database.** Paste the whole of
   `supabase/merged_migrations.sql` into the Supabase SQL Editor. Do this on a
   fresh project *and* on the existing one — it is idempotent, and the live
   database is currently behind: migrations **0014–0018** have not been applied.
   Until they are, taking a payment, creating a bill, deleting a bill, per-line
   tips and payroll all fail against the live data.
4. **Create a receptionist account** — none exists (see below).
5. **Test the three logins** on the live site: admin, receptionist, staff.

## Known gaps to fix before real customers use it

- **No receptionist account exists.** That role gates most day-to-day billing
  screens and has never been exercised on real data.
- **Sara Ahmed, Hina Malik and "oil" have no login** (`staff.profile_id` is
  null). They can be booked but cannot open `/my-schedule` to see their day.
- **"oil"** has no shifts and no services, so it can never be booked, but it
  still appears on the public site as a staff member.
- **Nobody works Sunday** — no shift rows for day 0, so Sunday shows no
  availability at all. Correct that in Settings if the salon does open.
- **No confirmation message is sent** after a booking. The customer only ever
  sees the code on screen.
