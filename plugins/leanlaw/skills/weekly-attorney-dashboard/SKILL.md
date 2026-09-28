---
name: weekly-attorney-dashboard
description: Build the firm's weekly billable hours report for one timekeeper or a roster of them — hours and value for the week, month to date and year to date, each timekeeper's top five matters, and progress against a monthly billable-hours goal — and render it as an email body the firm can send Monday morning. Use when someone asks for a weekly time report, a weekly billable hours email, a timekeeper summary, "how did my week look", "am I on pace", or when setting this up as a weekly scheduled task. Read-only against LeanLaw; it never edits time or invoices and never sends mail without confirmation. Not for AR aging, collections or partner compensation. Requires the LeanLaw MCP connector.
---

# Weekly billable hours report

One run produces the **weekly time report** for one timekeeper or for a roster of them. It
is read-only: every LeanLaw tool it calls is a `list_`, `get_` or `summarize_` call, and it
never creates, edits or approves anything.

The report answers one question per timekeeper: **is my time going in, and am I on pace?**
Three periods, each billable-first:

1. **The week just ended** — billable hours, value, non-billable hours, top five matters.
2. **Month to date** — the same, through the same closing day as the week.
3. **Year to date** — each month against the monthly goal, then goal vs actual through the
   last complete month.

## Tools

All tools come from the **LeanLaw MCP connector**. The names below are logical names; the
connector adds its own prefix, which differs per install, so match on the suffix.

| Need | Tool |
|---|---|
| Who the connection is, which firm, what it may read | `get_me` |
| The roster, resolving a timekeeper, and their custom field values | `list_users` |
| The firm's user custom field definitions | `list_custom_fields` |
| **Every hours and value figure in the report** | `summarize_time_entries` |
| Top five matters for a period | `list_time_entries` |

**Use `summarize_time_entries` for every total.** It takes up to 20 labelled date ranges in
one call and returns, per range, the entry count and hours split into billable,
non-billable and fixed fee, plus `billableAmount` when the connection has the rates scope.
Pass `allUsers: true` and it returns the same breakdown per user, so one call covers the
whole roster for every period in the report. Do not add up `list_time_entries` rows to
reach a total — that is slower, costs far more context, and silently truncates at the page
limit.

`summarize_time_entries` has no matter dimension, so it cannot produce the top five
matters. That is the one thing `list_time_entries` is for here (Step 4).

## Step 0: Preflight

Call `get_me` once. It returns the connection's `user`, its `firm` and its granted
`authorizations` as `action:resource` scopes.

- Needs `read:time-entries` and `read:users`. If either is missing, name it and stop.
- `read:rates` is what populates `billableAmount`. Without it, build the report on hours
  alone, drop the value columns, and say so at the top rather than showing zeros —
  `billableAmount` comes back as `0`, which is not the same as no revenue.

## Step 1: Setup questions

Ask these **once**, when the report is first set up. Put the answers in
[references/roster.md](references/roster.md) so later runs and scheduled runs don't ask
again. A scheduled run must never ask a question — if something is missing, it reports
what is missing and stops.

Ask all four together, not one at a time. Read the firm's user custom fields **before**
asking, so questions 1 and 2 offer the firm's real field names and values instead of
asking them to recall what they set up.

### Reading user custom fields

Two ways, depending on what the connector offers:

- `list_custom_fields`, limited to the **user** entity, returns the definitions: each
  field's id, name, value type, and for an enum its options.
- `list_users` with `select` **including `customFields`** returns each user's values. The
  `select` clause is all-or-nothing — one invented field name in the list and the whole
  clause is ignored, and you get the default columns back with no error. That silent
  failure reads exactly like "this firm has no custom fields", so if `customFields` comes
  back missing, re-check the `select` string before concluding anything.

A user's `customFields` entry looks like:

```json
{ "id": "a0891cf1-…", "name": "Monthly Hourly Target", "valueType": "Number", "value": 344 }
{ "id": "618fd324-…", "name": "Employment Status", "valueType": "Enum",
  "value": "Employee", "optionId": "6570f6d9-…" }
```

Match on the field `id`, not the name — names get edited. Users with the field unset come
back with it absent from the array, or with the array empty.

### 1. Who should get it?

1. **Everyone who logged time.** Pull the roster from `summarize_time_entries` with
   `allUsers: true` for the week — it returns only users with entries, which is usually
   what "everyone" means. Offer to exclude users with no billable hours at all.
2. **Certain roles.** `list_users` filters by `role`: `Principal`, `Attorney`,
   `Paralegal`, `Timekeeper`, `Operator`, `Accountant`. Ask which roles, and note that
   this is the reliable way to leave out back office and accounting.
3. **A user custom field**, such as "Receives Weekly Time Report" or "Employment Status".

   Show the firm their own **enum** fields with the options each one has, since an enum is
   what a recipient rule is normally built on, and propose the one that fits:

   ```
   Employment Status    Employee · Contractor · Partner
   ```

   Guess, then confirm — don't make them choose from a bare list:
   - A field whose name mentions report, weekly, digest or email, with yes/no-shaped
     options, is almost certainly the intended switch. Propose it and the affirmative
     option.
   - Otherwise propose the field that separates people who bill from people who don't, and
     the options to include. For the example above, that is Employment Status with
     Employee and Partner, leaving out Contractor.

   Then filter users on that field's `id` and the chosen `optionId`. A number or text
   field works too — match on `value`.

   Record the field id, the field name and the qualifying values in
   `references/roster.md`, so a later run resolves the roster from LeanLaw rather than a
   frozen list, and a reader of the file can see the rule.

   **People with the field unset are excluded**, and the skill says how many were dropped
   for that reason. Silently omitting someone whose field was never filled in is how a
   partner stops getting their report and nobody notices.

Whichever option is chosen, **show the resolved list of names and email addresses and get
a yes before the first send.** A roster is the thing most likely to be wrong, and a wrong
roster means someone's hours go to the wrong partner.

### 2. How is the monthly goal determined?

1. **A user custom field**, such as "Monthly Hourly Target". This is the option to
   recommend: goals differ by seniority, and a field keeps them in LeanLaw where the firm
   already maintains them instead of in a file that drifts.

   Offer the firm's **number** fields whose names suggest a target — target, goal, hours,
   billable — and propose the closest match. Then read each user's `value`.

   **Confirm what period the number is.** A field named "Monthly Hourly Target" says
   monthly, but firms store annual targets in similarly-named fields. Ask, and convert:
   annual ÷ 12 for a monthly goal. Getting this wrong scales every bar and every
   percentage in the report by twelve, and it looks plausible either way.

   Record the field id and the period in `references/roster.md`.

2. **One number for everyone.** Ask for it and write it to the roster file as the default.
3. **Leave it out.** Then drop the goal column, the goal-vs-actual block and the bar
   scaling, and show hours per month on their own. Do not invent a default goal.

Whichever the source, timekeepers whose goal is missing or zero get the report **without**
the goal column, the goal-vs-actual block and the bar scaling, rather than a borrowed
number or a division by zero. Say how many were affected.

### 3. When should they get it?

Ask for the day and time, and the time zone. Monday morning is the common answer, and the
report then covers the week that ended the day before.

If the agent supports scheduled tasks, offer to create it once the first report looks
right. If it doesn't, say so and give the firm the prompt to schedule elsewhere.

### 4. What should the report include?

1. Week only
2. Week and month to date
3. Week, month to date and year to date
4. All of the above (the default, and what the layout is designed around)

Year to date carries the chart and the goal-vs-actual block, so dropping it also drops
those.

## Step 2: The reporting window

Weeks run **Monday to Sunday**. Default to the **last complete week**. Every period in the
report closes on the same day as that week, so month to date and year to date both run
through the Sunday, not through today. A figure that closes on a different day than the
others invites exactly the arithmetic question the report should answer.

## Step 3: Every total, in one call

Build the ranges and make a single `summarize_time_entries` call with `allUsers: true`
(or `userId` for one person):

| Label | Range |
|---|---|
| `Week ending <date>` | Monday to Sunday of the reporting week |
| `MTD` | 1st of the month to the week's end date |
| `Jan` … the current month | each calendar month, the current one truncated to the week's end date |
| `Through <last complete month>` | Jan 1 to the end of the last complete month |

That is 12 ranges in September and 15 in December — inside the 20-range cap. If a request
ever needs more than 20, split it across calls rather than dropping ranges.

Read from each range: `billableHours`, `nonBillableHours`, `billableAmount`, and
`entryCount`. Ranges are totalled independently and may overlap, so MTD and the month row
returning the same numbers is correct, not a bug.

The `Through <last complete month>` range is what the goal-vs-actual block compares
against `goal × number of complete months`. Use complete months only — a partial September
against a full monthly goal reads as a shortfall that isn't real.

## Step 4: Top five matters

`summarize_time_entries` has no matter breakdown, so for each timekeeper in the roster
call `list_time_entries` with their `userId`, `billingType: "Billable"`, the period's
dates and `limit: 500`, then group by `matterId` locally and sum `hours` and `amount`.
Do this for the week and, if month to date is included, for the month.

- **List each matter separately.** Two matters for the same client are two rows. Never
  aggregate them into one client line — firms check this, and combining them hides which
  engagement the time went to.
- Show the client name under the matter name so the pairing is unambiguous.
- Below the five, add one **All other matters (n)** row with the remaining hours and
  value, so the rows sum to the period total shown above them.
- If the period has five or fewer matters, list them all and leave out the extra row.

A week of one timekeeper's billable entries is small. If a month exceeds the page limit,
page with `offset` — a partial total is a wrong number, not an approximate one.

## Step 5: Render the report

The layout, the Outlook-safe HTML and the full worked example are in
[references/email-layout.md](references/email-layout.md). **Read it before rendering.**

The two rules that matter most, because getting them wrong is invisible until someone
opens the mail:

- **No SVG, ever.** Outlook on Windows renders through Word and drops SVG silently,
  leaving a blank gap where the chart was. The bar chart is nested HTML tables with
  background colors on table cells.
- **No CSS classes, variables, flex or grid.** Every style is inline on the element.

Check contrast before sending. Grey text under about 4.5:1 against white is hard to read
on screen and worse in print; the layout file gives the values to use.

## Step 6: Deliver it

**The LeanLaw connector cannot send email.** It reads billing data; that is all.

- If the agent has an email tool, offer to send. **Show the recipient list, the subject and
  the rendered body, and get an explicit yes first** — and on the first run, send only to
  the person setting it up, so they see what their partners will see.
- If it doesn't, output the HTML body and say it needs to go through the firm's own mail
  system.

Never send to a roster without confirmation, and never on a scheduled run unless the firm
explicitly approved that roster for unattended sending.

## Running it every week

The scheduled prompt should name the report and the roster file, and nothing else:
*"Run the weekly billable hours report for the roster in references/roster.md."*

A scheduled run answers every question from the roster file. If the file is missing a
goal, a recipient or an email address, the run reports the gap and stops rather than
guessing or asking.

## What this skill can't do

Say so rather than approximating:

- **Send mail on its own.** See Step 6.
- **Create or edit a custom field.** It reads them. Adding a "Receives Weekly Time Report"
  field, or filling in a target for someone who has none, is done in LeanLaw.
- **Report on people the connection can't see.** A roster-wide run needs a connection with
  firm-wide read access. A single attorney's connection can only report on that attorney.
- **AR, collections or realization.** Out of scope. Billed and collected figures live in
  LeanLaw's receivables reports and the firm's accounting system.
- **Compensation or origination splits.** A different question and a different skill.

## Optional: flag entries before billing

Some firms want the weekly mail to double as a pre-bill nudge. If asked, check the week's
billable entries against [references/draft-checks.md](references/draft-checks.md) and add
a short **Tidy before billing** section listing at most ten flagged entries. Flag them;
never rewrite a narrative, and never describe work an entry doesn't mention. This is off
by default — it changes the mail from an encouraging summary into a task list, which not
every firm wants going to its partners.
