---
name: weekly-attorney-dashboard
description: Build an attorney's weekly time and billing summary from LeanLaw — billable and non-billable hours by client and matter against the prior week, pace against a billable-hours target, unbilled work on the matters they're responsible for, draft and in-review invoices waiting on them with links to the pre-bill, and flags on entries that may not survive review (block billing, vague narratives). Use when an attorney asks for their weekly dashboard, weekly summary, "how did my week look", "am I on pace", "what's waiting on me before month-end", or when running it as a weekly scheduled task. Read-only; it never edits time or invoices. Not for firm-wide or partner rollups, AR aging or collections. Requires the LeanLaw MCP connector.
---

# Weekly attorney dashboard

One run produces **one attorney's summary for one week**. It is read-only: every tool it
calls is a `list_` or `get_` call, and it never creates, edits or approves anything.

It answers three questions for the attorney:

1. **Is my time going in?** Hours for the week, by client and matter, and against the
   target. (Utilization.)
2. **Will it survive to the invoice?** Entries likely to be written down at review.
   (Realization.)
3. **What's stuck on me?** Unbilled work and invoices waiting for my review. (Billing
   velocity: work that sits in WIP or in a draft is work that isn't cash yet.)

## Tools

All tools come from the **LeanLaw MCP connector**. The names below are logical names;
the connector adds its own prefix, which differs per install, so match on the suffix.

| Need | Tool |
|---|---|
| What this connection may read | `list_authorizations` |
| Resolve the attorney | `list_users` |
| Hours for the week | `list_time_entries` |
| Matters they're responsible for | `list_matters` |
| Unbilled work | `list_time_entries`, `list_fixed_fees`, `list_expenses` (all with `billed: false`) |
| Invoices waiting on them | `list_invoices` |

If the connector isn't available, say so and stop. Don't estimate numbers from memory or
from anything other than the connector.

## Step 0: Preflight

Run in parallel:

- `list_authorizations`: needs read on time entries, matters and invoices. Fixed fees and
  expenses are optional; if either is missing, leave it out of WIP and say so in the
  report.
- `list_users` to resolve the attorney (Step 1).

## Step 1: Whose dashboard, and which week

**Attorney.** The connector acts as one named user but has no "who am I" call, so
resolve the attorney explicitly:

- If you know the user's email (from the conversation, the scheduled task prompt, or the
  agent's account context), call `list_users` with `email`.
- Otherwise ask for their name or email and use `list_users` with `query`.
- Require exactly one match. On a shared name, ask which person, listing email and role.

Keep `userId` for the rest of the run. When setting this up as a scheduled task, put the
attorney's email in the task prompt so later runs don't need to ask.

**Week.** Weeks run Monday to Sunday. Default to the **last complete week** (on a
Monday, that's the seven days that just ended). If the user asks for "this week", use
Monday through today and label it "week to date". Compare against the seven days before.

**Target.** The connector has no billable-target field. Use, in order:

1. A target the user states, or one written in the scheduled task prompt.
2. A firm table the user has added at [references/targets.md](references/targets.md).
3. Nothing. If there's no target, leave the pace section out of the report entirely. Don't
   invent a default.

A target can be weekly, monthly or annual. Convert it to the reporting week and state the
conversion: annual ÷ 48 working weeks, monthly × 12 ÷ 48, unless the firm's table says
otherwise.

## Step 2: Hours

Two calls, in parallel, each with `userId`, `startDate`, `endDate`, `limit: 1000`: one
for the week and one for the prior week. If `pagination.total` is more than the page,
page with `offset` until you have everything. A partial week of hours is a wrong number,
not an approximate one.

Group the week's entries by client and matter and sum `hours` by `billingType`:

- `Billable` counts toward the target.
- `FixedFee` is productive time on flat-fee work. Show it separately. It counts toward
  the target only if the firm's target says so.
- `NonBillable` is shown but never counts toward the target.

Show `amount` (value at rate) for billable time where the entry has one. Don't compute a
value for entries without a rate.

Pace: `billable hours ÷ weekly target`. Also show month-to-date if the target is monthly
and today is past the first week of the month; this needs one more `list_time_entries`
call from the first of the month.

## Step 3: Matters they're responsible for

`list_matters` with `responsibleId: userId`, `archived: false`, `limit: 500`. Page if
needed. Keep the set of `matterId`s. Steps 4 and 5 are about these matters, which covers
work logged by anyone on them, not only by the attorney.

## Step 4: Unbilled work

For each responsible matter, sum unbilled work:

- `list_time_entries` with `matterId`, `billed: false`: hours and `amount`.
- `list_fixed_fees` with `matterId`, `billed: false`: amounts.
- `list_expenses` with `matterId`, `billed: false`: amounts.

Run these in parallel batches. If the attorney is responsible for more than about 30
matters, it's cheaper to page each list once firm-wide with `billed: false` and keep only
rows whose `matterId` is in the set.

Report the matters with the most unbilled value first, with the date of the oldest
unbilled item. Old WIP is the most useful signal: work from 60 or more days ago that
hasn't been billed is the most likely to be written down or never collected. Leave out
matters with nothing unbilled.

## Step 5: Invoices waiting on them

`list_invoices` doesn't filter by responsible attorney, so resolve through the matters:
call it with `invoiceState: "Draft"` and again with `invoiceState: "Review"`, page through
all results, and keep invoices whose `matterId` is in the responsible set.

Link each one to its pre-bill in LeanLaw:

```
https://myleanlaw.co/#/billing/{clientId}/{matterId}/review/{draft|review}/{invoiceId}
```

Use `draft` or `review` to match the invoice's state. Show the invoice date, amount, and
how long it has been sitting (today minus `invoiceDate`).

## Step 6: Draft flags

Check the attorney's own **billable** entries for the week, plus the entries on the
invoices from Step 5 (`list_time_entries` with `invoiceId`), since those are about to be
reviewed. The checks are in
[references/draft-checks.md](references/draft-checks.md). Read it before flagging. Flag
entries; don't rewrite them, and don't suggest a narrative that describes work the entry
doesn't mention.

Keep the list short. If more than 10 entries are flagged, show the 10 with the most hours
and give the count of the rest.

## Step 7: Report

Lead with a one-line verdict: **Action needed** if there are invoices waiting on the
attorney or draft flags, otherwise **Info only**. Then list the actions, then the numbers.
Keep the whole thing readable on a phone.

```
Week of Sep 14–20 · Dana Whitfield
Action needed: 3 drafts waiting on you, 2 entries to tidy before billing

WAITING ON YOU
- Riverbend — Series B financing · draft · $8,420.00 · 6 days  → [open pre-bill]
- ...

HOURS                    This week   Prior week
Billable                    31.2        28.5
Fixed fee                    4.0         6.5
Non-billable                 5.5         4.0
Target (37.5/wk)            83%         76%

By matter (billable + fixed fee)
- Riverbend — Series B financing      12.4
- ...

UNBILLED ON YOUR MATTERS               Oldest item
- Harbor Point — Lease dispute   $14,210.00    Jul 2 (83 days)
- ...

TIDY BEFORE BILLING
- Sep 16 · Riverbend · 6.5h · "Work on financing docs": possible block billing; vague
- ...
```

If a section is empty, say so in one line ("Nothing waiting on you") rather than leaving
it out, except the target line, which is left out when there's no target.

## Running it every week

This skill is meant to run as a scheduled task, for example every Monday at 7am. If the
agent supports scheduled tasks, offer to set one up once the first report looks right,
with a prompt like: *"Run the weekly attorney dashboard for dana@firm.com, target 37.5
billable hours a week."* A scheduled run should never ask questions it can answer from
the prompt.

## What this skill can't do

Say so rather than approximating:

- **Other attorneys' dashboards.** It only reports what the connection's user may read.
  A partner rollup across the team, or sending the report to every attorney, needs an
  admin connection and is not part of this skill.
- **Edit or approve invoices.** The connector can't update an invoice; the links open the
  pre-bill in LeanLaw.
- **AR and collections.** Out of scope; outstanding balances live in the firm's
  accounting system or LeanLaw's receivables reports.
- **Targets stored in LeanLaw.** There is no target field on the connector, so the
  target comes from the user or the firm's table.
