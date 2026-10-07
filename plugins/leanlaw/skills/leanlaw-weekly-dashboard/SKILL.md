---
name: leanlaw-weekly-dashboard
description: Build the firm's weekly billable hours report for one timekeeper or a roster of them — hours and value for the week, month to date and year to date, each timekeeper's top five matters, and progress against a monthly billable-hours goal — and deliver it Monday morning by email through LeanLaw's own send_report_email tool, or else through the agent's Gmail, Slack or Teams connector. Use when someone asks for a weekly time report, a weekly billable hours email, a timekeeper summary, "how did my week look", "am I on pace", or when setting this up as a weekly scheduled task. Read-only against LeanLaw data; it never edits time or invoices and never sends without confirmation. Not for AR aging, collections or partner compensation. Requires the LeanLaw MCP connector.
---

# Weekly billable hours report

One run produces the **weekly time report** for one timekeeper or for a roster of them. It
is read-only against LeanLaw data: every LeanLaw tool it calls is a `list_`, `get_` or
`summarize_` call, except `send_report_email`, which delivers the finished report and
changes nothing. It never creates, edits or approves anything.

The report answers one question per timekeeper: **is my time going in, and am I on pace?**
Three periods, each billable-first:

1. **The week just ended** — billable hours, value, non-billable hours, top five matters.
2. **Month to date** — the same, through the same closing day as the week.
3. **Year to date** — each month against the monthly goal, then goal vs actual through the
   last complete month.

## Tools

The data comes from the **LeanLaw MCP connector**, and so, by default, does delivery:
`send_report_email` emails the report to people in the firm (Step 7). Another email, Slack
or Teams connector is the fallback. The names below are logical names; the connector adds
its own prefix, which differs per install, so match on the suffix.

| Need | Tool |
|---|---|
| Who the connection is, which firm, what it may read | `get_me` |
| The roster, resolving a timekeeper, and their custom field values | `list_users` |
| The firm's user custom field definitions | `list_custom_fields` |
| **Every hours and value figure in the report, including the top five matters** | `summarize_time_entries` |
| The week's entry narratives, only for the optional pre-bill flags | `list_time_entries` |
| **Emailing each timekeeper their report**, the default delivery (Step 7) | `send_report_email` |

**Use `summarize_time_entries` for every total.** It takes up to 20 labelled date ranges in
one call and returns, per range, the entry count and hours split into billable,
non-billable and fixed fee, plus `billableAmount` when the connection has the rates scope.
Pass `allUsers: true` and it returns the same breakdown per user, so one call covers the
whole roster for every period in the report. Pass `groupBy` and it breaks each range down
by user and matter server-side, which is where the top five matters come from (Step 4).

Never use `list_time_entries` for any figure in the report, and never add up its rows. It
is slower, costs far more context, and silently truncates at the page limit — a missed
page is a wrong total that looks entirely plausible.

The report is **two calls** for a roster of up to about 80 people: one grouped call for the week and
month to date (Step 4), and one ungrouped call for the monthly rows and the pace range
(Step 3).

## Step 0: Preflight

Call `get_me` once. It returns the connection's `user`, its `firm` and its granted
`authorizations` as `action:resource` scopes.

- Needs `read:time-entries` and `read:users`. If either is missing, name it and stop.
- `read:rates` is what populates `billableAmount`. Without it, build the report on hours
  alone, drop the value columns, and say so at the top rather than showing zeros —
  `billableAmount` comes back as `0`, which is not the same as no revenue.

## Step 1: Setup questions

Ask these **once**, when the report is first set up, then write the answers into the
**scheduled task prompt** — that prompt is the report's entire configuration. There is no
settings file: the recipient list and the goals live in LeanLaw as custom fields and are
resolved on every run, so the only thing to carry forward is which field to read and which
values qualify.

A scheduled run must never ask a question. If its prompt is missing something, it reports
what is missing and stops.

Ask all five together, not one at a time. Read the firm's user custom fields **before**
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

   Put the field **id** and the qualifying values in the scheduled prompt, with the field
   name alongside so the prompt stays readable. Ids survive a rename; names don't. The
   roster then resolves from LeanLaw on every run rather than being frozen at setup.

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

   Put the field id and the period in the scheduled prompt.

2. **One number for everyone.** Ask for it and put it in the scheduled prompt.
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

### 5. How should it reach them?

Before asking, look for a delivery channel as described in Step 7 and **propose the best
one you found**. Usually that is LeanLaw itself: "I can email these from LeanLaw, one per
timekeeper, with replies coming to you." Don't ask them to name a channel from a blank
slate.

If `send_report_email` isn't available, say why it's worth having and how to turn it on
(Step 7), then offer the best fallback you found. If there is no channel at all, the
reports come back to them to send until one is set up.

Put the channel, and the account or workspace it sends from, in the scheduled prompt.

## Step 2: The reporting window

Weeks run **Monday to Sunday**. Default to the **last complete week**. Every period in the
report closes on the same day as that week, so month to date and year to date both run
through the Sunday, not through today. A figure that closes on a different day than the
others invites exactly the arithmetic question the report should answer.

## Step 3: The year-to-date totals, in one call

Build the ranges and make a single `summarize_time_entries` call with `allUsers: true`
(or `userId` for one person), without `groupBy`. These ranges need no matter breakdown,
and grouping twelve months by matter would only add size:

| Label | Range |
|---|---|
| `Jan` … the current month | each calendar month, the current one truncated to the week's end date |
| `Through <last complete month>` | Jan 1 to the end of the last complete month |

That is 10 ranges in September and 13 in December — inside the 20-range cap. If a request
ever needs more than 20, split it across calls rather than dropping ranges.

Read from each range: `billableHours`, `nonBillableHours`, `billableAmount`, and
`entryCount`. With `allUsers: true` each range also has a `users` array with the same
fields per user. Ranges are totalled independently and may overlap, so the current month's
row here and MTD in Step 4 returning the same numbers is correct, not a bug.

The `Through <last complete month>` range is what the goal-vs-actual block compares
against `goal × number of complete months`. Use complete months only — a partial September
against a full monthly goal reads as a shortfall that isn't real.

## Step 4: The week, month to date and top five matters, in one call

Make one more `summarize_time_entries` call for the week and month to date, grouped by
user and then by matter:

```json
{
  "ranges": [
    { "label": "Week ending 2026-09-27", "startDate": "2026-09-21", "endDate": "2026-09-27" },
    { "label": "MTD", "startDate": "2026-09-01", "endDate": "2026-09-27" }
  ],
  "allUsers": true,
  "groupBy": [{ "by": "user" }, { "by": "matter", "top": 5 }],
  "sort": "billableHours"
}
```

For one person, pass `userId` instead of `allUsers`. Leave the month-to-date range out if the
report is week only.

- **`sort: "billableHours"` is required.** The default ranks by total hours, which lets a
  non-billable internal matter take a top-five slot.
- **`top` applies per user**, so every timekeeper gets their own five, however few hours
  they logged.

Each range's `groups` holds one entry per user. Each user entry has the range totals for
that user (`billableHours`, `nonBillableHours`, `billableAmount`), then:

- `groups`: up to five matters, in order. Each has `matter.name`, `client.name`,
  `billableHours` and `billableAmount`.
- `other`: the remaining matters, totalled, with `groupCount` saying how many.

The server guarantees that the five plus `other` add up to that user's totals. Render
straight from these fields; do no arithmetic of your own.

- **List each matter separately.** Two matters for the same client are two rows. Never
  aggregate them into one client line — firms check this, and combining them hides which
  engagement the time went to.
- Show the client name under the matter name so the pairing is unambiguous.
- **Drop any listed matter with zero billable hours.** It can only appear when the person
  had fewer than five billable matters, and it adds nothing to the billable figures.
- Below the matters, add one **All other matters (n)** row from `other`, with n =
  `other.groupCount`. Leave the row out when `other.billableHours` is 0.
- A user who logged no time in a range has no entry in that range's `groups`. Show their
  week as empty rather than dropping them from the report.

If the call fails because it would return too many groups, the roster is too large for one
call: each person costs six groups per range, and a call returns at most 1,000. Make one
call per range instead, week and month to date separately. Do not lower `top`.

## Step 5: Render the report

The layout, the Outlook-safe HTML and the full worked example are in
[references/email-layout.md](references/email-layout.md). **Read it before rendering.**

The two rules that matter most, because getting them wrong is invisible until someone
opens the mail:

- **No SVG, ever.** Outlook on Windows renders through Word and drops SVG silently,
  leaving a blank gap where the chart was. The bar chart is nested HTML tables with
  background colors on table cells.
- **No CSS classes, variables, flex or grid.** Every style is inline on the element.
- **Hex colors, with `bgcolor` beside every background.** Word drops any rule using
  `rgb()`/`rgba()`, and it ignores padding on a `<div>`; use a `<p>` margin or `<td>`
  padding instead.

Check contrast before sending. Grey text under about 4.5:1 against white is hard to read
on screen and worse in print; the layout file gives the values to use.

## Step 6: Preview on screen, and agree the format

**Nothing is sent until the firm has seen the report on screen and approved it.** The
order is fixed: preview and tweak here, then one test send to the person setting it up
(Step 7), then the roster, then the schedule. Don't skip ahead to a send because a
delivery connector is available.

Render a real report for a real timekeeper from their account. A layout agreed in the
abstract is not agreed; a firm only sees what they actually want changed once their own
names and numbers are in it.

### Show it visually

The preview is the rendered email, not its HTML source. Use the best surface the agent
has, in this order:

1. **An artifact, canvas or inline HTML preview** the user can see in the conversation.
2. **An HTML file opened in a browser**, if the agent can write files and open them.
3. **A PDF or screenshot** of the rendered page (`references/email-layout.md` has the PDF
   recipe).

Pasting raw HTML into the chat is not a preview. If none of these surfaces exists, say
so, and make the Step 7 test send to the setup user the preview instead — still before
anyone else receives anything.

Keep the preview at the email's own width, so what they approve is what the mail client
shows. For a Slack or Teams delivery, preview the message as it will be posted, not only
the HTML version.

Pick two people, not one:

1. **Someone with a full week** — the most entries in the reporting week, so every section
   is populated and the format can be judged.
2. **Someone sparse** — few matters, or no goal set. This is where a layout breaks: a
   top-five table with two rows, a missing goal column, a month with no time. Better the
   firm sees that now than in a partner's inbox.

Show the rendered result and walk through what is worth checking, rather than asking a
bare "does this look right?":

- **The greeting and the firm name** as they will appear.
- **The period labels** — that the week, the month and the year all close on the same day.
- **Matter naming** — whether matter-then-client reads correctly for how they name things,
  and that two matters for one client show as two rows.
- **The value columns** — some firms do not want hourly value in front of every
  timekeeper. Dropping them is a reasonable request; ask rather than assume.
- **The goal figures.** This is the one most likely to be wrong, and the sanity check is
  arithmetic: a monthly goal should be in the range a person could actually bill. A
  "monthly" goal reading 12, or 344, means the field holds something else — an annual
  number, a weekly one, or test data. Raise it rather than rendering it.
- **Non-billable placement**, and that it is clearly not counted toward the goal.

Take adjustments, re-render, and show the updated preview in the same place, so each
round replaces the last rather than stacking up copies. Repeat until they say it's right.
Only then move to the test send in Step 7.

If the firm wants a version to circulate before committing — to a managing partner, say —
render it to PDF as well; `references/email-layout.md` has the recipe and the flags that
matter.

## Step 7: Deliver it

**Finding a delivery channel is part of the job.** The report exists to land in someone's
inbox, so a run that ends with "I can't send email" has stopped short.

### Use `send_report_email` first

The LeanLaw connector has its own email tool, `send_report_email`, built for exactly this
report. Prefer it over every other channel:

- **The HTML arrives as rendered.** The layout in `references/email-layout.md` survives it
  intact, so what the firm approved in Step 6 is what lands.
- **It addresses people by LeanLaw user.** Pass the `userId` you already have from
  `list_users`; there's no address to look up or mistype.
- **Replies go to the person running it.** The email comes from LeanLaw on their behalf,
  with their name on it and a footer saying it was sent from Claude.

Call it once per timekeeper:

| Argument | Value |
|---|---|
| `to` | `[{ "userId": "<the timekeeper's userId>" }]` |
| `subject` | e.g. `Your week: 32.5 billable hours (week of 13 Oct)`, on one line |
| `htmlBody` | the Step 5 HTML, as the body, not an attachment |
| `reportName` | `weekly-billable-hours` |

It can only reach **users of the firm**, and a firm can email at most **200 recipients a
day**, with no more than **10 emails to one person a day**. A roster report fits easily,
but a run that is retried over and over won't. If it returns a limit error, stop and report
it rather than switching channels.

**If `send_report_email` isn't in the tool list**, the connection hasn't been allowed to send
email. Two things fix it:

1. **A firm admin turns it on.** In LeanLaw, go to Settings → Agent Access, open the agent's
   Permissions, and tick **Sending email**. That also allows seeing the firm's people,
   which the tool needs.
2. **The person running the report reconnects** the LeanLaw connector and approves
   "Email reports to people in your firm".

It also refuses to send over an API-key connection; it has to be a signed-in LeanLaw user.
Say this plainly, and recommend doing it before the first scheduled run. Then use a
fallback for now.

### Fallbacks

Only when `send_report_email` isn't available, look through **every** other tool the agent
has for one that can deliver to a person. Match on what the tool does, not on its prefix:

| Channel | A tool that… | Typical names |
|---|---|---|
| Email (Gmail, Outlook / Microsoft 365, other mail) | sends, or drafts, a message with recipients, a subject and a body | `send_message`, `send_email`, `send_mail`, `create_draft` |
| Slack | posts a message to a person or a channel | `send_message`, `post_message`, `schedule_message` |
| Microsoft Teams | posts a chat or channel message | `send_chat_message`, `post_message` |

Prefer them in this order:

1. **Gmail, one message per timekeeper.** It sends the HTML body as written, so the report
   arrives as designed.
2. **A Slack or Teams direct message** to each timekeeper.
3. **The Microsoft 365 / Outlook connector, reluctantly.** It strips the report's styling,
   so the layout arrives as unformatted text. Tell the firm that before using it, and
   recommend turning on `send_report_email` instead. If they still want it, send a
   plain-text version or attach the PDF (`references/email-layout.md` has the recipe)
   rather than an HTML body that will be mangled.
4. **A shared channel**, only for a single-person report or when the firm explicitly asks.
   Posting a roster's reports in one channel shows every timekeeper's hours to the others.

If the only email tool creates drafts, use it: drafts in the sender's mailbox are a good
first run, and the firm sends them by hand until they trust the output.

If nothing can deliver, recommend turning on `send_report_email`. If the agent can add
connectors (a connector directory or suggestion tool), Gmail is the next best. Until one of
those is in place, hand back the rendered reports and say which one would make the next run
deliver itself.

### Fit the report to the channel

- **`send_report_email` and Gmail:** send the Step 5 HTML as the message body, not as an
  attachment. For another mail tool, check it takes an HTML body (a `contentType`, `isHtml`
  or `htmlBody` parameter, or similar). If it only takes plain text, send a plain-text
  version rather than raw HTML tags.
- **Slack or Teams:** neither renders an HTML email. Send a short message in the
  channel's own formatting: billable hours and value for the week and month to date, the
  year-to-date pace against goal, and the top five matters as a list. If the tool can
  attach a file, attach the PDF version (`references/email-layout.md` has the recipe).
- **Match every recipient.** For Slack or Teams, look each timekeeper up by the email on
  their LeanLaw user with that connector's user-search tool. List anyone who can't be
  matched instead of skipping them.

### Test send, then the roster

Only after the preview in Step 6 is approved:

1. **Test send to the person setting it up, and nobody else.** Confirm the channel, their
   address and the subject first. Ask them to open it in the mail client the firm
   actually uses — desktop Outlook especially — and compare it with the approved
   preview. Clients drop styling a browser preview keeps; this is where that shows up.
2. **If it differs, fix it, re-preview on screen, and test send again.** Don't go to the
   roster with a known difference.
3. **Then the roster.** Show the channel, the full recipient list, the subject and one
   rendered body, and get an explicit yes before sending.

### Unattended runs

- Never send to a roster without confirmation. A scheduled run sends unattended only if
  the prompt says the firm approved that, and the prompt only says so after they have
  seen a real send. Otherwise the run renders the reports and hands them back for review.
- If a scheduled run can't reach the connector its prompt names, it renders the reports,
  hands them back, and says the connector was unavailable. It does not switch to a
  different channel on its own.

## Running it every week

Only offer this once the firm has approved the on-screen preview in Step 6 and seen a
test send in Step 7. Scheduling a format nobody has seen produces a Monday morning of
corrections.

The scheduled prompt carries the whole configuration, because there is no settings file
for it to read. Write it out in full when setting the schedule up:

> Run the weekly billable hours report for last week.
> Recipients: users whose "Employment Status" field (`618fd324-…`) is Employee or Partner.
> Goal: the "Monthly Hourly Target" field (`a0891cf1-…`), which holds a monthly number.
> Include week, month to date and year to date.
> Delivery: one email per timekeeper through LeanLaw's send_report_email, sent as the setup user.
> Render each report and hand them back for review; do not send.

Field ids belong in the prompt alongside the names — a renamed field breaks a
name-matched prompt silently, and the run would either pick the wrong field or report an
empty roster.

If the prompt is missing the selection rule, the goal source, the sections or the delivery channel, the run
reports what is missing and stops rather than guessing.

## What this skill can't do

Say so rather than approximating:

- **Send to anyone outside the firm.** `send_report_email` reaches LeanLaw users of the
  firm only. Reaching anyone else needs the firm's own email connector (Step 7).
- **Create or edit a custom field.** It reads them. Adding a "Receives Weekly Time Report"
  field, or filling in a target for someone who has none, is done in LeanLaw.
- **Report on people the connection can't see.** A roster-wide run needs a connection with
  firm-wide read access. A single attorney's connection can only report on that attorney.
- **AR, collections or realization.** Out of scope. Billed and collected figures live in
  LeanLaw's receivables reports and the firm's accounting system.
- **Compensation or origination splits.** A different question and a different skill.

## Optional: flag entries before billing

Some firms want the weekly mail to double as a pre-bill nudge. If asked, fetch the week's
billable entries with `list_time_entries` (`userId`, `billingType: "Billable"`, the week's
dates, paging with `offset` until every entry is read), check them against
[references/draft-checks.md](references/draft-checks.md), and add a short **Tidy before billing** section listing at most ten flagged entries. Flag them;
never rewrite a narrative, and never describe work an entry doesn't mention. This is off
by default — it changes the mail from an encouraging summary into a task list, which not
every firm wants going to its partners.
