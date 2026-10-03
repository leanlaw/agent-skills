---
name: leanlaw-missing-time-review
description: Find work an attorney did but hasn't logged yet — by comparing their calendar, sent email, and Slack or Teams messages with the time entries already in LeanLaw — work out the client and matter for each gap, and draft the missing time entries for them to review, then create the approved ones in LeanLaw. Use when someone asks "what time am I missing", "did I log everything today" or "yesterday", "catch up my time this week", "rebuild my day", "find unbilled work", or wants this set up as a morning or end-of-day task. Picks today, yesterday, this week or last week from when it's run, and matches each meeting or email to a client and matter by looking up the attendees' and correspondents' addresses in the firm's client contacts. Reports only the gaps, never re-lists time already logged, and never creates an entry without the attorney's confirmation. Not for editing existing entries or pre-bill review. Requires the LeanLaw MCP connector and at least one calendar or email connector.
---

# Missing time review

One run answers one question for one attorney: **what did I work on that isn't in LeanLaw
yet?** It reads the attorney's own activity for a period, takes away everything already
covered by a time entry, and proposes entries for what's left. The attorney reviews the
proposals, and only the ones they approve are created.

Time reconstructed days later comes in short, or not at all. Most entries are logged within
hours; the ones that matter here are the minority logged a week late, or never. That's
where this skill helps: it catches yesterday's call before it's forgotten, which raises
utilization and leaves less for the pre-bill to write down.

It reviews today, yesterday, this week or last week, depending on when it's run (Step 1),
or any range the attorney names.

## Tools

The names below are logical names. Each connector adds its own prefix, which differs per
install, so match on the suffix.

**LeanLaw MCP connector:**

| Need | Tool |
|---|---|
| Who the attorney is, their firm, and what the connection may do | `get_me` |
| Time already logged in the period, and recent history for matching | `list_time_entries` |
| Finding the matter for a piece of work | `list_matters`, `get_matter` |
| Client contact emails, for matching correspondents to clients | `list_clients`, `get_client` |
| Valid LEDES codes, only if the firm's matters require them | `get_codes` |
| **Creating the approved entries** | `create_time_entry` |
| Optional: emailing the morning list to the attorney | `send_report_email` |

**Activity sources**, from whichever connectors the agent has. Look through every tool it
has and match on what the tool does, not its name:

| Source | A tool that… | Typical names |
|---|---|---|
| Calendar (Google, Outlook / Microsoft 365) | lists or searches events in a date range | `list_events`, `search_events`, `outlook_calendar_search` |
| Sent email (Gmail, Outlook / Microsoft 365) | searches messages, including sent mail | `search_threads`, `outlook_email_search` |
| Slack | searches messages the user sent | `slack_search_public_and_private`, `search_messages` |
| Microsoft Teams | searches or lists chat and channel messages | `chat_message_search`, `teams_list_chats` |

Calendar and sent email are the core. Slack and Teams are used only if connected. With
**none** of these connectors, stop: say that the skill needs at least a calendar or an
email connector, and name the ones that would work.

## Step 0: Preflight

Call `get_me` once. It returns the connection's `user` (the attorney whose time this is),
the `firm`, and the granted `authorizations` as `action:resource` scopes.

- Needs `read:time-entries`, `read:matters` and `read:clients`. If any is missing, name it
  and stop.
- `create:time-entries` is what lets the skill create the approved entries. Without it, run
  the review anyway and hand back the proposals for the attorney to enter in
  [LeanLaw](https://myleanlaw.co). Say which permission would let it create them directly.
- Every entry is created for the `get_me` user. This skill never logs time for anyone else,
  even when the connection could.

Then list which activity sources are connected, and say which are missing. A run without
email will miss the work done in email, and the attorney should know that before trusting
"no gaps found".

## Step 1: The period

If the attorney named a period ("yesterday", "last week", "since the 1st"), use it. Otherwise
pick a recommended period from when the skill is run, in the attorney's local time:

| Run at | Recommend | Why |
|---|---|---|
| Monday before noon | **Last week**, Monday to Sunday | The week is fresh, weekend work is included, and this week has barely started |
| Any other day before 10:00 | **Yesterday**, the previous working day | The morning catch-up, before today's work buries it |
| From 16:00 | **Today**, up to now | The end-of-day check, while the day is still in their head |
| Any other time | **This week**, Monday up to now | A mid-day or mid-week catch-up |

Then offer it as a choice rather than starting on it. Recommend that period first, then the
nearest whole-day and whole-week alternatives, then a custom range, in one question:

```
Which period should I check?
  1. Yesterday, Thu 2 Oct (recommended)
  2. Today so far, Fri 3 Oct
  3. This week so far, Mon 29 Sep – Fri 3 Oct
  4. A different range — tell me the dates
```

- Keep to days and weeks. A custom range is the last option, for when the attorney already
  knows the dates.
- "Yesterday" means the previous working day. If that day had no calendar events and no sent
  mail, say so and offer the day before.
- **Today** and **this week** stop at the time the skill is run. Events still to come aren't
  work yet; leave them out, and don't propose time for a meeting that hasn't ended.
- Above about two weeks, warn that the review gets long, and offer to go a week at a time.
- A **scheduled run** doesn't ask. It uses the period its prompt names (see "Running it on a
  schedule").
- Use the attorney's own time zone, both for choosing the recommendation and for day
  boundaries. Take it from their calendar settings if the connector exposes them, otherwise
  ask once. An 11pm call belongs to the day it happened where they are, not to the next day in
  UTC.

## Step 2: What's already logged

Read the attorney's time entries for the period with `list_time_entries`: `userId` from
`get_me`, `startDate` and `endDate`, and page with `limit` and `offset` until every entry
is read. This is the one place the skill needs individual entries rather than totals,
because it matches on narratives. A missed page means real time gets proposed twice.

Then read a **history** of the attorney's entries for the 60 days before the period, the
same way. The history is what Step 4 learns from: which matters this attorney actually
works on, and how they phrase their narratives. Keep `date`, `matter`, `client`, `hours`,
`billingType` and `description`. Nothing else is needed.

If a calendar is connected, read its events for the same 60 days too, keeping only events
with someone outside the firm: date, title and attendee addresses. Step 4 lines these up with
the history to place people who aren't client contacts.

## Step 3: What they actually did

Collect the attorney's activity in the period from each connected source, then turn it into
**work blocks**: one block per piece of work, with a date, a start time where there is one,
an estimated duration, the people involved, and a short note of what it was.

The rules for reading each source, estimating time and leaving things out are in
[references/activity-sources.md](references/activity-sources.md). **Read it before
building blocks.** The rules that matter most:

- **Only what the attorney did.** Meetings they attended, email they sent, messages they
  wrote. Mail they received and didn't answer is not work. Neither is a meeting they
  declined.
- **Leave out personal events entirely.** Doctor, school pickup, gym, lunch, "Hold", and
  anything marked private. Don't propose them, and don't list them as left out by name;
  say "3 personal events skipped" at most.
- **Read subjects, recipients and the first lines of a message, not whole threads.** That's
  enough to say what the work was. Quote nothing from a client email into a narrative.
- **Merge before you estimate.** Six emails in one thread on one afternoon are one block,
  not six. A call and the follow-up email on the same matter are two blocks on one
  matter.

## Step 4: Which client and matter

Work out the client and matter for every block yourself. Don't ask the attorney to name a
matter first; ask only about the blocks you couldn't place.

### The contact index

The people on a block are the most reliable key to the client: the attendees and organizer
of a calendar event or invite, and the To and Cc of a sent email. Titles and subjects are
free text; an email address is not.

If `list_clients` takes an `email` parameter, look each external address up with it directly,
and each domain with `email: "@domain.com"`. Collect the run's distinct addresses first, so
each is looked up once.

Otherwise its `query` matches only name and reference, so build an index once per run:

1. Page through `list_clients` with `select: "contact"` and `limit: 500`, until every client
   is read. The `select` value must be exactly that; an unsupported field makes it ignored
   and returns clients without contacts. If no client comes back with a `contact`, check the
   `select` before concluding the firm has no contact emails.
2. From each client's `contact`, take `email` and every address in `emailCC` (it can hold
   several, separated by commas or semicolons). Lower-case them.
3. Map **address → client**, and **domain → clients**, leaving out public domains (gmail.com,
   outlook.com, hotmail.com, yahoo.com, icloud.com, and the like) and the firm's own domain.

Look up each block's external addresses in the index. Ignore the attorney's own address and
other addresses at the firm's domain. On a calendar invite **from** a client, the organizer
counts as an attendee.

### Matching

Match in this order, and stop at the first that gives a single matter:

1. **A matter name or reference** in the meeting title, invite description or email subject.
   Search for it with `list_matters` (`query`), leaving out archived matters.
2. **An attendee's or correspondent's address** exactly matches a client contact. That gives
   the client. Then list its open matters (`list_matters` with `clientId`, `archived: false`):
   - **One open matter:** use it.
   - **Several:** don't guess. Ask the attorney which one, listing the client's open matters
     with the ones they logged time to recently first. A matter named in the title or
     subject would already have been placed by 1.
3. **The same people, logged before.** Earlier calendar events in the 60-day history window
   with the same external attendees, on days when the attorney logged an entry whose
   narrative names those people or that meeting. Use that entry's matter. This is how
   opposing counsel, co-counsel and experts, who are never client contacts, get placed.
4. **The attorney's own history.** The same correspondent's name, meeting title or thread
   subject in an entry narrative in the last 60 days. Use that entry's matter.
5. **An address's domain** matches a client contact's domain. That points at the client, not
   a person, so it's weaker than an exact address. Pick the matter as in 2.
6. **The client name** in the title, subject or text, searched with `list_matters` (`query`).

When a block has several external people who point at different clients, don't pick one;
show the candidates.

If an exact address match placed a block the attorney then corrects, or a person matched
only by domain or history, mention once at the end that adding that address to the client's
contact in LeanLaw would let future runs match it directly. The skill doesn't change client
records itself.

Give every block a confidence:

- **High:** an exact matter reference; an exact address match to a client with one open
  matter; or a history match on the same people.
- **Medium:** one client from a domain match, with only one open matter.
- **Low:** several candidate matters, or a guess from a name. Show the top two or three
  candidates and let the attorney pick.
- **None:** no match. Ask, or leave it out.

**Internal and admin work** — firm meetings, training, business development, recruiting,
admin — goes on the firm's internal matter, the one whose `matterType` is `internal`, and
is non-billable. If the firm has several internal matters, use the one the attorney's
history uses for that kind of work. If it has none, list the block as unplaced; don't put
internal work on a client matter.

Never invent a matter. If no matter fits, the block is unplaced, and creating a matter is
outside this skill.

## Step 5: Only the gaps

Take away every block that's already covered. A block is **covered** when an entry on the
same date, on the same matter:

- describes the same work: the meeting, the correspondent, or the document named in the
  block, **or**
- has hours at least equal to the block's estimate, and no other block on that matter that
  day competes for it.

When in doubt, treat the block as covered. Proposing time that's already logged is worse
than missing a 0.1: a duplicate gets billed, and the attorney stops trusting the list.

If an entry on that matter and day exists but looks **short** — 0.3 logged against a
1.5-hour meeting — don't propose a second entry. Mention it as a note under the proposals
("Acme v. Ruiz, 14 Oct: 0.3 logged, calendar shows 1.5h") and let the attorney decide.
The skill never edits an existing entry.

## Step 6: Draft the entries

One proposed entry per remaining block. Each one has:

| Field | Value |
|---|---|
| Date | the block's date |
| Matter | from Step 4, with the client name shown next to it |
| Hours | the block's estimate, rounded **up** to the firm's increment |
| Billing | the matter's default, or non-billable for internal and admin work |
| Narrative | as below |

**Increments.** Use the increment the attorney's history shows: if every entry is a
multiple of 0.1, the increment is 0.1. If unclear, use 0.1. Never round a block to zero.

**Narratives.** Write in the attorney's own style, judged from their history: the same
tense, the same level of detail, the same way of naming people. In general:

- A verb, the object, and the purpose: "Call with J. Ruiz re: settlement terms", "Draft
  and send email to opposing counsel re: deposition schedule."
- Name the person, document or issue the activity names. Nothing more.
- **Never describe work the activity doesn't show.** A meeting titled "Ruiz call" is a call
  with Ruiz. It isn't "call with Ruiz to discuss strategy and review exhibits" unless
  something says so. The attorney knows what they did; the skill only knows what it saw.
- No email addresses, no quotes from a client's message, nothing privileged beyond what a
  narrative on an invoice would normally carry.

**LEDES codes.** Add `activityCode` and `taskCode` only if the matter's history shows them
in use. Take the codes from that history or from `get_codes`; don't guess codes.

Don't set a rate. The matter and the attorney's rate apply.

## Step 7: Review with the attorney

Show the proposals as one table, oldest first:

```
Missing time — Mon 13 Oct (logged so far: 5.2h)

 #  Matter (client)                    Hours  Billing      Narrative                                  Source       Confidence
 1  Ruiz v. Acme (Acme Holdings)        1.0   Billable     Call with J. Ruiz re: settlement terms     Calendar     High
 2  Estate of Park (Park, M.)           0.3   Billable     Emails with M. Park re: inventory filing   Sent email   Medium
 3  ? Harbor Lease / Harbor Renewal     0.5   —            Review of lease amendment draft            Sent email   Low
 4  Firm Admin (internal)               0.5   Non-bill.    Associate training session                 Calendar     High

Not proposed
 - Ruiz v. Acme, 13 Oct: 0.3 logged, calendar shows 1.5h. Check if this is short.
 - 2 events with no match: "Coffee w/ Dan", "Catch-up"
 - 3 personal events skipped
```

Then let the attorney:

- **Approve** all, or a list of numbers.
- **Change** any field: hours, matter, narrative, billing.
- **Pick** the matter for a low-confidence row, or **drop** any row.

Re-show the table after changes. Nothing is created until the attorney says to create a
specific set of entries. "Looks good" on a table that still has an unplaced row means
create the placed ones and ask about the rest.

If nothing is missing, say so in one line, with what was checked: "Nothing missing for Mon
13 Oct. 6 meetings and 14 sent emails, all covered by your 7.4h logged." A clean result
is the outcome the attorney wants most days.

## Step 8: Create the approved entries

Immediately before creating, read the period's entries again with `list_time_entries`.
If the attorney logged something themselves in the meantime that now covers a proposal,
drop it and say so. This is what keeps a rerun, or a second run on the same morning,
from creating duplicates.

Then call `create_time_entry` once per approved entry:

| Argument | Value |
|---|---|
| `matterId` | the matter's id |
| `date` | the entry's date |
| `hours` | the approved hours |
| `description` | the approved narrative |
| `billingType` | `nonBillable` for internal and admin work; otherwise omit it so the matter's default applies |
| `activityCode`, `taskCode` | only when Step 6 set them |

Don't pass `userId` or `rate`. The entry is for the `get_me` user, at their rate.

Report what was created: one line per entry with its matter, hours and the total, and any
that failed with the reason. Created entries are ordinary unbilled time in LeanLaw: the
attorney can edit or delete any of them there until they're billed.

## Running it on a schedule

Offer to schedule it once the attorney has been through one review and is happy with how
it places matters. A schedule set up before that produces a morning list nobody trusts.

**A scheduled run never creates entries.** It prepares the review and waits. Every entry is
confirmed by the attorney, every time.

The scheduled prompt carries the whole setup. Write it out when creating the task:

> Run the missing time review. Period: my previous working day, or last week when it runs on
> a Monday.
> Time zone: America/Denver.
> Sources: Google Calendar, Gmail sent mail, Slack.
> Prepare the proposals and stop for my review; do not create any entries.
> Also email me the list through LeanLaw's send_report_email.

If the agent supports scheduled tasks, create it with this prompt, at a time before the
attorney's day starts. An end-of-day schedule works too: set the period to today, and run it
late in the afternoon. If it doesn't, say so and give them the prompt to schedule
elsewhere.

**Emailing the list** is optional, and useful when the scheduled run happens somewhere the
attorney won't see it. Send it to the attorney only, with `send_report_email`: `to` is
`[{ "userId": "<the get_me user's id>" }]`, `reportName` is `missing-time-review`, and the
subject says how much was found, such as `2.3h of missing time for Mon 13 Oct`. The
body is the review table as a simple HTML table. It's a reminder, not an approval: the
email says to open the agent to approve, and the entries are created only from there.

If a scheduled run can't reach a source its prompt names, it runs with the rest and says
at the top which source was missing, since "nothing missing" without email isn't the same
result.

## What this skill can't do

Say so rather than approximating:

- **Log time for someone else.** It reviews the connection's own user. An assistant
  logging for an attorney does it in LeanLaw.
- **Edit, merge or delete existing entries.** It flags short ones; the attorney changes
  them in LeanLaw.
- **Create a client or matter.** Work with no matter is listed as unplaced.
- **See work that left no trace** in a connected source: a hallway conversation, an
  unscheduled phone call, drafting with no email after it. Say this when the attorney
  asks why something was missed, and suggest they add it by hand.
- **Phone call logs, document edits and billing-platform activity** are not read, even if a
  connector for them exists.
