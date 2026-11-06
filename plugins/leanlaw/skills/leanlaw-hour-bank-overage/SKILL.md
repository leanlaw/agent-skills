---
name: leanlaw-hour-bank-overage
description: Bill a subscription client's hours beyond the month's included hours at each attorney's own rate, with the lowest-rate time counted against the included hours first, before the monthly invoice is drafted in LeanLaw. Use whenever a firm bills a monthly fee that includes a set number of hours, or the user says "run month-end overage", "bill the overage", "hour bank", "included hours", "subscription overage", "who went over this month", "break out the overage before I bill", or is about to draft monthly invoices for subscription clients, even if they don't say "overage". Requires the LeanLaw connector.
compatibility: LeanLaw MCP connector (read and update time entries, read fixed fees). Best with the operator role.
---

# Bill monthly subscription overage at each attorney's rate

## The job

**Higher-Level Job:** Produce, review and approve invoices.

**Core Functional Job:** Bill a subscription client for the hours worked beyond the month's included hours, at the rate of the attorney who worked them, with the lowest-rate time counted against the included hours first, before that month's invoice goes out.

| | |
|---|---|
| **Who has it** | The billing manager at a firm that sells monthly subscriptions with a set number of included hours. |
| **Trigger** | The monthly billing date, once the period's time is in and before invoices are drafted. |
| **Done when** | Every included hour is covered by the monthly fee, every hour beyond it is on the invoice at the rate of the attorney who worked it, and the firm can trace each overage dollar to a time entry. |
| **Today** | Worked out by hand, then fixed in the draft editor one invoice at a time. |
| **What it costs** | Overage billed at one flat rate, or not billed at all, is time the firm worked and never collected. Hand rework delays subscription invoices. |

**Example.** A client pays a flat monthly fee for 35 included hours, and 44 hours were worked. The 35 cheapest hours are covered by the fee. The 9 most expensive hours bill as hourly time at each attorney's rate.

### Related jobs (handled elsewhere)

- **Assign time to the right fixed fee.** Time that isn't linked to the fee doesn't count toward the included hours. This skill reports unlinked time but doesn't link it without asking.
- **Set each attorney's billing rate for a matter.** The order and the overage amounts depend on these rates. This skill never changes a rate.

## Ground rules

This job changes what clients are billed, so:

1. **Show the preview and get an explicit yes before any write.** Approval covers that preview only. If anything changes, show it again.
2. **Only touch unbilled fixed-fee time linked to the fee being processed.** Never edit billed entries, other billing types, expenses, or the fee itself.
3. **When an entry becomes billable, detach it from the fee with the all-zeros ID.** Use `fixedFeeId: "00000000-0000-0000-0000-000000000000"` in the same call that sets `billingType: "billable"`.
   - Passing `null`, or leaving `fixedFeeId` out, **does not** detach it. LeanLaw treats null as "no change", and changing only the billing type leaves the link in place.
   - An entry that is billable but still linked is billed hourly **and** still counts against LeanLaw's own hour limit on the fee, so the client is double-charged.
4. **Never change an entry's description, date, attorney or matter.** Descriptions print on the invoice.
5. **When unsure, stop and ask.** Never guess included hours, rates or which fee a period belongs to.

## Job steps

### 1. Define: which clients, which period, how many included hours

- Call `get_me`. If the role isn't `operator` or `accountant`, `list_time_entries` returns only the user's own time.
  - First find who worked on the matter: `summarize_time_entries({ matterId, allUsers: true, groupBy: [{ by: "user" }], ranges: [<wide range>] })`.
  - Then list each of those attorneys' time with `userId`. This avoids querying every user at the firm.
- Confirm which subscription clients are being billed this cycle.
- **Included hours:**
  - **Preferred:** the fee's `limitHours`.
    - The field is **omitted**, not null, when no limit is set. So is `limitRate`.
  - **Otherwise:** the matter custom field the firm uses (for example "Included hours"), or ask the user.
  - Never infer it from the fee amount.

### 2. Locate: find the fee, its time and the rates

- **The fee:** `list_fixed_fees(filter: { clientId or matterId, billed: false })`. If more than one unbilled fee fits the period, ask which one.
- **The fee's time:** `list_time_entries(filter: { fixedFeeId, limit: 500 })`, without a `billed` filter.
  - If any linked entry is already billed, stop and ask. Some included hours were used on an earlier invoice, which fixes part of the order.
  - The unbilled `fixedFee` entries are the working set.
- **Unlinked time:** `list_time_entries(filter: { matterId, billingType: "fixedFee", billed: false })`, keeping the entries with no `fixedFeeId`. Report them; they don't count yet.
- **Rates:**
  - **Preferred:** each fixed-fee entry's `rate`, as returned by `list_time_entries`.
  - **Otherwise:** a rate table for the client from the user (for example "Jordan 250, Sam 275, Riley 350").
  - The rate decides the **order** only. The amount actually billed comes from the rate LeanLaw already stores on each entry.

### 3. Prepare: order the time and find the line

Sort the working set like this, so the same inputs always give the same answer:

1. rate, lowest first
2. date, earliest first
3. created time, earliest first
4. entry ID

Walk the sorted list with a running total of hours:

- **Included:** the total is still at or under the included hours.
- **Overage:** the included hours were already used up before this entry.
- **Split:** this entry crosses the line.
  - Included portion = included hours − hours before this entry.
  - Overage portion = the rest.

**Estimated overage amount** = overage hours × rate, rounded to cents.

If total hours are at or under the included hours, there's nothing to bill. Say so and move to the next client.

### 4. Confirm: preview and approval

```
Northwind Bakery — Monthly Fee (11/15) — 35.0 included hours
Worked: 44.0 h · Included: 35.0 h · Overage: 9.0 h ≈ $2,737.50

Overage by attorney
  Riley Chen        $350   4.0 h   $1,400.00
  Sam Patel         $275   3.5 h     $962.50
  Jordan Lee        $250   1.5 h     $375.00   (includes 1 split entry)

Changes
  Convert to billable + detach:  7 entries
  Split:                         1 entry (Jordan Lee 11/12, 2.0 h → 0.5 included / 1.5 overage)

Not counted (fixed-fee time not linked to any fee): 3 entries, 4.2 h
```

Ask: "Apply these changes for [clients]?" Wait for a clear yes.

### 5. Execute

**If the period's time is already on a draft invoice:** list with `select: "invoice"`. A billed entry whose `invoice.invoiceState` is `draft` is on a draft. The connector can't edit invoiced entries, drafts included. Make no changes. Instead, give the user the entries to break out of the fixed fee in the draft editor, highest rate first, plus any split, and say no changes were made.

**Otherwise, make the writes one at a time.**

**Overage entry:**
```
update_time_entry(id, { billingType: "billable",
                        fixedFeeId: "00000000-0000-0000-0000-000000000000" })
```

**Split entry.** This keeps the original stored rate on the overage portion, so no rate is typed in:
1. Convert and detach the original entry, as above.
2. Set its hours to the overage portion: `update_time_entry(id, { hours: <overage> })`.
3. Create the included portion back on the fee: `create_time_entry({ matterId, userId, date, time, description, hours: <included>, billingType: "fixedFee", fixedFeeId })`.
   - Copy the original's attorney, date, **time** and description.
   - Without `time`, the new entry is stamped with the current time.

If the firm requires hours in fixed increments and the split doesn't land on one, stop **before writing** and ask whether to round the included portion up or down.

### 6. Monitor: check each write as it lands

After each call, confirm all three:

- `billingType` is `billable`;
- the response has **no `fixedFeeId`**;
- `rate` matches the rate the order was based on.

If a link is still present, or the rate differs, stop. Report what changed and what didn't.

When all writes are done, run the same `summarize_time_entries` call on the matter and confirm all three:

- `fixedFeeHours` equals the included hours (or less, if there was no overage for that part);
- `billableHours` and `billableAmount` went up by exactly the overage hours and dollars in the preview;
- `totalHours` is unchanged, which shows no time was lost or duplicated by a split.

### 7. Modify: when something goes wrong

- **A call failed partway through:** report exactly what changed. Re-running is safe, because converted entries leave the working set.
- **A rate came back different from the table:** the order may be wrong. Show the corrected order and ask before continuing.
- **The user wants to undo,** before anything is invoiced:
  - set each converted entry back to `billingType: "fixedFee"` with its original `fixedFeeId`;
  - recombine any split entry (restore the original's hours and delete the included-portion entry you created).

### 8. Conclude: report and hand off

Report:

- clients processed, and overage hours and dollars **at the stored rates**;
- every changed entry: attorney, date, hours, and before → after;
- anything skipped, and why;
- unlinked fixed-fee time still waiting to be assigned.

Next step for the user: create the drafts as usual. The monthly fee bills on its own line, and the overage appears as hourly lines at each attorney's rate.

## Known limits (temporary)

- **Fall back to the user when the connector doesn't provide them.** If the connector doesn't return `limitHours` or entry `rate`, or the fee has no limit set, the user supplies the included hours and the rates. The order is then only as good as their rate table; the billed amount isn't affected.
- **The connector can't set or change a fee's hour limit** (`create_fixed_fee` / `update_fixed_fee` have no limit fields). Set it in LeanLaw.
- **Run this job before drafting.** If a fee has an hour limit in LeanLaw and the invoice is drafted while its linked hours exceed that limit, LeanLaw bills the overage as one line at the fee's limit rate, not at each attorney's rate.
- **Invoice line items can't be read through the connector.** Check the draft in LeanLaw after creating it.
