---
name: investigate
description: "Use when the user says /investigate, 'root-cause this', 'why did X happen', 'look into ticket', or a fleet watch hands over a Linear ticket labelled agent:investigate — any bug, data oddity, product question or ambiguous report that needs findings, not code."
---

# investigate — evidence in, findings out, no code

Owns the full lifecycle of an investigation ticket: claim it, find out what actually happened, post findings the reader can verify in under a minute, hand it back. Works for software bugs, data questions and product/behaviour puzzles alike.

## Iron Laws

Violating the letter of a law is violating the law.

1. **Claim first.** Before reading anything else: assignee me, state In Progress. A ticket under investigation must never sit in Todo.
2. **No writes except the findings comment and the state changes.** No code, no PR, no branch, no config, no new tickets. Suggest follow-ups in the comment; a human creates them.
3. **Every claim carries its evidence or says "not verified".** A record id, a query, a log line, a file:line, a live API response, a screenshot. A claim with none of those is a guess and must be labelled `not verified: <why>`.
4. **Findings are short.** Headline block ≤ 8 lines, whole comment ≤ 300 words before an optional `Detail` section. The reader gets the answer from the headline alone.
5. **Never redo a finished investigation.** If a root-cause comment already exists, verify its claims against current state and post only corrections or `Verified, nothing to add` — one line.

## Red Flags — STOP

- **Reading code before claiming the ticket** → STOP, claim it, then read.
- **Writing "probably", "likely", "I think" without an evidence tag** → STOP, either find the evidence or tag it `not verified`.
- **Comment past 300 words with no `Detail` heading** → STOP, move everything below the fold.
- **Reaching for a code fix "while I'm here"** → STOP, it goes in Fix options.
- **A data source you cannot reach (Honeycomb, prod DB, Sentry)** → STOP hiding it at the bottom; it goes in the headline block as `⚠️ not checked: <source> (<why>)`.
- **Inventing a flag for `fleet-status`** → STOP; the only forms are `fleet-status complete "<note>"` and `fleet-status awaiting "<note>"`.

## Rationalizations

| Excuse | Reality |
|---|---|
| I'll move it to In Progress once I know it's real | The label made it real. Claim first. |
| The evidence is obvious from the code | Obvious to you is not provable to the reader. Cite file:line. |
| More detail makes it more convincing | The reader stops at line 9. Detail goes below the fold. |
| Creating the follow-up ticket saves a step | Writes beyond the comment are not yours to make. |
| The prior comment is old, I'll redo it properly | Verify it against current state; post only the delta. |

## Steps

**0. Tools.** `ToolSearch(query="select:mcp__linear-server__get_issue,mcp__linear-server__list_comments,mcp__linear-server__save_issue,mcp__linear-server__save_comment,mcp__linear-server__get_user")`. If Linear is unavailable, stop and say so.

**1. Claim.** `get_issue` (with relations). If state is Done/Cancelled: stop and report. Otherwise `save_issue(id, assignee: "me", state: "In Progress")`. If the state name is rejected, report the error and stop. Then `list_comments`; if a root-cause comment exists, switch to Law 5 mode.

**2. Read.** Ticket, every comment, related and parent tickets, linked PRs. Write down, in one line each: the reported symptom, who saw it, when, and the reporter's own theory (kept separate from yours).

**3. Hypotheses before evidence.** List at least three, and at least one from outside the code: user behaviour, upstream data, timing/deploy, config, product expectation mismatch. Rank by cheapest to disprove. Do not start with the one the ticket suggests.

**4. Evidence, cheapest first.** For each hypothesis until one survives:
   - Reproduce or observe the actual record: Avo/admin link, DB row ids, timestamps.
   - Telemetry: Honeycomb trace/query link, Sentry issue, job logs. Record the query, not just the result.
   - Code: `file:line` on the current default branch, plus `git log -S`/blame for when it changed.
   - Upstream: live API response captured today, with the fields that matter.
   - Blast radius: one count query (how many users/records/events), with the query.
   Stop when the surviving hypothesis explains every observation in step 2. If none does, that is the finding.

**5. Post the comment** with `save_comment`, exactly this shape:

```
**<Headline: what happened, one sentence, no jargon>**

**Verdict:** ✅ confirmed | 🟡 likely (what is missing) | ❌ unknown     ← about the mechanism only
**Impact:** <who/how many, one line, with the query or record ids>
**Cause:** <one sentence, with file:line or record id>
⚠️ not checked: <source> (<why>)          ← only when a source you needed was unreachable

| Evidence | Link / id |
|---|---|
| <≤6 rows, one claim each> | <Avo/Honeycomb/PR/file:line/record id> |

**Fix options** (≤3, each: what, effort S/M/L, what it does not cover)
**Not verified:** <claims you could not prove, or "none">

```diff
- <what is broken, one line>
+ <what is safe / already handled, one line>
```

<details><summary>Detail</summary>
<everything else: queries run, full payloads, the disproved hypotheses and why>
</details>
```

Verdict grades the mechanism: ✅ when the cause is proven by file:line, a reproduction or a record; 🟡 when one link in the chain is unproven; ❌ when no hypothesis survived. How often it happens and to whom belongs in **Impact**, and when you could not measure it, Impact says so and the ⚠️ line names the source you lacked.

Before posting: count words above `Detail` (≤ 300), check every row has a link or id, check the ⚠️ line exists if a needed source was unreachable.

**6. Hand back.** `save_issue(id, state: "In Review")`. Then, in your own shell, `fleet-status complete "<TICKET>: <headline>"`. If you hard-stopped (Linear write failed, evidence impossible without access you lack, two consecutive tool failures): leave the ticket In Progress, post what you have with Verdict ❌ unknown, and run `fleet-status awaiting "<TICKET>: <what is blocked>"`. If `fleet-status` is not on PATH, skip it and say so.

## Running by hand

Applies only when a human typed `/investigate` in a terminal, never when the prompt arrived from a fleet watch. If the ticket carries `agent:investigate`, warn first: the watch will pick the same ticket up on its next poll unless the label is removed. Ask which one should own it; do not proceed until answered. No label: proceed.

## When NOT to use

The ticket asks for a change, not an answer (use e2e). The question is answerable from the ticket text alone (answer it, no lifecycle).
