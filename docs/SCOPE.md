# Ops Daily System — MVP Scope (v0.6)

> An MVP for the parking-operations team, built on **Microsoft 365 tools you already have**:
> **Teams** (what operators use), **Microsoft Lists** (where data is stored), and **Power
> Automate** (the automation). No custom app, no code, no extra subscriptions.

**Status:** for final confirmation before the build starts.
**Changes since v0.5:** scope narrowed to a no-code MVP that tests the process first.
Phone-call escalation is out for now. The full custom product is described in §15 as the
possible next step. Earlier versions are in the git history.

---

## 1. What we're trying to achieve

Today, tickets sit in an Excel sheet, updates live in people's heads and WhatsApp
groups, and nothing ensures that open work from yesterday gets picked up today.

**The MVP puts the team on one daily loop, run automatically in Teams:**

```
 09:00 Briefing ──► 1-3-5 plan ──► Check-ins 11/13/15/17 ──► 18:00 Close-out
   ▲ (yesterday's leftovers + open tickets)                       │
   └─────────── anything still open shows up the next morning ◄───┘

 Any time: 🚨 critical ticket → alert → escalate if nobody responds
```

**The MVP also answers one question:** does the team actually use this daily process? After a
2-week pilot we'll know which parts help, which annoy, and what (if anything) is worth
building as a custom app.

## 2. Who uses it

| Role | Who | What they do |
|---|---|---|
| **Operator** | 2 ops team members | Get the daily prompts in Teams, plan their day, work tickets, close out |
| **Backup** | 1 person from another team | Gets critical alerts if the owner doesn't respond (escalation level 2) |
| **Manager / Admin** | You | See everything, get escalations, edit dropdown lists and settings |

## 3. How it fits together

```
  WHAT PEOPLE USE                 AUTOMATION                     WHERE DATA LIVES
  Microsoft Teams                 Power Automate                 Microsoft Lists
  (phone + laptop)                (Microsoft's built-in          (spreadsheet-like tables
                                   automation tool)               with dropdowns)

  • 09:00 briefing      ◄────── scheduled flows ──────────────► • Tickets
  • 1-3-5 plan form               (09:00, 10:00, 11-17,          • Daily plan items
  • Check-ins                      18:00, 19:00)                 • Close-out answers
  • 18:00 close-out                                              • Missed calls, pending emails
  • 🚨 critical alerts  ◄────── "new critical ticket" trigger ◄─ • Dropdown lists (customers,
  • Tickets tab  ─────────────────────────────────────────────►    sites, issue types, people…)
```

- **Teams is the app operators use.** It's already on their phones. All prompts and
  alerts arrive there, and they open and update tickets from a tab inside Teams.
- **Lists replaces Excel.** It looks and feels like a spreadsheet, with enforced dropdowns,
  per-ticket history, and a mobile-friendly form.
- **Power Automate does the work** of sending prompts on time, saving answers, and escalating.

## 4. What gets built

### 4.1 Teams setup
- A team **"Ops"** with two channels:
  - **General**, for normal team chat.
  - **Ops – Critical**, where every 🚨 critical alert is posted for everyone to see.
- Tabs in the team: **Tickets** (open, mine, critical, all), **Today** (everyone's 1-3-5),
  and **Admin** (dropdown lists and settings, for you only).

### 4.2 Lists (the data)

| List | What it holds |
|---|---|
| **Tickets** | One row per ticket: number, date and time raised, customer, site, category, issue type, priority, status, owner, raised by, source, description, remarks, acknowledged at, resolved at |
| **Daily Items** | Each person's 1-3-5 items: tier (1/3/5/unplanned), title, done-by time, status, note, carried over from |
| **Close-outs** | One row per person per day: submitted at, answers to the close-out questions |
| **Missed Calls** | Who, number or site, reason, call back by, done |
| **Pending Emails** | To whom, subject, by when, done |
| **Dropdown lists** | Customers, Sites (linked to a customer and a default owner), Categories, Issue Types (with a default priority and an "always critical" flag), Priorities (with an age limit, e.g. Critical 1 h), Sources (WhatsApp group, phone call, email, site visit, internal) |
| **Team Members** | Name, role, active, on leave today |
| **Escalation Chain** | Level 1: ticket owner · Level 2: backup person · Level 3: you, with the minutes between levels |
| **Close-out Questions** | The questions asked at 18:00, editable by you |

### 4.3 Automations (Power Automate flows)

| # | Flow | When | What it does |
|---|---|---|---|
| 1 | **Critical alert** | A ticket becomes Critical | 🚨 card to the owner with an `Acknowledge` button, and a post in Ops – Critical. No acknowledgement in 10 min → alert to the backup. 20 min → alert to you. All in Teams |
| 2 | **Morning briefing** | 09:00, working days | Each operator gets yesterday's leftovers (unfinished items, missed calls, pending emails) and their open tickets with ticket age, with a status button per ticket |
| 3 | **1-3-5 plan** | Right after the briefing | Form with slots for the 1, 3, and 5, each with a done-by time, and yesterday's leftovers offered as one-tap picks. Saved to Daily Items |
| 4 | **Plan reminder** | 10:00 / 10:30 | No plan by 10:00 → reminder. By 10:30 → you're notified |
| 5 | **Check-ins** | 11:00, 13:00, 15:00, 17:00 | Lists each person's open items with a status choice for each: Done / On track / Need more time / Blocked / Drop, plus an optional note. Blocked → you're notified |
| 6 | **Close-out** | 18:00 | Confirm each unfinished item (carry over / hand over / drop), then missed calls, pending emails, and your close-out questions. Reminders at 18:15 and 18:30. Not done by 18:45 → you're notified |
| 7 | **Ticket age check** | Every hour | Flags tickets open longer than their priority's age limit to the owner and you |
| 8 | **Daily digest** | 19:00 | Your summary: each person's plan vs. done, carried-over items, tickets opened and closed, overdue tickets, late or missing close-outs |

### 4.4 Documentation (in this repo)
- **Build guide:** click-by-click instructions to create every list, flow, and tab, including
  the design of each Teams card.
- **Admin guide:** how to add a site, customer, issue type, person, or close-out question, and
  change a time.
- **Operator guide:** one page showing what arrives when and what to tap.
- **Test checklist:** every flow tested in a test team before going live.

## 5. A day with the system

| Time | Operator sees in Teams | You see |
|---|---|---|
| **09:00** | ☀️ Briefing: 3 carried-over items, 4 open tickets (🔴 #212 *open 16h*). Taps a status on each ticket | — |
| **09:05** | 🎯 Plan form: picks "Call back Lot C manager" from leftovers as a 3, types the 1 ("Fix Phoenix P2 exit barrier by 11:00"), adds the rest | The plan appears in the Today tab |
| **10:00** | *(only if no plan yet)* reminder | 10:30: notified if still no plan |
| **11:00** | 🕑 Check-in: marks the 1 as ✅ Done and one 3 as ⏰ "need until 14:00" | Today tab updates |
| **12:40** | 🚨 New critical ticket: "Payment not captured – City Hospital". Taps `Acknowledge` | Posted in Ops – Critical. If nobody acknowledged: backup alerted at 12:50, you at 13:00 |
| **13:00–17:00** | Check-ins every 2 hours | Notified only if something is ⛔ Blocked |
| **18:00** | 🌙 Close-out: 2 items carried to tomorrow, 1 missed call logged, 1 email pending. Submit | 18:45: notified only if not submitted |
| **19:00** | — | 📊 Daily digest |
| **Next day 09:00** | Yesterday's carried items, missed call, and pending email appear in the briefing | — |

## 6. What you can change yourself (no rebuild needed)

| Change | How |
|---|---|
| Add or edit customers, sites, categories, issue types, priorities, sources | Edit the dropdown lists. Changes show up immediately in the ticket form and Teams cards |
| Mark an issue type as "always critical" | Tick the flag in Issue Types |
| Change the age limit per priority | Edit Priorities |
| Add, remove, or reorder close-out questions | Edit Close-out Questions |
| Add, remove, or make required a ticket field | Edit the Tickets list's columns. The form updates automatically. *(If the new field should also appear in alerts, that's a 10-minute flow edit, covered in the admin guide.)* |
| Change the backup person or escalation minutes | Edit Escalation Chain |
| Add a team member or mark someone on leave | Edit Team Members |
| Change prompt times (e.g. briefing at 08:45) | A small edit in the flow's schedule (covered in the admin guide) |

## 7. Escalation rules (Teams only)

| Situation | Escalation |
|---|---|
| Critical ticket not acknowledged | 10 min → backup person · 20 min → you |
| No 1-3-5 plan | 10:00 reminder · 10:30 → you |
| Item marked Blocked | Immediately → you |
| Close-out not submitted | 18:15 and 18:30 reminders · 18:45 → you |
| Ticket older than its priority's age limit | Hourly flag → owner and you |

**Critical** = system not capturing payment, or a vehicle stuck or delayed at exit.
Parking runs 24/7, so critical alerts are sent at any hour. Regular prompts only run during
working hours (09:00–18:30).

## 8. Not included in the MVP

| Left out | Why / what happens instead |
|---|---|
| Phone calls and SMS for critical alerts | Out for now. Escalation is by Teams to the backup and you |
| Real SLA timers (counting working hours only, pausing while waiting) | Ticket age plus an age limit per priority instead |
| Automatic ticket creation from WhatsApp groups | Operators paste the customer's message into the New ticket form |
| AI (reading messages, drafting emails) | Later, if the process proves itself |
| Knowledge base | Later |
| Automatic leave and holiday handling | You mark "on leave" in Team Members |
| Deadline-based check-ins (pinging when each item is due) | Fixed check-ins at 11, 13, 15, and 17 |
| Branded bot name and icon | Messages come from Microsoft's "Workflows" sender |
| Dashboards and reports | Saved views and filters in Lists, plus Export to Excel |
| Customer portal, Jira, other integrations | Later, if needed |

## 9. Known limitations of this approach

- **"Do not disturb" in Teams can silence alerts.** The Ops – Critical channel and escalation
  to the backup and you are the safety net.
- **Old prompt cards stay clickable.** Answering yesterday's check-in late is harmless. The
  flows are designed so late answers are saved and marked late.
- **The automations depend on the account that owns them.** They are built under a shared
  account (e.g. ops-automation@yourcompany), not a person's account.
- **Changes are made by clicking, not code,** so there's no automatic undo history. The build
  guide documents every flow so it can be rebuilt.
- **Everyone with access to a list can edit it.** Admin lists are restricted to you. Tickets
  and plans are editable by the team.
- **Alerts can lag by about a minute** after a ticket is saved.

## 10. Risks and how we handle them

| Risk | Mitigation |
|---|---|
| Too many messages, so the team ignores them | ≤ 8 messages per person on a normal day. Review the volume after week 1 |
| Feels like surveillance | Present it as the team's memory and safety net. Unplanned work can be logged. You see exceptions, not every tap |
| Vague updates ("working on it") | Status is always a choice. Blocked and dropped items require a reason |
| Customer issues stay in WhatsApp and never become tickets | The morning briefing asks "Any customer issues not yet logged?" Paste-to-ticket takes under a minute |
| A flow silently stops working | Daily digest to you shows if prompts went out. Power Automate emails the owner account when a flow fails |
| Wrong timezone or working days | Set explicitly and tested in a test team first |

## 11. What we need from you before building

- [ ] **Microsoft 365 admin confirms** that Lists and Power Automate are available on your plan
- [ ] **A shared automation account** (e.g. ops-automation@yourcompany) with a Microsoft 365 licence
- [ ] **Your Excel column headers,** to finalise the Tickets list
- [ ] **Starting dropdown values:** customers, sites, issue types (a rough list is fine)
- [ ] **Backup person's name** (escalation level 2). You are level 3
- [ ] **Working days** (Mon–Fri or Mon–Sat?) and **timezone**
- [ ] **Who builds it on your side:** you, or someone comfortable clicking through Microsoft 365

## 12. Effort, time, and cost

| | Estimate |
|---|---|
| **My work** | Build guide, admin and operator guides, card designs, test checklist. 1–2 sessions, plus answering questions while you build |
| **Tokens** | Roughly 2–5 million in total, including support during the build. Rough estimate |
| **Your build time** | 2–4 days following the guide (1–2 days if the builder knows Power Automate) |
| **Testing** | 1–2 days in a test team |
| **Live** | About 1 week from start |
| **Pilot** | 2 weeks, then a review |
| **Running cost** | No new spend if Lists and Power Automate are in your plan. One licence for the shared automation account |

## 13. Rollout plan

1. **Build** the lists, flows, and Teams tabs following the guide.
2. **Test** every flow in a private test team, using the test checklist.
3. **Walk the 2 operators through it** (20 minutes) and adjust anything that feels off.
4. **Shadow week:** everything runs, but escalations to you are off. Tune times and wording.
5. **Go live:** escalations on. Enter the currently open tickets and archive the Excel file.
6. **Pilot review after 2 weeks** against the metrics in §14. Decide: keep as is, adjust, or
   build the custom product (§15).

## 14. How we'll know it works (pilot targets)

| Metric | Target |
|---|---|
| Plan submitted by 10:00 | ≥ 90% of working days |
| Close-out submitted by 18:30 | ≥ 95% |
| Check-ins answered within 20 min | ≥ 80% |
| Critical tickets acknowledged within 10 min | ≥ 95% |
| Tickets in Lists, not elsewhere | 100% after go-live |
| Carried-over items older than 3 days | Trending down |
| Operators say it helps them (quick poll) | Yes from both |

**Design principles for every prompt:** under 30 seconds to answer, pick instead of type, one
obvious next step, the same icons and wording everywhere (🔴 Critical · 🟠 High · 🟢 Normal ·
✅ Done · 🔄 On track · ⏰ Need more time · ⛔ Blocked · 🗑 Dropped), and a warm, neutral tone that never guilt-trips.

## 15. After the MVP: the full product (only if the pilot justifies it)

If the process works and the MVP's limits start to hurt, the next step is a custom product:
- A **branded Teams bot** with smarter prompts (deadline-based check-ins, multi-step escalation).
- A **web app** for tickets with real SLA timers, dashboards, and admin screens.
- Phone or SMS escalation for critical tickets.
- AI that turns pasted WhatsApp messages into tickets and drafts pending emails.
- A knowledge base and onboarding path, a customer portal, and an optional Jira link for software bugs.

The lists and dropdown values from the MVP move straight into it, so nothing is wasted.
