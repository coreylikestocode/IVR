---
name: ivr-gmail-check
description: Verify the Gmail connector is bound to info@islandviewretreat.com before doing any Island View Retreat mail work. Use BEFORE reading, drafting, labelling, or sending any Gmail in this project — and any time a session's first Gmail call happens here. Triggers: "check the gmail connection", "which mailbox am I on", "draft a reply to a guest", "reply to the booking enquiry", "send the quote", "what's in the IVR inbox", or any create_draft / send / label call against the Gmail connector.
---

# IVR Gmail connector check

The claude.ai Gmail connector is **single-account** and its binding is
account-global — it cannot be scoped to a directory. Re-authorizing it with a
different Google account **silently replaces** the binding rather than adding
an account. This was verified on 2026-07-19: authorizing Island View knocked
out `corey@useappello.com` entirely.

So the mailbox this project talks to is a runtime fact that must be checked,
not a setting that can be trusted.

## Why this matters

Island View Retreat mail is operational: booking quotes, availability holds,
guest correspondence signed by Kevin Shelson. Corey's `corey@useappello.com`
is a **different business**. Sending IVR correspondence from the Appello
identity — or Appello mail as Island View — is a real-world error that cannot
be undone once the message leaves.

Reading the wrong mailbox is merely confusing. Writing to it is not.

## The check

Run a read-only probe of the sent mail and inspect the sender:

```
search_threads(query="in:sent", pageSize=3)
```

Read the `sender` field on the returned messages.

## Interpreting the result

| Sender shows | Meaning | Action |
|---|---|---|
| `info@islandviewretreat.com` | Correctly bound | Proceed |
| `corey@useappello.com` | Connector was re-bound to Appello | **Stop.** Do not write. Tell Corey. |
| Empty `{}` or error | Connector unauthorized or disconnected | **Stop.** Report; do not guess. |

If it comes back as Appello, do not attempt to fix it yourself — the OAuth
flow lives on claude.ai and requires Corey in a browser. Report it plainly:
the connector needs re-authorizing with the Island View account, and restoring
Appello alongside it requires a *second, separate* connector instance, never a
re-auth of this one.

## Confirming a swap

If the top `in:sent` results look wrong but you want certainty before raising
it, the decisive check is a negative search:

```
search_threads(query="from:corey@useappello.com", pageSize=5)
```

An empty result confirms the Appello mailbox is genuinely unreachable on this
connector rather than merely ranked below the fold.

## Scope

This check is about **identity, not permission**. Passing it means you are
talking to the right mailbox — it does not authorize sending. Outbound mail,
drafts that will be sent, and any irreversible action still need Corey's
explicit go-ahead in chat, per normal rules.
