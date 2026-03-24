---
name: oss-discover
description: Find high-value PR opportunities on any open-source repo. Triggers on
  "oss discover", "find issues", "what should I work on", "next PR".
user_invocable: true
---

# OSS PR Opportunity Discovery

Surface the highest-value, most-likely-to-merge PR opportunities. Works on any repo with a profile.

## Invocation

- `oss-discover <repo>` — scan a specific repo
- `oss-discover` — scan all repos with profiles

## Step 0: Load Profile

Read `./oss-pilot-data/profiles/<repo>.md` for repo, fork, username, local_path. Profile schema: see `_template.md` (bundled) for expected sections.

**Cold start**: If the profile doesn't exist, this is a new repo. Copy `_template.md` to `./oss-pilot-data/profiles/<repo>.md`, fill in the 4 required fields (repo, fork, username, local_path), and ask the user to confirm before proceeding. Also ensure `./oss-pilot-data/context/` and `./oss-pilot-data/context/_archived/` directories exist.

## Step 0.5: Repo Openness Check

Before spending time scanning issues, check if this repo actually merges external contributions:
```bash
# Count recent merged PRs from external contributors vs maintainers
MAINTAINERS=$(gh pr list -R <REPO> --state merged --limit 30 --json mergedBy --jq '[.[].mergedBy.login] | unique | join(",")')
TOTAL=$(gh pr list -R <REPO> --state merged --limit 30 --json number --jq 'length')
EXTERNAL=$(gh pr list -R <REPO> --state merged --limit 30 --json author,mergedBy --jq "[.[] | select(.mergedBy.login as \$m | .author.login != \$m)] | length")
echo "Last 30 merged: $EXTERNAL external out of $TOTAL total"
echo "Active maintainers: $MAINTAINERS"
```

**Interpret the ratio:**
- **>30% external** → open repo, good for contributions. Proceed normally.
- **10-30% external** → selective repo, PRs need to be high quality. Proceed but be picky.
- **<10% external** → closed repo, maintainers do most work themselves. **Warn the user**: "This repo merges very few external contributions. Consider whether the time investment is worth it, or pick a more open repo."

This is NOT a hard blocker — some closed repos still merge great first PRs. But the user should know the odds before investing hours.

## Step 0.6: Velocity Check

Check how fast issues get claimed in this repo:
```bash
# Sample 5 recent good-first-issue PRs — how quickly did they get their first competing PR?
for issue in $(gh issue list -R <REPO> --label "good first issue" --state open --limit 5 --json number --jq '.[].number'); do
  prs=$(gh pr list -R <REPO> --search "$issue" --state open --json number --jq 'length')
  echo "#$issue: $prs competing PRs"
done
```

**Interpret the competition rate:**
- **<50% of issues have competing PRs** → LOW velocity. Normal flow (Source 1-8).
- **50-80% have competing PRs** → MEDIUM velocity. Prioritize Source 8 (codebase scan) and stale-PR reclaimable, but still check Source 1-2.
- **>80% have competing PRs** → HIGH velocity. **Skip Source 1-7** — issues get claimed within hours, scanning them is a waste of time. Go directly to:
  1. **Source 8** — codebase scan for safe cleanups (no issue needed, no competition)
  2. **Stale PR reclaimable** — find issues where competing PR is stale (>1 month no update) and redo the work
  3. **Ecosystem repos** — check if the project has plugin/extension/docs repos (often mentioned in CONTRIBUTING.md) that are less competitive
  4. **Suggest real-time watch** — tell the user: "This repo moves too fast for async discovery. Set up GitHub notifications for new `good first issue` labels to claim early."

## Step 1: Gather Exclusion List

```bash
# Issues we already have PRs for
gh pr list -R <REPO> --author <USERNAME> --state open --json body --jq '.[].body' | grep -oE 'Closes #[0-9]+' | grep -oE '[0-9]+' | sort -u

# Files our open PRs touch (for code-level exclusion)
for pr in $(gh pr list -R <REPO> --author <USERNAME> --state open --json number --jq '.[].number'); do
  gh pr diff $pr -R <REPO> --name-only
done | sort -u

# Issues we previously attempted (check archived context files)
grep -l "repo: <REPO>" ./oss-pilot-data/context/_archived/pr-*.md 2>/dev/null | while read f; do
  grep -oE 'issue: [0-9]+' "$f" | grep -oE '[0-9]+'
done | sort -u
```

For each previously attempted issue found in `_archived/`, read the `## Outcome` section. If the PR was **closed** (not merged), check why:
- **Closed for hygiene** (dirty, stale, too-many-prs) → issue may still be valid, don't auto-exclude but note the prior attempt
- **Closed for technical rejection** (wrong approach, won't fix) → exclude unless the Outcome suggests a different approach could work
- **Merged** → exclude (already done)

**Age relevance**: For archived context files older than 6 months, treat exclusions as soft — maintainers rotate, codebases evolve, and a previously rejected approach may now be welcome. Note the prior attempt but don't auto-exclude.

## Step 2: Search Sources

**IMPORTANT: Search order is by merge signal strength, not source number. Start from Source 1 (strongest signal) and work down. If Source 1-2 yield enough good candidates, don't waste time on weaker sources.**

**Source 1 — Maintainer-signal labels (HIGHEST PRIORITY):**
First, scan ALL labels to find high-signal ones — every repo uses different names:
```bash
gh label list -R <REPO> --json name --jq '[.[].name] | sort | .[:50]'
```
Look for labels that indicate **maintainer intent to accept contributions**:
- Bounty / reward labels (e.g., `💎 Bounty`, `bounty`, `$$`)
- Quick win / low-hanging fruit (e.g., `⚡ Quick Wins`, `easy fix`)
- Help wanted / good first issue (e.g., `🙋🏻‍♂️help wanted`, `✅ good first issue`)
- Regression labels (e.g., `📉 regressing`, `regression`) — maintainers want these fixed urgently
- Priority labels (e.g., `High priority`, `Urgent`)

Then search issues with each high-signal label found:
```bash
gh issue list -R <REPO> --label "<HIGH-SIGNAL-LABEL>" --state open --json number,title,createdAt,author --limit 10 --jq '.[] | {number, title: .title[:70], author: .author.login, created: .createdAt[:10]}'
```
**These candidates should be evaluated FIRST** — a bounty or quick-win issue has 3-5x the merge probability of an unlabeled bug.

**Source 2 — Recent bugs:**
```bash
gh issue list -R <REPO> --label "bug" --state open --json number,title,createdAt,author --limit 20 --jq '.[] | {number, title: .title[:70], author: .author.login, created: .createdAt[:10]}'
```
**Warning**: Do NOT use date string filtering with `--jq 'select(.createdAt > "YYYY-MM-DD")'` — this returns empty results due to timezone/format mismatches between local time and GitHub's UTC timestamps. Instead, use `--limit 30` and filter manually by recency.

Prefer bugs filed by maintainers or users with reproduction steps. Unlabeled bugs from unknown authors are the weakest signal.

**Source 3 — CI failures on default branch:**
```bash
# Auto-detect default branch (main, master, develop, etc.)
DEFAULT_BRANCH=$(gh repo view <REPO> --json defaultBranchRef --jq .defaultBranchRef.name)

# Check recent CI runs
gh run list -R <REPO> --branch $DEFAULT_BRANCH --limit 3 --json workflowName,conclusion,createdAt --jq '.[] | {conclusion, workflow: .workflowName, date: .createdAt[:16]}'
```

**Source 4 — Reclaimable closed PRs** (closed for hygiene, not rejection):
```bash
gh pr list -R <REPO> --state closed --label "good first issue" --limit 10
```

**Source 5 — Merged PR gaps ("What I Did NOT Verify"):**
Recently merged PRs often have explicit gaps the author acknowledged. These are the safest follow-up opportunities — the work is scoped, the maintainer already approved the direction.
```bash
gh pr list -R <REPO> --state merged --limit 10 --json number,title,body --jq '.[] | select(.body | test("not verif|TODO|follow.up|happy to revisit"; "i")) | {number, title: .title[:70]}'
```
Check for: unverified scenarios, deferred edge cases, declined bot suggestions marked "happy to revisit."

**Yield warning for "maintainer follow-up hints"**: In theory, maintainer comments like "this should be a follow-up" are goldmine opportunities. In practice, many repos' maintainers don't leave textual follow-up hints — they merge cleanly or push fixes themselves. If you scan maintainer comments for follow-up language, limit to 10 most recent merged PRs and move on quickly if nothing found. The structured "What I Did NOT Verify" approach (above) is much higher yield.

**Source 6 — Scoped modules** (extensions, plugins, packages):
Repos with modular structure have isolated areas with lower blast radius and faster review.
```bash
# Step 1: Discover module structure from repo directories (if local clone exists)
ls <LOCAL_PATH>/extensions/ <LOCAL_PATH>/plugins/ <LOCAL_PATH>/packages/ <LOCAL_PATH>/apps/ 2>/dev/null | head -20

# Step 2: Discover module labels (every repo uses different labels — don't hardcode)
gh label list -R <REPO> --json name --jq '[.[].name] | .[:30]'
# Look for labels that name specific components (e.g., "auth", "cli", "edge functions", "extension: telegram", "app")
# Then search issues with those labels
```
Don't hardcode label patterns — every repo is different. Scan the label list first, then pick relevant ones.

**Source 7 — Area merge rate signal** (tiebreaker only):
Areas with recent merges = maintainers paying attention = faster review.
```bash
gh pr list -R <REPO> --state merged --limit 30 --json title --jq '[.[].title[:20]] | group_by(.) | map({area: .[0], count: length}) | sort_by(.count) | reverse | .[:5]'
```
Hot area (+1 confidence) vs cold area (-1 confidence). Use as tiebreaker, not primary signal.

**Nuanced usage**: Area merge rate matters because maintainer attention shifts week by week. Check who specifically is merging in the area, not just the count:
```bash
gh pr list -R <REPO> --state merged --limit 20 --json mergedBy,title --jq '.[] | {merger: .mergedBy.login, title: .title[:50]}'
```
A maintainer actively reviewing one area this week will review yours fast — but if they shift focus next week, your PR may sit.

**Source 8 — Codebase scan for safe cleanups** (especially good for first PRs):
If local clone exists, scan for known-safe cleanup patterns that don't need an issue. **Detect the project language first**, then apply appropriate patterns:

**Auto-detect language:**
```bash
# Detect primary language from repo files
if [ -f "<LOCAL_PATH>/package.json" ]; then LANG="node"
elif [ -f "<LOCAL_PATH>/go.mod" ]; then LANG="go"
elif [ -f "<LOCAL_PATH>/Cargo.toml" ]; then LANG="rust"
elif [ -f "<LOCAL_PATH>/pyproject.toml" ] || [ -f "<LOCAL_PATH>/setup.py" ]; then LANG="python"
else LANG="unknown"; fi
echo "Detected: $LANG"
```

**Node.js / TypeScript:**
```bash
# Duplicate translation keys (JSON i18n files)
find <LOCAL_PATH> -name "*.json" -path "*/i18n/*" -o -name "*.json" -path "*/locales/*" | head -3 | while read f; do
  python3 -c "import json,re; lines=open('$f').readlines(); seen={}; dupes=[(i+1,m.group(1)) for i,l in enumerate(lines) if (m:=re.match(r'\s*\"([^\"]+)\"\s*:\s*(.*)',l)) and m.group(1) in seen and seen[m.group(1)]==m.group(2).rstrip().rstrip(',').strip() or (seen.setdefault(m.group(1),m.group(2).rstrip().rstrip(',').strip()) and False)]; [print(f'$f L{ln}: dup \"{k}\"') for ln,k in dupes]" 2>/dev/null
done
# Images missing alt text (a11y)
grep -rn '<img' --include="*.tsx" --include="*.jsx" <LOCAL_PATH> 2>/dev/null | grep -v 'alt=' | grep -v node_modules | head -5
```

**Python:**
```bash
# Unused imports (ruff check)
cd <LOCAL_PATH> && ruff check --select F401 --statistics . 2>/dev/null | head -5
# Type annotation gaps (mypy)
cd <LOCAL_PATH> && mypy --ignore-missing-imports --no-error-summary . 2>/dev/null | grep "error:" | head -5
```

**Go:**
```bash
# Unformatted files
cd <LOCAL_PATH> && gofmt -l . 2>/dev/null | head -5
# Static analysis
cd <LOCAL_PATH> && go vet ./... 2>/dev/null | head -5
```

**Rust:**
```bash
# Clippy warnings
cd <LOCAL_PATH> && cargo clippy --message-format short 2>/dev/null | grep "warning" | head -5
```

These produce XS PRs (1 file, <10 lines) that match patterns recently merged by maintainers. No issue needed — the PR itself is the justification.

## Step 3: Filter

- Exclude issues from exclusion list (Step 1)
- Skip issues with >5 comments
- Skip issues with linked PRs: check timeline for cross-references

## Step 4: Competition Check (HARD BLOCKER — mandatory for every candidate)

For each candidate, BOTH checks must be run before reporting:

1. **Issue-level**: `gh pr list -R <REPO> --search "<issue-number>" --state open`
2. **Code-level**: Identify root cause file/function, then search with context keyword + function name to avoid false positives:
   ```bash
   # Good: "Anthropic sanitizeToolCallIds" — narrow, relevant results
   # Bad:  "sanitizeToolCallIds" alone — returns 29 unrelated PRs about other providers
   gh pr list -R <REPO> --search "<context-keyword> <function-name>" --state open --json number,title,author --jq '[.[] | select(.author.login != "<USERNAME>")]'
   ```
   Skim returned titles/descriptions to filter noise. Only count PRs **actually targeting the same fix**.
3. Also check against our own PRs' files (from Step 1) — if candidate touches same files, skip.
4. If either level finds >2 competing PRs actually targeting the same fix → mark as ❌.

**This is NOT optional.** A candidate falsely reported as "no competition" is worse than no recommendation.

## Step 5: Code-Level Verification

**This is NOT optional.** An open issue does NOT mean the bug still exists. Maintainers push fixes without closing issues. Recommending a "fixed" issue wastes everyone's time.

For each top candidate (top 3-5), verify **the specific pattern described in the issue** still exists on main — not just that related keywords appear somewhere:
```bash
# Step A: Check git log for fix commits mentioning the issue or keyword
git log upstream/main --oneline --grep="<keyword>" | head -5
gh pr list -R <REPO> --search "<issue-number>" --state merged --json number,title --jq '.[].title' | head -3

# Step B: Verify the EXACT bug pattern still exists in code
# BAD: "server_default exists somewhere" → too broad, false positive
# GOOD: "server_default=sa.text('uuid_generate_v4()') exists in model files" → specific pattern from issue
gh search code "<exact-pattern-from-issue>" --repo <REPO> --json path --jq '.[].path' | head -5

# Step C: If no local clone, read the suspected file via API
gh api repos/<REPO>/contents/<file-path> --jq '.content' | base64 -d | grep "<pattern>" | head -5
```

**Common false positive**: A keyword exists in the codebase but the specific bug was already fixed. Always match the **exact pattern** described in the issue, not just related keywords.

Also check for AI-attempted labels (`codex`, `devin`, etc.) — if a previous AI attempt was closed, read the closure reason. The issue is valid but the approach may need to be different.

### Step 5B: Version Check

If the issue reports a specific version, check if a newer release exists that might fix it:
```bash
# What version does the issue report?
# Compare against latest release
gh release view -R <REPO> --json tagName,publishedAt --jq '{tag: .tagName, published: .publishedAt[:10]}'

# Scan release notes between reported version and latest for related fixes
gh release view -R <REPO> --json body --jq '.body' | grep -i "<keyword>"
```
If a newer release mentions fixes in the same area (packaging, provider registration, the specific component), **flag the candidate**: "Issue may be version-specific — check if still reproduces on latest release before investing time."

### Step 5C: Comment Intelligence

**Read the issue comments before committing to work.** Existing comments often contain:
- Root cause analysis that's already done (saves time)
- Signals that the issue is resolved ("fixed in latest", "workaround found")
- Contradictions in the root cause analysis (red flag for scope)
- Maintainer responses ("won't fix", "by design", "duplicate of #X")
```bash
gh api repos/<REPO>/issues/<NUMBER>/comments --jq '.[] | {user: .user.login, body: .body[:200]}' | head -10
```
**Red flags to watch for:**
- Comment says "likely fixed in version X" → verify before working
- Root cause analysis contradicts itself → scope risk, needs deeper investigation
- Maintainer says "by design" or "won't fix" → skip
- Multiple people offering different root causes → unclear problem, risky

### Step 5D: Fix Feasibility

For each top candidate, quickly assess if the fix is actually doable:
- **Root cause clear?** Is there a consistent, non-contradictory explanation of why the bug happens?
- **Scope bounded?** Can you identify which files/functions need to change without reading the entire codebase?
- **Testable?** Can you verify the fix without special infrastructure (running servers, specific hardware, paid APIs)?
- **Safe?** Does it touch security-sensitive code (auth, exec approval, permissions)?

If any answer is "no" and this is a first-time contribution, **downgrade the candidate** — save it for after you have trust and codebase familiarity.

Drop any candidate where the fix already landed. Mark remaining candidates as **verified**.

## Step 6: Present Candidates

Answer 3 yes/no questions per candidate:

| Question | Signal |
|---|---|
| **Maintainer cares?** | Has high-signal label (bounty, quick win, help wanted, regression), filed by maintainer, or has real (non-bot) user reports |
| **No competition?** | Zero PRs for same issue AND same code area |
| **Small scope?** | <200 lines, <5 files |

```
 #  Issue    Maintainer?  No comp?  Small?  Title
 1  #XXXXX   ✅ reason     ✅        ✅      description
 2  #YYYYY   ✅ reason     ❌ 3 PRs  ✅      description
```

For each include: suggested approach (1 sentence), risk factors.

## First PR on a New Repo

If this is our first contribution to a repo (no merged PRs yet), **prioritize low-risk scope over strong signal**:
- Prefer peripheral code (tests, i18n, a11y, docs tooling, CLI) over core infrastructure (auth, billing, availability, data pipeline)
- A merged small fix builds trust; a rejected core fix builds nothing
- Save high-signal core issues for after we have 1-2 merged PRs and understand the codebase

## What NOT to Work On

- Speculative features without an issue
- Large refactors (>10 files)
- Areas with 5+ competing PRs
- Documentation-only PRs without evidence
- Dependency bumps without security advisory
- **First PR**: core infrastructure changes (availability engine, auth, billing, data pipeline)
