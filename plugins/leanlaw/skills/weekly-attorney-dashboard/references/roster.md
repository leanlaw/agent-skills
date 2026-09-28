# Roster and monthly goals

Who gets the weekly billable hours report, and what each person's monthly goal is. The
skill fills this in from the setup questions and reads it on every later run, including
scheduled ones. Edit your installed copy.

Keep it here rather than in the scheduled prompt: the prompt is hard to audit, and a
roster is the thing most worth being able to look at before a Monday send.

## How recipients are chosen

Set one of these. The skill asks once and records the answer.

```
selection: everyone | roles | custom_field | named
roles: Principal, Attorney               # only when selection = roles
field_id:                                # only when selection = custom_field
field_name:                              # for humans reading this file
field_values:                            # the values that qualify someone
last_reviewed: YYYY-MM-DD
```

`selection: everyone` means every user who logged time in the reporting week.

**On `custom_field`.** This is resolved live from LeanLaw on every run, so the firm
maintains the list where it already works rather than here. Record the field's `id` as
`field_id` — names get edited, ids don't — and keep `field_name` alongside it so this file
stays readable.

```
selection: custom_field
field_id: 618fd324-6ce2-43b4-97be-1f1e4e81b567
field_name: Employment Status
field_values: Employee, Partner
```

Anyone whose field is unset is excluded, and the run reports how many were dropped that
way. With `selection: named`, the table below is the roster.

## Monthly goal

Where each timekeeper's monthly billable-hours goal comes from.

```
goal_source: custom_field | default | none
goal_field_id:                      # only when goal_source = custom_field
goal_field_name:                    # for humans reading this file
goal_field_period: monthly | annual # what the stored number means
default_goal_hours:                 # only when goal_source = default
```

**Prefer `custom_field`.** Goals differ by seniority and change as people move, and a
field keeps them in LeanLaw where the firm already maintains them.

```
goal_source: custom_field
goal_field_id: a0891cf1-30f5-46b8-907d-19820040a06e
goal_field_name: Monthly Hourly Target
goal_field_period: monthly
```

`goal_field_period` is not decoration. A field named for a month can hold an annual
number, and reading one as the other scales every bar and percentage in the report by
twelve while still looking plausible. An annual figure is divided by 12.

Anyone whose goal is missing or zero gets the report without the goal column, the
goal-vs-actual block and the bar scaling, rather than someone else's number.

## People

Only needed for `selection: named`, or to override what LeanLaw says for one person.
`goal_hours` here wins over the custom field and the default.

| email | goal_hours | include | notes |
|---|---|---|---|

Set `include` to `no` to keep a row for the record while dropping the person from the
send — it is worth leaving the row and the reason rather than deleting it.

Example rows, not used:

```
| dana@example.com    | 143 | yes | partner |
| j.ruiz@example.com  | 120 | yes | associate, reduced schedule |
| office@example.com  |     | no  | back office, no billable goal |
```

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
