---
description: "Generate Joel's morning chief-of-staff brief — Must act / Waiting on you / Today / Commitments detected, ≤200 words with hard caps, read-only. Use when the user says 'brief', 'morning brief', or 'what needs me today'."
argument-hint: [optional focus, e.g. "people" or "delivery"]
---

# Morning Brief — /brief

Chief-of-staff v0: one shot, push-format, ≤200 words. Born from the 2026-08 workflow analysis — machine-owned loops ran 9/9 while Joel-initiated loops ran ~0/9, so this command owns the initiation and leaves Joel only the judgment. The known-answer bar: replayed against Aug 2026 history it must surface the unanswered Kinza product signal, the Q3 forecast promise, the overdue compliance chase, and INT-592's month-long stall.

## Collect — read-only, metadata before bodies

Run collectors in parallel. Never fetch a message body until the item survives the metadata filter. If a Slack/Linear/Gmail MCP tool is deferred, load it via ToolSearch first.

| Source | Call | Keep only |
|---|---|---|
| Calendar | `list_events` today + tomorrow, Europe/London | double-bookings, meetings needing prep, unconfirmed attendees on meetings HE organises, meetings still accepted on leave days |
| Linear | `list_issues` assignee=me state=started, plus priority=1 | merge/park decisions, items stalled >7 days, due dates |
| GitHub | `gh search prs --review-requested=jjholmes927` and `--author jjholmes927`, open | review requests with age, own PRs conflicted or quiet >48h |
| Gmail | `search_threads` in:inbox newer_than:3d restricted to admin senders (trust@, people@, hibob, spendesk, ashbyhq, knowbe4, Drive access requests) | deadlines, overdue chases, access requests blocking a person |
| Slack | `slack_search_public_and_private` `to:me -from:me after:<last brief date>` | human asks with no reply from Joel; drop bots and social chat |
| Ledger | `~/.claude/cos/ledger.md` | every open commitment carries forward until done or dismissed |

## Compose — hard caps; never trim a warning

≤200 words across the four sections combined (title line and source footer excluded — `wc -w` on the section bodies is the check), exactly four sections:

- **Must act** — ≤3 items, each time-sensitive today, deadline named. A blocked person outranks a blocked ticket.
- **Waiting on you** — decisions only Joel can make. Never restate an item as "still waiting" more than twice — the third appearance must propose an escalation, reassignment, or explicit kill.
- **Today** — one line of schedule, conflicts flagged ⚠️, rota state noted.
- **Commitments detected** — ≤3, explicit and source-linked only, never inferred from vague discussion. Joel's own promises rank first.

Footer: one line of per-source freshness watermarks. A source that failed reads `source: FAILED` — a silently absent source is a defect of this command.

## Deliver and record

1. Write the brief to `~/.claude/cos/briefs/brief-YYYY-MM-DD.md` (create dirs as needed).
2. Print it in chat verbatim — the file is the archive, the chat is the delivery surface in v0.
3. Update `~/.claude/cos/ledger.md`: append new commitments as `date · source · promise · delete when`; remove entries Joel marked done or dismissed this run.

## Iron rules

- Read-only everywhere. No send, post, label, or ticket-write calls. If Joel asks for a reply, produce a draft for him to paste — never send it.
- An empty section prints "— none". An absent section is a failure.
- Caps are hard. Dropping the 4th Must-act item is correct behaviour — it lands in the ledger, not the brief.
- If a collector errors, name it in the footer and continue. Never fabricate a source's content, and never present a partial sweep as a full one.
