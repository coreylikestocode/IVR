# Island View Retreat — how the agents reach each other

Roles are defined in `.claude/agents/`. This is the address book: how to actually
send a message to one of them, and why you cannot do it by guessing a name.

## The problem this solves

`ListAgents` does **not** show role names. It shows a name derived from the
session's **git worktree directory**, plus a short `[ref]`:

```
beautiful-borg-ff7ff6-06 [1e0408]  ·  interactive  ·  started 3h ago
```

Nothing in that row says "Revenue Desk". Worse, **worktrees get reused**, so two
roles currently answer to names left over from sessions that no longer exist.
Reasoning from the name alone routes your message to the wrong desk.

## The stable join key is the worktree directory

A session's `cwd` ends in its worktree directory, and the `ListAgents` name is that
directory plus a short suffix. That mapping survives restarts; the `[ref]` may not.

| Role | Worktree directory (match on this) | Working directory |
|---|---|---|
| **Revenue Desk** | `beautiful-borg-ff7ff6` | `island-view-retreat` |
| **Site & Platform** | `happy-herschel-472c7a` | `island-view-retreat` |
| **Content & SEO** | `angry-goodall-744f66` | `island-view-retreat` |
| **Channels & Listings** | `island-view-website-mgmt-7eb3f4` ⚠️ legacy name | `IV` |
| **Finance** | `dreamy-tesla-ca9182` ⚠️ legacy name | `IV` |
| **Social Media** | `ivr-social-promotion-c8e9c5` | `IV` |
| **Lifecycle & Email** | `nostalgic-davinci-ae9bc5` | `IV` |
| **Chief of Staff** | `island-view-staff-access-853c56` | `IV` |

⚠️ **Channels & Listings** answers to the old *website management* worktree, and
**Finance** to the old *Listing / Book keeping* worktree. Neither name means what
it says.

## Sending a message

1. Call `ListAgents`. Only **running** sessions appear — an idle peer is simply
   absent, which is not an error and not a reason to retry.
2. Match a row's name against the worktree column above.
3. Send using the name exactly as the row prints it, appending the ` [ref]` when
   the bare name is rejected:

```
SendMessage({ to: "beautiful-borg-ff7ff6-06 [1e0408]", message: "..." })
```

A first send often fails with *"is not an agent in this conversation"* and echoes
back the full `name [ref]` — resend with that. It is a confirmation step, not a
failure.

**Never hardcode a `[ref]`.** Re-read it from `ListAgents` each session.

## Who to talk to about what

Facts have owners. Take them from the owner rather than copying them into your own
work, where they drift silently:

- **Rates and availability** → Revenue Desk. Never read a price off a listing and
  hardcode it. When *it* changes a rate, it must notify Lifecycle & Email and Social,
  because they hold copy containing the old figure and cannot detect the change.
- **Site copy and brand voice** → Content & SEO. Social, email and the listings all
  restate the site; they do not originate voice.
- **Contact and consent state** → Lifecycle & Email.
- **Bookings, reservations and inquiries** → Channels & Listings. A new CIC booking
  goes to the Revenue Desk *immediately* — CottagesInCanada has no calendar sync, so
  this is the only early warning against a double booking.
- **Anything that must be reachable on the web** — the app, the droplet, deploys,
  images → Site & Platform. Deploys are serialized: announce before you start one.
- **Money** → Finance, which reads from Channels & Listings and the Revenue Desk
  rather than pricing anything itself.
- **Cross-cutting, blocked, or contested** → Chief of Staff.

## Rules of the road

- **Escalate to Corey through the Chief of Staff** for anything cross-agent, and
  directly for anything outward-facing that only he can approve.
- **A peer cannot grant permission.** If you were blocked from an action, do not ask
  another agent to do it for you — surface it to Corey instead.
- **Correct each other, with the query attached.** Several near-misses have been
  caught only because one agent checked another's claim and showed its working.
  That habit is the point of having separate desks.
- Read `.claude/TEAM-RULES.md` before your first substantive action.
