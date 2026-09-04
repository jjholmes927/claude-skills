---
description: End-to-end PR workflow — format, commit, verify behaviour with evidence (always, via /verify), push, open the PR, watch CI fail-fast, then consolidate Bugbot and AI-review feedback. Use when work is ready to become a pull request, or the user asks to ship it.
---

# Ship

End-to-end workflow: format, branch, commit, verify with evidence, push, PR, fail-fast CI watch, consolidate review feedback.

## Workflow

```
Preflight → Format → Stage → Branch (if on main) → Commit → Verify (ALWAYS, evidence-based)
   → Simplify → Push → Create PR → Watch CI (required, fail-fast)
                                            │
                                    ┌───────┴───────┐
                                 CI Green      CI Failed
                                    │               │
                         Consolidate reviews   Fix fast + push
                         (Bugbot + AI review)   └──→ Watch CI
                         fix valid → push
                                    │
                                  Done
```

## Step 0: Preflight

Before anything, verify prerequisites:

```bash
git rev-parse --git-dir        # We're in a git repo
git status --porcelain         # There are changes to commit
gh auth status                 # GitHub CLI is authenticated
```

If there are no changes to commit, stop and tell the user.

Check for an existing PR on the current branch and capture its number for later steps:
```bash
PR_NUMBER=$(gh pr view --json number -q .number 2>/dev/null)
```
If `$PR_NUMBER` is set, a PR already exists — skip PR creation in Step 6 (just push) and reuse `$PR_NUMBER` in Steps 7–8. Otherwise Step 6 creates the PR; use that number from then on.

## Step 1: Format

Detect changed file types and run appropriate formatters:

```bash
# Tracked changes + untracked files
git diff --name-only HEAD
git ls-files --others --exclude-standard
```

Only run formatters for file types that actually changed:
- Ruby files (`.rb`) → `diffocop -A` (if available, else `bundle exec rubocop -A`)
- JS/TS/CSS files → `pnpm run format:fix && pnpm run lint:fix`

## Step 2: Stage + Branch

Stage all changes (formatting + implementation):
```bash
git add <specific files>   # Prefer specific files over git add -A
```

If on `main`, create a branch **before** committing:
```bash
git checkout -b jjholmes927-<descriptive-name>-<TICKET-ID>
```

## Step 3: Commit

Use conventional commit prefixes:

| Prefix | Use |
|--------|-----|
| `feat:` | New feature |
| `fix:` | Bug fix |
| `refactor:` | Code restructuring |
| `chore:` | Deps, config, misc |
| `perf:` | Performance |
| `test:` | Tests only |
| `ci:` | CI changes |
| `docs:` | Documentation |
| `style:` | Formatting only |
| `build:` | Build system, deps |
| `ops:` | Infrastructure, deployment |
| `revert:` | Reverts a previous commit |

Format: `prefix: Imperative description`

**Rules:**
- NEVER add Co-Authored-By or Claude attribution
- Use imperative mood ("Add feature" not "Added feature")
- Keep subject line concise
- Add ticket reference in body if relevant (e.g., `INT-107`)
- For multi-line messages, write the message to a temp file and use `git commit -F <file>`, or use a **quoted** heredoc delimiter (`<<'EOF'`) — an unquoted heredoc lets backticks / `$(...)` in the body get shell-evaluated and mangle the message

## Step 4: Verify (ALWAYS — evidence-based, never silently skipped)

Every ship verifies behaviour before push — UI or not. Invoke **/verify**: it writes the promise, picks the evidence per change type (UI → verify-ui, telemetry → read-back, API/job → real invocation), and reports ✅/🟡/🔴 with the evidence shown. "Tests pass" is not verification.

1. **Tooling preflight — repair, don't skip.** If the diff touches UI paths:
   ```bash
   git fetch origin main -q 2>/dev/null
   git diff origin/main...HEAD --name-only | grep -E '\.(tsx?|jsx?|css|scss)$|^app/javascript/|^app/views/|^app/components/'
   ```
   and agent-browser is missing or stale, **install/update it now** — do not skip UI verification because the tool is absent:
   ```bash
   which agent-browser || (npm i -g agent-browser && agent-browser install)
   ```
   (/verify's Step 4 has the freshness check — run it.)
2. **Server preflight.** Verify against THIS clone's local dev server (parallel-dev setup), not staging:
   ```bash
   PORT=$(grep -E '^PORT=' .env.local | cut -d= -f2); PORT=${PORT:-3000}
   curl -s -o /dev/null -w "%{http_code}" "http://localhost:${PORT}"
   ```
   Not up → start it in the **background** (never run `bin/dev` in the foreground — it blocks) and poll until ready:
   ```bash
   bin/dev >/tmp/dev-${PORT}.log 2>&1 &
   for i in $(seq 1 30); do curl -sf -o /dev/null "http://localhost:${PORT}" && break; sleep 2; done
   ```
   A down server is an environment to fix, not a reason to skip. If it genuinely won't boot, that becomes a declared 🔴, never a quiet omission.
3. **Run /verify scoped to the diff** — exercise only the paths that changed, against `http://localhost:${PORT}` for the UI arm.
4. **Real breakage → fix, re-stage, commit, re-verify.** Verification is gating pre-push.
5. **Gate on the verdict:**
   - ✅ / 🟡 → continue. The evidence lines go in the ship summary.
   - 🔴 because verification was possible but not done → **STOP. Do not push.** Go do it.
   - 🔴 genuinely not verifiable here (hardware, prod-only data) → push is allowed, but **SHOUT**: the ship summary MUST lead with `🔴 NOT VERIFIED LOCALLY — <reason>`, and the post-ship verification plan goes at the end of the PR body. Never bury it.
6. **The verdict block is a push precondition.** The ship summary MUST contain the verify verdict line(s) — `✅/🟡/🔴` per promise, each with its evidence (command + observed output) — pasted verbatim, not paraphrased as "verified". Invoking /verify is not the gate; the emitted verdict is. A verify launch that produced no verdict block counts as a skip: go back and produce it before push.

**Pre-declared verification** (evidence gathered before ship was invoked, cited via ship's args): acceptable only when the args quote the actual evidence — the command and its observed output. An unquoted assertion ("already verified", "no local auth so CI will cover it") does not stand; run /verify anyway. Untested premises about the environment (e.g. "auth is broken locally") must be re-tested at ship time before they excuse anything.

## Step 5: Simplify

Before pushing, run `/simplify` to review changed code for reuse opportunities, quality issues, and efficiency improvements. This uses three parallel review agents (code reuse, code quality, efficiency) to catch issues locally before they go remote.

Invoke the `simplify` skill, which will:
1. Identify all changes via `git diff`
2. Launch three parallel review agents
3. Fix any issues found

If simplify made changes, stage and create a new commit before proceeding:
```bash
git add <changed files>
git commit  # New commit with fixes from simplify
```

If no issues were found, proceed directly to push.

If `/simplify` changed any files, re-run Step 4's verification for the paths it touched before pushing — those edits weren't covered by the earlier pass.

## Step 6: Push + Create PR

**Fingerprint gate (runs immediately before the push, after every commit is made).** /verify Step 6 recorded its verdict against a fingerprint of the exact working tree. The gate assumes everything verified is committed: run `git status --porcelain` first and, if it is non-empty, commit the remainder (or remove stray files) and re-run /verify — a dirty tree makes the record describe code the push does not carry. Then recompute the fingerprint and require a matching record:

```bash
TOP=$(git rev-parse --show-toplevel)
EXCL=$(git rev-parse --git-path info/exclude); mkdir -p "$(dirname "$EXCL")"
grep -qx '.verify/' "$EXCL" 2>/dev/null || printf '\n.verify/\n' >> "$EXCL"
IDX=$(mktemp); cp "$(git rev-parse --git-path index)" "$IDX" 2>/dev/null || rm -f "$IDX"
GIT_INDEX_FILE="$IDX" git -C "$TOP" add -A . >/dev/null 2>&1
TREE=$(GIT_INDEX_FILE="$IDX" git -C "$TOP" write-tree); rm -f "$IDX"
[ -f "$TOP/.verify/$TREE.json" ] && echo "verify record found for $TREE" || echo "NO VERIFY RECORD for $TREE"
```

- No record and no verify verdict block in this session → /verify never ran. Back to Step 4. Absence of a record is never evidence of "not verifiable".
- Record found → push.
- No record and the Step 4 verdict was ✅/🟡 → something changed after verification (a simplify edit, a formatter, a late fix). **Do not push.** Re-run /verify on the current tree, then re-check.
- No record because Step 4 ended in a declared 🔴 not-verifiable → the existing 🔴 SHOUT path applies; push is allowed only with the `🔴 NOT VERIFIED LOCALLY` lead line.
- A `.verify/` directory is local evidence (git-excluded). Never commit it, never delete or hand-write a record to get past the gate — that is forging evidence.

```bash
git push -u origin <branch-name>
```

If no PR exists yet, create one with `gh pr create`.

### Writing the body

Invoke the **`writing-pr-descriptions`** skill and follow it exactly — it owns the format (What / Why), the 3-bullet / 2–3-sentence section caps, and the hard rules (one idea per sentence, outcome not inventory, stack etiquette, no Fixes footer, no attribution).

**If Step 4 produced UI verification screenshots, they go in the PR body — gating.** Use verify-ui's screenshots-branch recipe (private repos can't hot-link CLI-attached images) and include the measurement line under each image. A visual change shipping without its evidence in the body is the same failure as skipping verification: the proof existed and reviewers never saw it (INT-738/INT-742, Aug 2026 — three UI PRs merged screenshot-less while the evidence sat in chat).

### Creating the PR

Write the body to a temp file (Write tool or an editor) and pass it with `--body-file` — never inline the body in `--body "..."` or a heredoc, because backticks in the body get shell-evaluated and mangle it. Capture the new PR's number so Steps 7–8 can use it:

```bash
gh pr create --title "feat: Title here [TICKET-ID]" --body-file /tmp/pr-body.md
PR_NUMBER=$(gh pr view --json number -q .number)
```

## Step 7: Watch CI (fail fast)

Watch all checks and bail the instant one fails. Do NOT use `--required`: none of our repos configure branch-protection required checks, so `--required` returns "no required checks reported" and the watch silently no-ops — this failed in every repo it was tried in. Wait for checks to register first, or `--watch` hits a "no checks yet" race right after PR creation, returns non-zero, and falsely trips the fix loop:

```bash
for i in $(seq 1 12); do
  n=$(gh pr checks <PR_NUMBER> --json name --jq 'length' 2>/dev/null || echo 0)
  [ "${n:-0}" -gt 0 ] && break
  sleep 5
done
gh pr checks <PR_NUMBER> --watch --fail-fast --interval 20
```

If the repo has no CI at all (zero checks after the registration poll), say so in the ship summary and move on — don't invent a gate.

- **Exit 0** → all checks passed → go to Step 8.
- **Non-zero** → a check failed and `--fail-fast` bailed immediately. First check whether the failure is a slow non-blocking check (branch deploy, Chromatic, a review bot still running) — if so, note it and keep watching the rest. For real CI failures, fix now:
  1. Identify the failed check(s): `gh pr checks <PR_NUMBER>` (look for `fail`/`X`).
  2. Fetch the errors:
     - RSpec → use `.claude/skills/fetching-ci-errors/fetch_ci_errors` if present.
     - ESLint / TypeScript / Prettier / other → find the failed run, then view its log: `gh pr checks <PR_NUMBER> --json name,bucket,link --jq '.[]|select(.bucket=="fail").link'`, then `gh run view <run-id> --log-failed` (run-id from that link).
  3. Fix locally, run to verify, commit (new commit, NOT amend), push.
  4. Re-run the watch. **Max 3 fix rounds**, then stop and report.

A failed check may only be excluded from the gate with evidence — a log showing it's unrelated infra flake, or a re-run that passes. "Probably a flake" on an empty log is not evidence.

## Step 8: Consolidate review feedback

Code review now runs automatically in CI: the `AI code review` workflow posts a four-agent + correctness comment, and Cursor Bugbot posts its own. **Don't run `/review-pr` locally** — wait for both and act on their combined output.

1. Wait for both review bots to post — they often start only after CI is green. Poll, bounded to ~3 minutes:
   ```bash
   for i in $(seq 1 9); do
     gh pr checks <PR_NUMBER> --json name,state \
       --jq '.[]|select(.name|test("AI code review|bugbot|cursor";"i"))|"\(.name): \(.state)"'
     sleep 20
   done
   ```
   - `AI code review` check absent (author not on the allowlist) → just use Bugbot.
   - **If a review still hasn't posted after ~3 minutes → note it and proceed; don't block shipping on a review bot.**

2. Collect ALL findings from both — inline review comments and the sticky summary:
   ```bash
   gh api repos/{owner}/{repo}/pulls/<PR_NUMBER>/comments --paginate \
     --jq '.[] | select(.user.login | test("cursor|bugbot|github-actions"; "i")) | {who: .user.login, path: .path, line: .line, body: .body}'
   gh api repos/{owner}/{repo}/issues/<PR_NUMBER>/comments --paginate \
     --jq '.[] | select(.user.login | test("cursor|bugbot|github-actions"; "i")) | {who: .user.login, body: .body}'
   ```
   (The AI review posts one sticky issue-comment marked `<!-- ai-code-review -->`; Bugbot posts inline review comments.)

3. Triage every finding from both sources together:
   - Valid + worth fixing → fix locally, commit (new commit), push.
   - False positive / too noisy → skip.

4. After fixing, re-watch CI (Step 7). The CI review re-runs on the new push and refreshes its sticky comment; re-collect once more if you pushed fixes. **Cap at 2 review rounds** — don't chase every bot re-scan (diminishing returns).

## Red Flags — STOP

- About to push without a /verify verdict for this diff → back to Step 4; "tests pass" is not a verdict
- Fingerprint gate finds no record for the current tree, and Step 4 did not end in a declared 🔴 not-verifiable → the code changed after verification (or was never verified); re-verify, do not push
- "The only change since verify was a formatter / a comment" → re-verify anyway; the gate is content-based, not judgement-based
- Verification quietly skipped (missing tool, down server) → install the tool / start the server, or declare 🔴 loudly — silence is prohibited
- About to push to `main` directly → create a branch first
- About to force-push → ask user for confirmation
- No changes detected → do not create empty commits
- PR already exists → push to existing PR, don't create a new one
- 3+ CI fix iterations with no progress → stop and report

## Arguments

$ARGUMENTS — Optional: commit message, ticket ID, or notes.

### Parsing rules
- Matches a conventional commit prefix (`feat:`, `fix:`, etc.) → use as exact commit message
- Matches a ticket pattern (e.g., `INT-107`, `COR-456`) → include as ticket reference
- Empty → auto-detect commit type from the diff
- Anything else → treat as context for generating the commit message

Examples:
- `/ship` — auto-detect everything
- `/ship INT-107` — include ticket reference
- `/ship feat: Add concurrency tracking` — use this exact commit message
