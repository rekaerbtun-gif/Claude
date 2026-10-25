# Ops Daily Bot — Product Scope (Draft v0.1)

> Working name. A chat bot that runs the daily rhythm of the parking-operations
> team: morning briefing → 1-3-5 plan → check-ins → end-of-day close-out, with
> live alerts for critical tickets and a growing knowledge base.

Status: **Draft for review.** The decisions in §10 are needed before building starts.

---

## 1. Problem & goal

The ops team (2 people today) loses track of what carried over from yesterday,
which support tickets are still open, and what each person committed to deliver.
Updates sit in people's heads, WhatsApp threads, and missed-call logs.

**Goal:** every working day follows the same closed loop, and nothing falls
through the cracks between days:

```
 08:30 Morning briefing ──► 1-3-5 plan ──► Check-ins every ~2h ──► 18:00 EOD close-out
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

## 3. Channel recommendation: **Telegram**

| | Telegram | Discord | WhatsApp | Slack |
|---|---|---|---|---|
| Phone push notifications | ✅ Reliable | ⚠️ Noisy, easy to mute | ✅ | ✅ |
| One-tap buttons in messages | ✅ Inline keyboards | ✅ | ⚠️ Limited, template rules | ✅ |
| Cost | Free | Free | Paid per conversation + Meta approval | Free tier limits history |
| Setup effort | Minutes (BotFather) | Server + bot setup | Business verification, 24h messaging window rules | Workspace + app |
| Needs a public server/URL | No (long polling) | No | Yes (webhooks) | Optional |

**Recommendation:** Telegram for the first version. It is the fastest to build,
free, and works well on phones for people in the field. The code will keep the
messaging layer separate, so Slack or WhatsApp can be added later if needed.

## 4. The daily flow, step by step

Times are defaults and configurable per team. *(Timezone to be confirmed, §10.)*

### 4.1 08:30 — Morning briefing

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

Critical tickets must not wait until 08:30.

- When a ticket is created or escalated to **critical**, the assigned operator
  gets an immediate 🚨 alert with an `Acknowledge` button.
- **Escalation ladder:** not acknowledged in 10 min → re-alert. Not acknowledged
  in 20 min → alert the other operator and the manager.
- After acknowledging, the operator is asked for an update every 60 min until
  the ticket is resolved or downgraded.
- **Out of hours:** the alert goes to whoever is on call (see §10). With only 2 people,
  a simple weekly on-call rotation is enough.

*What counts as "critical"* needs your definition, e.g. gate or boom barrier
failure blocking entry or exit, payment system down, revenue loss, or a customer or
client escalation.

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

Plan window: 08:30–09:30. If no plan arrives by 09:30 → reminder. By 10:00 →
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

- **Unplanned work:** `/add` logs an interruption (e.g. "gate broke at Lot D")
  so the day is judged fairly when planned items slip.
- Items rescheduled **more than twice** are flagged in the manager summary.
- No reply within 20 min → one nudge. Two check-ins in a row with no reply → manager notified.

### 4.5 17:30–18:00 — End-of-day close-out (mandatory)

Starts **at 17:30**, so the close-out is finished *by* 18:00 rather than only started then.

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
- Reminders at 17:45, 18:00, and 18:10 until it is submitted.
- 18:15 → manager alerted: "Priya hasn't closed her day."
- The day stays **open** in the system, and tomorrow's briefing starts with "Finish
  yesterday's close-out" before anything else.

Everything captured here feeds the next 08:30 briefing.

### 4.6 Manager views

- **18:30 daily digest:** each person's plan vs. actual, items carried over,
  tickets opened and closed, missed calls outstanding, late or missed
  check-ins.
- **Friday weekly report:** completion rate of 1s, 3s, and 5s, average ticket age,
  critical acknowledgement times, most-rescheduled items.
- **Commands:** `/team` (live status), `/tickets` (all open), `/day <name>`.

### 4.7 Commands operators can use any time

| Command | Does |
|---|---|
| `/today` | Show my plan and status |
| `/add` | Log unplanned work |
| `/ticket` | Create or update a ticket |
| `/leave` | Mark a day off so prompts are skipped and tickets go to a colleague |
| `/kb <words>` | Search the knowledge base |
| `/help` | What can I do? |

## 5. Knowledge base

**Assumption to confirm:** "deploying systems" means the field SOPs for the
parking business, such as commissioning a new site, installing and configuring
ANPR cameras, boom barriers, FASTag readers, pay stations, and handling common faults.

- **Phase 1:** SOPs are written as markdown files in this repo (`kb/`),
  searchable from the bot with `/kb barrier calibration`.
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
| **Feels like surveillance** | Resentment, fake updates ("working on it") | Frame it as *their* tool: carryover memory, fewer things forgotten. `/add` credits unplanned work. Share metrics as team trends, not a leaderboard |
| **Low-quality updates** | Manager can't trust status | Status is always a button. Blocked, dropped, and rescheduled items require a reason |
| **Prompt ignored** (busy in the field) | Gaps in the record | One nudge, then manager escalation. Late answers are still accepted and marked late |
| **Leave, sick days, holidays** | Prompts and alerts to an absent person | `/leave` and a holiday calendar skip prompts and reroute tickets to the other operator |
| **Both people unavailable** | Critical tickets unattended | The manager is always the final escalation step |
| **After-hours critical ticket** (parking runs 24/7) | Revenue or customer impact overnight | On-call rotation. Out-of-hours alerts only for critical tickets |
| **Poor connectivity on site** | Missed prompts | Telegram queues messages. Late replies are accepted. A one-message plan works even on a weak connection |
| **Tickets out of sync** with the real helpdesk | Wrong briefing | Pick one source of truth (§10). The bot syncs from it rather than duplicating it |
| **Everything gets rescheduled** | The plan becomes meaningless | Reschedule counter. Items moved >2× are flagged. Weekly report shows patterns |
| **Bot server goes down** | No 08:30 prompt and nobody notices | Health check, plus an alert to the manager if the briefing didn't go out. Missed jobs run on restart |
| **Wrong timezone or clock** | Prompts at 3 am | Timezone set explicitly in config. Tested with a dry-run mode |
| **Lost phone or new number** | Locked out, or the wrong person receives data | Users are linked by Telegram ID plus a one-time code. An admin can unlink and relink |
| **Customer data** (phone numbers, plates) in chat | Privacy exposure | Only allow-listed users can talk to the bot. Store the minimum. Never post customer data in group chats |
| **Scope creep** | Never ships | Phase 1 is only the daily loop (§8) |

## 7. Data model (first cut)

```
User        id, name, telegram_id, role(operator|manager|admin), timezone, active
Day         id, user_id, date, briefing_done_at, plan_submitted_at, eod_submitted_at, status(open|closed|leave)
Item        id, day_id, tier(1|3|5|unplanned), title, due_at, status(open|done|blocked|dropped|carried),
            source(new|carryover|ticket|eod), ticket_id?, reschedule_count, parent_item_id? (carryover chain)
CheckIn     id, item_id, asked_at, answered_at, status, note
Ticket      id, external_id?, title, site, priority(critical|high|normal|low), status, assignee_id,
            opened_at, acknowledged_at, resolved_at
TicketNote  id, ticket_id, user_id, note, created_at
MissedCall  id, day_id, caller, contact, reason, callback_by, done_at
PendingEmail id, day_id, to, subject, notes, due_by, done_at
KBArticle   id, title, tags, body_path, status(draft|published)
Leave       id, user_id, date_from, date_to
```

## 8. Phased delivery

**Phase 1: MVP, the daily loop (≈1–2 weeks of build plus a 1-week pilot)**
- Telegram bot with user allow-list and linking
- 08:30 briefing with carryover and tickets
- Tickets managed inside the bot (`/ticket`) as the first source of truth
- Critical alerts with an acknowledge button and escalation ladder
- 1-3-5 planning with suggestions and one-message entry
- Check-ins (regular and deadline, merged)
- EOD close-out with reminders and manager escalation
- Manager daily digest
- `/leave`, `/today`, `/add`
- KB as markdown plus `/kb` search

**Phase 2: integrations and intelligence**
- Sync tickets from your real helpdesk, Google Sheet, or email
- AI assist: turn free-text updates into structured status, and draft the pending emails
- Weekly report, reschedule analytics
- KB capture from resolved tickets, and an onboarding path
- Holiday calendar, on-call rotation

**Phase 3: scale**
- Small web dashboard for the manager
- Voice-note updates (transcribed)
- More teams or sites, plus Slack or WhatsApp adapters

## 9. Technical approach (proposed)

- **Python** with `python-telegram-bot` (built-in job scheduler)
- **SQLite** to start (plenty for 2–10 users, easy to back up). Postgres later if needed
- **Hosting:** a small VPS or Railway or Fly.io (≈ $5/month). Long polling means no domain
  or HTTPS setup is needed
- **Config file** for working hours, check-in times, tier defaults, the critical
  definition, timezone, and holidays
- Tests for the scheduling logic (merging, escalations, timezone), since it is the
  riskiest part

## 10. Decisions needed from you

1. **Where do support tickets live today?** (Helpdesk tool, Excel or Google Sheet,
   WhatsApp, email?) Phase 1 can run tickets inside the bot. Phase 2 syncs
   from your existing system.
2. **What makes a ticket critical?** And who raises tickets: operators, clients, or
   site staff?
3. **Telegram okay?** Do both team members have it, ideally on work phones?
4. **Escalation contact:** you? Should you get every plan, or only the digest
   and exceptions?
5. **Working days, hours and timezone.** Saturdays? Is anyone covering after hours?
6. **Missed calls:** office line, or personal phones? (Phase 1 = entered by hand.)
7. **Knowledge base:** is "deploying systems" about on-site parking hardware and site
   commissioning, or software? Where do current SOPs live, if anywhere?
8. **Hosting:** okay with ≈$5/month on a small cloud server under your account?

## 11. How we'll know it works (pilot metrics)

| Metric | Target after 2 weeks |
|---|---|
| Plan submitted by 09:30 | ≥ 90% of working days |
| EOD submitted by 18:15 | ≥ 95% |
| Check-in reply within 20 min | ≥ 80% |
| Critical ticket acknowledged within 10 min | ≥ 95% |
| Carryover items older than 3 days | Trending down |
| Team says it saves them time (quick poll) | Yes from both |
