# The ServiceTitan Sync Blueprint

**by Ricky Graves — Graves Heating & Air, Southern Maryland**

> I synced my ServiceTitan data into my own PostgreSQL database, built my own platform on top of it, and deactivated ServiceTitan in March 2026. My system now runs my whole HVAC company. This document is the blueprint for the first part of that journey — getting **your** data into a database **you** own — written so a beginner with an AI assistant can follow it. Everything in here is the approach my production system actually uses, including the mistakes that cost me real money so you don't repeat them.

---

# PART 1 — READ THIS FIRST (you, the human)

## What you're about to build

A small program that logs into ServiceTitan's official API with your own credentials, downloads your business data — customers, locations, jobs, appointments, invoices, payments, estimates, memberships, installed equipment, and optionally every job photo and file — and stores it in a PostgreSQL database that belongs to you. Then it runs once a day and keeps itself up to date.

It is **read-only**. It pulls data *out* of ServiceTitan. Nothing you build here can write into or change your live ServiceTitan account — we make that impossible at the credential level, not just the code level.

You are not replacing ServiceTitan today. You're doing what I did first: making a full, live copy of your data that you own. What you do with it later — reports, a customer portal, your own CRM, leverage at renewal time — is up to you. Part Two at the end of this document sketches the road I took.

## What you need

1. **A ServiceTitan account** where you (or your admin) can access Settings → Integrations.
2. **A computer** you can install software on. Windows or Mac both work — I built mine on Windows.
3. **Claude Code** — Claude running in your terminal (claude.com/claude-code, included with a Claude Pro plan). This matters: Claude Code works *inside your project folder*, writing the files and running the commands itself while you watch and confirm. Copy-pasting code from a chat window into files by hand is the #1 way beginners lose a day — a chunk lands in the wrong file, half a file gets overwritten, and nothing works with no clue why.
4. **~$30/month**: Claude Pro (~$20), a starter PostgreSQL database on Render (~$7), and a few dollars of Google Cloud Storage if you do the photos phase.
5. **A few evenings.** Setup is an evening. The first full download of your history can run for hours by itself — that's normal, you don't watch it. The daily updates afterward take a couple of minutes.

## Do this ONE thing today

**Request your ServiceTitan API credentials.** You need four values: **Client ID, Client Secret, App Key, Tenant ID**. Getting them involves ServiceTitan's developer portal and can take days if approval is involved — it's the only step in this whole document with a waiting period, and everything is blocked until it's done. Phase 1 walks you (and your AI) through it click by click. If you already have all four: you're ahead, skip the wait.

## How to use this document

1. Create an empty folder on your computer, e.g. `my-servicetitan-sync`.
2. Save this file into that folder as `BLUEPRINT.md`.
3. Open a terminal **in that folder** and run `claude`.
4. Type: **"Read BLUEPRINT.md and follow it exactly. Start with the interview."**

From there, your AI runs the show one small step at a time: it explains what's about to happen in plain English, does it, shows you how to verify it worked, and waits for your confirmation before moving on. Everything below Part 1 is written *to your AI* — you're welcome to read it (the "rules that cost me real money" section is worth your time), but you don't need to understand it for this to work.

## My promises about this blueprint

- **It's proven.** This isn't theory — it's the design my company ran on for over a year, described from the actual code.
- **It's read-only.** Your ServiceTitan account cannot be modified by anything built here.
- **Your secrets stay yours.** Credentials live in a local file that is never uploaded anywhere, and the blueprint sets up the guard rails for that *before the first line of code*.
- **Re-running is always safe.** Every part of the sync can be run twice, ten times, whenever — it never duplicates data. That's a core design decision, not luck.

*— Ricky Graves, Graves Heating & Air, Southern Maryland*

---

# PART 2 — THE WORKING SPEC

**Everything from here down is addressed to the AI assistant (Claude) working with the owner.** Treat this document as your specification. Where it conflicts with your general instincts, this document wins — it encodes decisions proven in production and disasters already paid for.

## 2.1 Who you're working with, and how to behave

The owner is a smart, busy business person and a **beginner at software**. Assume they do not know what a terminal, environment variable, upsert, foreign key, migration, or watermark is until you've explained it. They can follow instructions carefully and they know their business data cold — use that: they can instantly tell you whether a customer count or a revenue number looks right.

**Non-negotiable working rules:**

1. **Interview first.** Before giving a single command, ask:
   - What operating system are they on? (Don't guess — nearly every command depends on it.)
   - Is anything already installed? Check for Node.js, git, and a code editor by having them run version commands; interpret the output for them.
   - Do they have a GitHub account? (Optional but recommended for backup of the *code* — never the secrets.)
   - Do they have the four ServiceTitan credentials yet? (Client ID, Client Secret, App Key, Tenant ID.) If not, Phase 1's credential walkthrough becomes today's priority because of the waiting period.
   - **Which parts of ServiceTitan do they actually use?** Memberships? Equipment tracking? Estimates? Multiple business units? Commercial with projects/purchase orders? Trim the entity list (Appendix A) to what they use — don't build syncs for empty tables.
2. **One step at a time.** Each step: one or two plain-English sentences on what's about to happen and why → the action (you write the file / run the command) → an explicit verification with expected output → **WAIT for the owner to confirm before continuing.** Never deliver five files at once.
3. **Explain before you code.** Introduce every new concept in one or two sentences of plain English at the moment it first matters. Example: *"An 'upsert' is a database write that means 'insert this if it's new, update it if it already exists' — it's what makes re-running our sync safe."*
4. **When something breaks, get the evidence.** Ask the owner to paste the exact error output. Do not guess from a description.
5. **You do the typing.** You write files and run commands directly. The owner's job is decisions and verification. If a step must happen in a web browser (Render, the ServiceTitan portal, Google Cloud), say clearly *"this part happens in your browser"*, walk them through it click by click, and then verify the artifact (a connection string, a key file, a credential) landed where it belongs.
6. **Respect phase gates.** Do not begin a phase until the previous phase's verification checkpoint passed. If the owner wants to skip ahead, explain what breaks without the gate.
7. **Mark the wins.** The first successful API call and the first full sync are genuinely big moments for a non-programmer. Say so.
8. **The current-reality clause.** The ServiceTitan facts in this document were battle-tested in 2025–2026. APIs drift. If the live API contradicts something here — an endpoint 404s, a field is renamed, a portal screen looks different — trust reality: check ServiceTitan's current developer docs (developer.servicetitan.io), adapt, and briefly tell the owner what changed. The *architecture* of this blueprint does not change; only surface details might.

## 2.2 The architecture (fixed decisions — do not redesign)

```
ServiceTitan API  ──►  sync scripts (Node.js)  ──►  YOUR PostgreSQL database
                                                        │
photos & files    ──►  Google Cloud Storage  ◄──  (Phase 7, optional)
                                                        │
                                              (Part Two: your own app)
```

**The stack, fixed:** Node.js (current LTS), the `pg` driver, direct SQL with parameterized queries (`$1, $2` — never string concatenation), `dotenv` for secrets, `axios` for HTTP. **No ORM. No framework. No build step.** This is deliberate — fewer moving parts for a beginner, and it's exactly what the reference system uses in production.

**The project skeleton (mirror it):**

```
my-servicetitan-sync/
├── .env                     # secrets — git-ignored, never committed
├── .gitignore
├── BLUEPRINT.md             # this file
├── CLAUDE.md                # standing safety rules (Appendix C)
├── package.json
├── integrations/
│   └── servicetitan/
│       └── index.js         # auth + one generic apiRequest() wrapper
├── modules/
│   └── sync/
│       ├── index.js         # syncCustomers(), syncJobs(), ... + syncAll()
│       └── attachments.js   # Phase 7: photos & files
├── shared/
│   └── database/
│       ├── index.js         # the db helper (query, findOne, ...)
│       └── migrations/      # 001_customers.sql, 002_..., numbered
└── scripts/
    ├── migrate.js           # migration runner
    ├── test-db.js           # smoke test: database connects
    ├── test-servicetitan.js # smoke test: ST auth works
    └── run-sync.js          # the CLI you actually run
```

**The four pillars.** These four decisions are the whole reason the reference system could eventually cut ServiceTitan loose. Copy them exactly:

1. **The owner's schema, not ServiceTitan's.** Design clean local tables named for the business (customers, jobs, invoices...). Do not mirror ServiceTitan's data model. Every synced table gets: `servicetitan_id BIGINT UNIQUE` (nullable), `created_on`/`modified_on` timestamps, and a `deleted_at` column for soft deletes (mark deleted, never actually delete rows). ServiceTitan's id becomes just a cross-reference column on tables the owner controls — that's what makes the data truly theirs.
2. **One `upsert(table, servicetitanId, data)` helper** used by every sync — `INSERT ... ON CONFLICT (servicetitan_id) DO UPDATE`. One writer, used everywhere. Re-running never duplicates. (Appendix B2.)
3. **One `paginateAll()` helper** — the `hasMore` loop with a ~200ms delay between pages and a hard 500-page safety cap, with progress logging. One pagination implementation, used everywhere. (Appendix B1.)
4. **Pre-loaded lookup maps for relationships.** ServiceTitan records reference each other by *ServiceTitan's* ids (a job carries `customerId`, `locationId`, `businessUnitId`). Before syncing a batch, load `SELECT id, servicetitan_id FROM <table>` into a Map and translate ServiceTitan id → local id on every insert. That translation is what a "foreign key" means here: the job row points at *your* customer row. (Appendix B3.)

**Three supporting behaviors, also fixed:**

- **Dependency order.** Sync parents before children, always in this order: settings (business units, job types, technicians, payment types, tax zones, tags, campaigns) → customers → locations → contacts → jobs → appointments → appointment assignments → invoices (line items arrive embedded) → payments → estimates → memberships → installed equipment. (Full map: Appendix A.)
- **On-demand fill.** When a child references a parent that isn't in the local database yet (a job pointing at an unknown customer), fetch that *single* record from the API, insert it, add it to the map, and continue. **Never silently drop the child row.**
- **Fail per-record, not per-batch.** Wrap each record in its own try/catch; count `created / updated / errors`; keep going; print a summary at the end. One bad record must never kill a 10,000-record sync.

**One database for now.** While ServiceTitan is still the owner's system of record, this local database is a *rebuildable mirror* — if something goes wrong you can wipe it and re-sync. A second (dev) database becomes mandatory the day the owner builds features that **write** data existing nowhere else; that's Part Two (Appendix E). Don't burden them with two databases before then.

## 2.3 The build, phase by phase

Every phase ends with a **verification checkpoint**. Do not proceed past a failed checkpoint.

### Phase 0 — Setup (git first, secrets protected before they exist)

1. Run the interview (§2.1). Record the answers — especially the entity trim list.
2. Install what's missing: Node.js LTS and git, with per-OS instructions. Verify with `node --version` and `git --version` and interpret the output for the owner.
3. Inside the project folder: `git init`, then **write `.gitignore` before anything else**, containing at minimum:
   ```
   .env
   node_modules/
   *.log
   ```
   Explain why in one sentence: *if a secret is ever committed, it lives in git history forever, even after you delete it — so we make it impossible before the secret exists.*
4. Write `CLAUDE.md` from Appendix C — the standing safety rules every future session inherits.
5. `npm init -y`, then `npm install pg dotenv axios`.
6. Create `.env` with placeholders and explain each line:
   ```
   DATABASE_URL=
   ST_CLIENT_ID=
   ST_CLIENT_SECRET=
   ST_APP_KEY=
   ST_TENANT_ID=
   ```
7. First commit (which must NOT include `.env` — show the owner `git status` proving it's ignored). Offer to set up a **private** GitHub repository as an offsite backup of the code; explain that the `.gitignore` keeps secrets out of it.

**✅ Checkpoint:** `node --version` and `git --version` succeed; `git log` shows one commit; `git status` shows `.env` untracked-and-ignored.

### Phase 1 — Credentials + two independent smoke tests

**Getting ServiceTitan API credentials (if the owner doesn't have them).** This happens in the browser; screens change, so follow the current portal (developer.servicetitan.io) — the goal is these four values. The shape of the process:

1. Sign in to ServiceTitan's **developer portal** and create an "app" (this is just a registration, not code). This yields the **App Key**.
2. When choosing the app's API scopes/permissions, select **read-only** scopes, and only for the APIs in Appendix A that the owner actually needs. This is a hard rule: the app must be *incapable* of writing to ServiceTitan. If a screen offers read/write toggles, everything is read.
3. In ServiceTitan itself (the office product, not the portal): **Settings → Integrations → API Application Access**, connect/authorize the new app to the owner's tenant. This is where the **Client ID** and **Client Secret** are issued and where the **Tenant ID** is visible.
4. ⚠️ **The Client Secret is typically shown ONCE.** Have the `.env` file open and paste it straight in.
5. Approval/provisioning may take time — if the app isn't active yet, park here and prep Phase 2's database in the meantime.

**The database (Render).** In the browser: render.com → New → PostgreSQL → the ~$7/mo starter plan (the free tier expires and lacks backups — not suitable for data you care about) → pick the region nearest the owner. Copy the **external connection string** into `DATABASE_URL` in `.env`. Then verify, in Render's dashboard, that the plan includes **automatic backups / point-in-time recovery** — this is Blood Rule #4, and it must be true *before* real data lands. (Any managed Postgres host works if the owner prefers; Render is the reference choice because it's what the proven system runs on.)

**Smoke test A — the database.** Write `scripts/test-db.js`: connect using `DATABASE_URL` (SSL on), run `SELECT NOW()`, print the result, exit. Run it with `node -r dotenv/config scripts/test-db.js`. (That `-r dotenv/config` prefix is how every script loads `.env` — it works identically on Windows and Mac; never tell the owner to `source .env`.)

**Smoke test B — ServiceTitan.** Write `scripts/test-servicetitan.js`: fetch an OAuth token (§2.4), then request **one page of one customer** (`/crm/v2/tenant/{tenant}/customers?page=1&pageSize=1`) and print the customer's name. Prove the two halves separately before combining them — when something fails later, the owner already knows both ends work.

**✅ Checkpoint:** both smoke tests pass. The owner just saw a real customer of theirs come back from the API — say congratulations, because from here it's all downhill.

### Phase 2 — Foundation (db helper, migrations, first tables)

1. **The db helper** (`shared/database/index.js`): a `pg` Pool (SSL on) exposing `query(text, params)`, `getClient()` for transactions, and `findOne(table, conditions)`. All database access goes through it. (Appendix B6.)
2. **The migration system.** Explain in plain English: *a migration is a numbered SQL file that changes the database's shape; a tracker table records which ones have run, so the database's structure is version-controlled just like code — we never change tables by hand.* Build `scripts/migrate.js`: reads `shared/database/migrations/*.sql` in number order, diffs against a `schema_migrations` tracker table, applies what's missing (each file in a transaction), supports `--dry-run`. (Appendix B7.) Two rules from the reference system's scars: **never renumber or edit an already-applied migration** (add a new one instead), and check the folder for the current max number before creating a file (duplicate numbers made the reference system's sequence untrustworthy).
3. **Migration 001:** the `customers` table and the `sync_log` table (exact SQL: Appendix B8). Walk the owner through the column conventions from §2.2 — this table shape repeats for every entity. Introduce `sync_log` in plain English: *the sync's diary — every run writes one row per entity saying when it ran, what it did, and whether it succeeded. It's how tomorrow's run knows where yesterday's left off, and how you check on the sync without reading logs.*
4. Run the migration, then prove it: `--dry-run` first, then apply, then `SELECT * FROM schema_migrations;`.

**✅ Checkpoint:** migration applied; `\d customers` (or an information_schema query) shows the expected columns; re-running `migrate.js` reports "nothing to do."

### Phase 3 — Customers, end to end (the phase where all the learning happens)

Build in this order, explaining each piece as it lands:

1. **The ServiceTitan client** (`integrations/servicetitan/index.js`): token fetch + in-memory cache with a 60-second refresh buffer, and one generic `apiRequest()` that stamps every call with the three required things (§2.4). (Appendix B1.)
2. **`paginateAll()`** and **`upsert()`** in `modules/sync/index.js`. (Appendix B1, B2.)
3. **`syncCustomers(options)`**: paginate `/crm/v2/tenant/{tenant}/customers` (passing `modifiedOnOrAfter` when `options` provides it), map each ServiceTitan customer to the local column shape, `upsert` each one inside a per-record try/catch, count created/updated/errors, print a summary, write a `sync_log` row. (Appendix B4 is the canonical example — every later entity follows its shape.)
4. **First real run** — the full customer pull. Set expectations first: a few thousand customers takes a few minutes because of the polite 200ms page delay.

**✅ Checkpoint — all three, in order:**
- `SELECT COUNT(*) FROM customers;` roughly matches the customer count the owner sees in ServiceTitan (ask them to check the ST screen — they know where).
- Spot-check: the owner names a customer; `SELECT name, address_street, address_city FROM customers WHERE name ILIKE '%<name>%';` returns them with a correct address.
- **The idempotency proof:** run the sync a second time. Explain what to watch for: same total count, `0 created / N updated`. That's "idempotent" — safe to re-run forever — and it's the property everything else builds on.

One honest note to pass to the owner: customer phone numbers and emails may look sparse until the **contacts** wave lands in Phase 4 — modern ServiceTitan keeps them on a separate contacts record, and the reference system fills customer phone/email from contacts.

### Phase 4 — The rest of the entities, in waves

Each wave = migration(s) → sync function(s) modeled on `syncCustomers` → full run → verify → commit. Between waves, ask the owner to sanity-check a number they know (job count, an invoice total). Skip anything the interview trimmed. Wave-by-wave notes — these encode the reference system's landmines, don't relearn them:

**Wave 1 — settings & lookup tables** (small, fast, everything else references them): `business_units`, `job_types`, `employees` (from `/settings/v2/.../technicians` — includes positions/skills), `payment_types`, `tax_zones`, `tag_types`, `campaigns`. No `modifiedOnOrAfter` needed — they're small enough to pull in full every run.

**Wave 2 — locations & contacts:**
- `locations`: carries `customerId` → translate via the customer lookup map. Store address parts *and* `latitude`/`longitude` if present.
- `contacts`: **the bulk endpoint returns 403** — that's permanent, not a bug. Fetch per-customer instead: `/crm/v2/tenant/{tenant}/customers/{id}/contacts`, in batches of ~30 customers with ~50ms staggered starts inside the batch and ~500ms between batches (Appendix B5). Individual fetches for inactive customers return **409 — treat as "skip", never "error."** After saving contacts, backfill each customer's primary `email`/`phone_number` from them.

**Wave 3 — jobs, appointments, assignments:**
- `jobs`: the wave where lookup maps earn their keep — customer, location, job type, business unit, campaign, tags all translate. Use **on-demand fill** for missing customers/locations (§2.2). Store the fields the owner will report on: number, status, summary, scheduled/completed times, total. If a job carries `tagTypeIds`, maintain a `job_tags` join table (delete-and-reinsert per job is fine).
- `appointments`: carries `jobId` → job map; store start/end and the customer-promised `arrival_window_start/end` — **they are different things; never copy one onto the other.**
- `job_assignments` (who was assigned): from `/dispatch/v2/tenant/{tenant}/appointment-assignments`, mapping technician → employees.
- After the wave, run an **orphan check**: `SELECT COUNT(*) FROM jobs WHERE customer_id IS NULL;` — a handful is normal (deleted customers); hundreds means a map bug.

**Wave 4 — the money:**
- `invoices`: **line items arrive embedded in the invoice response** — upsert them into `invoice_items` in the same pass; there is no separate fetch needed. Store subtotal, tax, total, balance, dates. Use NUMERIC columns for money, never floating point.
- `payments`: the shape everyone trips on — the customer is **nested** (`payment.customer.id`) and the invoice it applies to is inside an **`appliedTo[]` array** (`appliedTo[0].appliedTo` is the invoice's ServiceTitan id), not top-level. Also guard the nonsense date ServiceTitan sometimes sends (`0001-01-01T00:00:00Z`) — treat it as null. After payments land, recalculate each affected invoice's balance from its payments.
- `estimates`: from `/sales/v2/tenant/{tenant}/estimates`. **`status` arrives as an object** (`{ value, name }`) — store `status.name`. Items arrive embedded; upsert in the same pass. Estimates referencing a job you don't have get skipped and counted, not crashed on.

**Wave 5 — memberships & equipment:**
- `memberships`: membership **type names must be fetched individually** (`/memberships/v2/.../membership-types/{id}`) because inactive types don't appear in list calls — cache them per run. Normalize status; if ServiceTitan marks one deleted/inactive, set `deleted_at` (soft delete). Membership from/to are **date-only values — store in DATE columns** (see Blood Rule #7). If memberships carry `renewedById`, resolve those links in a post-pass after all rows exist.
- `installed_equipment`: from `/equipmentsystems/v2/tenant/{tenant}/installed-equipment` — manufacturer, model, serial, install date, per location. For an HVAC/plumbing/electrical shop this table is gold: it's the "aging systems to quote a replacement" list.

**Wave 6 — optional extras** (only if the interview said they're used): call logs (`/telecom/v2/.../calls`), job notes (**no unique id on ST notes** — dedupe by job + created timestamp; 404 on deleted jobs = skip), projects (`/jpm/v2/.../projects`), purchase orders (`/inventory/v2/.../purchase-orders` — items embedded; translate `vendorId` through a vendor map, storing raw ServiceTitan ids in local FK columns is the reference system's longest-lived data bug), timesheets (endpoint under the payroll API — check current docs).

Finish the phase with **`syncAll()`**: runs every entity in dependency order, each step isolated so one failure doesn't stop the rest, prints a grand summary (Appendix B4 note). And **`scripts/run-sync.js`** — the CLI the owner actually uses: `node -r dotenv/config scripts/run-sync.js` (full), `--recent` (incremental), `--customers`/`--jobs`/etc. (one entity). It validates the five env vars before doing anything.

**✅ Checkpoint:** a full `run-sync.js` completes; the summary shows errors at or near zero; the owner confirms job count and a recent invoice's total against the ServiceTitan screen; `SELECT * FROM sync_log ORDER BY id DESC LIMIT 20;` shows a row per entity.

### Phase 5 — The daily incremental sync

**How incremental works (explain to the owner):** every list endpoint accepts `modifiedOnOrAfter` — "only give me what changed since this moment." The first run pulled everything; from now on, each run pulls a small window and the upserts make overlap harmless.

**The window rule (this is the reference system's actual production behavior):** each daily run pulls **the last 24 hours** of changes — not a razor-thin "since the exact last run" watermark. Overlap costs nothing (idempotent upserts) and generous windows catch records ServiceTitan touched without reliably bumping `modifiedOn`. One improvement on top: read `sync_log` first, and **if the last successful run is older than ~24h, widen the window to cover the whole gap** (laptop was off for three days → that run pulls three days). Small lookup tables just re-pull in full every run. (Appendix B8.)

**Scheduling — offer three tiers and let the owner pick:**
1. **Manual** (start here): run `node -r dotenv/config scripts/run-sync.js --recent` with morning coffee. Zero setup, and thanks to the gap-widening window, missed days heal themselves.
2. **Their computer runs it**: Windows Task Scheduler (a daily task; note it only fires while the machine is on/awake) or Mac `launchd`/cron.
3. **The cloud runs it** (the graduation move): a Render Cron Job pointed at the GitHub repo, running the same command daily — it fires whether their laptop is on or not, and it's the natural step if they're headed toward Part Two anyway. Requires the (private) GitHub repo from Phase 0 and the env vars entered into Render's dashboard — walk them through that securely; secrets go into Render's environment settings, never into git.

**Backups, re-verified:** confirm point-in-time recovery is active on the database (Blood Rule #4). Additionally teach the owner the one-line safety net and how to run it monthly, or automate it:
```
pg_dump "<DATABASE_URL>" > backup-YYYY-MM-DD.sql
```
That file is their entire business history in a single portable file. It's also the moment to say out loud: **this database is now worth protecting.**

**✅ Checkpoint:** two consecutive daily runs in `sync_log` with status `success`; a change made in ServiceTitan today (e.g. edit a customer's name) shows up in the local database after the next `--recent` run.

### Phase 6 — The payoff: ask your data anything

The whole point, made tangible. Set the owner up with a friendly way in — either you querying on their behalf in Claude Code, and/or a free GUI (DBeaver, TablePlus, or pgAdmin) for browsing — then run their history through questions like:

```sql
-- Revenue by month, all time
SELECT date_trunc('month', invoice_date) AS month,
       SUM(total_amount) AS revenue
FROM invoices
WHERE deleted_at IS NULL
GROUP BY 1 ORDER BY 1;

-- Top 25 customers by lifetime revenue
SELECT c.name, SUM(i.total_amount) AS lifetime
FROM invoices i JOIN customers c ON c.id = i.customer_id
GROUP BY c.name ORDER BY lifetime DESC LIMIT 25;

-- Systems 12+ years old (replacement conversations waiting to happen)
SELECT c.name, l.address_street, e.manufacturer, e.model_number, e.install_date
FROM installed_equipment e
JOIN locations l ON l.id = e.location_id
JOIN customers c ON c.id = l.customer_id
WHERE e.install_date < NOW() - INTERVAL '12 years'
ORDER BY e.install_date;

-- Memberships expiring in the next 60 days
SELECT c.name, m.membership_type_name, m.to_date
FROM memberships m JOIN customers c ON c.id = m.customer_id
WHERE m.status = 'Active' AND m.to_date BETWEEN NOW() AND NOW() + INTERVAL '60 days'
ORDER BY m.to_date;
```

Adjust to their schema and their questions. This is where the owner *feels* the ownership — every one of these took a support ticket or an export in the old world.

**✅ Checkpoint:** the owner has asked at least three questions of their own data and believes the answers.

### Phase 7 — Photos & files (optional, after the daily sync is stable)

Job photos and attachments are the other half of "own your data" — and they work completely differently from records, which is why they're their own phase:

- **There is no bulk attachments list and no `modifiedOnOrAfter` for files.** You crawl **job by job**: `GET /forms/v2/tenant/{tenant}/jobs/{jobId}/attachments` lists a job's files (404 = the job has none / is gone — skip), then `GET /forms/v2/tenant/{tenant}/jobs/attachment/{attachmentId}` downloads the raw bytes (request as binary/arraybuffer).
- **Files don't go in the database.** The database gets a `job_attachments` row per file (which job, original filename, type, size, where it's stored); the bytes go to **Google Cloud Storage** — the reference choice, proven in production. Storage path convention: `jobs/<local job id>/<attachmentId>.<ext>`. Cost reality for the owner: a shop's photo history is typically a few dollars a month.
- **GCS setup** (browser, click-by-click): create a Google Cloud project → enable billing → create a **private** bucket (uniform access, no public access) → create a service account with Storage Object Admin on that bucket → download its JSON key → reference it from `.env` (add `GCS_BUCKET_NAME` and the key file path — and add the key file to `.gitignore` **before** downloading it). Then a smoke test: upload a tiny text file, generate a signed URL, open it.
- **The crawl, with the reference system's two tricks:**
  1. **The placeholder trick.** After checking a job that has zero attachments, insert a `NO_ATTACHMENTS` placeholder row for it — so the crawler never wastes an API call re-checking that job. Exception: jobs from the **last 7 days** with only a placeholder DO get re-checked, because technicians upload photos after the job first syncs.
  2. **Two crawl modes.** A *recent* pass (jobs from the last ~30 days) that runs with the daily sync, and a *backlog* pass (any never-checked job, ~500 jobs per run, ~100ms between jobs) that the owner re-runs — or schedules hourly — until the entire history is drained. With years of history this takes days of polite crawling in the background. That's fine.
- **Viewing:** files in a private bucket are accessed via short-lived **signed URLs** (the reference uses 60-minute expiry) generated on demand — the bucket never goes public. (Full reference implementation: Appendix B9.)
- **Honesty note, from the reference system's real experience:** the API crawl may not surface *every* historical file. When the reference company deactivated ServiceTitan, it obtained a full backup export from ServiceTitan — a multi-gigabyte archive with every table as CSV **plus attachments** — which recovered a couple hundred files the API had never returned. Tell the owner this export exists; when they're serious about completeness (or about leaving), they should ask ServiceTitan for it.

**✅ Checkpoint:** the recent crawl runs clean; `SELECT COUNT(*) FROM job_attachments WHERE file_type != 'none';` is growing; the owner opens a signed URL and sees an actual photo from an actual job. (This one lands emotionally — it's *their techs' photos* in *their* bucket.)

## 2.4 ServiceTitan API field guide (hard-won facts — don't relearn them)

**Authentication.** OAuth2 client-credentials: form-urlencoded `POST https://auth.servicetitan.io/connect/token` with `grant_type=client_credentials`, `client_id`, `client_secret`. The token lives ~15 minutes. **Cache it in memory and refresh with a ~60-second buffer** — never re-authenticate per call, and never write the token to disk.

**Every API call needs three things** — #2 is the one everyone misses:
1. `Authorization: Bearer <token>` (header)
2. `ST-App-Key: <app key>` (header — nothing works without it, and the error won't tell you that's why)
3. The **tenant id in the URL path**: `https://api.servicetitan.io/crm/v2/tenant/{tenantId}/customers`

**Pagination contract.** `page` + `pageSize` (max 100) on every list endpoint; responses carry `data[]` and a `hasMore` boolean. **Loop on `hasMore`, never on whether the page came back full.** ~200ms between pages; hard cap at 500 pages so a misbehaving response can't spin forever (if the cap ever fires, something is wrong — investigate, don't raise it blindly).

**Incremental.** `modifiedOnOrAfter=<ISO timestamp>` on every list endpoint. Caveat that shaped the reference design: ServiceTitan doesn't reliably bump `modifiedOn` for every kind of change — hence generous overlapping windows (§ Phase 5), not razor-thin watermarks.

**The quirks table:**

| Quirk | What to do |
|---|---|
| Bulk contacts endpoint returns **403** | Permanent. Fetch per-customer (`/customers/{id}/contacts`) in batches of ~30, ~50ms staggered starts, ~500ms between batches |
| **409** fetching an individual inactive/deleted record | Skip and count it, don't error |
| **404** on a job's notes/attachments | Job deleted/archived in ST — skip |
| Payment shape | Customer is nested `customer.id`; invoice is `appliedTo[0].appliedTo`; guard `0001-01-01T00:00:00Z` dates |
| Invoice / estimate / purchase-order **line items arrive embedded** | Upsert them in the same pass as the parent — no separate fetch |
| Estimate `status` is an object | Store `status.name` |
| Job notes have **no unique id** | Dedupe by job + created timestamp |
| Inactive membership types missing from list calls | Fetch types individually by id, cache per run |
| Attachments have no list-all or modified filter | Crawl per job (Phase 7) |
| **429 / rate limiting** | The pacing above prevented it entirely in production. If one ever appears: honor `Retry-After` if present, otherwise double the delays |

## 2.5 The rules that cost real money (binding, non-negotiable)

These come from disasters the reference system actually paid for. They bind **you**, the AI — enforce them even when the owner asks for a shortcut.

1. **NEVER derive a real business event time from a "last modified" timestamp.** Modified timestamps move whenever *any* process touches a row. The reference system backfilled job completion dates from `modified_on` as a fallback and silently overwrote **~5,800 real completion dates** — some by 5+ years — keeping no copy. Recovery required a point-in-time database restore. If the true source column is null, **leave the value alone.**
2. **Any script that overwrites existing data must preserve the old values first** — snapshot them to a file or backup column — so the change is reversible without a restore. When unsure, make it a no-op.
3. **Every data-changing maintenance script defaults to dry-run** — it prints what it *would* do and changes nothing until an explicit `--apply` flag. (The sync itself is exempt: it's the designed, idempotent write path.)
4. **Automatic backups / point-in-time recovery must be verified ON before real data lands.** This is what saved the reference system. Teach `pg_dump` as the belt-and-suspenders.
5. **One piece of logic = one place.** The reference system accumulated ~28 hand-written copies of the same insert across 12 files; they drifted and caused real billing bugs, and consolidating them into one writer function was the fix. If logic would live in two places, extract it into one function *first* — the refactor is the fix.
6. **Store text raw; decode on the way in; escape only at display.** External data sometimes arrives pre-escaped (`&amp;` instead of `&`). Stored as-is, it eventually renders as literal `&amp;` in front of a customer.
7. **One business timezone, and date-only values are landmines.** Parsing a date-only string (`"2026-07-14"`) through a timezone-aware formatter is a classic off-by-one-day bug. Date-only facts (membership from/to) go in DATE columns and never round-trip through `new Date('YYYY-MM-DD')` + a timezone-aware formatter. Warn the owner whenever you're near this.
8. **Read-only credentials, forever.** This project pulls data out. If a future feature ever needs to write to ServiceTitan, that's a separate app with separate credentials and a separate conversation — never a scope added to this one.
9. **All SQL is parameterized (`$1, $2`).** Never build SQL by concatenating values into the string.

---

# PART 3 — APPENDICES

## Appendix A — Entity map (endpoint → table → wave)

| Entity | ServiceTitan endpoint (`/tenant/{tenantId}/` implied) | Local table | Wave |
|---|---|---|---|
| Business units | `/settings/v2/.../business-units` | `business_units` | 1 |
| Job types | `/jpm/v2/.../job-types` | `job_types` | 1 |
| Technicians | `/settings/v2/.../technicians` | `employees` | 1 |
| Payment types | `/accounting/v2/.../payment-types` | `payment_types` | 1 |
| Tax zones | `/accounting/v2/.../tax-zones` | `tax_zones` | 1 |
| Tag types | `/settings/v2/.../tag-types` | `tag_types` | 1 |
| Campaigns | `/marketing/v2/.../campaigns` | `campaigns` | 1 |
| Customers | `/crm/v2/.../customers` | `customers` | Phase 3 |
| Locations | `/crm/v2/.../locations` | `locations` | 2 |
| Contacts | `/crm/v2/.../customers/{id}/contacts` (per-customer!) | `contacts` | 2 |
| Jobs | `/jpm/v2/.../jobs` | `jobs` (+ `job_tags`) | 3 |
| Appointments | `/jpm/v2/.../appointments` | `appointments` | 3 |
| Assignments | `/dispatch/v2/.../appointment-assignments` | `job_assignments` | 3 |
| Invoices + items | `/accounting/v2/.../invoices` (items embedded) | `invoices`, `invoice_items` | 4 |
| Payments | `/accounting/v2/.../payments` | `payments` | 4 |
| Estimates + items | `/sales/v2/.../estimates` (items embedded) | `estimates`, `estimate_items` | 4 |
| Memberships | `/memberships/v2/.../memberships` (+ `/membership-types/{id}`) | `memberships` | 5 |
| Installed equipment | `/equipmentsystems/v2/.../installed-equipment` | `installed_equipment` | 5 |
| Call logs *(optional)* | `/telecom/v2/.../calls` | `call_logs` | 6 |
| Job notes *(optional)* | `/jpm/v2/.../jobs/{id}/notes` (per-job, no unique id) | `notes` | 6 |
| Projects *(optional)* | `/jpm/v2/.../projects` | `projects` | 6 |
| Purchase orders *(optional)* | `/inventory/v2/.../purchase-orders` (items embedded) | `purchase_orders`, `purchase_order_items` | 6 |
| Job attachments *(optional)* | `/forms/v2/.../jobs/{id}/attachments` + `/forms/v2/.../jobs/attachment/{attId}` | `job_attachments` + cloud storage | 7 |

## Appendix B — Reference implementations

Adapted from the production system this blueprint describes (sanitized; simplified where the original carried historical baggage). **These are references, not paste targets** — the AI writes the owner's version, explaining as it goes, and may adapt names and columns to the owner's business. The *behavior* is the contract.

### B1 — ServiceTitan client: token cache + `apiRequest` + `paginateAll`

```js
// integrations/servicetitan/index.js
const axios = require('axios');

let accessToken = null;
let tokenExpiry = null;

async function getAccessToken() {
  // Reuse the cached token until 60s before it expires
  if (accessToken && tokenExpiry && Date.now() < tokenExpiry - 60_000) return accessToken;

  const res = await axios.post(
    'https://auth.servicetitan.io/connect/token',
    new URLSearchParams({
      grant_type: 'client_credentials',
      client_id: process.env.ST_CLIENT_ID,
      client_secret: process.env.ST_CLIENT_SECRET,
    }),
    { headers: { 'Content-Type': 'application/x-www-form-urlencoded' }, timeout: 30_000 }
  );
  accessToken = res.data.access_token;              // token lives ~15 minutes
  tokenExpiry = Date.now() + res.data.expires_in * 1000;
  return accessToken;
}

// One generic request wrapper — every call goes through here,
// so the three required pieces can never be forgotten.
async function apiRequest(method, endpoint, { params = null, responseType = 'json' } = {}) {
  const token = await getAccessToken();
  const url = 'https://api.servicetitan.io' +
    endpoint.replace('{tenant}', process.env.ST_TENANT_ID);

  const res = await axios({
    method, url, params, responseType,
    headers: {
      Authorization: `Bearer ${token}`,
      'ST-App-Key': process.env.ST_APP_KEY,
    },
    timeout: 60_000,
  });
  return res.data;
}

// The hasMore loop: polite delay, hard safety cap, progress logging.
async function paginateAll(fetchPage, params = {}, pageSize = 100, label = '') {
  const all = [];
  let page = 1, hasMore = true;
  const MAX_PAGES = 500;

  while (hasMore && page <= MAX_PAGES) {
    const res = await fetchPage({ ...params, page, pageSize });
    const data = res.data || [];
    all.push(...data);
    if (label) console.log(`[SYNC]   ${label} page ${page}: ${data.length} records (total ${all.length})`);
    hasMore = res.hasMore !== undefined ? res.hasMore : data.length === pageSize;
    page++;
    if (hasMore) await new Promise(r => setTimeout(r, 200)); // rate-limit courtesy
  }
  if (page > MAX_PAGES) console.warn(`[SYNC] ${label}: hit the ${MAX_PAGES}-page safety cap — investigate.`);
  return all;
}

module.exports = { getAccessToken, apiRequest, paginateAll };
```

### B2 — The one upsert helper (every table write goes through this)

```js
// modules/sync/index.js
const db = require('../../shared/database');

// "Insert if new, update if it already exists" — keyed on servicetitan_id.
// The UNIQUE index on servicetitan_id (created in each table's migration)
// is what makes ON CONFLICT work. Table/column names come from OUR code,
// never from user input; the VALUES are parameterized.
async function upsert(table, servicetitanId, data) {
  const keys = Object.keys(data);
  const values = Object.values(data);
  const allKeys = ['servicetitan_id', ...keys];
  const allValues = [servicetitanId, ...values];
  const placeholders = allValues.map((_, i) => `$${i + 1}`).join(', ');
  const updates = keys.map((k, i) => `${k} = $${i + 2}`).join(', ');

  const result = await db.query(
    `INSERT INTO ${table} (${allKeys.join(', ')})
     VALUES (${placeholders})
     ON CONFLICT (servicetitan_id) DO UPDATE SET ${updates}, modified_on = NOW()
     RETURNING (xmax = 0) AS inserted, *`,
    allValues
  );
  const row = result.rows[0];
  return { action: row.inserted ? 'created' : 'updated', record: row };
}
```

### B3 — Lookup maps + on-demand fill (how relationships resolve)

```js
// Before syncing a batch of children, load the parents' id translations once:
async function loadMap(table) {
  const res = await db.query(
    `SELECT id, servicetitan_id FROM ${table} WHERE servicetitan_id IS NOT NULL`
  );
  return new Map(res.rows.map(r => [String(r.servicetitan_id), r.id]));
}

// Inside the jobs loop:
//   let customerId = customerMap.get(String(stJob.customerId)) || null;
//
// On-demand fill — the job references a customer we've never seen:
//   if (!customerId && stJob.customerId) {
//     const rec = await syncCustomerById(stJob.customerId);  // fetch ONE record from ST, upsert it
//     if (rec) { customerId = rec.id; customerMap.set(String(stJob.customerId), rec.id); }
//   }
// Never silently drop the child row because its parent was missing.
```

### B4 — The canonical entity sync (every entity copies this shape)

```js
async function syncCustomers(options = {}) {
  console.log('[SYNC] Starting customer sync...');
  const stats = { total: 0, created: 0, updated: 0, errors: 0 };
  const started = new Date();

  const params = {};
  if (options.modifiedOnOrAfter) params.modifiedOnOrAfter = options.modifiedOnOrAfter;

  const customers = await paginateAll(
    p => st.apiRequest('GET', '/crm/v2/tenant/{tenant}/customers', { params: p }),
    params, 100, 'Customers'
  );
  stats.total = customers.length;

  for (const c of customers) {
    try {                                    // fail per-record, not per-batch
      const data = {
        name: c.name || `${c.firstName || ''} ${c.lastName || ''}`.trim(),
        type: c.type || 'Residential',
        address_street: c.address?.street || null,
        address_unit:   c.address?.unit || null,
        address_city:   c.address?.city || null,
        address_state:  c.address?.state || null,
        address_zip:    c.address?.zip || null,
        do_not_service: c.doNotService || false,
        balance:        c.balance || 0,
        custom_fields:  c.customFields ? JSON.stringify(c.customFields) : null,
        created_on:     c.createdOn ? new Date(c.createdOn) : new Date(),
      };
      const result = await upsert('customers', c.id, data);
      result.action === 'created' ? stats.created++ : stats.updated++;
    } catch (err) {
      console.error(`[SYNC] customer ${c.id}: ${err.message}`);
      stats.errors++;
    }
  }

  console.log(`[SYNC] Customers: ${stats.created} created, ${stats.updated} updated, ${stats.errors} errors`);
  await logSync('customers', stats, started);   // B8 — the sync's diary
  return stats;
}
```

`syncAll()` then calls every entity's function **in dependency order**, each wrapped so a failure logs and continues:

```js
async function runStep(results, name, fn) {
  try { results[name] = await fn(); }
  catch (e) {
    console.error(`[SYNC] ${name} FAILED, continuing: ${e.message}`);
    results[name] = { created: 0, updated: 0, errors: 1 };
  }
}
```

### B5 — The contacts crawl (bulk endpoint is 403 — this is the workaround)

```js
const BATCH = 30, STAGGER_MS = 50;

async function syncContacts(options = {}) {
  // Which customers to check: all on a full run; only recently-modified ones on incremental
  const { rows: customers } = await db.query(
    options.modifiedOnOrAfter
      ? `SELECT id AS local_id, servicetitan_id FROM customers
         WHERE servicetitan_id IS NOT NULL AND modified_on >= $1`
      : `SELECT id AS local_id, servicetitan_id FROM customers
         WHERE servicetitan_id IS NOT NULL`,
    options.modifiedOnOrAfter ? [options.modifiedOnOrAfter] : []
  );

  const stats = { created: 0, updated: 0, skipped: 0, errors: 0 };

  for (let i = 0; i < customers.length; i += BATCH) {
    const batch = customers.slice(i, i + BATCH);
    await Promise.all(batch.map(async (cust, idx) => {
      await new Promise(r => setTimeout(r, idx * STAGGER_MS));   // staggered starts
      try {
        const res = await st.apiRequest('GET',
          `/crm/v2/tenant/{tenant}/customers/${cust.servicetitan_id}/contacts`);
        for (const contact of (res.data || [])) {
          const email = contact.type === 'Email' ? contact.value : null;
          const phone = ['Phone', 'MobilePhone'].includes(contact.type) ? contact.value : null;
          await upsert('contacts', contact.id, {
            customer_id: cust.local_id,
            type: contact.type || 'Contact',
            email, phone_number: phone,
            memo: contact.memo || null,
          });
          stats.created++; // (split created/updated via the upsert result if you care)
        }
        // then backfill customers.email / customers.phone_number from what arrived
      } catch (err) {
        if (err.response?.status === 409) { stats.skipped++; return; } // inactive customer — skip
        stats.errors++;
      }
    }));
    await new Promise(r => setTimeout(r, 500));                  // breather between batches
  }
  return stats;
}
```

### B6 — The db helper

```js
// shared/database/index.js
const { Pool } = require('pg');

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  ssl: { rejectUnauthorized: false },   // hosted Postgres (Render) requires SSL
  max: 10,
});

async function query(text, params) { return pool.query(text, params); }
async function getClient() { return pool.connect(); }   // for BEGIN/COMMIT transactions

async function findOne(table, conditions) {
  const keys = Object.keys(conditions);
  const where = keys.map((k, i) => `${k} = $${i + 1}`).join(' AND ');
  const res = await query(`SELECT * FROM ${table} WHERE ${where} LIMIT 1`, Object.values(conditions));
  return res.rows[0] || null;
}

module.exports = { query, getClient, findOne, pool };
```

### B7 — The migration runner (sketch)

```js
// scripts/migrate.js — run with: node -r dotenv/config scripts/migrate.js [--dry-run]
// 1. CREATE TABLE IF NOT EXISTS schema_migrations (filename TEXT PRIMARY KEY, applied_at TIMESTAMPTZ DEFAULT NOW())
// 2. Read shared/database/migrations/*.sql, sorted by filename (NNN_ prefix)
// 3. Skip files already in schema_migrations
// 4. --dry-run: print what WOULD run, exit
// 5. Otherwise, per file: BEGIN → run the SQL → INSERT INTO schema_migrations → COMMIT
//    (any error → ROLLBACK and stop, so a half-applied file never counts as applied)
// Rules: never edit or renumber an applied migration — add a new file.
```

### B8 — Migration 001 + the sync diary

```sql
-- shared/database/migrations/001_customers_and_sync_log.sql

CREATE TABLE customers (
  id               BIGSERIAL PRIMARY KEY,
  servicetitan_id  BIGINT UNIQUE,                -- the cross-reference to ST; ours is `id`
  name             TEXT NOT NULL,
  type             TEXT,                          -- Residential / Commercial
  address_street   TEXT,
  address_unit     TEXT,
  address_city     TEXT,
  address_state    TEXT,
  address_zip      TEXT,
  email            TEXT,
  phone_number     TEXT,
  do_not_service   BOOLEAN DEFAULT FALSE,
  balance          NUMERIC(12,2) DEFAULT 0,       -- money is NUMERIC, never float
  custom_fields    JSONB,
  created_on       TIMESTAMPTZ,                   -- from ServiceTitan's createdOn
  modified_on      TIMESTAMPTZ DEFAULT NOW(),     -- when WE last touched the row (machine time, not a business fact!)
  deleted_at       TIMESTAMPTZ                    -- soft delete: set, never DELETE
);

CREATE TABLE sync_log (
  id                 BIGSERIAL PRIMARY KEY,
  sync_type          TEXT,          -- 'full' | 'incremental'
  table_name         TEXT,          -- which entity
  records_processed  INTEGER,
  records_created    INTEGER,
  records_updated    INTEGER,
  started_at         TIMESTAMPTZ,
  completed_at       TIMESTAMPTZ,
  status             TEXT,          -- 'success' | 'partial' | 'failed'
  error_message      TEXT
);
```

```js
// The diary writer + the incremental window (Phase 5's rule)
async function logSync(entity, stats, startedAt) {
  await db.query(
    `INSERT INTO sync_log (sync_type, table_name, records_processed, records_created,
                           records_updated, started_at, completed_at, status)
     VALUES ($1, $2, $3, $4, $5, $6, NOW(), $7)`,
    [stats.syncType || 'incremental', entity,
     (stats.created || 0) + (stats.updated || 0) + (stats.errors || 0),
     stats.created || 0, stats.updated || 0, startedAt,
     (stats.errors || 0) > 0 ? 'partial' : 'success']
  ).catch(e => console.log(`[SYNC] could not write sync_log: ${e.message}`));
}

async function getLastSyncTime(entity) {
  const res = await db.query(
    `SELECT MAX(completed_at) AS last FROM sync_log
     WHERE table_name = $1 AND status IN ('success','partial')`, [entity]);
  return res.rows[0]?.last || null;
}

// The window: at least the last 24h (generous overlap is FREE thanks to upserts,
// and it catches changes ST doesn't reliably stamp modifiedOn for).
// If the last success is older than that — laptop was off — widen to cover the gap.
async function getSince(entity) {
  const last = await getLastSyncTime(entity);
  if (!last) return null;                                  // first ever run → full pull
  const dayAgo = Date.now() - 24 * 3600 * 1000;
  const gapCover = new Date(last).getTime() - 3600 * 1000; // last success minus 1h overlap
  return new Date(Math.min(dayAgo, gapCover)).toISOString();
}
```

### B9 — Photos & files (Phase 7)

```js
// modules/sync/attachments.js
const { Storage } = require('@google-cloud/storage');
const db = require('../../shared/database');
const st = require('../../integrations/servicetitan');

let _bucket;
function bucket() {
  if (!_bucket) {
    const storage = new Storage({ keyFilename: process.env.GCS_KEY_FILE });
    _bucket = storage.bucket(process.env.GCS_BUCKET_NAME);
  }
  return _bucket;
}

async function listJobAttachments(stJobId) {
  try {
    const res = await st.apiRequest('GET', `/forms/v2/tenant/{tenant}/jobs/${stJobId}/attachments`);
    return res.data || [];
  } catch (err) {
    if (err.response?.status === 404) return [];   // job gone in ST — treat as none
    throw err;
  }
}

async function downloadAttachment(attachmentId) {
  const data = await st.apiRequest('GET',
    `/forms/v2/tenant/{tenant}/jobs/attachment/${attachmentId}`, { responseType: 'arraybuffer' });
  return Buffer.from(data);
}

// One job: list → skip known → download → upload → record.
// Zero attachments → a placeholder row so the crawler never re-checks this job
// (except jobs < 7 days old: techs upload photos after the job first syncs).
async function syncJobAttachments(localJobId, stJobId) {
  const stats = { downloaded: 0, skipped: 0, errors: 0 };
  const atts = await listJobAttachments(stJobId);

  if (atts.length === 0) {
    await db.query(
      `INSERT INTO job_attachments (servicetitan_id, job_id, filename, file_type)
       VALUES ($1, $2, 'NO_ATTACHMENTS', 'none') ON CONFLICT (servicetitan_id) DO NOTHING`,
      [-stJobId, localJobId]   // negative ST job id = the placeholder's unique key
    );
    stats.skipped++;
    return stats;
  }
  await db.query(`DELETE FROM job_attachments WHERE job_id = $1 AND file_type = 'none'`, [localJobId]);

  for (const att of atts) {
    try {
      if (await db.findOne('job_attachments', { servicetitan_id: att.id })) { stats.skipped++; continue; }
      const buffer = await downloadAttachment(att.id);
      const ext = (att.fileName || 'file').split('.').pop().toLowerCase();
      const path = `jobs/${localJobId}/${att.id}.${ext}`;
      await bucket().file(path).save(buffer, { contentType: contentTypeFor(ext) });
      await db.query(
        `INSERT INTO job_attachments
           (servicetitan_id, job_id, filename, original_filename, file_type, file_size, gcs_path, created_on)
         VALUES ($1,$2,$3,$4,$5,$6,$7,$8)`,
        [att.id, localJobId, `${att.id}.${ext}`, att.fileName,
         fileTypeFor(ext), buffer.length, path, att.createdOn || null]
      );
      stats.downloaded++;
    } catch (err) { console.error(`  attachment ${att.id}: ${err.message}`); stats.errors++; }
  }
  return stats;
}

// Crawl drivers: `recent` (last ~30 days, runs daily) and `backlog`
// (any never-checked job, ~500/run, 100ms apart — re-run until drained).
// Viewing: private bucket + short-lived signed URLs:
async function getSignedUrl(gcsPath, minutes = 60) {
  const [url] = await bucket().file(gcsPath)
    .getSignedUrl({ action: 'read', expires: Date.now() + minutes * 60_000 });
  return url;
}
```

*(Note: the production system keys ST-synced attachment rows by the ServiceTitan job id — a historical artifact its own docs flag as a gotcha that later forced extra plumbing. This reference keys by the local job id like every other table. Learn from the scar, don't copy it.)*

## Appendix C — Starter `CLAUDE.md` for the owner's project

Write this file in Phase 0 so **every future AI session** inherits the rules without the owner remembering to restate them:

```markdown
# My ServiceTitan Sync — Standing Instructions

Read BLUEPRINT.md before doing anything — it is the specification for this project.

## Who I am
I'm a business owner, not a programmer. Explain in plain English before showing code.
One step at a time; tell me how to verify each step; wait for my confirmation.
When something errors, ask me to paste the exact output.

## Hard safety rules (never break these, even if I ask casually)
- This project is READ-ONLY against ServiceTitan. Never add write scopes or write calls.
- `.env` and any credential/key files are git-ignored and never committed, printed, or uploaded.
- All SQL is parameterized ($1, $2). Never concatenate values into SQL.
- Schema changes only through numbered migration files + the runner. Never hand-edit tables.
- Never edit or renumber a migration that already ran — add a new one.
- Any script that changes existing rows defaults to DRY-RUN and requires --apply.
- Any overwrite of existing data must snapshot the old values first, so it's reversible.
- NEVER derive a business event time (like a completion date) from modified_on/updated_at.
  If the true source is null, leave the value alone.
- Soft deletes only (deleted_at) — never DELETE rows of business data.
- One piece of logic lives in one place. Extract shared functions instead of copy-pasting.
- Store text raw (decode escaped input on the way in); escape only when displaying.
- My business timezone is: <FILL IN, e.g. America/New_York>. Date-only values go in DATE
  columns; warn me about the date-only off-by-one trap whenever we touch them.
- Before anything that loads a lot of data, confirm database backups are on.

## How to run things
- Scripts: `node -r dotenv/config scripts/<name>.js` (loads .env on every OS — never `source .env`)
- Migrations: `node -r dotenv/config scripts/migrate.js [--dry-run]`
- Sync: `node -r dotenv/config scripts/run-sync.js [--recent | --customers | --jobs | ...]`
```

## Appendix D — Troubleshooting

| Symptom | What it usually means | Fix |
|---|---|---|
| `401 Unauthorized` | Token expired/invalid, or wrong client id/secret | Check `.env` values for stray spaces/quotes; confirm the token cache refreshes; re-run the auth smoke test |
| Everything fails with 401/403 even though the token works | **Missing `ST-App-Key` header** — the one everyone misses | Confirm the header is on every request (it is, if all calls go through `apiRequest`) |
| `403 Forbidden` on one endpoint | App lacks that API's scope, or it's a restricted bulk endpoint (contacts) | Add the read scope in the portal, or use the per-record pattern (B5) |
| `404` on a specific job's notes/attachments | Job deleted/archived in ST | Skip it — that's the designed behavior |
| `409 Conflict` fetching one record | Record is inactive/deleted in ST | Skip and count, don't error |
| `429 Too Many Requests` | Rate limit (rare with the standard delays) | Honor `Retry-After` if present; otherwise double the delays |
| First sync seems stuck | It's just big — check the page-progress logs | Let it run; hours for a big history is normal |
| Counts don't match ServiceTitan | Filters differ (active-only vs all), or a wave hasn't run | Compare like-for-like; run the orphan check; a small gap of deleted/inactive records is normal |
| `SELF_SIGNED_CERT_IN_CHAIN` / SSL errors to the DB | Hosted Postgres needs SSL | The helper's `ssl: { rejectUnauthorized: false }` handles Render; confirm you're using the external connection string |
| Script can't find env vars | `.env` not loading | Run via `node -r dotenv/config ...` from the project root |
| A change made in ST today isn't local yet | Daily window hasn't run since the change | Run `--recent`; confirm `sync_log` shows a fresh success |

## Appendix E — Part Two: growing it into your own platform

This is the road I took after the sync was boringly reliable. It took me about a year of nights and weekends to a full replacement — but every milestone below was independently useful long before the end, and value shows up at milestone 1, not milestone 7.

1. **A tiny read-only web app.** One Express server, plain HTML/JS pages, module pattern (each feature = a folder with its own `routes.js` mounted at `/api/<name>`). First page: a searchable customer list with their jobs and equipment. You'll use it within a week — it's faster than ServiceTitan's screens for lookups. Put a password on it from day one, even self-hosted.
2. **Reports ServiceTitan wouldn't give you.** Revenue by month/tech/type, membership renewals coming due, aging equipment for replacement outreach. All SELECT statements with a page around them.
3. **The moment you write data that exists nowhere else** (a note, a custom field, a new feature), the rules change: add a **second database** (dev) and never develop against the live one again; give the migration runner an explicit `--target=dev|prod` flag; back up like it's the only copy of your business — because for that data, it is.
4. **Deploy it** (Render web service, auto-deploy from your repo's main branch). The day you set that up, a push to main IS a production deploy — treat it with that respect. Work on a dev branch; promote deliberately.
5. **Grow features toward the pain**: whatever annoys you most about ServiceTitan that month — dispatching board, invoicing, a customer booking page, membership billing. One module at a time, each writing to *your* tables.
6. **Dual-run.** For a while, ServiceTitan and your platform both run; the sync keeps your side current while more and more daily work happens on your side. This phase is safe *because* of the read-only sync — ST stays intact underneath you.
7. **The cutover.** When the platform is doing the real work: export everything final, **ask ServiceTitan for a complete backup export** (mine was a multi-gigabyte archive — every table as CSV plus attachments; it recovered files the API never surfaced), keep the `servicetitan_id` columns forever as historical cross-references, and deactivate. Mine happened March 2026. The `servicetitan_id` columns are now just archaeology — and the business runs on tables I own.

Nobody has to go past milestone 2. Owning your data with a daily sync and your own reports is already a different negotiating position at renewal time — and a different level of safety — than where you started.

---

*Written by **Ricky Graves — Graves Heating & Air, Southern Maryland**, from the system that runs my company. Share it freely with credit (CC BY 4.0 — see LICENSE). If it helped you, star the repo so other contractors can find it.*
