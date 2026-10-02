# Conflict pre-check

A search of the firm's existing LeanLaw clients and matters for every party to a new engagement,
run before anything is created, and presented in the same format every time so the firm's
conflict reviewer knows where to look.

It's a pre-check, not a clearance. It finds names in LeanLaw. It can't tell whether a hit is the
same person or company, whether a conflict exists, or whether one can be waived.

## The parties

Build the list from the letter, then ask the user to add to it:

| Role | Examples |
|---|---|
| Client | The client, and any former or trade names |
| Client affiliates | Parent, subsidiaries, principals, officers, guarantors, a spouse in a family matter |
| Adverse | The party on the other side |
| Opposing | Opposing counsel, if the firm tracks them |
| Related adverse | Co-defendants, insurers, lenders, counterparties to the transaction |

Show the list and ask "anyone else?" once. In litigation the letter often names only the client.

## The searches

For each party, search both clients and matters, **including archived matters**: a former client
can be a conflict.

- `list_clients` with `query` — matches client name or reference.
- `list_matters` with `query` and no `archived` filter — matches matter name, client name, matter
  reference and client reference, so it also finds a party named in a matter title such as
  "Ames v. Riverbend".

`query` is a substring match, so search the **distinctive part** of a name, not the whole string:

- Companies: drop suffixes like LLC, Inc., Corp., LLP, Holdings, Group, and search the core word.
  "Riverbend", not "Riverbend Holdings LLC".
- Individuals: search the surname alone, then the first name too if the surname is common.
- Initials, ampersands and punctuation vary between records. Search "Ames" rather than
  "Ames & Co.", and try both spellings of anything with an obvious variant.
- Two or three narrow searches beat one broad one. If a search returns more than a page, the term
  is too broad; narrow it rather than paging through hundreds of rows.

Record every search term used. It goes in the report, so a reviewer can see what was and wasn't
searched.

## Classifying a hit

| Hit | Why it matters |
|---|---|
| **Adverse party is an existing client** | The most serious: the firm may be asked to act against its own client |
| **Adverse party is named in a matter** | It may be a current or former opponent, or it may be a client under another name |
| **Client is an existing client** | Usually expected — a new engagement for a known client. Note it, don't flag it |
| **Client is named in another client's matter** | The client may have been adverse to the firm's client before |
| **Related party is a client, or named in a matter** | Usually less serious, but the reviewer decides |
| **Name similarity only** | A partial match on a common word. List it lower down; don't drop it |

For each hit that matters, `get_matter` gives the responsible attorney, whether the matter is
archived, and the conflict information recorded on it. Include those in the report.

Don't decide that a hit is or isn't the same entity. Say what matched and let the reviewer decide.

## The report

Use this format every time:

```
CONFLICT PRE-CHECK — Riverbend Holdings LLC / Series B financing
Run 2026-10-01 by Dana Whitfield via the LeanLaw connector

Parties searched
  Client          Riverbend Holdings LLC      terms: "Riverbend"
  Affiliate       Riverbend Capital LP        terms: "Riverbend"
  Principal       Jordan Ames                 terms: "Ames", "Jordan Ames"
  Adverse         Northgate Ventures          terms: "Northgate"

Hits to review (2)
  ! Adverse party is an existing client
    Northgate Ventures Inc.  client ref 2210 · 3 matters, 1 open
    Open: "Northgate – lease review" · responsible Priya Shah
  · Related party is named in a matter (archived)
    "Ames v. Coastal Lending" · client Coastal Lending · archived 2024 · responsible Dana Whitfield

Expected
  Riverbend Capital LP is an existing client (ref 1042, 2 open matters)

No matches
  Jordan Ames (as a client)

Not searched: contacts, conflict information stored on other matters, and anything outside
LeanLaw. This is a name search, not a conflict clearance.
```

- Hits to review come first, the most serious first, each with enough detail to judge without
  opening LeanLaw.
- Every party appears somewhere in the report, including those with no matches, so it's clear
  they were searched.
- The closing line about what wasn't searched always appears.

### Adjusting the format

Firms run conflicts differently, so the format can change. If the user asks for different columns,
an extra party role, a different order or wording, use it for the rest of the run, and suggest
they add it to their agent's instructions for this skill so every run uses it. Keep these whatever
the format:

- every party searched, with the terms used;
- the hits;
- the line saying what wasn't searched.

## After the report

Ask whether the firm's conflict process has seen it and onboarding should go ahead. If the user
says yes, carry the outcome into the read-back as "firm review confirmed by user". If they want to
wait, stop: nothing has been created, and the report is the result of the run.

The parties go into the new matter's `conflictInformation`, so the next conflict search can find
them:

| Field | Parties |
|---|---|
| `adverse` | Adverse parties |
| `opposing` | Opposing parties or counsel |
| `relatedAdverse` | Related adverse parties |
| `relatedClient` | Client affiliates and principals |

Separate several names in one field with semicolons.
