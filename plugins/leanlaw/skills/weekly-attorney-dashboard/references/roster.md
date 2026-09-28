# Roster and monthly goals

Who gets the weekly billable hours report, and what each person's monthly goal is. The
skill fills this in from the setup questions and reads it on every later run, including
scheduled ones. Edit your installed copy.

Keep it here rather than in the scheduled prompt: the prompt is hard to audit, and a
roster is the thing most worth being able to look at before a Monday send.

## How recipients are chosen

Set one of these. The skill asks once and records the answer.

```
selection: everyone | roles | named
roles: Principal, Attorney          # only when selection = roles
source_field:                       # the LeanLaw custom field this list came from, if any
last_reviewed: YYYY-MM-DD
```

`selection: everyone` means every user who logged time in the reporting week.

**On `source_field`.** The LeanLaw connector does not expose user custom fields — a
`list_users` call returns only `userId`, `name`, `firstName`, `lastName`, `initials`,
`role` and `email`, and `select` does not widen that. So a field like "Receives Weekly
Time Report" can stay the firm's source of truth in LeanLaw, but the skill can't read it.
Name it here, list the people it currently resolves to in the table below, and update the
table when the field changes. If the connector later exposes user fields, this note is
what tells the skill it can read the field directly instead.

## People

One row per recipient. `goal_hours` is billable hours per month.

| email | goal_hours | include | notes |
|---|---|---|---|

Leave `goal_hours` blank for someone who has no goal; they get the report without the
goal column, the goal-vs-actual block and the bar scaling, rather than someone else's
number. Set `include` to `no` to keep a row for the record while dropping the person from
the send.

Example rows, not used:

```
| dana@example.com    | 143 | yes | partner |
| j.ruiz@example.com  | 120 | yes | associate, reduced schedule |
| office@example.com  |     | no  | back office, no billable goal |
```

## Default goal

Used when the firm sets one number for everyone rather than per-person goals. A row in the
table always wins over this.

```
default_goal_hours:
```

Leave it empty to run without goals entirely.

## Delivery

```
send_day: Monday
send_time: 07:00
time_zone: America/New_York
sections: week, mtd, ytd          # any of: week, mtd, ytd
approved_for_unattended_send: no
```

`approved_for_unattended_send` stays `no` until the firm has seen a real send and said
the roster is right. While it is `no`, a scheduled run renders the report and hands it
back for review instead of mailing it.
