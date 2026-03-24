# OSS Pilot

I'm a PM, not an engineer. I don't write code for a living — I build products.

But I wanted to contribute to open-source projects I use every day. So I built this system: a set of OpenClaw skills that handle the entire contribution lifecycle — from finding the right issue to getting a PR merged. I used it to land PRs on [OpenClaw](https://github.com/openclaw/openclaw) (329K+ stars), [Cal.com](https://github.com/calcom/cal.com), and other repos.

Every feature in this system exists because I hit a real wall. I wasted hours on issues that were already fixed. I picked issues with 6 competing PRs. I tried to fix core infrastructure on my first contribution and got nowhere. Each failure became a rule in the system, so the next attempt was better.

If you want to contribute to open source but don't know where to start, or if you're tired of rejected PRs — this is for you.

Built by [@Cypherm](https://github.com/Cypherm).

---

## What It Does

```
discover ──→ auto ──→ pr ──→ check ──→ retrospective
   ↑                   ↑       │              │
   │                   └───────┘              ▼
   │              (review loop)          profile grows
   │                                    (lessons, patterns)
   └──── learns from history ───────────────┘
```

| Skill | What it does | When to use |
|-------|-------------|-------------|
| **discover** | Find high-value issues with highest merge probability | "What should I work on?" |
| **auto** | Issue → implemented, reviewed, bot-responded, maintainer-pinged PR | "Fix this issue end-to-end" |
| **pr** | Quality gate: root cause analysis, description, bot strategy | "Is this PR ready?" |
| **check** | Monitor CI, bot comments, stale status, take action | "What needs attention today?" |

## Install

```bash
# Manual install (recommended)
git clone https://github.com/Cypherm/oss-pilot.git ~/.claude/skills/oss-pilot

# Via ClawHub (coming soon)
# openclaw skills install oss-pilot
```

## Quick Start

```bash
# 0. Make sure gh CLI is authenticated
gh auth status

# 1. Find something to work on
> oss discover https://github.com/some-org/some-repo
# (First run auto-creates a profile and forks the repo for you)

# 2. Fix an issue end-to-end
> oss auto some-repo #12345

# 3. Check your pending PRs next morning
> oss check
```

On first run, the system creates a **profile** for each repo — a living document that captures build commands, maintainer styles, bot behavior, and lessons learned. See `_template.md` for the schema and `example.md` for what a mature profile looks like.

## How It Works

### The Learning Loop

Every PR you open goes through this cycle:

1. **Discover** finds issues by scanning 8 sources (bounty labels → bugs → CI failures → codebase cleanup), checking repo openness, issue velocity, and competition
2. **Auto** implements the fix: reads code, traces root cause, writes tests, opens PR, responds to bots, pings maintainer
3. **PR** validates quality: root cause at the right layer? Description matches diff? Bot comments all answered?
4. **Check** monitors daily: CI status, new reviews, stale pings. When a PR is merged or closed, runs a **retrospective** — writes the outcome and lessons back to the profile
5. Next time **Discover** runs, it uses the profile to make better choices and checks archived PRs to avoid past mistakes

### What Makes It Different

- **Profile system**: Each repo builds institutional knowledge over time — maintainer preferences, bot behaviors, architecture patterns. Your 10th PR is informed by lessons from your 1st.
- **Competition check**: Two-level (issue + code) to avoid wasting time on crowded issues.
- **Repo openness check**: Measures external contributor merge rate before you invest hours.
- **Velocity check**: Detects fast-moving repos where issues get claimed in hours.
- **Version/comment intelligence**: Reads issue comments and release notes to avoid working on already-fixed bugs.
- **Maintenance**: Auto-prunes stale lessons and old context files to keep signal-to-noise ratio high.

## File Structure

```
oss-pilot/
├── SKILL.md          # Entry point — routing + quick start guide
├── discover.md       # Issue discovery skill (8 sources + verification)
├── auto.md           # End-to-end PR automation (orchestrator)
├── pr.md             # PR quality validation (root cause + description)
├── check.md          # Daily PR monitoring + retrospective + maintenance
├── _template.md      # Profile template for new repos
└── example.md        # Example mature profile (anonymized real data)
```

## Data Directories

The system creates and manages:

```
~/.claude/oss-profiles/        # One profile per repo (grows with experience)
~/.claude/oss-auto/            # Active PR context files
~/.claude/oss-auto/_archived/  # Completed PRs (approach + decisions + outcome)
```

## Prerequisites

- [OpenClaw](https://openclaw.ai) installed
- `gh` CLI authenticated (`gh auth status`)
- A GitHub account

## Language Support

Tested on TypeScript, Python, and Go repos. Source 8 (codebase scan) has language-specific patterns for:
- **Node.js/TypeScript**: Duplicate i18n keys, missing alt text
- **Python**: Unused imports (ruff), type annotation gaps (mypy)
- **Go**: Unformatted files (gofmt), static analysis (go vet)
- **Rust**: Clippy warnings

## Built From Real Contributions

This system was built and refined through actual open-source contributions across multiple repos (cal.com, ollama, dify, and others). Every feature exists because we hit a real problem:

- **Velocity check** → Added after Dify issues got claimed within hours
- **Version check** → Added after investigating an issue already fixed in a newer release
- **Comment intelligence** → Added after missing a comment that said "likely fixed in next version"
- **First PR rules** → Added after attempting a core infrastructure fix on first contribution
- **Repo openness check** → Added after investing time on a repo that merges <10% external PRs

## License

[MIT](LICENSE)
