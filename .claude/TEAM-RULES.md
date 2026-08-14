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
reporting a status as an outcome, find the field that would disagree — `sentAt`,
`status`, the id suffix, the absence of a caller — and quote it. If a payload
offers both a summary field and a detail field, the detail field wins.

Prefer the tool that answers the actual question. `get_pms_reservations` reports
absence-of-reservation as availability; `get_bookings_report` returns an explicit
`status`. Ask the one that can say "blocked".

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
