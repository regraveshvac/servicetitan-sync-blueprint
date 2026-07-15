# Own Your ServiceTitan Data — The Blueprint

**by Ricky Graves — Graves Heating & Air, Southern Maryland**

I run an HVAC company. Over the past year I synced all of my ServiceTitan data into my own database, built my own field-service platform on top of it, and **deactivated ServiceTitan in March 2026**. My platform now runs the whole business — dispatching, scheduling, invoicing, memberships, AI phone agents, a technician mobile app.

It all started with the thing this document teaches: **pulling your ServiceTitan data into a PostgreSQL database you own.**

You don't need to know how to code. You need a computer, a ServiceTitan account, about $30/month, and an AI assistant (Claude) doing the typing while you steer. That's how I built mine.

## What you end up with

- Every customer, location, job, appointment, invoice, payment, estimate, membership, and piece of installed equipment — in **your** database, refreshed daily
- Your job photos and file attachments downloaded and stored in your own cloud storage (optional phase)
- The ability to ask your own data anything ("revenue by month for the last 5 years", "which customers have 15-year-old systems")
- A foundation you can grow into your own CRM later, like I did — or just keep as insurance that your data is yours

## How to use this

1. Install **Claude Code** (claude.com/claude-code) — it's Claude running in a terminal, inside your project folder, where it can write the files and run the commands itself. Do **not** try to copy-paste code out of a chat window; that's the #1 way beginners lose a day.
2. Create an empty folder on your computer, e.g. `my-servicetitan-sync`.
3. Save [`BLUEPRINT.md`](BLUEPRINT.md) from this repo into that folder.
4. Open a terminal in that folder and run `claude`.
5. Type: **"Read BLUEPRINT.md and follow it exactly. Start with the interview."**

The blueprint takes over from there — one step at a time, in plain English, waiting for you to confirm each step worked.

> **Do one thing today even if you build nothing else:** request your ServiceTitan API credentials (Client ID, Client Secret, App Key, Tenant ID). It's the only step with a waiting period, and everything is blocked until you have them. The blueprint walks you through it.

## What it costs

| Thing | Cost |
|---|---|
| Claude (Pro plan, includes Claude Code) | ~$20/mo |
| PostgreSQL database on Render | ~$7/mo |
| Google Cloud Storage for photos (optional phase) | a few dollars/mo |
| ServiceTitan API access | included with your ST account |

## Credit & license

Written by **Ricky Graves, Graves Heating & Air** (Southern Maryland), from a system running a real business in production. Share it freely — the license just asks that you keep the credit: document text is [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/), code samples are MIT. See [LICENSE](LICENSE).

If this helped you, star the repo so other contractors can find it.
