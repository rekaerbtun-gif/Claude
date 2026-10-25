# Ops Daily Bot — Product Scope (Draft v0.1)

> Working name. A chat bot that runs the daily rhythm of the parking-operations
> team: morning briefing → 1-3-5 plan → check-ins → end-of-day close-out, with
> live alerts for critical tickets and a growing knowledge base.

Status: **Draft v0.4 for review.** The decisions in §10 are needed before building starts.
The quality bar every feature must meet is in §12–§14.

---

## 1. Problem & goal

The ops team (2 people today) loses track of what carried over from yesterday,
which support tickets are still open, and what each person committed to deliver.
Updates sit in people's heads, WhatsApp threads, and missed-call logs.

**Goal:** every working day follows the same closed loop, and nothing falls
through the cracks between days:

```
 09:00 Morning briefing ──► 1-3-5 plan ──► Check-ins every ~2h ──► 18:00 EOD close-out
        ▲  (carryover + open tickets)                                     │
        └──────────── anything still open becomes tomorrow's carryover ◄──┘

 At any time: 🚨 critical ticket alert → acknowledge → update
```

**Design principle:** the bot should *pre-fill and ask people to confirm*, not
make them retype. The fewer seconds each prompt takes, the more likely people
keep using it.

## 2. Users & roles

| Role | Who | What they do |
|---|---|---|
| **Operator** | 2 ops team members | Get prompts, plan their day, post updates, close the day |
| **Manager** | You (owner) | See every plan, check-in, and EOD summary; get escalations when something is missed or a critical ticket is left unacknowledged |
| **Admin** | You (or a delegate) | Add/remove users, set working hours, holidays, and what counts as "critical" |

## 3. Channel: **WhatsApp (official WhatsApp Business Platform)**

Decision: the team prefers WhatsApp, and customers already raise issues there, so
WhatsApp is the channel for both the team's daily prompts **and** customer ticket
intake (§17). Telegram is dropped.

**How it works**
- One dedicated company number, e.g. "*<Company> Parking Support*", connected to
  Meta's official **WhatsApp Cloud API**. This number can't also be used in the
  normal WhatsApp app.
- Our server receives every message sent to that number and replies with
  buttons and dropdown lists (WhatsApp "reply buttons" and "list messages").
- Team members chat with the same number 1-to-1 for their briefing, plan,
  check-ins, and close-out.

**Rules we have to design around**
| Rule | What it means for us |
|---|---|
| **24-hour window:** the bot can send free-form messages only within 24h of the person's last message | Business-initiated messages (09:00 briefing, critical alerts, customer "resolved" updates) use pre-approved **templates**. Once the person replies, normal chat continues |
| **Templates cost money** | About ₹0.15 per utility template in India (Meta's rate card, checked Oct 2026). For 2 operators plus customer updates this is a few hundred rupees a month |
| **Groups API:** only for verified Official Business Accounts, max **8 people** per group, and only groups the business creates (people join by invite link) | The bot **cannot be added to existing customer groups**. See §17 for how we bridge them |
| **Meta's AI policy (Jan 2026):** general-purpose AI chatbots are banned, business-specific bots are allowed | Our use (ticket triage and team workflow) is allowed. It is not a "chat with AI about anything" bot |
| **Business verification** | Meta Business account verification and display-name approval take a few days to a couple of weeks. Start this first |

**Not recommended:** unofficial tools that automate a normal WhatsApp account to read
groups (e.g. "WhatsApp Web" bots). They break WhatsApp's terms, and the number can be
banned without warning, which would take down your support line.

## 4. The daily flow, step by step

**Working hours:** 09:00–18:30 regular shift. Anything outside is logged as **extra time**.
Parking runs 24/7, so **critical** tickets alert around the clock (on-call). Non-critical tickets
raised after hours get an auto-reply and wait for the next morning's briefing.
All times below are defaults and configurable. *(Timezone to be confirmed.)*

### 4.1 09:00 — Morning briefing

The bot sends each operator **one message** containing:

1. **Carryover from yesterday:** unfinished 1-3-5 items, EOD "open items",
   missed calls to return, emails still to draft.
2. **Open support tickets:** *all* open tickets assigned to them (not only
   yesterday's — on Monday, Friday's tickets must still show), sorted by
   priority then age, each showing how long it has been open.
3. **Action required:** for each ticket the operator taps a status button
   (`✅ Resolved` / `🔄 Update` / `⛔ Blocked` / `↪️ Reassign`). Choosing
   **Update** asks for a one-line note.

The briefing counts as complete only when every **critical** ticket has an
update. Other items can be carried into the plan.

```
☀️ Good morning, Ravi — Tue 14 Oct

🔁 Carried over (3)
 • Call back Lot C manager re: billing   [missed call, due 10:00]
 • Draft email to vendor re: barrier AMC
 • Replace ticket printer at Mall P2     [was in your 3s yesterday]

🎫 Open tickets (4)
 🔴 #212 Boom barrier stuck open — Lot B   (open 16h)  [Update]
 🟠 #208 FASTag reader intermittent — P1   (open 2d)   [Update]
 🟢 #205 Signage request — Tower A         (open 4d)   [Update]
 🟢 #199 Monthly pass refund               (open 6d)   [Update]

Update 🔴 tickets first, then I'll help you plan your 1-3-5 ➜
```

### 4.2 Critical ticket alerts (any time)

Critical tickets must not wait until the morning.

- When a ticket is created or escalated to **critical**, the assigned operator
  gets an immediate 🚨 alert with an `Acknowledge` button.
- **Escalation ladder:** not acknowledged in 10 min → re-alert. Not acknowledged
  in 20 min → alert the other operator and the manager.
- After acknowledging, the operator is asked for an update every 60 min until
  the ticket is resolved or downgraded.
- **Out of hours:** the alert goes to whoever is on call (see §10). With only 2 people,
  a simple weekly on-call rotation is enough.

**Critical = anything that blocks money or traffic:**
1. The system is **not capturing payment** (pay station, FASTag, UPI, or card failure).
2. **A vehicle is stuck at exit**, or exit is delayed, for any reason (barrier, ANPR,
   ticket reader, payment, network).

The issue-type master (§16) marks which issue types are automatically critical. The AI
intake (§17) also flags messages like "car stuck at exit" or "payment not going through" as critical.

Each critical alert also starts an **extra-time log** if it happens outside 09:00–18:30.

### 4.3 After the briefing — the 1-3-5 plan

The bot helps the operator build the day's plan:

| Tier | Meaning | Default "done by" time (editable) |
|---|---|---|
| **1** | The single most important outcome today | 12:00 |
| **3** | Three important tasks | 15:00 |
| **5** | Five smaller tasks needed to close the day well | 18:00 |

**Making it fast:**
- The bot **suggests candidates** first: critical or high tickets, then carryover
  items. One tap adds a candidate to a tier.
- Free-text entry works in one message, e.g.:
  ```
  1 Fix Lot B barrier @11
  3 Call Lot C manager @10, Vendor AMC email @13, Printer at Mall P2 @16
  5 ...
  ```
  If a time isn't given, the tier default is used.
- Not every slot has to be filled (e.g. a field day might be just 1 + 2). The plan just needs
  at least the **1**.
- When the plan is submitted, the manager gets a copy.

Plan window: 09:00–10:00. If no plan arrives by 10:00 → reminder. By 10:30 →
manager notified.

### 4.4 Check-ins (~every 2 hours)

Two triggers, merged so nobody gets pinged twice:

1. **Regular check-in** at 11:00, 13:00, 15:00, 17:00. It lists every open
   item in one message.
2. **Deadline check-in**: when an item's "done by" time arrives, ask about that item.
   If it falls within 30 min of a regular check-in, fold it into that one
   instead.

Each item gets one-tap buttons:

`✅ Done` · `🔄 On track` · `⏰ Need more time` (asks for a new time) ·
`⛔ Blocked` (asks for a reason, then alerts the manager) · `🗑 Drop` (asks for a reason)

- **Unplanned work:** `add` logs an interruption (e.g. "gate broke at Lot D")
  so the day is judged fairly when planned items slip.
- Items rescheduled **more than twice** are flagged in the manager summary.
- No reply within 20 min → one nudge. Two check-ins in a row with no reply → manager notified.

### 4.5 18:00–18:30 — End-of-day close-out (mandatory)

Starts **at 18:00**, so the close-out is finished *by* the 18:30 end of shift.

1. **Open items from today:** the bot lists everything not yet ✅ and the operator
   confirms each one: carry over to tomorrow, hand over to a colleague, or drop.
2. **Missed calls:** who, number or site, what it was about, call back by when.
   ("None" is a valid one-tap answer.)
3. **Emails pending:** to whom, subject, by when. *(Phase 2: the bot drafts the
   email from these notes.)*
4. **Optional one-liner:** "Anything I should know?"
5. `Submit day` produces a summary card for the operator and manager.

**Enforcement** (we can't physically stop someone leaving, so we make skipping
visible and easy to fix):
- Reminders at 18:15, 18:30, and 18:40 until it is submitted.
- 18:45 → manager alerted: "Priya hasn't closed her day."
- The day stays **open** in the system, and tomorrow's briefing starts with "Finish
  yesterday's close-out" before anything else.

Everything captured here feeds the next 09:00 briefing.

### 4.6 Manager views

- **19:00 daily digest:** each person's plan vs. actual, items carried over,
  tickets opened and closed, missed calls outstanding, late or missed
  check-ins.
- **Friday weekly report:** completion rate of 1s, 3s, and 5s, average ticket age,
  critical acknowledgement times, most-rescheduled items.
- **Commands:** `team` (live status), `tickets` (all open), `day <name>`.

### 4.7 Commands operators can use any time

WhatsApp has no command menu, so these are plain words the bot understands, also available
from a **Menu** button (a WhatsApp dropdown list).

| Command | Does |
|---|---|
| `today` | Show my plan and status |
| `add` | Log unplanned work |
| `ticket` | Create or update a ticket |
| `leave` | Mark a day off so prompts are skipped and tickets go to a colleague |
| `kb <words>` | Search the knowledge base |
| `menu` / `help` | Show everything I can do |

## 5. Knowledge base

**Assumption to confirm:** "deploying systems" means the field SOPs for the
parking business, such as commissioning a new site, installing and configuring
ANPR cameras, boom barriers, FASTag readers, pay stations, and handling common faults.

- **Phase 1:** SOPs are written as markdown files in this repo (`kb/`),
  searchable from the bot with `kb barrier calibration`.
- **Phase 2:**
  - When a ticket is resolved, the bot asks "Worth an SOP? What fixed it?" Those notes
    become KB drafts that the manager approves.
  - Ticket alerts link the relevant SOP (e.g. a barrier ticket → barrier troubleshooting).
  - An **onboarding path** for new hires: an ordered list of SOPs and a short check-off
    so they "progress into the job".
- **Later:** move to Notion or Google Docs if non-technical editing becomes important.

## 6. What can go wrong — and how we handle it

| Risk | Impact | Mitigation |
|---|---|---|
| **Notification fatigue:** team mutes the bot | The whole system fails silently | Few, batched messages. Deadline pings fold into check-ins. Only 🚨 alerts break through. Review the volume after week 1 |
| **Feels like surveillance** | Resentment, fake updates ("working on it") | Frame it as *their* tool: carryover memory, fewer things forgotten. `add` credits unplanned work. Share metrics as team trends, not a leaderboard |
| **Low-quality updates** | Manager can't trust status | Status is always a button. Blocked, dropped, and rescheduled items require a reason |
| **Prompt ignored** (busy in the field) | Gaps in the record | One nudge, then manager escalation. Late answers are still accepted and marked late |
| **Leave, sick days, holidays** | Prompts and alerts to an absent person | `leave` and a holiday calendar skip prompts and reroute tickets to the other operator |
| **Both people unavailable** | Critical tickets unattended | The manager is always the final escalation step |
| **After-hours critical ticket** (parking runs 24/7) | Revenue or customer impact overnight | On-call rotation. Out-of-hours alerts only for critical tickets |
| **Poor connectivity on site** | Missed prompts | WhatsApp delivers queued messages when the phone reconnects. Late replies are accepted. A one-message plan works even on a weak connection |
| **Tickets out of sync** with the real helpdesk | Wrong briefing | Pick one source of truth (§10). The bot syncs from it rather than duplicating it |
| **Everything gets rescheduled** | The plan becomes meaningless | Reschedule counter. Items moved >2× are flagged. Weekly report shows patterns |
| **Bot server goes down** | No 09:00 prompt and nobody notices | Health check, plus an alert to the manager if the briefing didn't go out. Missed jobs run on restart |
| **Wrong timezone or clock** | Prompts at 3 am | Timezone set explicitly in config. Tested with a dry-run mode |
| **Lost phone or new number** | Locked out, or the wrong person receives data | Users are identified by their WhatsApp number. An admin changes the number in the web app |
| **Customer data** (phone numbers, plates) in chat | Privacy exposure | Only allow-listed users can talk to the bot. Store the minimum. Never post customer data in group chats |
| **WhatsApp number banned or restricted** | Support line and team prompts go down | Official Cloud API only, approved templates, no spam or bulk marketing. The web app keeps working if WhatsApp is down |
| **Customers keep using the old groups** | Tickets bypass the system | Operators forward group messages to the bot (§17). The pinned group message points to the support number |
| **AI picks the wrong site or category** | Wrong ticket data | The bot always shows its guess with Confirm / Change buttons. Low-confidence guesses ask instead |
| **Scope creep** | Never ships | Phase 1 is only the daily loop (§8) |

## 7. Data model (first cut)

```
User        id, name, whatsapp_number, role(operator|manager|admin), timezone, active
Day         id, user_id, date, briefing_done_at, plan_submitted_at, eod_submitted_at, status(open|closed|leave)
Item        id, day_id, tier(1|3|5|unplanned), title, due_at, status(open|done|blocked|dropped|carried),
            source(new|carryover|ticket|eod), ticket_id?, reschedule_count, parent_item_id? (carryover chain)
CheckIn     id, item_id, asked_at, answered_at, status, note
Ticket      id, number, customer_id, site_id, category_id, issue_type_id, priority_id, status_id,
            source_id, raised_by_contact_id?, owner_id, description, remarks,
            opened_at, acknowledged_at, resolved_at, is_after_hours, raw_message?
            (all *_id fields point to masters, §16)
TicketNote  id, ticket_id, user_id, note, created_at
MissedCall  id, day_id, caller, contact, reason, callback_by, done_at
PendingEmail id, day_id, to, subject, notes, due_by, done_at
KBArticle   id, title, tags, body_path, status(draft|published)
Leave       id, user_id, date_from, date_to
ExtraTime   id, user_id, ticket_id?, started_at, ended_at, note
```

## 8. Phased delivery

**Phase 1: MVP, the daily loop plus the tickets web app (≈2–3 weeks of build plus a 1-week pilot)**
- WhatsApp bot (official Cloud API) for the team, with approved message templates
- Masters (§16) managed from the web app
- 09:00 briefing with carryover and tickets
- Tickets web app (§15): list, filters, create/edit, history, login. The bot and the web app share one database
- One-time Excel import, plus an "Export to Excel" button
- Critical alerts with an acknowledge button and escalation ladder
- 1-3-5 planning with suggestions and one-message entry
- Check-ins (regular and deadline, merged)
- EOD close-out with reminders and manager escalation
- Manager daily digest
- After-hours critical alerts (on-call) and extra-time log
- `leave`, `today`, `add`
- KB as markdown plus `kb` search

**Phase 2: customer intake and intelligence**
- Customers raise tickets by messaging the support number, with AI reading the message (§17)
- "Forward to bot" from existing WhatsApp groups (§17)
- Customers get ticket number and "resolved" updates on WhatsApp
- QR-code ticket form at each site, so site staff and clients can raise tickets directly
- Email-to-ticket (forward an email and it becomes a ticket)
- AI assist: turn free-text updates into structured status, and draft the pending emails
- Weekly report, reschedule analytics
- KB capture from resolved tickets, and an onboarding path
- Holiday calendar, on-call rotation

**Phase 3: scale**
- Small web dashboard for the manager
- Voice-note updates (transcribed)
- Customer portal: each client sees their own sites' tickets and SLA report
- Bot-run WhatsApp groups per customer (if the Official Business Account is approved)

## 9. Technical approach (proposed)

- **Python / Django** for the web app (login, database, admin, migrations built in)
- **WhatsApp Cloud API** (Meta, direct, so no reseller markup) for the bot, in the same project, using the same database
- **Claude API** to read free-text WhatsApp messages and match them to masters (Phase 2)
- **SQLite** to start (plenty for 2–10 users, easy to back up). Postgres later if needed
- **Hosting:** a small VPS or Railway or Fly.io (≈ $5/month) with a domain and HTTPS,
  which WhatsApp needs to deliver messages to us
- **Config file** for working hours, check-in times, tier defaults, the critical
  definition, timezone, and holidays
- Tests for the scheduling logic (merging, escalations, timezone), since it is the
  riskiest part

## 10. Decisions needed from you

1. ~~Where do support tickets live today?~~ **Answered: Excel.** Proposal: replace it
   with the tickets web app (§15). Still needed: the column headers or a sample file.
2. ~~What makes a ticket critical?~~ **Answered:** payment not captured, or a vehicle stuck or
   delayed at exit (§4.2).
3. ~~Telegram okay?~~ **Answered: WhatsApp.** Needed: a new phone number for the bot, and access to
   your Meta Business account (or we create one) to start verification.
4. **Escalation contact:** you? Should you get every plan, or only the digest
   and exceptions?
5. ~~Working hours~~ **Answered: 09:00–18:30, critical 24/7.** Still needed: working days
   (Saturdays?), timezone, and who is on call after hours.
6. **Missed calls:** office line, or personal phones? (Phase 1 = entered by hand.)
7. **Knowledge base:** is "deploying systems" about on-site parking hardware and site
   commissioning, or software? Where do current SOPs live, if anywhere?
8. **Hosting:** okay with ≈$5/month on a small cloud server under your account?
9. **Masters:** your current list of customers, sites, and issue types, so we can pre-load them (§16).

## 11. How we'll know it works (pilot metrics)

| Metric | Target after 2 weeks |
|---|---|
| Plan submitted by 09:30 | ≥ 90% of working days |
| EOD submitted by 18:15 | ≥ 95% |
| Check-in reply within 20 min | ≥ 80% |
| Critical ticket acknowledged within 10 min | ≥ 95% |
| Carryover items older than 3 days | Trending down |
| Team says it saves them time (quick poll) | Yes from both |

---

## 12. Product principles — what makes this work

These are the tests every feature has to pass. If a feature breaks one, we
change or cut it.

| # | Principle | What it means in practice | How we check it |
|---|---|---|---|
| 1 | **Faster than not using it** | Each prompt takes under 30 seconds. The whole day's overhead is under 10 minutes | Time the pilot: morning ≤ 3 min, each check-in ≤ 20 s, EOD ≤ 3 min |
| 2 | **Confirm, don't type** | The bot already knows yesterday's items, tickets, and plans, so people tap to confirm and type only what's new | Fewer than 1 in 3 interactions need typed text |
| 3 | **Reliable like a utility** | The 09:00 prompt arrives every working day, on time. Data is never lost. If something breaks, *we* know before the team does | 100% of scheduled prompts sent in the pilot. Daily backups. Health alerts |
| 4 | **Calm by default** | Predictable times, batched messages. Only 🚨 critical tickets interrupt | ≤ 8 bot messages per person on a normal day |
| 5 | **Gives back to the operator** | It remembers for them, shows what they got done, and protects them when fire-fighting derails the plan | Both operators say it helps *them* (pilot poll) |
| 6 | **Fair** | Unplanned work counts. Late answers are accepted. Leave is respected | No one is flagged for a day spent on a logged critical ticket |
| 7 | **Forgiving** | Anything can be edited or undone. Typos are understood. Nothing gets a person stuck | Every action has an Undo or Edit path |
| 8 | **One owner, one time** | Every item and ticket has exactly one owner and a "by when" | No ownerless items in the database |
| 9 | **One source of truth** | Tickets and tasks live in one place, and the bot never contradicts the real system | Sync checks once tickets are integrated |
| 10 | **Visible, not policed** | The manager sees exceptions (blocked, late, missed), not a minute-by-minute feed | Manager receives ≤ 3 messages a day plus the digest |

## 13. Experience design standards

A professional feel comes from consistency more than features.

**Identity**
- A proper name, a bot avatar, and a short bio. The bot introduces itself on first use.
- One consistent tone: brief, warm, and direct. It uses first names, and it doesn't
  guilt-trip or overuse exclamation marks.

**Message design system:** every bot message follows the same layout:

```
<icon> <Title> — <context: date / time / count>     ← what this is, at a glance
<body: max ~7 lines, grouped, most urgent first>    ← only what's needed now
<progress: "2 of 4 tickets updated">               ← where they are
[Primary action] [Secondary] [Later]               ← one obvious next step
```

**Fixed visual vocabulary** (always the same meaning, nothing else):

| Priority | Status | Message types |
|---|---|---|
| 🔴 Critical · 🟠 High · 🟢 Normal | ✅ Done · 🔄 On track · ⏰ Rescheduled · ⛔ Blocked · 🗑 Dropped | ☀️ Briefing · 🎯 Plan · 🕑 Check-in · 🌙 Close-out · 🚨 Alert · 📊 Digest |

**Interaction rules**
- Buttons first. Typing is only for genuinely new information.
- A long flow is broken into steps with progress shown ("Step 2 of 3: missed calls").
- Every completed flow ends with a short confirmation card. Nothing is left ambiguous.
- A **Menu** button lists every command, so nobody has to remember them.
- If the bot doesn't understand something, it offers the likely options instead of "invalid input".
- Escalation messages to the manager stay factual and neutral, e.g. "Priya's close-out is
  pending (18:15)", never accusatory.

**Copy examples**

| Instead of | Write |
|---|---|
| "ERROR: Plan not submitted." | "No plan yet today. Want to start with your carryover items?" |
| "You have failed to update ticket #212." | "#212 (Lot B barrier) still needs an update. 1 tap ➜" |
| "Submit EOD now." | "🌙 Time to close out. About 2 minutes, then you're done for the day." |

## 14. Running it like a real product

**Reliability and trust**
- A separate **test bot** and dry-run mode. Changes are tried there before reaching the team.
- Monitoring: if the 09:00 briefing hasn't gone out by 09:05, the manager gets alerted.
- Automatic daily database backups, with a restore that has been tested once.
- An **audit trail**: every status change records who, when, and the old and new values.
- Data retention policy, e.g. customer phone numbers deleted after 90 days.

**Onboarding and adoption**
- **Co-design:** walk the 2 operators through the flow *before* building, and adjust it with them. People
  use what they helped shape.
- A first-run tour in the bot (2 minutes), plus a one-page guide.
- **Shadow week:** the bot runs fully but escalations to the manager are switched off. We tune timings and
  wording. Then go live.
- A `feedback` command available any time. A 15-minute review at the end of each pilot week.

**Evolving it**
- Versioned releases with a short "What's new" message to the team.
- A monthly look at the metrics in §11. Cut prompts nobody values, and add only what removes
  real friction.
- Configuration (times, tiers, critical rules) can be changed without code changes.

## 15. Tickets web app (replaces the Excel tracker)

**Why not keep Excel as the live system?**
- No instant trigger when a critical ticket is added, so alerts would lag.
- Two people editing the same file at once causes conflicts and lost rows.
- Free-typed columns ("Critcal", "high ", blank) break the logic.
- There's no history of who changed what.

**Proposal:** a small web app that is the single source of truth for tickets. It shares
one database with the bot, so a ticket created on the web triggers a 🚨 alert within
seconds, and an update from WhatsApp shows on the web immediately.

```
   Web app (browser, laptop or phone)       WhatsApp bot
              │                                   │
              └────────── one database ───────────┘
                             │
                 Excel import (once) / Excel export (any time)
```

**Phase 1 screens**
| Screen | What it does |
|---|---|
| Login | Email and password for the 3 users. Roles: operator, manager, admin |
| Masters | Add and edit customers, sites, categories, issue types, priorities, and more (§16) |
| Tickets | Table with filters (status, priority, site, assignee, age) and search. Overdue and critical tickets highlighted |
| Ticket detail | Fields, a timeline of every update (web and WhatsApp), assign, change status |
| New ticket | Short form where every field is a dropdown from the masters (§16). Only description and remarks are free text |
| Today | Each operator's 1-3-5 plan and status (read-only for the manager) |
| Export | One click downloads an Excel file, so existing reports keep working |

**Excel migration**
1. Map your current columns to ticket fields (needs the header row or a sample file).
2. Run an import with a dry-run report: rows imported, rows with problems, duplicates.
3. Run both side by side for 2–3 days, then the web app becomes the source of truth and the Excel file is archived.

**Later**
- A QR code at each parking site that opens a public "Report a problem" form. Tickets
  then come straight from site staff and customers.
- Knowledge base pages (§5) inside the same web app.
- The manager dashboard (§4.6) as a web page.

## 16. Masters (dropdown lists, built for many customers)

Every ticket field is chosen from a **master list**, except the description and
remarks. The admin edits masters in the web app, and changes show up immediately in
the web form and in WhatsApp dropdowns. No code change is needed to add a customer, site, or issue type.

| Master | Example values | Linked to |
|---|---|---|
| **Customer** (client) | Phoenix Mall, City Hospital, Tech Park A | — |
| **Site / location** | Phoenix Mall – P2, City Hospital – Basement | Customer |
| **Equipment / lane** *(optional)* | Exit lane 2 barrier, Entry ANPR cam 1 | Site |
| **Category** | Payment, Entry/Exit, Hardware, Software, Billing, Request | — |
| **Issue type** | Payment not captured, Vehicle stuck at exit, Barrier not opening, FASTag not reading | Category, with a default priority and an **auto-critical** flag |
| **Priority** | Critical, High, Normal, Low | Response and resolution targets (SLA), e.g. Critical: acknowledge 10 min, resolve 1 h |
| **Status** | Open, Acknowledged, In progress, Blocked, Resolved, Closed | — |
| **Owner** | Ravi, Priya | Default owner per customer or site |
| **Raised by (contact)** | Site supervisor at Phoenix P2 (+91 …) | Customer or site. Lets the bot recognize who is messaging |
| **Source / channel** | WhatsApp, Phone call, Email, Site visit, Internal | — |

**Smart defaults, so most tickets need 2–3 taps:**
- Pick the **site**, and the customer and default owner fill in automatically.
- Pick the **issue type**, and the category and priority fill in. Auto-critical types trigger 🚨 alerts.
- If a known contact messages on WhatsApp, the customer and site are already known.
- Dates, ticket number, and "after hours" are filled in automatically.

**Multi-customer from day one:** every ticket belongs to a customer, so reports, SLAs,
and later a customer portal can be split per client without restructuring.

Masters are **soft-deleted** (marked inactive), so old tickets keep their history.

## 17. Customer tickets from WhatsApp

**The problem:** customers report issues in WhatsApp groups today, and a bot can't read
existing groups (§3).

**The plan has three routes, from most to least automated:**

**A. Customers message the support number directly (target state)**
```
Customer: "Exit barrier at P2 not opening, 4 cars waiting"
Bot:      🚨 Got it — looks critical.
          Site: Phoenix Mall – P2   Issue: Vehicle stuck at exit
          [✅ Correct]  [✏️ Change]
Bot:      Ticket #231 raised. Ravi has been alerted. We'll update you here.
   ...
Bot:      ✅ #231 resolved at 10:42 — barrier reset. Reply if the issue returns.
```
- The AI reads the message and matches it to the masters (site, issue type, priority).
  If it isn't confident, it asks with a dropdown instead of guessing.
- Known contacts skip the site question.
- Customers get the ticket number immediately and an update when it's resolved.
- Rollout: share the number with every client, add it to site signage with a QR code, and pin it in the existing groups.

**B. "Forward to bot" from existing groups (bridge, works from day one)**
- An operator sees an issue in a customer group, long-presses it, and **forwards it to the
  support number**.
- The bot reads it, proposes a ticket (site, issue, priority), and the operator confirms with one tap.
- Takes about 5 seconds, needs no change from customers, and nothing is lost.

**C. Bot-run groups (optional, later)**
- If Meta approves an Official Business Account, the bot can create a group per
  customer (max 8 people) and read issues posted there.
- The 8-person limit and approval process make this a later option, not the plan.
