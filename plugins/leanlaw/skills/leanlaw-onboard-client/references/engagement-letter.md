# Reading an engagement letter

How to turn a signed engagement letter into the fields LeanLaw needs, without adding anything the
letter doesn't say.

## Before extracting

Check these first, and raise any that fail before going further:

- **It's signed.** Look for the client's signature or countersignature block. An unsigned draft
  can still be used for the conflict pre-check, but say that the terms may still change, and ask
  before creating anything from it.
- **It's the final version.** If the search turned up several versions, use the latest signed one
  and say which.
- **It's one engagement.** A letter covering several distinct engagements is several matters. Ask
  whether to onboard them one at a time.
- **It's readable.** A scanned letter with no text layer needs OCR. If the text comes out garbled,
  say which sections couldn't be read rather than guessing at them.

## What to extract

| Field | Where it usually is | LeanLaw field |
|---|---|---|
| Client name, as it should appear on an invoice | Addressee block, "Client" definition, signature block | client `name` |
| Contact person, title, email, phone, address | Addressee block, signature block, notices clause | client `contact` |
| Client type: individual or entity | Signature block ("By: … Its: …" means an entity) | how `contact` is filled |
| Principals, parents, subsidiaries, affiliates | "Client" definition, scope, "who we represent" clause | conflict pre-check; `conflictInformation.relatedClient` |
| Adverse and opposing parties | Scope of engagement, recitals, "the matter" | conflict pre-check; `conflictInformation.adverse`, `opposing` |
| Other named parties (co-defendants, lenders, counterparties) | Scope of engagement | conflict pre-check; `conflictInformation.relatedAdverse` |
| What the engagement is | Scope of engagement | matter `name` (proposed, then confirmed) |
| Responsible attorney | "Responsible attorney", "lead attorney", the signer for the firm | `responsibleId` |
| Originating attorney | Rarely in the letter; ask | `originatorId` |
| Practice area | Implied by the scope | `practiceAreaId` |
| Fee arrangement | Fees, compensation, or billing section | `matterType`, fixed fees, billing instructions |
| Retainer or trust requirement | Retainer, advance fee, or deposit clause | billing instructions, a to-do |
| Invoice cadence and payment terms | Billing section | billing instructions |
| Date signed | Signature block | matter `opened` |

The firm's signer is often the responsible attorney but not always. A managing partner who signs
every letter is not the responsible attorney on every matter. If the letter doesn't name the
responsible attorney outright, propose the signer and ask.

## Mapping the fee terms to a billing type

| The letter says | `matterType` | What gets created |
|---|---|---|
| Hourly rates, "billed for time at", a rate schedule | `Hourly` | No fee records. Rates go in the billing instructions; setting them is a to-do |
| A flat fee, a fixed fee, "$X for the engagement" | `FixedFee` | A fixed fee for the total, or one per installment |
| A flat fee paid in installments, or on milestones | `FixedFee` | One fixed fee per installment, dated |
| A percentage of recovery, a contingent fee | `Contingency` | No fee records. The percentage goes in the billing instructions |
| Pro bono, no fee | `Probono` | Nothing |

Mixed arrangements need a decision, so ask rather than pick:

- **Hourly up to a cap.** `Hourly`, with the cap in the billing instructions so it's checked at
  invoicing.
- **Flat fee plus hourly for work outside the scope.** Usually `FixedFee` for this matter, with
  the hourly rates for out-of-scope work in the billing instructions. Some firms open a second
  hourly matter instead.
- **Hybrid contingency, a reduced hourly rate plus a percentage.** Usually `Hourly`, with the
  contingency terms in the billing instructions.
- **An evergreen retainer billed hourly.** `Hourly`. The retainer is a trust deposit, not a fee,
  so it is never a fixed fee.

**A retainer is not a fixed fee.** A deposit held in trust and drawn down against invoices is the
client's money until it's earned. Creating it as a fixed fee would bill the client for it. Only a
fee the letter says is earned on receipt, or a flat fee for the work, becomes a fixed fee.

## Billing instructions

Write a short summary of the fee terms for the matter's `billingInstructions`, quoting the
letter's own figures and citing the section:

```
Per engagement letter signed 2026-09-28. Hourly: partners $475, associates $310, paralegals
$165 (§3). Fees capped at $25,000 for phase one without written approval (§3.2). $10,000
evergreen retainer in trust, replenished when below $2,500 (§4). Monthly invoices, due in 30
days (§5).
```

Keep it to the terms that affect the invoice. Don't copy whole clauses, and leave out anything
that isn't about billing.

## Showing the extraction

Show a table of field, value and the letter's words, then a list of what's missing:

| Field | Value | From the letter |
|---|---|---|
| Client | Riverbend Holdings LLC | "Riverbend Holdings LLC (the "Company")" (p. 1) |
| Fee | Fixed, $12,000 in 4 monthly installments | "a flat fee of $12,000, payable in four equal monthly installments" (§3) |
| Responsible | Dana Whitfield | signed for the firm (p. 4) — proposed, please confirm |

**Not in the letter:** originating attorney, matter reference number.
