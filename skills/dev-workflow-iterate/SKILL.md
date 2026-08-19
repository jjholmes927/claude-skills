---
name: dev-workflow-iterate
description: "Periodic review of the dev workflow itself — what's working, what's missing, what's rotting — via instruments-first evidence, an adversarial panel, and deletion-first output. Use when workflow friction accumulates, after a big process change beds in, or roughly quarterly. Two modes: light (~30 min, no fleet) and deep (multi-agent, expensive)."
---

# Dev workflow iterate

Reviews the development workflow the way we review production: measured evidence, adversarial verification, and changes sequenced deletion-first. Born from the 2026-08-18/19 control-plane review (artifacts: The Missing Control Plane, The Operator's Day; memory: `workflow-control-plane-plan`).

## Mode selection

Ask which mode unless the request states it:

- **light** — ~30 minutes, no subagent fleet. Read the instruments, answer three questions, cut scope. Default for a monthly pulse or a single felt friction.
- **deep** — multi-agent panel + blind cross-model opinion. Several million tokens and hours. Reserve for quarterly reviews, a pile of accumulated friction, or before/after a structural change (new tooling era, team change, new machine).

## Iron laws (both modes)

1. **Prior art before proposals.** Read `second-brain/01-decisions/` for standing decisions, the previous iteration's memory + artifact, and MEMORY.md's workflow rules — BEFORE drafting anything. A recommendation that contradicts a settled decision must name it and argue the reversal explicitly. (The first run nearly shipped a "stream registry" the 2026-07-19 work-OS record had already ruled out.)
2. **Instruments before archaeology.** Start from what's already measured: `closure-sweep` output, `clone-status`, the workflow-ledger Honeycomb dataset (skill/verify usage), `gh pr list` stats (cadence, size, unreviewed rate), rate-limit history, MEMORY.md Active-work aging (entries violating the entry contract). Only excavate raw sources (shell history, transcripts) for questions the instruments can't answer.
3. **Live-verify every tool claim before recommending it.** Docs and web sources are hypotheses; run the command on this machine and tag findings LIVE / DOCS / WEB. (Docs-optimism produced two near-misses in the first run: agent-view JSON's real schema, and `.worktreeinclude` vs `WORKTREE_OFFSET`.)
4. **No new surfaces.** The fix is fewer places to look, not another dashboard. Any proposal adding a standing thing-to-check must displace something bigger.
5. **Deletion-first output.** Phase 0 of any plan is pure scope cuts (rot, dead config, stale worktrees/stashes, leaked secrets) executed immediately; builds come after and must pass the fast-scope-cuts / slow-architecture-bets test.
6. **Consent boundaries hold.** Team-facing changes (shared repos, review rotas, CI) are proposed, not shipped, unless explicitly asked. Never merge past a failing guard.

## Light mode

1. Run the instruments (law 2) and skim the previous iteration's plan memory — mark each of its items done / rotted / abandoned.
2. Answer three questions with evidence, one line each per item:
   - **Working:** what earned its keep since last time (usage evidence, not fondness)?
   - **Missing:** what friction did the operator actually feel (ask them for their list too)?
   - **Rotting:** what shipped last iteration and is now unused, drifted, or bypassed? (Check: does the plugin cache match source? Do the helpers still run? Are memory delete-whens overdue?)
3. Output: a short chat summary + up to five actions, at least one of which is a deletion. Execute the scope cuts now; anything bigger gets a memory entry per the active-work entry contract.

## Deep mode

1. **Evidence pack** (light mode steps first, then): fresh gh/Linear throughput stats, live tmux/fleet snapshot, any new-tool questions the operator raised. Write it to a scratchpad file — panel agents read the file, not the conversation.
2. **Research only the open questions** — parallel agents per question (named tools, cloud state, practitioner patterns), each returning findings with source evidence. Don't re-survey settled ground; cite the previous run's verdicts and check only what could have changed.
3. **Draft the recommendation, then attack it.** Adversarial panel, one agent per lens, each reading the evidence pack:
   - build-steelman (argue for MORE custom investment),
   - pragmatist (argue the boring existing thing already does it — hunt for wiring bugs before builds),
   - attention-economics (argue for removal; every addition must fund itself in attention),
   - claim-verifier (live-run the load-bearing tool claims; tag LIVE/DOCS/WEB; may probe in a scratch repo only, never the lanes),
   - completeness critic (what's missing: prior art, team angle, cost, portability).
4. **Blind cross-model opinion:** brief Sol via codex-collab with the evidence pack ONLY (not the draft), xhigh effort. Reconcile where Sol and the panel disagree; unresolved forks get a decision rule ("do X for four weeks; if Y still happens, build Z"), not a compromise.
5. **Output:** artifact (visual, verdict-first, provenance-tagged) + 3-5 line chat summary + phased plan where Phase 0 scope cuts are executed immediately. Update `dotfiles/docs/operator-workflow.md` for anything that changes the operating model, and write the iteration memory with expected outcome, next action, and delete-when.

## Red flags

- Recommending a tool from docs or stars without a live probe → law 3.
- A plan whose Phase 1 is a build → re-check laws 2 and 5; the first run's biggest find was a wiring bug (`/e2e` bypassing `bin/create_worktree`), not a missing system.
- The review producing only additions → the attention lens failed; re-run it.
- Skipping the operator's own "what felt bad" question → instruments miss feelings; both are evidence.
