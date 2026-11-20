# Fixed-fee installment schedules

A fixed-fee matter has no amount of its own. The fee is one or more fixed fees on the matter,
each with an amount and a date, created with `create_fixed_fee`. A fixed fee dated in the future
waits, unbilled, until an invoice run covers its date. So an installment schedule is simply one
fixed fee per installment, all created now with their future dates.

## What to ask for

Take these from the letter, and ask for anything it leaves out:

- the total fee;
- the number of installments, or the milestones;
- the first date (often the signing date);
- the cadence: monthly, quarterly, or by milestone;
- whether the split is even or the first installment is larger.

## Splitting the total

Work in whole cents, and make every installment exactly two decimal places.

**Even split.** Divide the total in cents by the number of installments and round down. That
leaves a remainder of a few cents. Add one cent to each of the first installments until the
remainder is used up. The installments then add up to the total exactly.

$10,000.00 over 3 is 1,000,000 cents. 1,000,000 ÷ 3 = 333,333 with 1 cent left over:

| # | Date | Amount |
|---|---|---|
| 1 | 2026-10-01 | $3,333.34 |
| 2 | 2026-11-01 | $3,333.33 |
| 3 | 2026-12-01 | $3,333.33 |
| | **Total** | **$10,000.00** |

**A larger first installment.** Work out the first installment, subtract it from the total, and
split what's left evenly. "$15,000, 40% on signing and the rest over three months" is $6,000.00
first, then $9,000.00 split three ways: $3,000.00 each.

**Amounts the letter states outright.** Use them as written, then check they add up to the
total. If they don't, show the difference and ask; don't absorb it into one installment.

## Dates

- **Monthly or quarterly**: keep the day of the month from the first date. When a month is
  shorter, use its last day, then go back to the original day the next month. A schedule starting
  on the 31st runs Jan 31, Feb 28, Mar 31.
- **Milestones**: ask for a date for each milestone, and name the milestone in the description.
  If a milestone has no date yet, don't invent one: create the dated installments and list the
  undated one as a to-do.
- **A first installment dated in the past** goes on the next invoice. That's often what the firm
  wants for a payment due on signing, but confirm it.
- Write every date as `YYYY-MM-DD`.

## Each fixed fee

| Field | Value |
|---|---|
| `matterId` | The new matter |
| `date` | The installment date |
| `amount` | The installment amount, greater than zero |
| `description` | `Fixed fee 2 of 4 – Series B financing`, since it appears on the invoice as written |
| `userId` | The timekeeper credited with the fee. Default to the responsible attorney and say so; leave it out only if the firm wants the fee credited to the firm |
| `activityCode`, `taskCode` | Only if the client requires LEDES codes (`get_codes`) |

## Checks before the read-back

- The installments add up to the total exactly.
- Every amount is greater than zero and has no more than two decimal places.
- The dates are in order, and the first one is the date intended.
- Each description says which installment it is.
- `userId` is set, or deliberately left out.

## If a call fails

Create the fixed fees in date order and stop at the first failure. Report which installments were
created, with their ids, and which weren't, and let the user decide how to finish. Never start the
schedule again from the top; that bills the client twice for the installments that already exist.

To check what's on a matter, use `list_fixed_fees` with its `matterId`, and `billed: false` for
the ones not yet invoiced.
