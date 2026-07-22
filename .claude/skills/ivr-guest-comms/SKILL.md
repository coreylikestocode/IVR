---
name: ivr-guest-comms
description: Write, send, and follow through on Island View Retreat guest communications — outreach copy in the house voice, reply triage in info@, quote/hold handling, and contract drafting. Use for "write the outreach email", "draft a reply to this guest", "triage the inbox", "send the next wave", "update the campaign copy", "draft the rental agreement", or any guest-facing email work. Improves over time: every campaign learning gets logged in this file.
---

# IVR guest communications

One system for the whole guest lifecycle: outreach → reply → quote/hold →
agreement → stay → review ask. The 2027 early-access campaign is the first
full run; log what works below so the next campaign starts smarter.

**Precondition for ALL Gmail work:** run the `ivr-gmail-check` skill first.
The connector must be bound to info@islandviewretreat.com. Never send or
draft from the Appello identity.

## The house voice

Kevin and Corey, not a marketing department. Every rule below exists because
the counter-move reads as spam:

- **Plain text.** No HTML templates, no images, no buttons. One link max
  (islandviewretreat.com).
- **Warm, specific, brief.** Reference *their* stay, *their* dates, *their*
  holiday. A guest should feel remembered, not mail-merged.
- **Facts only, and only true ones.** Four bedrooms + the waterfront bunkie,
  nine beds, groups of 10–16, big sandy beach, A/C in every bedroom (bunkie
  goes without), gas fireplace, cleaning/landscaping/exterior staging always
  included. Never invent amenities; never quote prices above the published
  calendar; never claim scarcity that isn't real.
- **Sign-off:** "Corey & Kevin Shelson · Island View Retreat · Big Gull Lake ·
  Arden, Ontario", then the opt-out P.S.: *"If you'd rather not hear from us,
  just reply 'no thanks' and we won't email again."* Always. It's both CASL
  hygiene and good manners.
- Kevin's phone (226-577-7767) may appear in 1:1 replies, never in bulk sends.

## Outreach copy structure

Every campaign email is: greeting → **hook** (the personalized sentence — the
whole game is here) → price line for *their* week → optional group-size line
(party ≥9) → close (link + deadline + first-come-first-served + "reply with
dates and we'll hold them") → segment outro → sign-off.

Segment hooks (2027 campaign versions):
- **S1 repeat guest** — "Returning families get first call on dates, and
  you're at the top of that list." Name their years and most recent week.
- **S2 past guest** — name the exact week and year they stayed. "Past guests
  hear first."
- **S3 hot inquiry** — quote the exact dates they asked about and the 2027
  price for the matching week. Neutral about why it didn't work out.
- **S4 holiday asker** — lead with their holiday; explain Fri–Mon + the
  Thursday-before option with both prices.
- **S5 earlier inquiry** — light general announcement; "everyone who's ever
  asked about the place hears first."

**Where the copy lives (keep in sync — this is the one sharp edge):**
1. `build_outreach.py` in the session scratchpad (see memory
   `ivr-2027-outreach-tracker`) — generates tracker rows + reference bodies.
2. The Apps Script in the tracker's "Mail merge setup" tab — builds the real
   sent bodies from row fields at send time.
A copy change means updating BOTH, or deliberately editing only the Apps
Script (source of truth for what actually sends).

## Sending discipline

- Ramp: start ≤20/day on a cold sender, grow ~30% daily, cap 160–170/day.
  3s between sends. Warmest segments first — their replies build reputation.
- Only status=`Approved` rows send. Statuses: Draft → Approved → Sent →
  Replied → Booked / Bounced / Unsub.
- Stop the ramp if bounces exceed ~2% in a day.
- SPF/DKIM/DMARC on islandviewretreat.com verified 2026-07-22.

## Reply triage playbook

Check info@ daily during a campaign. For each reply:
1. Update the tracker row status + a one-line note.
2. **"no thanks" / any opt-out flavour** → status Unsub. Never email again.
   No reply needed (or one line: "Done — all the best.").
3. **Date request** → check the CIC calendar for conflicts FIRST (double-book
   risk vs Airbnb too), then draft: confirm the exact dates + price from the
   published calendar, state the hold ("we'll hold these for 5 days"), and
   the next step (rental application). Summer short-stay asks follow the
   week-only rule — offer the full week, note shorter stays may open closer
   to the date.
4. **Price pushback** → do not discount in writing. Flag for Corey/Kevin.
5. **Questions** → answer from the deck's facts (the live site is the
   canonical copy source). Never improvise amenity or rule details.
6. Drafts for anything consequential go to Gmail Drafts for a human to send
   unless Corey has explicitly authorized direct sending in the session.

## Contract drafting

When a guest confirms dates and the deposit conversation starts:
- Source of truth for terms: the signed agreements in Kevin's Dropbox Sign
  history (pattern: "Cottage Rental Agreement - {Guest Name}").
- Standard commercial terms: 50% books the dates, balance due 30 days before
  arrival, $1,500 fully refundable security deposit, 13% HST on rent, linen
  service $250 optional, check-in Fri 4:00 PM / check-out 10:00 AM sharp.
- Draft the agreement details (names, dates, amounts) and the cover email;
  Kevin sends via Dropbox Sign from kevin.shelson10@gmail.com.
- After signing: Final Details document + a phone call scheduled ~1 week out;
  gate/lockbox codes go out by email the week before check-in.
- Log every booking in the tracker (status Booked) AND tell Corey to update
  the deck calendar's booked dates (`.day.bk` state is built and waiting).

## Learnings log (append after every campaign / wave)

- **2026-07-22 (pre-launch):** 805-contact early-access campaign built.
  Hypotheses to verify: (a) exact-dates hooks (S3) out-reply generic (S5);
  (b) holiday framing (S4) converts shoulder weeks; (c) "third night on us"
  reframing beats "3-night minimum". Check reply rates per segment in the
  tracker's Send log after wave 3 and record results here.
- **Data lesson:** the master booking workbook has name/phone/email on every
  booking row — it is the authoritative contact source, ahead of inquiry
  threads. Mine it first next time.
- **Search lesson:** guest names in the booking sheet are unreliable
  (Byran/Bryan, Green/Greene, Gebrael/Occhinero, "Fareeha Amded"). When a
  name search fails, retry with date-bounded searches around their stay and
  check e-transfer / Dropbox Sign notification emails.
