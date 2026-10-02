---
name: leanlaw-onboard-client
description: Take a new client from a signed engagement letter to a live, billable matter in LeanLaw — read the letter (a PDF or a document in a connected system), extract the client, contacts, responsible and originating attorney, practice area and fee arrangement, run a conflict pre-check against existing clients and matters, check whether the client already exists, then create the client, the matter and the fee structure, including a fixed-fee installment schedule. Shows a full read-back and writes nothing until the user confirms. Use when someone says "onboard this client", "new client intake", "here's the signed engagement letter", "open a matter for X", "run conflicts on this new client", or "set up a fixed fee schedule". Requires the LeanLaw MCP connector.
---

# Onboard a client from an engagement letter

One run takes **one engagement letter** to **one client and one matter** in LeanLaw:

1. Read the letter and extract the terms.
2. Conflict pre-check, before anything is created.
3. Find or plan the client, resolve the attorneys and practice area.
4. Read back everything that will be created, and wait for a yes.
5. Create the client, the matter and the fixed fees.
6. Verify and link to the matter.

The point is that the matter can be billed from the day the letter is signed, and that the
invoice reflects the terms the client agreed to. A matter opened weeks late, or under the wrong
billing type, is time that sits unbilled and fees that get written down when the client
compares the invoice with their letter.

Steps 1 to 4 only read. Nothing is written until the user has confirmed the read-back in Step 4.

## Tools

All LeanLaw tools come from the **LeanLaw MCP connector**. The names below are logical names; the
connector adds its own prefix, which differs per install, so match on the suffix.

| Need | Tool |
|---|---|
| Who the connection is, which firm, what it may do | `get_me` |
| The attorney roster | `list_users` |
| Practice areas | `list_practice_areas` |
| Conflict pre-check and duplicate check | `list_clients`, `get_client`, `list_matters`, `get_matter` |
| Create the client | `create_client` |
| Create the matter | `create_matter` |
| Fixed-fee installments | `create_fixed_fee`, `list_fixed_fees` |
| LEDES activity and task codes, only if the client requires them | `get_codes` |
| All three writes in one call, if the connector offers it | `leanlaw_flow_onboard_client` |

The engagement letter comes from whatever the agent can read: a file the user attached, or a
document in a connected system such as Google Drive, OneDrive or SharePoint, Box, or a document
management system. Match on what a tool does (searches or reads files), not on its name.

If the LeanLaw connector isn't available, say so and stop.

## Guardrails

- **Nothing is created before the read-back is confirmed.** `create_client`, `create_matter` and
  `create_fixed_fee` write to the firm's live billing data. A fixed fee is a charge the client
  will be invoiced for.
- **The conflict pre-check is not conflict clearance.** It shows what LeanLaw's names turn up.
  Whether there is a conflict, and whether it can be waived, is the firm's decision. Never
  describe a result as "cleared" or "no conflict".
- **The letter is the source of truth, and it is quoted, not paraphrased.** Every extracted term
  in the read-back cites where in the letter it came from. A term the letter doesn't state is
  asked, not inferred.
- **Never invent** a reference number, a matter name, a rate, an amount, a date or an attorney.
- **Stop on ambiguity.** Two plausible existing clients, or two attorneys with the same surname,
  is a question for the user.
- Collect only what billing needs. If the letter contains an SSN, a bank or card number, or a
  government ID, leave it out of every record and every summary.

## Step 0: Preflight

Run these together, once, before reading the letter:

- `get_me` — the user and firm the connection acts as, and its granted scopes. The run needs to
  read users, clients and matters, and to create clients, matters and fixed fees. If a create
  scope is missing, name it and say the run can still do the extraction and the conflict
  pre-check, but a firm admin must grant the scope (Settings → Agent Access in LeanLaw) and the
  user must reconnect before anything can be created.
- `list_users` — the roster, for resolving the responsible and originating attorney.
- `list_practice_areas` — for the matter.

Keep the results for the rest of the run.

## Step 1: Read the engagement letter

The default input is the signed engagement letter. If the user named a file, read it. If they
named a client but no file, search the connected document systems for the letter (the client's
name plus "engagement", "retainer" or "fee agreement") and confirm the file with the user before
reading it. If there's no letter at all, run the same intake from questions instead, asking for
everything in one message.

What to extract, how to map fee language to a LeanLaw billing type, and the checks to run on a
letter before trusting it are in
[references/engagement-letter.md](references/engagement-letter.md). **Read it before
extracting.**

Show the extraction as a table of field, value and the letter's own words for it, and list what
the letter didn't say. Ask for the missing required fields — matter name, responsible attorney —
in one message.

## Step 2: Conflict pre-check

**Before the client exists in LeanLaw**, search the firm's existing clients and matters for every
party in the letter: the client, its principals and affiliates, adverse and opposing parties, and
any related parties the letter names. Ask the user for parties the letter doesn't name — in
litigation the opposing party is often only in the complaint.

The searches, how to classify a hit, and the report format are in
[references/conflict-precheck.md](references/conflict-precheck.md). **Read it before
searching.**

Present the report and ask the user to confirm that the firm's conflict process has seen it and
that onboarding should go ahead. If they say no, or want to wait for the firm's conflict review,
stop there: nothing has been created, and the report is the output.

If the connector offers a conflict-check tool, use it in place of the name search and present its
results in the same report format.

## Step 3: Plan the client and the matter

### Does the client already exist?

The Step 2 searches already answer this. Classify:

- **Clear match** — an existing client with this name or reference. Show it (name, reference,
  open matters) and confirm that the new matter goes under it. This is common: an existing client
  with a new engagement. No `create_client`.
- **Possible matches** — list them with what distinguishes them, and ask. Never merge, and never
  pick one silently.
- **No match** — a new client.

For a new client, plan the `create_client` call: `name` as it should read on an invoice,
`contact` with the individual's name, company, email, phone and address from the letter,
`reference` only if the firm gave one, and `notes` if useful.

### Attorneys

The **responsible attorney** (`responsibleId`, required) and the **originating attorney**
(`originatorId`) drive compensation and origination reporting, so both are confirmed by name.

- Resolve each against the Step 0 roster. Require one match, and show their full name and email
  in the read-back.
- A shared surname is a question. A name with no match may be someone who isn't a LeanLaw user
  yet; a matter can't be created without a valid responsible attorney.
- Don't default the originator to the responsible attorney. If the letter doesn't name an
  originator, ask, and if the user leaves it unset, say so in the read-back.
- If origination is split between several attorneys, set the primary one here and say the split
  is configured on the matter in LeanLaw.

### The matter

- **Name** — from the letter's description of the engagement, confirmed with the user. Follow the
  firm's naming pattern if their existing matters show one.
- **Billing type** — from the fee terms, using the mapping in the engagement-letter reference.
- **Practice area** — resolved to `practiceAreaId` from the Step 0 list. If nothing matches, offer
  the closest existing ones or leave it unset.
- **Opened** — the date the letter was signed, or today if it isn't dated.
- **Conflict information** — the parties from Step 2 go into `conflictInformation`: `adverse`,
  `opposing`, `relatedAdverse` and `relatedClient`. This is what the firm's next conflict search
  on this matter will rely on.
- **Billing instructions** — a short summary of the agreed fee terms, quoted from the letter:
  rates, caps, the contingency percentage, retainer and replenishment terms, invoice cadence.
  This keeps the agreed terms in front of whoever prepares the invoice. Show it in the read-back.

### The fee structure

- **Hourly** — no fee records. Rates come from the firm's rate groups and user rates, which the
  connector can't set. If the letter's rates differ from what the firm normally charges, list
  them as a to-do in the read-back.
- **Fixed fee** — one `create_fixed_fee` per installment. The split and dating rules are in
  [references/fixed-fee-schedules.md](references/fixed-fee-schedules.md). **Read it before
  computing a schedule.** The installments must add up to the total exactly.
- **Contingency** — no fee records. The percentage goes in the billing instructions.
- **Trust or retainer requirement** — the connector can't take a deposit or set a trust
  requirement. Put the amount and terms in the billing instructions and list it as a to-do.

## Step 4: Read-back

Show one read-back with everything that will be created, then wait for an explicit yes. A yes
covers exactly what was shown; any change means a new read-back.

```
Engagement letter   Riverbend Holdings LLC – engagement letter, signed 2026-09-28

Conflict pre-check  3 parties searched · 1 hit (see report) · firm review confirmed by user

Client       Riverbend Holdings LLC                      NEW
             Contact: Jordan Ames, CFO · jordan@riverbend.example · (555) 010-2234
Matter       Riverbend – Series B financing              NEW
             Type FixedFee · Practice area Corporate · Opened 2026-09-28
             Responsible  Dana Whitfield <dana@…>
             Originator   Marcus Ellery  <marcus@…>
             Conflict info  Adverse: none · Related client: Riverbend Capital LP
             Billing instructions  "Flat fee of $12,000, payable in four monthly
                                   installments beginning on signing." (§3)
Fixed fees   4 installments, total $12,000.00
             1  2026-09-28  $3,000.00  Fixed fee 1 of 4 – Series B financing
             2  2026-10-28  $3,000.00  Fixed fee 2 of 4 – Series B financing
             3  2026-11-28  $3,000.00  Fixed fee 3 of 4 – Series B financing
             4  2026-12-28  $3,000.00  Fixed fee 4 of 4 – Series B financing
             Credited to Dana Whitfield

Not done here (do in LeanLaw)
             Trust deposit of $5,000 required before work starts (§4)
```

Below it, list anything the letter left open that was filled from the user's answers, so it's
clear which terms came from the letter and which didn't.

## Step 5: Create

If the connector offers `leanlaw_flow_onboard_client`, use it to create the client, matter and
fee structure in one call, with exactly the confirmed values. It reports what was created, and
reports a partial failure rather than leaving an orphaned record.

Otherwise, in order, stopping at the first failure:

1. `create_client`, unless the client exists. If it fails, report the error verbatim and stop.
   Don't retry with an altered name; that's how duplicate clients are made.
2. `create_matter`. If the firm uses QuickBooks Online, this also creates the matter, and the
   client if it isn't linked yet, in QuickBooks. A QuickBooks failure does not fail the matter, so
   if the response mentions one, the matter exists in LeanLaw and the QuickBooks side needs a look.
3. `create_fixed_fee` for each installment, in date order.

**If anything fails partway**, stop and report exactly what was created, with ids, and what
wasn't. Never re-run the whole sequence: the client exists, and repeating the fixed fees bills the
client twice. Let the user decide how to finish.

## Step 6: Verify and report

Read back what was created rather than trusting the write responses: `get_matter` on the new
matter, and `list_fixed_fees` filtered by its `matterId` for a fixed-fee matter. Then a short
summary: client (new or existing), matter, billing type, attorneys, the fee schedule as created,
the conflict pre-check outcome, and the to-dos for LeanLaw (rates, trust deposit, split
origination, a QuickBooks check).

End with a link to the matter, so the user can check the real record:

```
https://myleanlaw.co/#/matters-next/client/<clientId>/matter/<matterId>/info
```

Use both ids exactly as the connector returned them, client first. The app routes on the `#/…`
fragment, so never trim it. Show it as a markdown link titled with the matter name. If a link
can't be built, offer `https://myleanlaw.co` instead of guessing at another route.

## What this skill can't do

Say so rather than approximating:

- **Clear conflicts, or record a conflict check in LeanLaw.** The pre-check is a name search
  across clients and matters. It doesn't search contacts, the conflict information on other
  matters, or anything outside LeanLaw, and it doesn't write a conflict record.
- **Set rates or rate groups.** Hourly rates are configured in LeanLaw.
- **Take a trust or retainer deposit**, or set a trust requirement.
- **Split origination** across several attorneys, or set per-matter compensation.
- **Change a fixed fee after it has been invoiced.**
