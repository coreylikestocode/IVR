---
name: cic-owner-account
description: Read and manage the Island View Retreat listing on CottagesInCanada (owner portal, listing DI-32361) — including the full rates CRUD used to configure seasonal pricing. Use for setting up or editing rates for a season or year, checking new guest inquiries, reviewing reservations, auditing calendar availability, spotting double-booking risk against Airbnb/VRBO, or editing listing content. Triggers: "set up the 2027 rates", "configure rates", "add a rate period", "change the nightly/weekly price", "what are our current rates", "update pricing", "check CottagesInCanada", "any new inquiries", "what's booked", "is the CIC calendar current", "which weeks are still open", "update the listing", "CIC availability".
---

# CottagesInCanada owner account

Owner portal for the Island View Retreat listing. Verified 2026-07-19.

- **Listing:** `DI-32361` — "Luxury Log Cottage on Big Gull Lake, Frontenac,
  Ontario — 9 Beds, 4 Rooms + Bunkie, Beachfront, WIFI"
- **Numeric id:** `32361`
- **Owner name on inquiries:** Kevin
- **Login:** info@islandviewretreat.com (see [[ivr-gmail-connector]] — same address)

## Login is a human step

Claude does not enter credentials. The session must be established by a person
logging in at `https://www.cottagesincanada.com/owner/login.aspx` in the
Browser pane (`preview_start` → `navigate`), after which Claude can drive
everything behind the auth wall.

Two traps, both seen in practice:

1. **Two browsers.** Logging into real Chrome does nothing for the Browser
   pane — separate cookie jars. The login must happen in the pane Claude is
   driving, unless the claude-in-chrome extension is connected.
2. **Session timeouts.** This is ASP.NET with `__VIEWSTATE`. Sessions lapse
   fairly quickly. If pages start bouncing to the login form mid-task, that's
   all it is — ask for a re-login, don't debug it.

Never store, request, or write the password anywhere.

## Hosts and paths

Login is on `www.`, the app is on `app.` — and the app lives under `/owner/`.
Hitting `app.cottagesincanada.com/rm-inquiries.aspx` without `/owner/` yields
`Error.aspx`. Full base: `https://app.cottagesincanada.com/owner/`

| Page | Path |
|---|---|
| Dashboard | `default.aspx` |
| Inquiries | `rm-inquiries.aspx` |
| Calendar | `rm-calendar.aspx` |
| Reservations | `rm-reservations.aspx` |
| Payments | `rm-payments.aspx` |
| Reviews | `rm-reviews.aspx` |
| Statistics | `statistics.aspx` |
| Listings | `lm-listings.aspx` |
| Featured Cottages | `featured-listings.aspx` |
| Special Offers | `special-offers.aspx` |
| Sign Out | `sign-out.aspx` |

## Prefer these over scraping

Two paths avoid the fragile parts of the site entirely.

**Calendar — keyed iCal, no login required:**

```
https://app.cottagesincanada.com/process/calendar.ics?id=32361&k=<KEY>
```

The `k` value is a capability token and is **deliberately not stored in this
file — this repo is public.** Retrieve it from the portal: Calendar →
Import/Export, under "Export your calendar" (takes about ten seconds). It is
stable, so paste it in at the point of use, don't commit it.

Verified HTTP 200 unauthenticated, 54 `VEVENT`s. Events carry
`SUMMARY:Reserved` with `DTSTART`/`DTEND`. This is the correct way to answer
any "which weeks are open" question — `curl` it, parse it, done. No session,
no VIEWSTATE, no pagination.

**Reservations — Export to CSV** button at the bottom of
`rm-reservations.aspx`. Better than paging through 15-row tables.

## Scraping notes, where scraping is unavoidable

- **Calendar day state is in CSS classes, not text.** `calDayRes` = reserved,
  `calDayAva` = available, `calTdNoSpace` = padding cell. Reading the page
  text gives you a wall of bare numbers with no state. Query classes instead:

  ```js
  [...document.querySelectorAll('td')]
    .filter(td => /^\d+$/.test(td.textContent.trim()))
    .map(td => ({ day: td.textContent.trim(), state: td.className }))
  ```

  Blocked / tentative / imported-iCal states have their own classes not yet
  observed (the calendar had none at time of writing).

- **Inquiries and Reservations paginate at 15 rows.** As of 2026-07-19: 826
  inquiries, 125 reservations. Don't assume page one is the whole story.

- **Most controls are `__doPostBack` with VIEWSTATE**, e.g. Import/Export is
  `__doPostBack('M$ContentPlaceHolder1$lnkImportExport','')`. They depend on
  page state, so they must be clicked in a live session — you cannot construct
  a URL for them. Clicking by element id works:
  `document.getElementById('ContentPlaceHolder1_lnkImportExport').click()`

- **`get_page_text` reports only the origin**, not the full URL. To find out
  where you actually are, evaluate `location.href`.

## Rates — full CRUD map

Verified 2026-07-19 by opening every form; nothing was saved.

The listing editor is at direct URLs, no click-through needed:

| Screen | URL |
|---|---|
| Listing tabs | `listing-overview.aspx?ida=32361` |
| Rates list | `listing-rates.aspx?ida=32361` |
| Rate create/edit | `listing-rates-item.aspx?ida=32361&idTarif=<id>` |

`ida` is the listing (32361). `idTarif` is the rate row. Omit `idTarif` for a
new rate. Rate IDs are **not exposed in the grid's client data** — the
`getDataKeyValue` call returns nulls, so the only way to learn an ID is to
click the row's action icon (`radGridRates_ctl00_ctl<NN>_gbcSelection`, where
NN runs 04, 06, 08 … stepping by 2) and read the resulting URL. Known so far:
`idTarif=1` is BASE RATE, `idTarif=23` is FALL 2026.

**BASE RATE is a protected singleton.** It's `idTarif=1`, has no name and no
date fields, and its form offers only *Save Changes* / *Cancel* — **no Delete
button**. Seasonal and event rates get the full form plus *Delete*. Don't
plan a "wipe all rates and rebuild" flow; the base rate can only be edited in
place, and it's the fallback for every date not covered by a period.

### Create/edit form fields

All ids are prefixed `M_ContentPlaceHolder1_`:

| Field | Element id | Notes |
|---|---|---|
| Type | `cboType_Input` | Telerik combo — `Seasonal` or `Event` only |
| Start date | `radDateDebut_dateInput` | displays `MM-DD-YYYY`; paired hidden `radDateDebut` holds `YYYY-MM-DD` |
| End date | `radDateFin_dateInput` | same pairing with `radDateFin` |
| Period name EN | `txtNomPeriode1` | e.g. "FALL 2026" |
| Period name FR | `txtNomPeriode2` | site is bilingual; FR shows on `/fr/` |
| Nightly | `txtPrixNuitSem` | currency-masked, renders as `$0` |
| Weekend night | `txtPrixNuitFinSem` | the `$750 Weekend Night` seen on Aug rows |
| Weekend (2 nights) | `txtPrixFinSem` | |
| Weekly | `txtPrixSem` | |
| Monthly | `txtPrixMois` | |
| Minimum stay | `cboMinStay_Input` | combo, see values below |
| Extra guest fee toggle | `chkFraisSuppOui` | checkbox, reveals the two fields below |
| Extra guest fee amount | `txtFraisAdditionnels` | renders as `0 $` — note trailing symbol, unlike the others |
| Applies above group size | `cboLargerGroup_Input` | combo, 2–99 |
| Note toggle | `chkNoteOui` | checkbox, reveals note fields |
| Note EN / FR | `txtNote1` / `txtNote2` | this is the "Check in Friday at 4:00PM…" line |
| Save | `btnEnregistrer_input` | "Save Rate" (new) / "Save Changes" (edit) |
| Cancel | `btnAnnuler_input` | |

**Minimum stay** options: (blank), 1–6 Nights, 10, 28, 29, 30, 31, 32 Nights,
1–3 Weeks, 1–11 Months, 1 Year. Note the gap — **there is no 7, 8, or 9
Nights**; a one-week minimum must be `1 Week`, not `7 Nights`.

Dropdowns are Telerik RadComboBox, not `<select>`, so `document.querySelector`
finds nothing useful. Read them with
`$find('M_ContentPlaceHolder1_cboMinStay').get_items().toArray()` and set them
by calling `.select()` on the matching item — setting `.value` on the visible
`_Input` text box alone will not register.

### Setting dates — read this before any bulk rate entry

**The date fields will silently keep their old value.** Each date is two
elements: a visible `_dateInput` and a hidden field (`radDateDebut` /
`radDateFin`) that is what actually posts. Setting the visible one updates
what you see while the hidden one keeps the default — and the form saves the
*hidden* value with no warning. Verified 2026-07-19: a form displaying
`11-15-2028` still held `2026-07-20` underneath.

These do **not** work: assigning `.value` in JS, dispatching synthetic
`input`/`change`/`keyup`, or typing via `computer{action:"type"}` (that one
also landed in the wrong field when a stale ref shifted focus).

What does work:

1. `form_input` on the visible date textbox (reaches the widget's client state)
2. then click that field's calendar icon (`radDateDebut_popupButton` /
   `radDateFin_popupButton`) — it opens on the typed month
3. click the day cell, or click the icon again to close and commit
4. **verify the hidden field** before saving:
   ```js
   document.getElementById('M_ContentPlaceHolder1_radDateDebut').value  // want YYYY-MM-DD
   ```

Never click Save until both hidden fields read the intended `YYYY-MM-DD`. A
mis-set start date defaults to *today*, so a botched save creates a live rate
covering the current week rather than a harmless future one.

Two related notes:

- **Combos**: `$find('M_ContentPlaceHolder1_cboMinStay')` then `.select()` on
  the matching item works. Calling Telerik's `set_selectedDate()` on the date
  pickers throws a strict-mode error about `arguments.callee` — use the
  calendar-click route instead.
- **Combo values are French-coded**: "2 Nights" posts as `2-J` (*jour*).
- **Refs go stale** after every postback. Re-run `read_page` before using a
  `ref_N`, or drive by element id through `javascript_tool`.

### Verified CRUD cycle

Run end-to-end 2026-07-19 on a throwaway rate, then removed:

| Op | How | Result |
|---|---|---|
| Create | Add New Rate → fill → `btnEnregistrer_input` | row 8 created, `idTarif=26` |
| Read | row menu → `listing-rates-item.aspx?...&idTarif=26` | all values round-tripped |
| Update | edit fields → `Save Changes` | name, nightly, weekly, min-stay all persisted; dates untouched |
| Delete | `btnDelete_input` | row gone; the id now redirects to the list |

No confirm dialog fires on Delete — it is immediate and unrecoverable. There
is no undo and no draft state.

### Expiry behaviour

Rates whose end date has passed stay in the list but render
`*** This rate is expired and it is not displayed with your listing ***`.
They are not auto-deleted. As of 2026-07-19 three of the seven rows were
already expired, so the list will accumulate clutter across 2027/2028 — worth
deciding whether to delete or leave them as a pricing record.

### Before writing rates

Rates are public and drive what guests are quoted. Confirm with Corey before
any save. Two specific traps:

- **Overlapping date ranges.** The existing rows are contiguous but not
  continuous (e.g. nothing covers Jun 20 – Jun 25, 2026, which silently falls
  back to BASE RATE). Check for gaps and overlaps before saving a 2027 set —
  the form does not appear to warn about either.
- **A wrong rate is live immediately.** There's no draft or preview state.

## Guest PII

Inquiries carry guest names, emails, and phone numbers — 826 of them. This is
other people's personal data. Do not paste it into artifacts, published pages,
commits, or anything leaving the machine. Summarize in-conversation; quote a
single inquiry when working it. Never bulk-export it without a specific reason
from Corey.

## Writes — ask first

Reading is free. These are not, and each needs explicit confirmation:

- **Replying to an inquiry** — goes to a real guest, cannot be recalled.
  Default to drafting for Corey to send, not sending.
- **Calendar edits** — blocking/unblocking dates changes what guests can book.
- **Listing content, rates, policies** — public-facing.
- **Adding an import calendar** — see below; correct move, but confirm the URL
  first, since a wrong feed silently blocks real availability.

## Known issue: no external calendar sync

As of 2026-07-19 the Import/Export panel has **zero calendars imported** —
both fields empty, no configured rows — while the listing is also live on
Airbnb (a Jul 16 inquiry from a guest who tried to book there first). The CIC
calendar's own "Updated" stamp read **Jun 17, 2026**, a month stale, with 278
of 365 days showing available.

Nothing is preventing a CIC guest from booking a week that Airbnb already
sold. The fix is pasting the Airbnb (and VRBO, if used) iCal export URL into
the Import a calendar fields — up to 5 feeds. Raise this before doing
availability work; any "which weeks are open" answer drawn from CIC alone is
only as good as this sync, which currently doesn't exist.
