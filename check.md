---
name: oss-check
description: Morning check-in for pending PRs on any open-source repo. Reads repo
  profile, checks CI/bot/stale status, takes action. Triggers on "oss check",
  "check PRs", "morning check", "PR status".
user_invocable: true
---

# OSS PR Check-In Skill

Check all pending PRs and handle everything that needs attention. Works on any repo with a profile at `./oss-pilot-data/profiles/<repo-name>.md`.

## Invocation

- `oss-check` — check all repos with profiles that have active context files
- `oss-check <repo>` — check a specific repo (e.g., `oss-check openclaw`)

## Step 0: Load Profile

Read the profile from `./oss-pilot-data/profiles/<repo>.md` (schema: see `_template.md` bundled with this skill) to get:
- `repo` — owner/repo (e.g., `some-org/some-repo`)
- `fork` — our fork (e.g., `your-username/some-repo`)
- `username` — our GitHub username
- `local_path` — local clone path

If no repo specified, scan `./oss-pilot-data/context/` for context files and infer repos from them. If no context files exist and no repo specified, ask the user which repo to check or suggest running `oss-discover` first.

## Step 1: Load Active PRs

```bash
ls ./oss-pilot-data/context/
```

Read each `pr-*.md` context file. Also check for PRs not in context files:
```bash
gh pr list -R <REPO> --author <USERNAME> --state open --json number,title
```

## Step 2: For Each PR, Check Status

```bash
# CI — count fails and pending
FAILS=$(gh pr checks <NUMBER> -R <REPO> 2>&1 | grep -c "fail")
PENDING=$(gh pr checks <NUMBER> -R <REPO> 2>&1 | grep -c "pending")

# Unanswered bot comments — bot root comments with no reply from us
gh api repos/<REPO>/pulls/<NUMBER>/comments --jq '
  [.[] | select(.user.login != "<USERNAME>" and .in_reply_to_id == null) | .id] as $bot_roots |
  [.[] | select(.user.login == "<USERNAME>" and .in_reply_to_id != null) | .in_reply_to_id] as $replied |
  [$bot_roots[] | select(. as $r | $replied | index($r) | not)] | length
'

# Human reviews (exclude bots and self)
gh pr view <NUMBER> -R <REPO> --json reviews --jq '[.reviews[] | select(.author.login != "<USERNAME>" and (.author.login | test("bot"; "i") | not))] | length'

# PR state
gh pr view <NUMBER> -R <REPO> --json state --jq .state

# Stale check — ping timestamp with no human response after it
gh api repos/<REPO>/issues/<NUMBER>/comments --jq '
  ([.[] | select(.user.login == "<USERNAME>" and (.body | test("^@")))] | last.created_at // empty) as $ping |
  if $ping == "" then "not_pinged"
  else
    ([.[] | select(.created_at > $ping and .user.login != "<USERNAME>" and (.user.login | test("bot"; "i") | not))] | length) as $responses |
    if $responses > 0 then "responded"
    else $ping
    end
  end
'
```

**CI rules:** ✅ = 0 fail + 0 pending. ❌ = any fail. ⏳ = 0 fail + some pending. 🔒 = all jobs skipping (fork PR — needs maintainer to trigger CI, this is normal, not a failure).

**Bot rules:** ✅ = 0 unanswered. ❌ = any unanswered → auto-respond before reporting.

**Cross-reference rules:**
- Filter bot noise before acting. Only count as genuine merge signal if: (a) author is a real user who hit the bug, or (b) maintainer opened/triaged it
- Still add `Closes #XXXX` for bot-opened duplicates (same bug), but don't inflate urgency claims
- Report honestly: e.g., "+2 cross-refs (1 real user, 1 bot duplicate)"

## Step 3: Triage and Act

| State | Action |
|---|---|
| **MERGED** | Celebrate. Run retrospective (Step 3.5). Archive context file. |
| **CLOSED** | Check why. Run retrospective (Step 3.5). Archive context file. |
| **New human review** | Report to user — needs human judgment. After addressing feedback, run `/oss-pr review` to verify quality before pushing. |
| **Unanswered bot comments** | Auto-respond using context file for approach details |
| **CI fail (our code)** | Read failure, fix, push |
| **CI fail — needs triage** | Not all red CI means "wait." Triage the failures into 3 buckets (see CI Triage below). |
| **CI green + bots answered + no human review + not pinged** | Find who to ping: CODEOWNERS first, else `git log --since="30 days ago" --format="%an" -- <changed-files>` top result |
| **Stale (48h+ since ping, no response)** | Enrich the ping: update triage comment with area-specific pass list, add new linked issues if found. Makes the PR look active and gives maintainer fresh reason to look. |
| **Stale (>3 days since ping, no response)** | Re-ping same person with updated context, or try a different maintainer via `git log` |
| **Everything done, waiting** | Report "waiting for review" |

### CI Triage (replacing binary green/red)

When CI has failures, don't just report ❌ — **classify each failure**:

```bash
# Get our PR's changed files
OUR_FILES=$(gh pr diff <NUMBER> -R <REPO> --name-only | tr '\n' '|' | sed 's/|$//')

# For each failed job, check if the failure is in our files or upstream
gh pr checks <NUMBER> -R <REPO> 2>&1 | grep "fail" | while read line; do
  echo "$line"
done
```

**Three buckets:**
1. **Our code failing** — the failed test/check is in a file we changed → fix and push
2. **Upstream failing, our area passes** — failures are in unrelated files, AND checks covering our area (e.g., `extension-fast (telegram)` for telegram changes) all pass → **PR is ready for review.** Post/update triage comment listing our passing checks.
3. **Upstream failing, can't tell** — failures are in shared infra (build, lint, type-check) that cover everything → check if main has same failure. If yes, post triage comment.

**Report CI with nuance:**
- ✅ = all pass
- ⏳ = still running
- 🟡 = upstream failures only, our area passes → **ready for review with triage**
- ❌ = our code failing, or can't determine

**Key insight:** In repos with flaky CI (>20% failure rate on main), waiting for fully green CI is a losing strategy. The goal is giving the maintainer enough evidence that our code is clean — not achieving a green badge that may never come.

To verify main CI, check the actual CI workflow — NOT `gh run list` which includes non-CI workflows (Labeler etc.) that are always green.

**Anomaly-only checks** — surface in Action Taken only when triggered:
- **Merge conflict**: Only act on `CONFLICTING` (not `UNKNOWN`). Rebase if main CI green.
- **Competing PR**: `gh pr list -R <REPO> --search "<issue>" --state open` — alert if non-us PRs found for same issue.
- **New cross-references**: `gh api repos/<REPO>/issues/<NUMBER>/timeline --jq '[.[] | select(.event == "cross-referenced")] | length'` — only report if new ones found since last check. Filter bot noise per cross-reference rules above.

## Step 3.5: Retrospective (on MERGED/CLOSED PRs)

For each PR that was MERGED or CLOSED this check, before removing the context file:

1. **Collect data** — run ALL of these, not just issue comments:
   ```bash
   # Timeline
   gh pr view <NUMBER> -R <REPO> --json createdAt,mergedAt,closedAt,mergedBy --jq '{created: .createdAt[:16], merged: .mergedAt[:16], closed: .closedAt[:16], mergedBy: .mergedBy.login}'

   # Human reviews (what did the reviewer actually say?)
   gh api repos/<REPO>/pulls/<NUMBER>/reviews --jq '.[] | select(.state == "APPROVED" or .state == "CHANGES_REQUESTED") | {author: .author.login, state: .state, body: .body[:300]}'

   # Bot review scores (e.g., Greptile confidence)
   gh api repos/<REPO>/issues/<NUMBER>/comments --jq '.[] | select(.user.login | test("bot"; "i")) | {user: .user.login, body: .body[:200]}'

   # Inline review comments (what specific code did bots/reviewers flag?)
   gh api repos/<REPO>/pulls/<NUMBER>/comments --jq '[.[] | .user.login] | group_by(.) | map({user: .[0], count: length})'

   # Our ping → first response time
   gh api repos/<REPO>/issues/<NUMBER>/comments --jq '[.[] | {user: .user.login, date: .created_at[:16]}] | .[-5:]'
   ```

2. **Calculate timeline**: opened → pinged → first human response → merged/closed. How many days at each stage?

3. **Write retrospective into the context file** — append an `## Outcome` section:
   ```markdown
   ## Outcome
   - **Result**: merged / closed (reason)
   - **Timeline**: opened [date] → pinged [date] → reviewed [date] → merged [date] ([N] days total)
   - **Reviewed by**: @who — what they specifically cared about (quote their review comment)
   - **Bot scores**: Greptile [N]/5, Codex [N] rounds ([summary of key concerns])
   - **What worked**: (fast merge? clean review? good approach? what specifically made this succeed/fail?)
   - **What surprised us**: (unexpected rejection? bot concern we missed? maintainer preference we didn't know?)
   - **Lesson**: (one actionable takeaway — not "clean merge, no lessons" unless truly nothing was learned)
   ```
3. **Route the lesson** (if any):
   - **Repo-specific** (maintainer name, bot quirk, label, convention) → also append to profile's Lessons Learned section
   - **Universal** (methodology improvement, new technique) → flag to user: "This lesson may be universal — consider updating oss-* skill"
4. **Then** archive context file: `mv pr-<REPO>-<N>.md` → `./oss-pilot-data/context/_archived/`

**Why write retrospective into the context file**: The archived file becomes a complete record — approach → decisions → outcome → lesson. When oss-discover encounters a similar issue in the future, checking `_archived/` reveals not just "we tried this" but "we tried this and here's what happened."

### Maintenance Check (after retrospective)

After archiving, check if accumulated knowledge needs pruning. Skip if no PRs were merged/closed this run.

**Profile Lessons Learned (trigger: >15 entries)**:
1. Read all entries in Lessons Learned section
2. For each entry: has this lesson been absorbed into a more structured section?
   - Encoded in Architecture Patterns → remove from Lessons Learned
   - Encoded in Maintainer Styles → remove from Lessons Learned
   - Encoded in Bot Behavior → remove from Lessons Learned
3. For remaining entries: is the lesson still accurate?
   - Stale (references a bot that no longer exists, a maintainer who left, a process that changed) → remove or update
4. Report: "Pruned N lessons (M absorbed, K stale). Lessons Learned now has X entries."

**Archived context files (trigger: >30 files for this repo)**:
1. Count files matching `pr-<REPO>-*.md` in `_archived/`
2. If >30, identify cleanup candidates:
   - Older than 6 months AND Outcome is "clean merge, no new lessons" → delete
   - Older than 6 months AND Outcome has lessons → keep (valuable reference)
   - Any age AND Outcome is "closed for technical rejection" → keep (prevents re-attempts)
3. Report: "Pruned N archived context files. X remain."

**Profile total size (trigger: >200 lines)**:
1. `wc -l` the profile
2. If >200: flag to user which sections are largest, suggest review
3. Do NOT auto-prune structured sections (Architecture Patterns, Maintainer Styles) — these require human judgment

## Step 4: Report

```
╔═══════════════════════════════════════════════════════════════╗
║              OSS Check-In: <repo>                             ║
╠═══════════════════════════════════════════════════════════════╣

 PR       Status    CI    Bot     Action Taken
 #123     open      ✅    ✅      Waiting (pinged 25h ago)
 #456     open      ❌    ✅      Upstream fail, main recovered → rebase
 #789     merged    🎉    -       Archived context file

╚═══════════════════════════════════════════════════════════════╝
```

## Step 5: Update Context Files

For each PR where action was taken, update the context file Decisions section.

## Context Files

Live at `./oss-pilot-data/context/pr-<REPO-SHORT>-<NUMBER>.md` (e.g., `pr-openclaw-52644.md`). See `/oss-auto` for format.

## Error Handling

- Missing context file for open PR → create from GitHub data
- Merged/closed PR → archive context file to `./oss-pilot-data/context/_archived/`
- GitHub API failure → report error, skip that PR
