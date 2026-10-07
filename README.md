# Cold Outreach Worker

A Slack bot that runs B2B cold-email campaigns end to end for a small sales team: find local businesses, find a real contact email, research each company, write a personal email in the salesperson's own voice, gate it with a reviewer, send it from the salesperson's Gmail at a human pace, and track the replies.

> Built as an internal tool at a small AI hardware company and used by its sales reps. This public copy uses a fictional company (**Acme Devices**, products **VoiceClip** and **PhonePilot**) and fictional reps (Alex, Sam). No prospect data, campaign logs, credentials or history are included.

---

## Pipeline

```
Slack: "prospect dental clinic in LA, CA | auto repair in Houston, TX --template comprehensive"
   │
   ▼
1. Discover      Google Maps scraper (subprocess) ──► businesses + websites
2. Dedupe        against everyone the team has ever contacted (domain / phone / company key)
                 + in-batch dedupe by email
3. Find email    crawl contact pages ──► mailto ▸ JSON-LD ▸ visible text ▸ raw HTML
                 score candidates ──► drop placeholders/junk ──► MX record check
4. Upload        lead CSV to the rep's Google Drive folder
   │
Slack: "auto"  (watches the rep's Drive folder for new lead CSVs)
   ▼
5. Research      per company, concurrent, cached 30 days, validated before caching
6. Write         layered prompt: company facts + agent role + this rep's template
7. Review        reviewer agent, up to 3 rounds; still failing ──► discarded, never sent
8. Send          Gmail API with the rep's own OAuth token, human-like pacing
   │
Slack: "track"
   ▼
9. Replies       classify bounce / out-of-office / auto-reply / human
                 bounces feed a shared junk list; human replies get an analysis
                 (sentiment, intent, objections, suggested follow-up)
10. Report       campaign stats + model-written insights
```

## Design decisions worth a look

**Each rep writes in their own voice, without forking the prompts.** Prompts are composed in layers (`agents/soul_loader.py`):

1. `agents/souls/_shared.md` — company and product facts, rules every agent follows
2. `agents/souls/<agent>.md` — the agent's job and output format
3. `config/<rep>/copywriters/<template>.md` — only what is different for this rep and template

A rep can have several templates (`config/users.json`): a researched "comprehensive" pitch, a short "discovery" email, a casual neighbor-style note. Greeting and signature are injected by code, never generated, so the model can't sign as the wrong person or invent a Calendly link. A template can also be fully **static** (`static_subject` / `static_body`): no model call at all.

**The reviewer can reject, and rejection is final.** Drafts go through up to 3 review rounds. A draft that still fails is recorded as `rejected_by_reviewer` and never sent. Every email has one lifecycle field (`drafted → reviewing → approved → sending → sent | failed | rejected_by_reviewer | rejected_by_human | canceled`), so "what happened to this lead" is one query (`firestore_schema.md`).

**Email finding is scored, not first-match.** `services/email_finder.py` collects candidates from four sources with different confidence, prefers role addresses that are likely to be read (`info@`, `contact@`, `hello@` …), filters placeholders (`name@domain.com`, `example@example.com`) and technical addresses, prefers emails on the company's own domain, and requires an MX record before accepting. Bounces from real sends are learned: a domain that bounces three different addresses is blocked for the whole team (`services/junk_list.py`).

**Sending looks like a person, not a script.** Delays are drawn from a normal distribution (mean ~25 s) with a small chance of a 1–2 minute break, all configurable (`SEND_DELAY_*`). Each rep sends from their own Gmail through an OAuth flow completed inside Slack.

**Being polite to Google Maps.** The scraper runs behind a process-wide semaphore (`PROSPECT_MAX_CONCURRENT`, default 2): several reps prospecting at once from the same IP would otherwise trigger CAPTCHAs and IP bans.

**Moving from JSON files to Firestore without losing anything.** The bot started with per-user JSON logs. `migrations/001–004` move prospects, the research cache, the email history and processed Drive files into Firestore. Every migration has `--dry-run` / `--commit`, uses deterministic document IDs so re-running is safe, and filters invalid cached research instead of copying it blindly. The schema, its indexes and the reasoning behind each collection are in `firestore_schema.md`.

## Slack commands

| Command | What it does |
|---|---|
| `prospect <query> [ \| <query> …] [--template t] [--depth n]` | Find businesses and their emails, upload a lead CSV to Drive |
| `auto` / `stop auto` / `status auto` | Research, write, review and send for new lead CSVs |
| `track` / `stop track` | Watch Gmail for replies |
| `report` / `insights` | Campaign stats / model-written analysis |
| `templates` | Show this rep's templates |
| `usage` | Token cost per rep |
| `auth` | Connect the rep's Gmail and Drive |

## Project layout

```
main.py                 Slack Bolt app (Socket Mode)
core/                   command router, per-user state, user registry
auto/                   prospect, auto (write+send), track, report pipelines
agents/                 researcher, copywriter, reviewer, reply analyzer, reporter + soul loader
agents/souls/           shared facts and per-agent instructions
config/users.json       reps: Slack ID → folder, Gmail, Drive folder, templates
config/<rep>/           per-rep, per-template copywriter instructions
services/               email finder, Gmail sender, reply tracker, Drive, Firestore, junk list, usage
migrations/             JSON → Firestore migrations (dry-run first)
```

## Running

```bash
python -m venv .venv && source .venv/bin/activate     # Windows: .venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env                                   # Slack, OpenAI, GCP settings
python main.py
```

Prospecting needs a build of [gosom/google-maps-scraper](https://github.com/gosom/google-maps-scraper) (MIT); point `GMAPS_SCRAPER_PATH` at the binary. This project drives it as a subprocess and does everything after the scrape. Gmail sending needs a Google OAuth client (`GMAIL_CREDENTIALS_PATH`); each rep then runs `auth` in Slack.

## Status and honest notes

- Used by a real sales team; this copy is a snapshot, not maintained.
- There is no automated test suite in this repository; the migrations have `--dry-run` and prospecting has `--debug` for checking before anything is written or sent.
- Code comments are partly in Chinese (the team worked in Chinese and English).
- Cold email is regulated (CAN-SPAM in the US). The reviewer checks for spam-trigger wording and every email is a 1-to-1 message from a named person; anyone reusing this is responsible for their own compliance and for respecting the terms of the sites they scrape.

## Stack

Python · Slack Bolt · OpenAI (GPT-4.1 for writing, a reasoning model for reply analysis) · Gmail API · Google Drive API · Firestore · BeautifulSoup · dnspython · gosom/google-maps-scraper
