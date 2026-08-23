# Deploy this to Netlify — read this first

**Do not use Netlify's drag-and-drop box for this app. It will publish and
show 100% errors.** Confirmed 2026-08-15: dropping a folder onto an existing
site's Deploys page does **not** run a build — it just publishes whatever you
drop, as-is. That's fine for a static site (like the gym app, which *is* a
finished `index.html` + `assets/` folder). It is fatal here, because this zip
is source code — there is no `index.html`, no server, nothing a browser can
run directly. Every route errors because there is no built app behind it.

**The environment variables also come BEFORE the deploy.** If they're missing,
you get a different failure — a page that loads but shows "Your project's URL
and Key are required to create a Supabase client!" That one *is* a successful
build, just with no database to talk to.

---

## Step 1 — Set the variables (once, before your first real deploy)

Netlify **Site configuration > Environment variables** on `lucky-parfait-a30f71`.
The values are in the `.env` file in your project folder on your PC — open it
in Notepad and copy each one. No quotes, no spaces at the ends.

| Variable | Value |
|---|---|
| `NEXT_PUBLIC_SUPABASE_URL` | from `.env` (Supabase > Project Settings > API) |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | from `.env` — safe to be public |
| `SUPABASE_SERVICE_ROLE_KEY` | from `.env` — **SECRET, bypasses all security** |
| `NEXT_PUBLIC_SITE_URL` | your live address, e.g. `https://lucky-parfait-a30f71.netlify.app` |

> Never put these in a zip you email, upload or share. `.env` is deliberately
> excluded from `salon-netlify-v*.zip`.

## Step 2 — Deploy with the Netlify CLI (builds locally, no Git needed)

This is the no-Git equivalent of drag-and-drop that actually works for a
dynamic Next.js app. Run these in PowerShell, from the project folder
(`c:\Users\Ammar Shafi\Downloads\salon 2`), one time each unless noted:

```powershell
npx netlify-cli login      # opens your browser once, to authorize this machine
npx netlify-cli link       # pick "Use current git remote" -> No -> select lucky-parfait-a30f71 from the list
npx netlify-cli deploy --build --prod   # run this one every time you want to publish
```

(`npx netlify-cli` — not `npx netlify`, which is a different, unrelated npm
package. It isn't installed as a project dependency, so each `npx` call
fetches/reuses a cached copy on demand.)

`--build` makes the CLI run `npm run build` itself (via `netlify.toml` and
`@netlify/plugin-nextjs`) and package the server functions Next.js needs for
its dynamic routes, then uploads the real result. `--prod` publishes it to
your live domain instead of a preview link. Only the last command needs
repeating for future deploys.

If you'd rather not touch a terminal at all, the reliable alternative is
connecting the GitHub repo (see `DEPLOYMENT.md`) so Netlify builds on every
push automatically.

## Step 3 — After the variables change, build AGAIN

This is the step almost everyone misses.

Variables named `NEXT_PUBLIC_*` are **compiled into the site when it builds**.
They are not read fresh on each visit. So if you added the variables after an
upload, the already-built site still contains the old empty values, and it will
keep showing the same error until a new build runs.

- CLI: run `npx netlify-cli deploy --build --prod` again.
- GitHub-connected site: *Deploys > Trigger deploy > Clear cache and deploy site*.

## Step 4 — Set up the database

Open Supabase > **SQL Editor**, paste the entire contents of
`supabase/merged_migrations.sql`, and run it.

It is safe to run on an existing database and safe to run more than once —
every table is `create table if not exists`, every function is
`create or replace`, and nothing in it drops or deletes anything.

Until you do this, taking a payment, creating a bill, deleting a bill and
sending WhatsApp messages will all fail even though the site loads.

## Step 5 — Point Supabase back at the live site

Supabase > **Authentication > URL Configuration** > add your Netlify address to
the redirect URLs, or logging in on the live site will bounce you back to the
login page.

---

## Why this can't be a "just drop the files" zip

Your gym site was static: the browser downloaded the files and ran everything
itself, so Netlify only had to serve them. Dropping that zip onto an existing
site's Deploys page works, because Netlify doesn't need to do anything except
copy files — they're already the finished product.

This app is not that. 29 of its 31 pages are built on the server for each
visitor, because:

- payments, booking and billing run as server actions,
- logging in reads cookies on every request,
- `SUPABASE_SERVICE_ROLE_KEY` must stay on the server — putting it in a file
  the browser downloads would hand every visitor full control of the database.

There is no `index.html` to serve, and dropping the *source* onto Netlify's
existing-site Deploys page does not build it — that box publishes exactly what
you drop, no more. (This was tried on 2026-08-15 and produced a 100% error
rate: real symptom of source files being served as if they were the site.)
Netlify has to build and run this app, which is what `netlify.toml` and
`@netlify/plugin-nextjs` set up — but something has to trigger that build:
either the CLI's `--build` flag (Step 2 above) or a Git-connected site.

## Strongly recommended: connect GitHub instead

The CLI in Step 2 avoids Git entirely, but it does mean running one command
per deploy from this machine. Connecting the repo once turns every later
change into a `git push` and Netlify builds it automatically — no local step
at all, and no risk of deploying a stale copy. See `DEPLOYMENT.md`.
