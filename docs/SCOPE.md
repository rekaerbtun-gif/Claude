# Ops Daily Bot — Product Scope (Draft v0.5)

> Working name. An internal product for the parking-operations team: a **Microsoft
> Teams bot** that runs the daily rhythm (morning briefing → 1-3-5 plan → check-ins →
> end-of-day close-out), and a **web app** for tickets, SLAs, and admin. Both share one
> database.

Status: **Draft v0.5 for review.** The open decisions are in §14.
The quality bar every feature must meet is in §16–§18.

**Changes since v0.4:** Teams replaces WhatsApp as the channel. Excel is gone: the web
app is the ticket system from day one. Added SLA rules (§7) and a note on Jira (§19).

---

## 1. Problem & goal

The ops team (2 people today) loses track of what carried over from yesterday,
which support tickets are still open, and what each person committed to deliver.
Tickets sit in an Excel sheet. Updates sit in people's heads, WhatsApp groups, and
missed-call logs.

**Goal:** every working day follows the same closed loop, and nothing falls
through the cracks between days:

```
 09:00 Morning briefing ──► 1-3-5 plan ──► Check-ins every ~2h ──► 18:00 close-out
        ▲  (carryover + open tickets)                                    │
        └─────────── anything still open becomes tomorrow's carryover ◄──┘

 At any time: 🚨 critical ticket alert → acknowledge → update
```

**Design principle:** the product should *pre-fill and ask people to confirm*, not
make them retype. The fewer seconds each prompt takes, the more likely people
keep using it.

## 2. Users & roles

| Role | Who | What they do |
|---|---|---|
| **Operator** | 2 ops team members | Get prompts in Teams, plan their day, work tickets, close the day |
| **Manager** | You (owner) | See plans, check-ins, and close-outs; get escalations when something is missed, a critical ticket is unacknowledged, or an SLA is about to breach |
| **Admin** | You (or a delegate) | Manage users, masters (dropdown lists), SLA rules, working hours, and holidays in the web app |

## 3. The product: two parts, one database

```
   Microsoft Teams bot                      Web app (also inside Teams as a tab)
   - 09:00 briefing                         - Tickets: list, create, update, history
   - 1-3-5 plan                             - Masters: customers, sites, issue types…
   - Check-ins                              - SLA rules and breach tracking
   - 18:00 close-out                        - Today: everyone's 1-3-5 status
   - 🚨 critical alerts                     - Reports and knowledge base
            │                                           │
            └────────────────── one database ───────────┘
```

- **Teams bot** = the *nudges*. It comes to people at the right time and asks for quick updates.
- **Web app** = the *system of record*. Tickets, admin, SLAs, and reports all live here.
- An update in one shows up in the other immediately. There is no Excel and no
  second system to keep in sync.

## 4. Channel: Microsoft Teams

The company already uses Teams every day, so there's no new app to install or keep open.

| Need | How Teams handles it |
|---|---|
| Scheduled prompts (09:00, check-ins, 18:00) | The bot messages each person in a private 1-to-1 chat |
| Quick structured input | **Adaptive Cards**: dropdowns, text boxes, time pickers, and buttons inside the chat message |
| Shared visibility of emergencies | A team channel **"Ops – Critical"** where every 🚨 alert is posted |
| Web app access | Pinned as a **tab** in Teams, so tickets are one click away |
| Login | **Microsoft sign-in (SSO)** with existing company accounts. No new passwords |
| Commands | Teams shows the bot's command list as you type (`today`, `add`, `ticket`, …) |

**Setup needs:**
- Your Microsoft 365 admin approves the custom app once and can pre-install it for the
  team. Pre-installing is required so the bot can start conversations.
- A free bot registration in Microsoft Azure. The Teams channel costs nothing.

## 5. The daily flow, step by step

**Working hours:** 09:00–18:30 regular shift. Anything outside is logged as **extra time**.
Parking runs 24/7, so **critical** tickets alert around the clock (on-call). Non-critical
tickets raised after hours wait for the next morning's briefing.
All times below are defaults, configurable in the web app. *(Working days and timezone to be confirmed.)*

### 5.1 09:00 — Morning briefing

The bot sends each operator **one card** containing:

1. **Carryover from yesterday:** unfinished 1-3-5 items, close-out "open items",
   missed calls to return, emails still to draft.
2. **Open tickets:** *all* open tickets they own (not only yesterday's, so on Monday,
   Friday's tickets still show), sorted by priority then age, with SLA time left.
3. **Action required:** each ticket has a status button
   (`✅ Resolved` / `🔄 Update` / `⛔ Blocked` / `↪️ Reassign`). Choosing
   **Update** opens a one-line note box.

The briefing counts as complete only when every **critical** ticket has an
update. Other items can be carried into the plan.

```
☀️ Good morning, Ravi — Tue 14 Oct

🔁 Carried over (3)
 • Call back Lot C manager re: billing   [missed call, due 10:00]
 • Draft email to vendor re: barrier AMC
 • Replace ticket printer at Mall P2     [was in your 3s yesterday]

🎫 Open tickets (4)
 🔴 #212 Vehicle stuck at exit — Phoenix P2   (SLA: 20 min left)  [Update]
 🟠 #208 FASTag reader intermittent — P1      (SLA: 3 h left)     [Update]
 🟢 #205 Signage request — Tower A            (open 4d)           [Update]
 🟢 #199 Monthly pass refund                  (open 6d)           [Update]

Update 🔴 tickets first, then plan your 1-3-5 ➜
```

### 5.2 Critical ticket alerts (any time)

**Critical = anything that blocks money or traffic:**
1. The system is **not capturing payment** (pay station, FASTag, UPI, or card failure).
2. **A vehicle is stuck at exit**, or exit is delayed, for any reason (barrier, ANPR,
   ticket reader, payment, network).

The issue-type master (§8) marks which issue types are automatically critical.

What happens:
- The owner gets an immediate 🚨 card with an `Acknowledge` button, and the alert is posted
  in the **Ops – Critical** channel.
- **Escalation ladder:** not acknowledged in 10 min → re-alert. Not acknowledged in
  20 min → alert the other operator and the manager.
- After acknowledging, the owner is asked for an update every 60 min until it is resolved.
- **Out of hours:** alerts go to whoever is on call (a simple weekly rotation for 2 people),
  and the time spent is logged as extra time.
- **Phase 2:** an SMS or automated phone call if nobody acknowledges, because "Do not disturb"
  in Teams can silence notifications.

### 5.3 After the briefing — the 1-3-5 plan

| Tier | Meaning | Default "done by" time (editable) |
|---|---|---|
| **1** | The single most important outcome today | 12:00 |
| **3** | Three important tasks | 15:00 |
| **5** | Five smaller tasks needed to close the day well | 18:00 |

**Making it fast:**
- One card with the 1, 3, and 5 slots, each with a title and a time picker.
- **Suggested items** are listed first (critical or high tickets, then carryover). One tap
  adds a suggestion to a tier.
- Not every slot has to be filled (a field day might be just 1 + 2). The plan just needs at
  least the **1**.
- When the plan is submitted, the manager can see it in the web app's Today view.

Plan window: 09:00–10:00. No plan by 10:00 → reminder. By 10:30 → manager notified.

### 5.4 Check-ins (~every 2 hours)

Two triggers, merged so nobody gets pinged twice:

1. **Regular check-in** at 11:00, 13:00, 15:00, 17:00, listing every open item in one card.
2. **Deadline check-in** when an item's "done by" time arrives. If it falls within 30 min of
   a regular check-in, it is folded into that one instead.

Each item gets one-tap buttons:

`✅ Done` · `🔄 On track` · `⏰ Need more time` (asks for a new time) ·
`⛔ Blocked` (asks for a reason, then alerts the manager) · `🗑 Drop` (asks for a reason)

- **Unplanned work:** `add` logs an interruption (e.g. "gate broke at Lot D"), so the day
  is judged fairly when planned items slip.
- Items rescheduled **more than twice** are flagged in the manager summary.
- No reply within 20 min → one nudge. Two check-ins in a row with no reply → manager notified.

### 5.5 18:00–18:30 — End-of-day close-out (mandatory)

Starts **at 18:00**, so the close-out is finished *by* the 18:30 end of shift.

1. **Open items from today:** everything not yet ✅ is listed, and the operator chooses for each:
   carry over to tomorrow, hand over to a colleague, or drop.
2. **Missed calls:** who, number or site, what it was about, call back by when.
   ("None" is a valid one-tap answer.)
3. **Emails pending:** to whom, subject, by when. *(Phase 2: the bot drafts the email
   from these notes.)*
4. **Optional one-liner:** "Anything I should know?"
5. `Submit day` produces a summary card for the operator and manager.

**Enforcement** (we can't physically stop someone leaving, so we make skipping
visible and easy to fix):
- Reminders at 18:15, 18:30, and 18:40 until it is submitted.
- 18:45 → manager alerted: "Priya's close-out is pending."
- The day stays **open**, and tomorrow's briefing starts with "Finish yesterday's close-out".

Everything captured here feeds the next 09:00 briefing.

### 5.6 Manager views

- **19:00 daily digest** in Teams: each person's plan vs. actual, items carried over,
  tickets opened and closed, SLA breaches, missed calls outstanding, late check-ins.
- **Friday weekly report:** completion of 1s, 3s, and 5s, ticket volume per customer,
  SLA performance, critical acknowledgement times, most-rescheduled items.
- **Web app:** live Today view and ticket dashboards (§6).

### 5.7 Bot commands (shown in Teams' command list)

| Command | Does |
|---|---|
| `today` | Show my plan and status |
| `add` | Log unplanned work |
| `ticket` | Create a ticket from a card, or paste a message to turn it into one (§9) |
| `leave` | Mark days off, so prompts are skipped and tickets go to a colleague |
| `kb <words>` | Search the knowledge base |
| `help` | Show everything I can do |

## 6. Web app — the system of record

Everything about tickets and administration happens here, in a browser or as a tab in Teams.

| Screen | What it does |
|---|---|
| **Tickets** | All tickets in one table, filtered by status, priority, customer, site, owner, or age, with search. Critical, overdue, and near-breach tickets highlighted |
| **Ticket detail** | All fields, SLA clock, and a timeline of every update and status change, whether from the web or Teams |
| **New ticket** | Short form where every field is a dropdown from the masters (§8). Only description and remarks are free text |
| **Today** | Each operator's 1-3-5 plan and live status |
| **Masters** | Admin manages customers, sites, categories, issue types, priorities, and more (§8) |
| **SLA rules** | Admin defines response and resolution targets (§7) |
| **Settings** | Working hours, check-in times, tier defaults, holidays, on-call rotation |
| **Reports** | SLA performance, tickets by customer, site, and issue type, team completion. Download as Excel or CSV for sharing with clients |
| **Knowledge base** | SOPs and how-to guides (§10) |

**Going live:** currently open tickets are entered fresh in the web app on day one
(usually a manageable number). The Excel file is then archived. If there are many open
tickets, a one-off import can be done, but it isn't a dependency.

## 7. SLA rules

Defined in the web app, no code changes needed.

| Setting | Example |
|---|---|
| **Per priority** | Critical: acknowledge in 10 min, resolve in 1 h · High: 1 h / 8 h · Normal: 4 h / 2 days · Low: 1 day / 5 days |
| **Clock** | Critical runs 24/7. Others count only working hours (09:00–18:30, working days) |
| **Per-customer override** *(optional)* | A client with a stricter contract gets its own targets |
| **Pause** | The clock pauses while a ticket is "Waiting on customer" or "Waiting on vendor" |

**What the product does with SLAs:**
- Shows "time left" on every open ticket, in the web app and in the bot's cards.
- Warns the owner at **80%** of the time, and alerts the manager on **breach**.
- Reports SLA performance per customer, site, and issue type.

## 8. Masters (dropdown lists, built for many customers)

Every ticket field is chosen from a **master list**, except the description and
remarks. The admin edits masters in the web app, and changes show up immediately in
web forms and Teams cards.

| Master | Example values | Linked to |
|---|---|---|
| **Customer** (client) | Phoenix Mall, City Hospital, Tech Park A | Optional SLA override |
| **Site / location** | Phoenix Mall – P2, City Hospital – Basement | Customer, default owner |
| **Equipment / lane** *(optional)* | Exit lane 2 barrier, Entry ANPR cam 1 | Site |
| **Category** | Payment, Entry/Exit, Hardware, Software, Billing, Request | — |
| **Issue type** | Payment not captured, Vehicle stuck at exit, Barrier not opening, FASTag not reading | Category, with a default priority and an **auto-critical** flag |
| **Priority** | Critical, High, Normal, Low | SLA targets (§7) |
| **Status** | Open, Acknowledged, In progress, Waiting on customer, Waiting on vendor, Resolved, Closed | Whether the SLA clock runs |
| **Owner** | Ravi, Priya | — |
| **Raised by (contact)** | Site supervisor at Phoenix P2 (+91 …) | Customer or site |
| **Source / channel** | WhatsApp group, Phone call, Email, Site visit, Internal | — |

**Smart defaults, so most tickets need 2–3 taps:**
- Pick the **site**, and the customer and default owner fill in automatically.
- Pick the **issue type**, and the category and priority fill in. Auto-critical types trigger 🚨 alerts.
- Ticket number, dates, SLA deadlines, and "after hours" are filled in automatically.

**Multi-customer from day one:** every ticket belongs to a customer, so reports, SLAs,
and later a customer portal can be split per client without restructuring.

Masters are **deactivated, not deleted**, so old tickets keep their history.

## 9. Getting tickets out of WhatsApp groups

Customers will keep reporting issues in WhatsApp groups, and no tool can read those
groups automatically. So we make the hand-over take about 5 seconds:

1. The operator long-presses the customer's message in WhatsApp → **Share** → **Teams** → the bot.
2. The bot reads the message and proposes a ticket:
   ```
   🎫 New ticket from message
   "Exit barrier at P2 not opening, 4 cars waiting"
   Site: Phoenix Mall – P2 ▾   Issue: Vehicle stuck at exit ▾   Priority: 🔴 Critical
   [✅ Create ticket]  [✏️ Edit]
   ```
3. One tap creates it, and critical tickets trigger the 🚨 alert flow.

- If sharing from WhatsApp to Teams doesn't work smoothly on a particular phone, copy and
  paste into the bot (`ticket` + paste) does the same thing. We'll test both on the team's
  phones during the pilot.
- Phase 1 suggests fields with simple keyword rules. Phase 2 uses AI (Claude) to read
  the message and match it to the masters more accurately. The operator always confirms.

## 10. Knowledge base

**Assumption to confirm:** "deploying systems" means field SOPs for the parking business,
such as commissioning a new site, installing and configuring ANPR cameras, boom barriers,
FASTag readers, pay stations, and handling common faults.

- **Phase 1:** SOP pages in the web app, searchable from Teams with `kb barrier calibration`.
- **Phase 2:**
  - When a ticket is resolved, the bot asks "Worth an SOP? What fixed it?" Those notes become
    drafts that the manager approves.
  - Ticket cards link the relevant SOP (e.g. a barrier ticket → barrier troubleshooting).
  - An **onboarding path** for new hires: an ordered list of SOPs with a short check-off,
    so they "progress into the job".

## 11. What can go wrong — and how we handle it

| Risk | Impact | Mitigation |
|---|---|---|
| **Notification fatigue:** team mutes the bot | The whole system fails silently | Few, batched messages. Deadline pings fold into check-ins. Only 🚨 alerts break through. Review the volume after week 1 |
| **Feels like surveillance** | Resentment, fake updates ("working on it") | Frame it as *their* tool: carryover memory, fewer things forgotten. `add` credits unplanned work. Share metrics as team trends, not a leaderboard |
| **Low-quality updates** | Manager can't trust status | Status is always a button. Blocked, dropped, and rescheduled items require a reason |
| **Prompt ignored** (busy in the field) | Gaps in the record | One nudge, then manager escalation. Late answers are still accepted and marked late |
| **"Do not disturb" silences a critical alert** | Vehicle stuck, nobody responds | Alert also posted in the Ops – Critical channel. Escalation ladder to the other operator and manager. Phase 2: SMS or call fallback |
| **Leave, sick days, holidays** | Prompts and alerts to an absent person | `leave` and a holiday calendar skip prompts and reroute tickets to the other operator |
| **Both people unavailable** | Critical tickets unattended | The manager is always the final escalation step |
| **After-hours critical ticket** (parking runs 24/7) | Revenue or customer impact overnight | On-call rotation. Out-of-hours alerts only for critical tickets |
| **Customer issues stay in WhatsApp** | Tickets never get logged | Share-to-bot takes 5 seconds (§9). The 09:00 briefing asks "Any issues from customer groups not yet logged?" |
| **Everything gets rescheduled** | The plan becomes meaningless | Reschedule counter. Items moved >2× are flagged. Weekly report shows patterns |
| **Bot server goes down** | No 09:00 prompt and nobody notices | Health check, plus an alert to the manager if the briefing didn't go out. Missed jobs run on restart |
| **Microsoft 365 sign-in problem** | Nobody can log in to the web app | One emergency admin login that doesn't depend on Microsoft sign-in |
| **App not installed for a user** | The bot can't message them | Admin pre-installs the app for the team. The web app shows who hasn't connected |
| **Wrong timezone or clock** | Prompts at 3 am | Timezone set explicitly in settings. Tested in a test team first |
| **Customer data** (phone numbers, plates) | Privacy exposure | Company sign-in only. Store the minimum. Customer details never posted in channels |
| **Scope creep** | Never ships | Phase 1 is only what's listed in §12 |

## 12. Phased delivery

**Phase 1: the daily loop and the ticket system (≈2–3 weeks of build plus a 1-week pilot)**
- Web app with Microsoft sign-in: tickets, ticket detail and history, new ticket form
- Masters and SLA rules managed in the web app, with SLA clocks, 80% warnings, and breach alerts
- Teams bot: 09:00 briefing, 1-3-5 plan, check-ins, 18:00 close-out with reminders and escalation
- Critical alerts: 🚨 card, Ops – Critical channel, escalation ladder, on-call, extra-time log
- "Ticket from message" via share or paste, with simple keyword suggestions
- Today view and manager daily digest
- `leave`, `today`, `add`, `help`
- Knowledge base pages with `kb` search

**Phase 2: smarter and louder**
- AI (Claude) reads shared messages and fills in ticket fields more accurately
- AI drafts the pending emails from the close-out notes
- SMS or phone-call fallback for unacknowledged critical alerts
- Weekly report, SLA and reschedule analytics
- Knowledge base capture from resolved tickets, and an onboarding path
- QR code at each site that opens a "Report a problem" form

**Phase 3: scale**
- Customer portal: each client sees their own sites' tickets and SLA report
- Voice-note updates (transcribed)
- Optional Jira link for software bugs (§19)

## 13. Technical approach (proposed)

- **Python / Django** for the web app (database, admin, migrations, and permissions built in)
- **Microsoft Teams bot** through Azure Bot Service (free Teams channel), with
  **Adaptive Cards** for all prompts. Same project, same database
- **Microsoft Entra ID (SSO)** for sign-in to the web app and the Teams tab
- **SQLite** to start (plenty for 2–10 users, easy to back up). Postgres when it grows
- **Hosting:** a small cloud server (≈ $5/month) with a domain and HTTPS, which Teams
  needs to deliver messages to the bot
- **Claude API** in Phase 2 for reading messages and drafting emails
- Tests for the scheduling and SLA logic (merging, escalations, business-hours clock,
  timezone), since that is the riskiest part
- A separate **test team** in Teams, so changes are tried before they reach the operators

## 14. Decisions needed from you

1. ~~Where do tickets live?~~ **Answered:** in the new web app. Excel is retired at go-live.
2. ~~What makes a ticket critical?~~ **Answered:** payment not captured, or a vehicle stuck
   or delayed at exit (§5.2).
3. ~~Which channel?~~ **Answered: Microsoft Teams.** Needed: who your Microsoft 365 admin is,
   for about 10 minutes to approve the app.
4. ~~Working hours?~~ **Answered: 09:00–18:30, critical 24/7.** Still needed: working days
   (Saturdays?), timezone, and who is on call after hours.
5. **SLA targets:** are the example targets in §7 close? Do any clients have contractual SLAs?
6. **Masters:** your current list of customers, sites, and issue types, so they're pre-loaded (§8).
7. **Escalation contact:** you? Should you see every plan, or only the digest and exceptions?
8. **Missed calls:** office line, or personal phones? (Phase 1 = entered by hand.)
9. **Knowledge base:** is "deploying systems" about on-site parking hardware and site
   commissioning, or software? Where do current SOPs live, if anywhere?
10. **Software bugs:** is there an in-house developer team or a vendor that fixes the parking
    software, and what do they use to track work? (§19)
11. **Hosting:** okay with ≈$5/month on a small cloud server under your account?

## 15. How we'll know it works (pilot metrics)

| Metric | Target after 2 weeks |
|---|---|
| Plan submitted by 10:00 | ≥ 90% of working days |
| Close-out submitted by 18:30 | ≥ 95% |
| Check-in reply within 20 min | ≥ 80% |
| Critical ticket acknowledged within 10 min | ≥ 95% |
| Tickets within SLA | Baseline set in week 1, then improving |
| Carryover items older than 3 days | Trending down |
| Team says it saves them time (quick poll) | Yes from both |

---

## 16. Product principles — what makes this work

These are the tests every feature has to pass. If a feature breaks one, we
change or cut it.

| # | Principle | What it means in practice | How we check it |
|---|---|---|---|
| 1 | **Faster than not using it** | Each prompt takes under 30 seconds. The whole day's overhead is under 10 minutes | Time the pilot: morning ≤ 3 min, each check-in ≤ 20 s, close-out ≤ 3 min |
| 2 | **Confirm, don't type** | The system already knows yesterday's items, tickets, and plans, so people tap to confirm and type only what's new | Fewer than 1 in 3 interactions need typed text |
| 3 | **Reliable like a utility** | The 09:00 prompt arrives every working day, on time. Data is never lost. If something breaks, *we* know before the team does | 100% of scheduled prompts sent in the pilot. Daily backups. Health alerts |
| 4 | **Calm by default** | Predictable times, batched messages. Only 🚨 critical tickets interrupt | ≤ 8 bot messages per person on a normal day |
| 5 | **Gives back to the operator** | It remembers for them, shows what they got done, and protects them when fire-fighting derails the plan | Both operators say it helps *them* (pilot poll) |
| 6 | **Fair** | Unplanned work counts. Late answers are accepted. Leave is respected | No one is flagged for a day spent on a logged critical ticket |
| 7 | **Forgiving** | Anything can be edited or undone. Nothing gets a person stuck | Every action has an Undo or Edit path |
| 8 | **One owner, one time** | Every item and ticket has exactly one owner and a "by when" | No ownerless items in the database |
| 9 | **One source of truth** | Tickets and tasks live only in the web app's database. Teams shows the same data | No spreadsheets or side lists after go-live |
| 10 | **Visible, not policed** | The manager sees exceptions (blocked, late, missed, breaching), not a minute-by-minute feed | Manager receives ≤ 3 messages a day plus the digest |

## 17. Experience design standards

A professional feel comes from consistency more than features.

**Identity**
- A proper name, an icon, and a short description in Teams. The bot introduces itself on first use.
- One consistent tone: brief, warm, and direct. It uses first names, and it doesn't
  guilt-trip or overuse exclamation marks.

**Card design system:** every bot card follows the same layout:

```
<icon> <Title> — <context: date / time / count>     ← what this is, at a glance
<body: max ~7 lines, grouped, most urgent first>    ← only what's needed now
<progress: "2 of 4 tickets updated">               ← where they are
[Primary action] [Secondary] [Later]               ← one obvious next step
```

The web app uses the same colors, icons, and wording as the cards, so both parts feel like one product.

**Fixed visual vocabulary** (always the same meaning, nothing else):

| Priority | Status | Message types |
|---|---|---|
| 🔴 Critical · 🟠 High · 🟢 Normal | ✅ Done · 🔄 On track · ⏰ Rescheduled · ⛔ Blocked · 🗑 Dropped | ☀️ Briefing · 🎯 Plan · 🕑 Check-in · 🌙 Close-out · 🚨 Alert · 📊 Digest |

**Interaction rules**
- Buttons and dropdowns first. Typing is only for genuinely new information.
- A long flow is broken into steps with progress shown ("Step 2 of 3: missed calls").
- Every completed flow ends with a short confirmation card. Nothing is left ambiguous.
- If the bot doesn't understand something, it offers the likely options instead of "invalid input".
- Escalation messages to the manager stay factual and neutral, e.g. "Priya's close-out is
  pending (18:45)", never accusatory.

**Copy examples**

| Instead of | Write |
|---|---|
| "ERROR: Plan not submitted." | "No plan yet today. Want to start with your carryover items?" |
| "You have failed to update ticket #212." | "#212 (Phoenix P2 exit) still needs an update. 1 tap ➜" |
| "Submit EOD now." | "🌙 Time to close out. About 2 minutes, then you're done for the day." |

## 18. Running it like a real product

**Reliability and trust**
- A separate **test team** in Teams and a test copy of the web app. Changes are tried there first.
- Monitoring: if the 09:00 briefing hasn't gone out by 09:05, the manager gets alerted.
- Automatic daily database backups, with a restore that has been tested once.
- An **audit trail**: every change records who, when, and the old and new values.
- Data retention policy, e.g. customer phone numbers removed after 90 days.

**Onboarding and adoption**
- **Co-design:** walk the 2 operators through the flow *before* building, and adjust it with
  them. People use what they helped shape.
- A first-run tour in the bot (2 minutes), plus a one-page guide.
- **Shadow week:** everything runs, but escalations to the manager are switched off. We tune
  timings and wording. Then go live.
- A `feedback` command available any time. A 15-minute review at the end of each pilot week.

**Evolving it**
- Versioned releases with a short "What's new" card to the team.
- A monthly look at the metrics in §15. Cut prompts nobody values, and add only what removes
  real friction.
- Settings (times, tiers, SLA rules, masters) change in the web app without code changes.

## 19. Jira and other tools

**Jira is not needed for the ticket system.** The web app already covers tickets, admin,
SLAs, and reporting. Adding Jira underneath would mean two systems to keep in sync, extra
licenses, and a more complicated tool for the operators.

**Where Jira could help later:** if a ticket turns out to be a *software bug* (for example the
parking software miscalculating fees), and a developer team or vendor tracks its work in Jira:
- An `Escalate to Jira` button on the ticket creates a Jira issue with the details.
- The ticket shows the Jira link and status, and moves to "Waiting on vendor" (SLA paused).
- When the Jira issue is closed, the ticket owner is notified to confirm with the customer.

The ops team never has to open Jira. This is Phase 3, and only if a developer team actually uses Jira (§14, question 10).
