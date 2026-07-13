# Vercel Auto-Deploy Pipeline (Cron + Claude Loop)

## Feature Specification for Porting

This document describes the two-tier CI/CD pipeline that keeps a **Vercel Hobby** project deployed continuously even when contributors (non-owner GitHub accounts) raise PRs. Vercel Hobby rejects deploys when the HEAD commit author is not the project owner — this system wraps every merge in an owner-authored release commit to unblock the pipeline automatically.

---

## 1. Overview

Two tiers work together to cover 100% of cases:

- **Tier 1 — OS cron** runs every hour with no Claude session needed. Merges safe PRs, bumps semver, writes CHANGELOG, creates an owner-authored release commit, waits for Vercel to be Ready, then emails + Slacks the dev team.
- **Tier 2 — `/loop` watcher** (Claude Code session) polls every 15 min. A cheap Sonnet watcher dispatches a subagent **only** when a PR appears, and the subagent does a full security + breaking-change review before merging.

The cron is the **autonomous floor** (survives Mac sleep, no terminal needed); the loop is the **smart tier** (careful review, session-only).

```
Contributor PR / commit lands on GitHub
        │
        ▼
  ┌─────────────────────────────────────────────────────────────┐
  │  Tier 1 — OS cron (scripts/auto-merge.py, runs hourly)     │
  │  git pull → gh pr list → classify blockers                 │
  │  Squash-merge safe PRs → bump semver (deterministic) →     │
  │  write CHANGELOG + /changelog page → owner-authored PR →   │
  │  squash-merge → poll vercel ls until Ready →               │
  │  notify-deploy.mts (email + Slack)                         │
  │                                                             │
  │  HOLDS (skips + notifies) if PR is:                        │
  │    • Draft                                                  │
  │    • Touches drizzle/ (DB migration)                        │
  │    • Has merge conflicts                                    │
  └─────────────────────────────────────────────────────────────┘
        │
        ▼ (held PRs sit until the loop or human acts)
  ┌─────────────────────────────────────────────────────────────┐
  │  Tier 2 — /loop watcher (claude --model sonnet session)    │
  │  Polls every 15 min. Cheap Sonnet watcher — pure bash on   │
  │  idle cycles. On PR detected: dispatches a subagent that   │
  │  does full review, conflict resolution in isolated          │
  │  worktree, npm run build gate, owner-authored release,     │
  │  Vercel wait, notify-deploy.mts.                           │
  └─────────────────────────────────────────────────────────────┘
        │
        ▼
  Vercel production deploy is Ready
  email + Slack → dev team notified
```

---

## 2. Why Owner-Authored Commits Matter

Vercel Hobby blocks deploys when the HEAD commit author is not the project owner registered on Vercel. Every merge from a contributor (GitHub login ≠ owner) triggers a blocked deploy.

The fix: after every contributor merge, immediately create and merge a new **owner-authored** commit (a changelog/version-bump PR). This becomes the new HEAD on `main`, unblocking Vercel.

Both tiers do this:
1. Squash-merge the contributor PR (HEAD is still contributor-authored → Vercel blocked)
2. Create a `chore: document N contributor commits (vX.Y.Z)` branch → commit as owner → open PR → squash-merge → HEAD is now owner-authored → Vercel deploys

---

## 3. Tier 1 — Hourly OS Cron

### 3.1 What It Does (In Order)

| Step | Action |
|---|---|
| 1 | `git checkout main && git pull` |
| 2 | `gh pr list` — find open PRs by non-owner contributors |
| 3 | `classify_blockers()` — inspect each PR for draft / migration / conflict |
| 4 | Squash-merge safe PRs; hold blocked ones and send `notify-blocked.mts` (once per head SHA) |
| 5 | Walk `git log` to find non-owner commits not yet in CHANGELOG.md |
| 6 | Determine semver bump by conventional-commit rules (deterministic, no LLM) |
| 7 | Generate changelog bullets (baseline = cleaned commit subjects; upgraded to diff-based prose when LM Studio is running) |
| 8 | `npm version patch/minor/major --no-git-tag-version` |
| 9 | Update `CHANGELOG.md` and `/changelog` page |
| 10 | Create branch → commit → `gh pr create` → `gh pr merge --squash` → `git pull` |
| 11 | Poll `vercel ls` every 15 s until Production is **Ready** (5-min timeout) |
| 12 | `npx tsx scripts/notify-deploy.mts` — email + Slack the dev team |
| 13 | Any crash → `alert_cron_failure()` posts to `SLACK_ERROR_WEBHOOK_URL` before exit |

### 3.2 Semver Bump Rule

Deterministic — no LLM involved. Regex over conventional-commit subjects:

| Condition | Bump |
|---|---|
| `BREAKING CHANGE` in any subject, or any `type!:` prefix | **major** |
| Any `feat:` / `feat(...):` prefix | **minor** |
| Everything else | **patch** |

### 3.3 Changelog Generation

Two quality levels that degrade independently per commit:

| Mode | When active | How it works |
|---|---|---|
| **Baseline** | Always | Strips conventional-commit prefix, sentence-cases raw subject |
| **ornith-35b upgrade** | When LM Studio is running at `localhost:1234` with `ornith-1.0-35b` loaded | Reads the actual diff per commit; map-reduces large diffs across 16k-char chunks |

Each commit's bullet falls back to baseline independently — if LM Studio closes mid-run, completed bullets stay accurate; pending ones use baseline.

> **ornith note:** it is a reasoning model whose hidden thinking consumes the token budget before the visible answer. Use `max_tokens=1500` minimum or `content` returns empty.

### 3.4 Blocked-PR Hold Rules

```
classify_blockers(pr) → { blocked, reasons[], actions[] }
```

| Rule | Reason | Action told to team |
|---|---|---|
| `pr.isDraft` | Work-in-progress | Mark PR "Ready for review" |
| Any file starts with `drizzle/` | DB migration must be applied manually | Run `drizzle-kit push`, then merge manually |
| `pr.mergeable == "CONFLICTING"` | Has merge conflicts | Rebase or merge main into the branch |

Blocked-PR notifications are deduped per head SHA in `scripts/.notify-blocked-state.json` — the team gets one message per revision, not hourly spam.

### 3.5 Cron Script

```python
# scripts/auto-merge.py — key constants to adapt per project

OWNER_LOGIN = "your-github-username"
OWNER_NAMES = {"Your Name", "your-github-username"}
REPO        = "your-github-username/your-repo"
REPO_ROOT   = Path(__file__).parent.parent
HOMEBREW_BIN = "/opt/homebrew/bin"   # macOS/Homebrew — adjust for Linux

# Hold rules in classify_blockers() — customise which file paths block
# (e.g. change "drizzle/" to "migrations/" for your ORM)
```

### 3.6 Installing the Cron

```bash
# Add — runs at the top of every hour
(crontab -l 2>/dev/null; \
 echo "0 * * * * /opt/homebrew/bin/python3 /path/to/project/scripts/auto-merge.py \
   >> /path/to/project/scripts/auto-merge.log 2>&1") | crontab -

# Verify registered
crontab -l | grep auto-merge

# Verify healthy (last run should not contain "Traceback")
tail -20 scripts/auto-merge.log

# Dry-run preview (safe to run during dev)
python3 scripts/auto-merge.py --dry-run
```

A healthy idle run ends with:
```
INFO  No open contributor PRs
INFO  No undocumented contributor commits — already up to date
```

---

## 4. Headless Auth Fix

macOS cron can't unlock the GUI keychain, so both `git push` and `gh pr` fail with `fatal: could not read Username` / `HTTP 401`. Fix both at once with a plaintext credential store — one file, no keychain:

```bash
# 1. Write your GitHub token to a plaintext credential file
umask 077
printf 'https://x-access-token:%s@github.com\n' "$(gh auth token)" > ~/.git-credentials
chmod 600 ~/.git-credentials

# 2. Register 'store' as a global git credential helper fallback
#    (keychain stays primary for interactive use; store is the cron fallback)
git config --global --add credential.helper store

# 3. Verify headless auth — must print a SHA, not a username prompt
env -i HOME="$HOME" PATH="/opt/homebrew/bin:/usr/bin:/bin" \
  git ls-remote https://github.com/OWNER/REPO.git HEAD
```

The cron script reads `~/.git-credentials` and injects `GH_TOKEN` for `gh` automatically — both `git` and `gh` authenticate from the same single file.

**Token expiry:** refresh with:
```bash
umask 077; printf 'https://x-access-token:%s@github.com\n' "$(gh auth token)" > ~/.git-credentials
```

---

## 5. Tier 2 — `/loop` PR-Watcher Session

A Claude Code session that polls cheaply and escalates **only when there's real work**.

| Aspect | Detail |
|---|---|
| **Model** | `claude --model sonnet` (watcher) → subagent `model 'sonnet'` (can change to `'opus'` for payment-critical review) |
| **Interval** | `/loop 15m` — adjust as needed |
| **Idle cost** | Near zero — pure bash on idle cycles, no model inference |
| **Session-only** | Dies when the terminal closes; the OS cron keeps running independently |

### 5.1 How to Start

```bash
cd /path/to/project && claude --model sonnet
```

Then paste this as the first message (edit the `OWNER/REPO` and the quoted subagent task for your project):

```
/loop 15m You are a low-cost PR watcher for OWNER/REPO. Poll cheaply and escalate ONLY when there is real work. Each run:

1. Run `git fetch origin --prune`, then `gh pr list --state open --json number,title,author,isDraft` and compare local HEAD to origin/main.
2. If NO open PRs AND no new non-owner commits: stay silent. Do NOT review, build, or edit anything.
3. If an open PR or non-owner commit exists: dispatch a subagent via the Agent tool with model 'sonnet' and this task:
   "Review, merge, and deploy the open PR(s)/non-owner commits on OWNER/REPO.
    Do a careful security/breaking-change review; ship by default unless it breaks something major
    (build fails, data-loss migration, security/payment regression).
    Resolve conflicts in an ISOLATED git worktree off origin/main.
    Verify with `npm run build` (exit 0) before pushing.
    Cut an OWNER-authored release commit (bump semver: feat=minor, fix/other=patch, breaking=major;
    update CHANGELOG.md) so Vercel deploys.
    Wait for the Vercel production deploy to be Ready.
    Run `scripts/notify-deploy.mts --version X.Y.Z` to email+Slack the devs.
    If a PR must be HELD (draft, migration, data-loss, security regression),
    run `scripts/notify-blocked.mts` explaining why and how to fix.
    Never merge/branch/push from the shared working copy without a worktree."
4. Relay a ONE-line summary of what shipped or was held.
```

---

## 6. Notify Scripts

### 6.1 `scripts/notify-deploy.mts`

Emails + Slacks the dev team when a release is live on Vercel.

```bash
npx tsx scripts/notify-deploy.mts \
  --version 2.51.0 \
  --deploy-url https://your-project-abc123.vercel.app \
  [--summary "Add hotel booking"] \
  [--summary "Fix email fallback"] \
  # --summary repeats; omit to auto-derive from latest git commit subject
  [--dry-run]
```

**Key fields to adapt in `notify-deploy.mts`:**

| Field | What to change |
|---|---|
| `RECIPIENTS` | Email list for the dev team |
| `FROM` | Verified sender in your email provider (`Name <email@domain>`) |
| `DEFAULT_PROD_URL` | Your production URL |

Email provider: tries **SendGrid** first (`SENDGRID_API_KEY`), falls back to **Resend** (`RESEND_API_KEY`) on 401/5xx. Also posts a **Slack Block Kit** message to `SLACK_DEPLOY_WEBHOOK_URL`.

### 6.2 `scripts/notify-blocked.mts`

Emails + Slacks the dev team when a PR is held.

```bash
npx tsx scripts/notify-blocked.mts \
  --pr 82 \
  --title "feat: hotel bookings" \
  --author tonytha \
  --url https://github.com/OWNER/REPO/pull/82 \
  --reason "PR is a draft" \
  --action "Mark the PR 'Ready for review'" \
  [--reason "..."] [--action "..."] \
  [--dry-run]
```

`--reason` and `--action` repeat (one bullet each in the email and Slack message).

---

## 7. Environment Variables

All variables live in `.env.local`. The cron script reads them from the file directly (no shell env needed in cron):

| Variable | Used by | Purpose |
|---|---|---|
| `SENDGRID_API_KEY` | notify-deploy, notify-blocked | Primary transactional email |
| `RESEND_API_KEY` | notify-deploy | Email fallback if SendGrid returns 401/5xx |
| `SLACK_DEPLOY_WEBHOOK_URL` | notify-deploy, notify-blocked, auto-merge.py | Deploy + held-PR Slack messages |
| `SLACK_ERROR_WEBHOOK_URL` | auto-merge.py | Cron crash alerts (can be the same webhook) |

---

## 8. Session-Start Checklist

Run at the start of every dev session:

```bash
# 1. Is the cron registered and running cleanly?
crontab -l | grep auto-merge
tail -5 scripts/auto-merge.log    # must not contain "Traceback"

# 2. Is the /loop watcher running?
pgrep -fl "claude.*sonnet" | grep -v grep
# If not: cd /path/to/project && claude --model sonnet  → paste /loop prompt
```

---

## 9. Known Failure Modes

### Cron auth breaks after Mac restart or token expiry

**Symptom:** `fatal: could not read Username` or `gh: HTTP 401` in the log

**Fix:**
```bash
umask 077; printf 'https://x-access-token:%s@github.com\n' "$(gh auth token)" > ~/.git-credentials
```

### Cron missed runs (Mac was asleep)

**Symptom:** gap in log timestamps

**Fix:** run manually to catch up:
```bash
python3 scripts/auto-merge.py --dry-run   # preview first
python3 scripts/auto-merge.py             # apply
```

### Never run `auto-merge.py` while doing dev work in the main checkout

`main()` starts with `git checkout main && git pull`, which switches your working tree mid-task. Test with `--dry-run` or run in a separate terminal.

### The script is untracked local tooling

`scripts/auto-merge.py` is **not committed to the repo** — it lives only on the host machine and the cron runs it directly from disk. Edits take effect immediately with no commit/PR. Back it up externally; `git reset --hard` on the repo will not delete it (it's untracked).

---

## 10. Porting Checklist

- [ ] **Copy `scripts/auto-merge.py`** — update `OWNER_LOGIN`, `OWNER_NAMES`, `REPO`, and `REPO_ROOT`
- [ ] **Adapt hold rules** — update the `drizzle/` path check in `classify_blockers()` to match your ORM/migration folder
- [ ] **Copy `scripts/notify-deploy.mts`** — update `RECIPIENTS`, `FROM`, `DEFAULT_PROD_URL`
- [ ] **Copy `scripts/notify-blocked.mts`** — update `RECIPIENTS` and `FROM`
- [ ] **Set env vars** — add `SENDGRID_API_KEY` (or `RESEND_API_KEY`), `SLACK_DEPLOY_WEBHOOK_URL`, `SLACK_ERROR_WEBHOOK_URL` to `.env.local`
- [ ] **Headless auth** — write `~/.git-credentials` and run `git config --global --add credential.helper store`
- [ ] **Verify headless** — `env -i HOME="$HOME" PATH="/opt/homebrew/bin:/usr/bin:/bin" git ls-remote https://github.com/OWNER/REPO.git HEAD` must print a SHA
- [ ] **Install the cron** — `(crontab -l 2>/dev/null; echo "0 * * * * ...") | crontab -`
- [ ] **Dry-run test** — `python3 scripts/auto-merge.py --dry-run` — confirm output looks correct
- [ ] **Live test** — raise a trivial contributor PR, wait for the next cron run, confirm it merges + Vercel deploys
- [ ] **Verify owner-authored HEAD** — `git log -1 --format="%an"` on `main` after deploy should be the owner
- [ ] **Start the /loop watcher** — `claude --model sonnet` → paste the `/loop 15m ...` prompt
- [ ] **Session-start habit** — check `crontab -l | grep auto-merge` and `pgrep -fl "claude.*sonnet"` at each session start
- [ ] **Test blocked-PR path** — open a draft PR and wait for a cron run; `notify-blocked.mts` should fire once, not every hour (check `.notify-blocked-state.json`)
- [ ] **Check email + Slack on first real deploy** — confirm the dev team receives the notification

---

<!-- Part of the ShipFactory feature spec library — https://github.com/vishalquantana/shipfactory -->

## About Us

We are [Quantana](https://quantana.com.au), an AI-first design and development agency working with Fortune 500s to build bespoke AI solutions and provide the audit and training needed to ensure success. [Click here to learn more](https://quantana.com.au).

## License

MIT
