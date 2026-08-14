# Island View Retreat — shared operating rules

Applies to every agent in `.claude/agents/`. Each agent's own file says what it
owns; this says how they work alongside each other.

## 1. Memory is context, not a control surface

Memory files under `~/.claude/projects/-Users-coreyshelson-IV/memory/` are a
**shared write surface with no staleness signal.** Several agents both read and
write them, and nothing marks a line as out of date.

On 2026-08-13 this failed exactly as you'd predict. A memory file said "run
`2026-fall-shortstay` first". It had been true when written; the campaign was
repositioned afterwards. The Chief of Staff read it in good faith and
recommended sending a superseded campaign to 399 people. The owning agent caught
it before it reached Corey. **Nobody was careless — the surface was.** An audit
then found five more stale claims in the same file: a branch described as
unmerged that was deployed, a contact count off by one, secrets described as
missing that were on the server, and a setting described as OFF that Corey had
already turned on.

So:

- **Never quote a memory figure into anything outward-facing without re-checking
  it against the live system.** Numbers, statuses, and "not yet done" claims rot
  fastest.
- **Put operational constraints in code, where they can refuse.** A note saying
  "don't send this" is documentation. A `CANCELLED` status that makes
  `sendBroadcast` throw is a guard. Prefer the guard.
- **When you correct a memory file, correct the whole file**, not just the line
  that bit you. Rot clusters.
- Memory is for hard-won context — traps, rationale, why a thing is the way it
  is. That ages well. Operational state does not.

## 1a. "Harmless" is itself a claim worth checking

The same day, the Chief of Staff noted that two test contacts sat in the send
audience and called it *"harmless, arguably useful as a delivery canary"* — a
judgement offered without checking. The owning agent checked, and it was wrong on
both halves:

- `info@islandviewretreat.com` was at **position 69** — it would arrive on **day
  2** of a 20-day ramp, putting a marketing email addressed "Hi Kevin" into the
  live booking inbox that handles real guests.
- `corey.shelson@gmail.com` was at **position 786** — arriving on **day 20**,
  useless as an early warning. `send_test` already covers that need properly:
  on demand, any campaign, delivered in seconds, recorded nowhere.

Both are now `HOLD` rather than `doNotContact` — dropped from `MAILABLE` and every
audience, without implying they misbehaved. Reversible. The audience is **794**.

The rule: dismissing something as harmless is a factual claim about blast radius.
It costs a query or two to check, and here the check reversed the answer.

## 1b. A status word is not an outcome

**Three times in one day (2026-08-14) a status field was read as a fact about the
world, and every time the correcting evidence sat in the same payload as the claim:**

| Field said | Reality | The tell, already in hand |
|---|---|---|
| `booking_status: "available"` | An owner-side block | `reservation_id: "…__blk"` in the same row |
| campaign named "transactional" | Nothing triggers it | no caller anywhere in the repo |
| a post "scheduled for tomorrow" | A **draft** whose time had already passed | `status: "draft"`, `sentAt: null` |

Each cost a correction that reached Corey. None needed new information — only
reading the rest of the object.

So: **a label describes intent; a different field describes what happened.** Before
reporting a status as an outcome, find the field that would disagree and quote it.
If a payload offers both a summary field and a detail field, the detail field wins.

**Those three tells are examples, not a checklist.** They are simply the ones we
happened to find in one afternoon. The rule is *find the field that would disagree*
— read as "check `sentAt`, the id suffix, and for a caller", it will sail straight
past the fourth instance. The value is in the looking.

**And note who caught them: in all three cases it was someone re-checking a peer's
claim, never the claimant catching their own.** That is the real lesson. A single
agent re-reading its own payload did not catch these; a second agent asking "is that
actually true?" caught every one. Which argues for the cross-checking habit far more
than for any table of fields.

Prefer the tool that answers the actual question. `get_pms_reservations` reports
absence-of-reservation as availability; `get_bookings_report` returns an explicit
`status`. Ask the one that can say "blocked".

## 1c. Size a finding from the values it actually covers

Twice on 2026-08-14 a real finding reached Corey with an inflated headline number,
both times because a rate was carried past the dates it covered:

- **"~$10,000 of unsold September"** — the range was blocked, not open.
- **"~$6,000 of over-blocked October"** — the eight nights are real, but they are
  worth **$3,999**. The $750 figure used to size them is an override covering
  **Oct 2–7 only**; the actual recommended prices for Oct 18–25 run $468–$558.

**The finding survives the correction. It just gets smaller and more credible.**
That is the trade: an inflated number does not make a problem more persuasive, it
makes the next number you report less trusted.

So: price a range from the values for **those dates** — `get_listing_prices` for the
actual window — never from a neighbouring band, a season average, or a rate card that
"basically covers it". Before reporting a round, dramatic figure, check which dates
the rate you used actually applies to.

## 1d. Before a sentence becomes a headline, ask where its check was run

The single test that catches every error we made on 2026-08-14:

> **Was the check supporting this claim run in this session, or inherited?**

Every one fails it. `welcome-inquiry` "firing" was inherited from a campaign *name*.
The "$4,250 on CIC" was inherited from a rate row that was true in July and
superseded by a promo. "Sep 5–20 available" was a field read without its sibling.
"~$6,000" was a rate applied outside its dates. None needed new access — only
re-running the check at the moment of asserting it.

This bites hardest at the **summarising** step, and hardest of all on the Chief of
Staff, because compression *is* that desk. The failures were not in gathering; the
`__blk` and the $750's date range were both in hand and lost while compressing.

### A related trap, inside the analysis rather than the summary

**Two derivations agreeing can be redundancy, not corroboration.** Five 2026 price
overrides matched the 2027 card by both `weekly ÷ 7` and `short-stay ÷ 3`. The
agreement felt like independent confirmation; it was a property of the rate card
being internally consistent. The matches proved the **source** of the numbers and
said nothing about the **mechanism** that wrote them — which was then reported as
though established.

So: when two routes to the same answer agree, ask whether they could ever have
disagreed. If not, you have one piece of evidence, not two. And keep *source* and
*mechanism* as separate claims with separate evidence — conflating them closes a
question that is still open, which is worse than leaving it open.

## 2. Facts have owners; consumers do not transcribe

If a fact belongs to another agent, take it from them and cite where it came
from. Do not copy it into your own artifact where it can drift silently.

- **Rates and availability** → `ivr-revenue-desk`. Never read a price off a page
  and hardcode it.
- **Site copy and brand voice** → `ivr-content-engine`. Social and email restate;
  they don't originate.
- **Contact and consent state** → `ivr-lifecycle-email`.
- **Booking and reservation facts** → `ivr-channel-listings`.

When you change a fact others hold copies of, **push the notification** — they
cannot detect your change.

## 3. Deploys are serialized

One 1-vCPU droplet, one `main`. A concurrent deploy OOM-killed the box on
2026-08-13 and took the site down ~10 minutes; a push was rejected the same day
because another session had pushed twice. Announce before deploying, wait if
another agent is mid-build, and never force-push `main`. Builds run 15–25 minutes.

## 4. Two repos, one of them public

`/Users/coreyshelson/IV` is **public** and holds 782 people's contact details,
kept out of git by `.gitignore` alone. The app repo `island-view-retreat` nested
inside it is private. `finance/` is gitignored.

**Verify `.gitignore` before every `git add` in the parent repo.** Never move
guest data, financial records, or secrets upward into it. Never paste keys into
a transcript — two Resend credentials already leaked that way and need rotating.

## 5. Outward-facing actions are Corey's

Sending to the list, publishing a post or page, changing a public rate, editing a
listing, and deploying are all confirmed with Corey first — every time, including
when the change is obviously correct. Standing responsibility for a domain does
not remove that step; it is what makes moving quickly on everything else safe.

Claude never enters credentials, changes DNS, or moves money. Those are human
steps by design, not obstacles to route around.

## 6. Report what actually happened

If a check was skipped, say so. If a figure came from memory rather than the live
system, label it. Both near-misses today were caught because an agent volunteered
a correction against its own earlier work.
