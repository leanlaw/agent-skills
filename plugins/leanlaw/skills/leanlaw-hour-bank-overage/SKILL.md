---
name: leanlaw-hour-bank-overage
description: Bill a subscription client's hours beyond the month's included hours at each attorney's own rate, before the monthly invoice is drafted in LeanLaw. Which hours count against the included hours (lowest rate first, or earliest first) is the firm's choice; the skill asks. Use whenever a firm bills a monthly fee that includes a set number of hours, or the user says "run month-end overage", "bill the overage", "hour bank", "included hours", "subscription overage", "who went over this month", "break out the overage before I bill", or is about to draft monthly invoices for subscription clients, even if they don't say "overage". Requires the LeanLaw connector.
compatibility: LeanLaw MCP connector (read and update time entries, read fixed fees) with the rates permission. Best with the operator or accountant role.
---

# Bill monthly subscription overage at each attorney's rate

## The job

**Higher-Level Job:** Produce, review and approve invoices.

**Core Functional Job:** Bill a subscription client for the hours worked beyond the month's included hours, at the rate of the attorney who worked them, counting hours against the included hours in the order the firm's policy sets, before that month's invoice goes out.

| | |
|---|---|
| **Who has it** | The billing manager at a firm that sells monthly subscriptions with a set number of included hours. |
| **Trigger** | The monthly billing date, once the period's time is in and before invoices are drafted. |
| **Done when** | Every included hour is covered by the monthly fee, every hour beyond it is on the invoice at the rate of the attorney who worked it, and the firm can trace each overage dollar to a time entry. |
| **Today** | Worked out by hand, then fixed in the draft editor one invoice at a time. |
| **What it costs** | Overage billed at one flat rate, or not billed at all, is time the firm worked and never collected. Hand rework delays subscription invoices. |

**Example.** Kestrel Aerial Imaging pays a monthly fee for 20 included hours, and 24.5 hours were worked:

| Attorney | Rate | Entries |
|---|---|---|
| Marcus Halloway | $210 | 11/03 5.0 h, 11/10 3.0 h |
| Devin Park | $265 | 11/04 4.0 h, 11/12 3.0 h |
| Adrienne Cole | $320 | 11/05 2.5 h, 11/14 3.5 h |
| Rosa Iglesias | $385 | 11/06 2.0 h, 11/18 1.5 h |

Either way 4.5 hours are overage. Which 4.5 depends on the firm's policy:

- **Lowest rate first:** Marcus's and Devin's 15 hours, then Adrienne's 2.5 h and 2.5 h of her 3.5 h entry, fill the 20 included hours. The overage is Adrienne 1.0 h ($320.00) and all 3.5 h of Rosa's time ($770.00 + $577.50): **$1,667.50**.
- **Earliest first:** the six entries through 11/12 (19.5 h) and 0.5 h of Adrienne's 11/14 entry fill the included hours. The overage is Adrienne 3.0 h ($960.00) and Rosa's 11/18 1.5 h ($577.50): **$1,537.50**.

### Related jobs (handled elsewhere)

- **Assign time to the right fixed fee.** Time that isn't linked to the fee doesn't count toward the included hours. This skill reports unlinked time but doesn't link it without asking.
- **Set each attorney's billing rate for a matter.** The order and the overage amounts depend on these rates. This skill never changes a rate.
- **Set a fee's included hours.** The hour limit is set on the fixed fee in LeanLaw. This skill reads it and never changes it.

## Ground rules

This job changes what clients are billed, so:

1. **Show the preview and get an explicit yes before any write.** Approval covers that preview only. If anything changes, show it again.
2. **Only touch unbilled fixed-fee time linked to the fee being processed.** Never edit billed entries, other billing types, expenses, or the fee itself.
3. **When an entry becomes billable, detach it from the fee with the all-zeros ID.** Use `fixedFeeId: "00000000-0000-0000-0000-000000000000"` in the same call that sets `billingType: "billable"`.
   - Passing `null`, or leaving `fixedFeeId` out, **does not** detach it. LeanLaw treats null as "no change", and changing only the billing type leaves the link in place.
   - An entry that is billable but still linked is billed hourly **and** still counts against the fee's hour limit, so the client is double-charged.
4. **Never change an entry's description, date, attorney, matter or rate.** Descriptions print on the invoice.
5. **When unsure, stop and ask.** Never guess included hours, rates, the firm's overage order, or which fee a period belongs to.

## Reading rates and limits

- **Use each entry's `hourlyRate`** (the attorney's rate on this matter, saved on the entry) and `hourlyValue` (hours × `hourlyRate`). Every entry carries them, fixed-fee time included.
- **Never read `rate` or `amount` on fixed-fee entries.** Those are what the client is charged for billable time, and fixed-fee entries don't have them.
- **Permission:** the rate fields (`rate`, `amount`, `hourlyRate`, `hourlyValue`, and the fee's `limitRate`) are **omitted** when the connection lacks the rates permission, and **`null`** when it has the permission but nothing is stored.
- **The fee's `limitHours`** is the included hours. It is omitted when the fee has no limit.
- **The fee's `limitRate`** is the rate LeanLaw uses for overage on its own, at drafting. Treat it as blank when it is absent, `null` or `0.00`.
- `limitHours` and `limitRate` are read-only through the connector. They are set on the fee in LeanLaw.

## Job steps

### 1. Define: access, clients, period, included hours and the firm's overage order

- **Check access first.** Call `get_me` and look at the granted scopes and the user's role.
  - **No rates permission (`read:rates`): stop here, before anything else.** Explain plainly: without it the skill can't read anyone's hourly rate, so it can't put the time in order or price the overage. Ask the user to reconnect with the rates permission, or to have someone who has it run the job.
  - **Role isn't `operator` or `accountant`:** say plainly that this connection sees only the user's own time across the firm, so the skill will look up who worked on each matter and list their time one person at a time. Then continue. For **every** `list_time_entries` call in this skill, including the checks in step 6:
    1. Find who has time: `summarize_time_entries({ matterId, allUsers: true, groupBy: [{ by: "user" }], ranges: [<the period, widened to cover the fee>] })`.
    2. List each of those people's time with `userId` added to the filter, and combine the results.
- **Confirm which subscription clients** are being billed this cycle.
- **Included hours:** read `limitHours` from the fee (found in step 2). If it's absent, ask the user. Never infer it from the fee amount or the description.
- **Ask how the firm's overage works**, once per run unless the user says it differs by client:
  - **Lowest rate first:** the lowest-rate time counts against the included hours, and the highest-rate time is billed as overage.
  - **Earliest first:** time counts against the included hours in the order it was worked, and the latest time is billed as overage.

  Use the answer for the ordering in step 3. Don't assume either.

### 2. Locate: find the fee, its time and the rates

- **The fee:** `list_fixed_fees(filter: { clientId or matterId, billed: false })`. If more than one unbilled fee fits the period, ask which one.
- **The fee's time:** `list_time_entries(filter: { fixedFeeId, limit: 500 })`, without a `billed` filter.
  - If any linked entry is already billed, stop and ask. Some included hours were used on an earlier invoice, which fixes part of the order.
  - The unbilled `fixedFee` entries are the working set.
- **Unlinked time:** `list_time_entries(filter: { matterId, billingType: "fixedFee", billed: false })`, keeping the entries with no `fixedFeeId`.
  - If the fee has a `limitHours`, stop and ask the user to link this time to the right fee, or make it billable, before you continue. LeanLaw can count unlinked fixed-fee time toward the fee's limit when the invoice is drafted, which adds a second overage line at the fee's limit rate.
  - If the fee has no limit, report it; it doesn't count toward this fee.
- **Rates:** each working-set entry's `hourlyRate`.
  - If any is `null`, stop and ask. No rate is stored for that attorney on this matter, so the order and the overage price can't be worked out.
- **Rate check:** for each attorney in the working set, list their billable time on other matters over a recent window (`list_time_entries(filter: { userId, billingType: "billable", startDate, endDate })`) and note the rate on most of it.
  - If an attorney's `hourlyRate` on this fee is well below that rate (for example, under 70% of it), flag it for the preview. The stored rate may be a cost figure rather than a billing rate, and converting would bill the overage at it.
  - If the attorney has no billable time to compare against, say so in the preview.
- **Before totals:** call `summarize_time_entries({ matterId, allUsers: true, ranges: [<the fee's period>] })` and keep `fixedFeeHours`, `billableHours`, `billableAmount` and `totalHours` for step 6.

### 3. Prepare: order the time and find the line

Sort the working set by the firm's order from step 1, then by the tiebreaks, so the same inputs always give the same answer:

| Lowest rate first | Earliest first |
|---|---|
| 1. `hourlyRate`, lowest first | 1. date, earliest first |
| 2. date, earliest first | 2. created time, earliest first |
| 3. created time, earliest first | 3. entry ID |
| 4. entry ID | |

Walk the sorted list with a running total of hours:

- **Included:** the total is still at or under the included hours.
- **Overage:** the included hours were already used up before this entry.
- **Split:** this entry crosses the line.
  - Included portion = included hours − hours before this entry.
  - Overage portion = the rest.

**Estimated overage amount:** for each overage entry or overage portion, round hours × `hourlyRate` to cents, then add them up.

If total hours are at or under the included hours, there's nothing to bill. Say so and move to the next client.

### 4. Confirm: preview and approval

The example above, with the firm's order set to lowest rate first:

```
Kestrel Aerial Imaging — Monthly Fee (11/30) — 20.0 included hours
Order: lowest rate first
Worked: 24.5 h · Included: 20.0 h · Overage: 4.5 h ≈ $1,667.50

Overage by attorney
  Rosa Iglesias     $385   3.5 h   $1,347.50
  Adrienne Cole     $320   1.0 h     $320.00   (split entry)

Changes
  Convert to billable + detach:  2 entries
  Split:                         1 entry (Adrienne Cole 11/14, 3.5 h → 2.5 included / 1.0 overage)

Rate check: no flags
```

Add a line under **Rate check** for each flagged attorney, with both rates, and ask whether to go ahead for that attorney.

Ask: "Apply these changes for [clients]?" Wait for a clear yes.

### 5. Execute

**If the period's time is already on a draft invoice:** list with `select: "invoice"`. A billed entry whose `invoice.invoiceState` is `draft` is on a draft. The connector can't edit invoiced entries, drafts included. Make no changes. Instead, give the user the entries to break out of the fixed fee in the draft editor, in the firm's overage order, plus any split, and say no changes were made.

**Otherwise, make the writes one at a time.**

**Overage entry:**
```
update_time_entry(id, { billingType: "billable",
                        fixedFeeId: "00000000-0000-0000-0000-000000000000" })
```

**Split entry.** Create the included portion first, then convert the original, so the overage portion keeps the original's saved rate and no rate is typed in:
1. Create the included portion on the fee: `create_time_entry({ matterId, userId, date, time, description, hours: <included>, billingType: "fixedFee", fixedFeeId })`.
   - Copy the original's attorney, date, **time** and description.
   - Without `time`, the new entry is stamped with the current time.
2. Convert the original to the overage portion in one call:
   ```
   update_time_entry(id, { billingType: "billable",
                           fixedFeeId: "00000000-0000-0000-0000-000000000000",
                           hours: <overage> })
   ```

If the firm requires hours in fixed increments and the split doesn't land on one, stop **before writing** and ask whether to round the included portion up or down.

### 6. Monitor: check each write as it lands

After each `update_time_entry`, confirm all three:

- `billingType` is `billable`;
- the response has **no `fixedFeeId`**;
- `rate` equals the `hourlyRate` the order was based on.

For a created included portion, confirm it's `fixedFee` and linked to the fee.

If a link is still present, or the rate differs, stop. Report what changed and what didn't.

When all writes are done, run both checks:

- **Matter totals:** call `summarize_time_entries` with the same arguments as the before totals in step 2.
  - `fixedFeeHours` equals the included hours, plus any other fixed-fee time on the matter;
  - `billableHours` rose by exactly the overage hours in the preview, and `billableAmount` by the overage dollars;
  - `totalHours` is unchanged, which shows no time was lost or duplicated by a split.
- **Per fee:** the hours still linked to the fee (`list_time_entries(filter: { fixedFeeId })`) add up to exactly the included hours. The matter totals can't show this on their own, because they also count other fixed-fee time on the matter.

### 7. Modify: when something goes wrong

- **A call failed partway through:** report exactly what changed.
  - A converted entry leaves the working set, so re-running is safe after a failed conversion.
  - A split that stopped between its two calls is not safe to re-run: the included portion exists, but the original is still whole on the fee. Fix that entry by hand (delete the new entry, or finish the update), then re-run.
- **A rate came back different from the one the order was based on:** the order may be wrong. Show the corrected order and ask before continuing.
- **The user wants to undo,** before anything is invoiced:
  - set each converted entry back to `billingType: "fixedFee"` with its original `fixedFeeId`;
  - recombine any split entry (restore the original's hours and delete the included-portion entry you created).

### 8. Conclude: report and hand off

Report:

- clients processed, the overage order used, and overage hours and dollars **at the stored rates**;
- every changed entry: attorney, date, hours, and before → after;
- anything skipped, and why (including any attorney held back by the rate check);
- unlinked fixed-fee time still waiting to be assigned.

Next step for the user: create the drafts as usual. The monthly fee bills on its own line, and the overage appears as hourly lines at each attorney's rate.

## Known limits

- **The connector can't set or change a fee's hour limit or limit rate.** Set them on the fee in LeanLaw.
- **Run this job before drafting.** If a fee has an hour limit in LeanLaw and the invoice is drafted while its hours exceed that limit, LeanLaw bills the overage as one line at the fee's `limitRate`, not at each attorney's rate. Those hours are the fee's linked time, and can include the matter's unlinked fixed-fee time.
- **Invoice line items can't be read through the connector.** Check the draft in LeanLaw after creating it.
