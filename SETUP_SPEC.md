# Quickly Fork — Setup Spec

Status: **draft / ideation**. Nothing below is provisioned yet.

- Fork: https://github.com/BanananaBreadd/Quickly (`origin`)
- Upstream: https://github.com/AbdelftahZowail/Quickly (`upstream`) — sync with `git fetch upstream && git merge upstream/main`

---

## 1. Hosting (Railway)

Two services:

| Service | Source | Notes |
|---|---|---|
| App | Build from this fork's `Dockerfile` | Pushes to the fork redeploy our changes. Prebuilt `azowail/quickly:latest` would ignore fork changes. Listens on port 8000. |
| Postgres | Railway Postgres plugin | Wire `DATABASE_URL` into the app service. |

- No Caddy — Railway terminates HTTPS.
- Plan: Hobby (~$5/mo incl. usage, unverified). App must stay always-on: APScheduler runs sends in-process, so no serverless sleep.
- Backups: compose stack writes dumps to `/app/backups` (`QUICKLY_LOCAL_DISK_BACKUPS=1`). On Railway either attach a volume there or rely on Railway Postgres backups.
- Env to set: `BASE_URL`, `CORS_ORIGINS` (Railway domain or custom), `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, optionally `OFFICE365_*`. Everything else is configured in the UI and stored in the DB.

## 2. Sending domains

- 7 domains available; not yet configured in Gmail.
- Never send cold from the primary brand domain.
- 2–3 inboxes per domain → **14–21 inboxes**.

### DNS per domain

- SPF, DKIM (from Google Workspace), DMARC starting at `p=none` with reporting, MX.
- Redirect domain root to the main site.
- Open/click tracking **off** initially (pixels and wrapped links hurt deliverability). If wanted later, use Quickly Beacon for a per-domain custom tracking hostname.
- Keep unsubscribe headers on (`add_unsubscribe_header`, `add_one_click_unsubscribe` default true; required by Gmail/Yahoo).

## 3. Inbox provider

Default choice: **Google Workspace**.

| Option | Approx. cost/inbox/mo (unverified) | Deliverability | Quickly fit |
|---|---|---|---|
| Google Workspace direct | ~$7–8 | Strong | Native OAuth send + reply sync |
| Microsoft 365 Business Basic | ~$6 | Good; tightening on cold | Native OAuth |
| Reseller inboxes | ~$3–5 | Varies; often shared tenants | SMTP/IMAP or OAuth |
| Amazon SES / SMTP relay | cents | Poor for cold; SES AUP bans unsolicited mail | Supported — avoid |

- Mixing Google + Microsoft inboxes enables Quickly's **lead provider matching** (MX lookup routes Google leads to Gmail senders, Microsoft leads to O365 senders).

### Gmail OAuth shape

- One Google Workspace org holding all 7 domains as secondary domains.
- Google Cloud OAuth consent screen set to **Internal** → no Google app verification / CASA assessment for restricted Gmail scopes.
- Risk: single tenant = shared fate; a suspension takes all domains down. Mitigation if volume grows: split across 2–3 Workspace tenants (one OAuth app each, or external app + verification).

## 4. Volume plan

- Steady state: ~25–40 cold sends/inbox/day → **~350–800/day** (~7–16k/month).
- Ramp from ~5/day over 3–4 weeks (Quickly per-inbox `ramp_up_enabled`, `ramp_up_period_days`, `max_jitter_seconds`).

## 5. Warmup — gap in Quickly

Quickly's "warm-up" is only a **volume ramp** (daily limit increases over `ramp_up_period_days`, default 42). It has **no peer warmup network** (inboxes mailing each other, replying, rescuing from spam).

Options:

1. Paid standalone warmup tool (Mailreach, Warmup Inbox, Lemwarm, …) — proven; cost scales per inbox.
2. **Build warmup into this fork** — our inboxes send/reply to each other on a schedule. Free to run; weaker than a large network (closed pool of ~15–20 inboxes).
3. None — domain aging + slow ramp + plain text + low volume.

This is the main reason the fork carries value beyond the prebuilt image.

## 6. Open questions

1. Are the 7 domains lookalikes of a main brand or the brand itself? Registration dates (aging)?
2. Target daily volume and audience (B2B? UK/EU recipients → PECR/GDPR obligations).
3. Warmup approach: paid tool, build into fork, or none?
4. One Workspace tenant or split from day one?
5. Railway generated subdomain or custom domain for the app?
6. AI reply classification provider (Quickly supports 19, incl. Anthropic/OpenAI/Ollama).
