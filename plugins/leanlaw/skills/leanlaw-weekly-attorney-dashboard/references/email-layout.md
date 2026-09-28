# Email layout

How to render the weekly billable hours report so it survives the mail clients law firms
actually use. Read this before rendering.

## The constraints that drive everything here

Attorneys read mail in Outlook, and **Outlook on Windows renders through Word**, not a
browser engine. That single fact rules out most modern HTML:

| Don't | Why | Do instead |
|---|---|---|
| `<svg>` | Dropped silently — a blank gap where the chart was | Nested tables with cell background colors |
| CSS classes, `<style>` blocks | Stripped or ignored | `style="…"` inline on every element |
| CSS variables, `flex`, `grid` | Not supported | Tables for all layout |
| Google Fonts, webfonts | Never load; falls back to something arbitrary | Georgia, Arial, Courier New |
| Background images | Unreliable | Background colors on `<td>` |
| `<div>` for columns | Collapses | `<td>` for columns |

Write the mail body as a table of tables. It looks dated as source and it is the only
thing that renders the same in Outlook, Gmail and Apple Mail.

## Width

Outer table `width="560"` with `style="width:560px;max-width:100%;"`, centered in a
full-width wrapper table. Mobile clients scale a fixed-width table to fit rather than
reflowing it, so proportions hold and text doesn't overflow. Don't try to build a
responsive email; there is no reliable way without media queries Outlook ignores.

## Palette

Small grey text is the most common legibility complaint, in print especially. Nothing
below about 4.5:1 against white.

| Role | Hex | Contrast on white |
|---|---|---|
| Body text, headings | `#151c26` | 15.8:1 |
| Secondary text, dates, "all other matters" rows | `#4a5666` | 8.0:1 |
| Column labels, client names under matter names | `#5b6674` | 6.3:1 |
| Empty future months, footnotes | `#6b7685` | 5.0:1 |
| Bar, under goal | `#3a6ab0` | — |
| Bar, goal met | `#2e7d5b` | — |
| Bar, partial month | `#9db4d6` | — |
| Bar track | `#e8ecf1` | — |
| Panel fill | `#f5f7fa` | — |
| Rules | `#d9e0e8` light, `#c3ccd7` firm | — |
| Percent of goal | `#b4741a` | 4.6:1 |

**Never go below `#6b7685` for text.** `#7b8697` is about 4.0:1 and `#9aa4b2` about
3.3:1; both read fine on a designer's monitor and badly on a printed page.

## Type

- Headings: `Georgia,'Times New Roman',serif`
- Body and labels: `Arial,Helvetica,sans-serif`
- All figures: `'Courier New',Courier,monospace` — digits line up in columns without
  needing `tabular-nums`, which Word ignores

Put `white-space:nowrap` on every figure cell. Three summary cells across 560px leaves
about 158px each, and a value like `$40,142.50` in Courier is close enough to the edge
that Outlook will wrap it.

## Structure

```
Masthead        firm name · WEEKLY TIME SUMMARY
                Good morning, <first name>.
                Here is your time for the week of <range>.

Week ending     3 cells: billable hours | value | non-billable hours
                Top five matters table + "All other matters (n)" + Total

Month to date   same 3 cells, same table shape

Year to date    one bar row per month, Jan–Dec
                Year to date total line

Through <last complete month>
                3 cells: goal | actual | % of goal

Note            closing note
```

Non-billable belongs in the summary cells, quieter than the two billable figures
(`#4a5666` rather than `#151c26`), and never in the chart or the year-to-date table. It is
context, not performance.

## The bar chart

One table row per month. The full track is the monthly goal, so a bar that fills the track
means the goal was met — the goal reads without needing an axis or a separate line.

```html
<tr>
  <td width="34" style="width:34px;font-family:Arial,Helvetica,sans-serif;font-size:12px;color:#4a5666;padding:3px 0;">Mar</td>
  <td style="padding:3px 8px;">
    <table role="presentation" cellpadding="0" cellspacing="0" border="0" width="100%" style="border-collapse:collapse;background-color:#e8ecf1;"><tr>
      <td width="100%" style="width:100%;background-color:#2e7d5b;font-size:0;line-height:0;height:13px;">&nbsp;</td>
    </tr></table>
  </td>
  <td width="62" align="right" style="width:62px;font-family:'Courier New',Courier,monospace;font-size:12px;color:#151c26;padding:3px 0;white-space:nowrap;">151.2</td>
</tr>
```

Rules:

- Width is `round(hours / goal × 100)%`, **capped at 100**. A month over goal fills the
  track and turns `#2e7d5b`; the hours figure on the right carries the real magnitude.
- A month under goal is `#3a6ab0` and needs a second, unstyled `<td>` after the bar cell
  to hold the remaining track.
- The current, partial month is `#9db4d6` so nobody reads a short bar as a bad month.
- Months with no time yet: no bar cell, track `#f0f3f6`, an em dash in the figure column.
- `font-size:0;line-height:0;height:13px;` with a `&nbsp;` is what keeps an empty cell
  from collapsing in Outlook. The `&nbsp;` is load-bearing.
- With no goal configured, scale bars to the largest month instead and say so in the
  caption, or drop the chart.

Caption it: *Each full bar is the 143-hour monthly goal. Green means the goal was met.*

## Top five matters

Matter name on the first line, client name beneath it at `#5b6674`, 12px. Two matters for
the same client are **two rows** — never combined. Add a line under the heading saying so;
firms ask, and showing it is better than describing it.

The rows plus **All other matters (n)** must sum to the Total row, and the Total must
equal the summary cell above. Someone will add them up.

## Closing note

One short paragraph. Say what non-billable means here and when the figures were taken:

> **Note:** Non-billable hours are shown for reference only and are not included in
> billable hour totals. Figures reflect time entered as of the date and time this report
> was run.

That last clause matters: time entered after the run changes next week's numbers, and an
attorney who spots a discrepancy should know why before emailing the billing manager.

## Checking it

Before the first send:

```
grep -c '<svg'            → 0
grep -c 'var(--'          → 0
grep -cE 'display:(flex|grid)' → 0
grep -c 'class='          → 0
```

Then send one to yourself in the real client. A browser preview does not tell you what
Outlook will do.

## Producing a PDF version

To hand the layout to someone as a sample, wrap the body in a print document with the
email headers above it and render with headless Chrome:

```bash
chrome --headless --disable-gpu --no-pdf-header-footer \
  --virtual-time-budget=4000 --print-to-pdf=out.pdf file://$PWD/print.html
```

Set `-webkit-print-color-adjust: exact; print-color-adjust: exact;` on `*`, or the bar
chart prints as empty tracks. Do **not** put `page-break-inside:avoid` on `tr` — the
outer wrapper is a single giant row, and making it unbreakable pushes content onto extra
pages.

## Worked example

[example-email.html](example-email.html) is a complete rendered report using illustrative
figures — a timekeeper against a 143-hour goal, with one client appearing twice in the top
five. Open it in a browser to see the finished layout, or copy its structure and swap the
numbers. Every figure in it reconciles: the matter rows sum to each period total, and the
twelve months sum to the year-to-date line.
