# Parallel Multi-Agent Dev System (Orchestrator + Worker Agents in cmux)

## Feature Specification for Porting

This document describes a **multi-agent software-development system**: one **Orchestrator** agent supervises a fleet of **Worker** coding agents (Claude Code, Codex, Gemini, etc.) running in parallel [cmux](https://github.com/) terminal panes. Workers each build a feature in an **isolated git worktree** on a `feat/*` branch; a single-writer **merge-train** continuously integrates every branch into `master`, stamps one version, pushes, and a prod box auto-deploys — usually within ~30s of a worker's last commit.

The design goal: **many agents writing code at the same time with zero merge conflicts and zero human babysitting**, because (a) no worker may ever touch `master` (git-hook enforced), and (b) the only writer to `master` is a deterministic integration script.

It is runtime-agnostic — the workers can be any CLI coding agent, mixed freely.

---

## 1. Overview

| Concept | Summary |
|---------|---------|
| **Orchestrator** | A supervising agent. Maps the fleet, dispatches tasks, runs/monitors the merge-train, unblocks stuck workers, never writes app code itself. |
| **Worker (Dev)** | A coding agent in its own cmux pane. Creates a worktree, builds one task on a `feat/*` branch, runs tests, commits, and stops. Never merges or deploys. |
| **Worktree isolation** | Every worker gets a private working directory (`../<repo>-wt-<task>`) on a fresh branch off latest `master`. No two workers share a working tree. |
| **Single-writer master** | A shared git hook **rejects any commit/push to `master`** unless the env var `KLAV_ORCHESTRATOR=1` is set. Only the merge-train sets it. This makes master conflicts structurally impossible. |
| **Merge-train** | A loop that assembles every `feat/*` branch with new commits into `master` (`-X theirs`), runs an integrity gate, stamps ONE version across all manifests, and pushes. |
| **Autodeploy** | The prod box polls `origin/master` and pulls+restarts on change (with health-rollback). The orchestrator does not SSH-deploy per change. |
| **Control plane** | The orchestrator drives worker panes via the cmux CLI (`read-screen`, `send`, `send-key`) — it can read any pane's screen and type into it. |

**Why this shape works:** parallel agents normally collide on shared files and the human becomes a merge bottleneck. Here, isolation (worktrees) + a single deterministic writer (merge-train) + hook enforcement removes both failure modes. Workers can be "dumb and local" — commit and stop — while integration, versioning, and deploy are centralized and automatic.

---

## 2. System Topology

```
                      ┌──────────────────────────────────────────────┐
                      │              ORCHESTRATOR (1 agent)           │
                      │  • maps fleet (list-panes/list-pane-surfaces) │
                      │  • dispatches tasks (send + send-key enter)   │
                      │  • runs merge-loop daemon                     │
                      │  • status ticks, unblocks, reassigns          │
                      └───────────────┬──────────────────────────────┘
            drives panes (read-screen │ send)        runs
            ┌───────────────┬─────────┼─────────┬───────────────┐
            ▼               ▼         ▼         ▼               ▼
      ┌───────────┐   ┌───────────┐  ...  ┌───────────┐   ┌─────────────────┐
      │  Dev 1    │   │  Dev 2    │       │  Dev N    │   │   merge-loop.sh │
      │ (Claude)  │   │ (Codex)   │       │ (Gemini)  │   │  (sole writer)  │
      │ cmux pane │   │ cmux pane │       │ cmux pane │   └────────┬────────┘
      └─────┬─────┘   └─────┬─────┘       └─────┬─────┘            │
            │ new-worktree.sh <task>            │            every ~25s:
            ▼               ▼                   ▼            merge-train.sh
   ../repo-wt-taskA  ../repo-wt-taskB    ../repo-wt-taskN          │
   feat/taskA        feat/taskB          feat/taskN                │
            │ commit         │ commit            │ commit          │
            └────────────────┴───────────────────┴─────────────────┤
                                                                    ▼
                                          ┌──────────────────────────────────┐
                                          │  master (single writer only)     │
                                          │  merge -X theirs every feat/*    │
                                          │  → integrity gate                │
                                          │  → stamp ONE version             │
                                          │  → git push origin master        │
                                          └─────────────────┬────────────────┘
                                                            │ push
                                                            ▼
                                              ┌──────────────────────────┐
                                              │  GitHub origin/master    │
                                              └────────────┬─────────────┘
                                                           │ poll + pull (health-rollback)
                                                           ▼
                                              ┌──────────────────────────┐
                                              │  PROD box (autodeploy)   │
                                              └──────────────────────────┘
```

**Single shared `.git`:** worktrees share one `.git` directory, so the hooks in `.git/hooks/` apply to **every** worktree at once — one enforcement point covers the whole fleet.

---

## 3. Roles & Responsibilities

### 3.1 Orchestrator

| Does | Never does |
|------|------------|
| Re-discovers the fleet every session (surface IDs change) | Edits app code / commits in the main checkout |
| Dispatches one task per idle worker | Pushes or merges to master by hand |
| Runs exactly ONE merge-loop daemon | Runs per-change SSH deploys (autodeploy owns that) |
| Status-ticks: reads each pane, classifies working/idle/blocked | Lets two merge-loops run at once |
| Unblocks workers (conflicts, hook rejections, parked input) | Resolves a worker's conflict against master (theirs-wins handles it) |
| Assigns hot files to a single owner per round (lane discipline) | |

### 3.2 Worker (Dev)

The worker contract (5 rules, enforced by hooks + convention):

1. **Never touch `master`** — a git hook rejects it.
2. **Always work in your own worktree** on a `feat/*` branch (`bash scripts/new-worktree.sh <task>`).
3. **You don't deploy, but you test** — run the suite green before calling a task done; pull latest at the end.
4. **Don't bump the version** — the merge-train stamps exactly one version per integration; editing manifests/CHANGELOG just causes churn (it owns + overwrites them).
5. **If blocked** — `git merge --abort` and keep committing on your branch; the orchestrator integrates with theirs-wins. Don't fight master.

### 3.3 Heterogeneous backends (optional)

Workers can be different agent runtimes to compare them. Key adaptations:

| Runtime | Reads rules from | "Working" signature on screen |
|---------|------------------|-------------------------------|
| Claude Code | `CLAUDE.md` | `Verb… (timer · esc to interrupt)` |
| Codex | `AGENTS.md` | `● Working (Ns · esc to interrupt)` |
| Gemini / Antigravity | `GEMINI.md` | `⣾ Generating… / esc to cancel`, prompt `>` |

> **Mirror the worker rules into `AGENTS.md` and `GEMINI.md` at the workspace root** (ancestors of every worktree) so non-Claude agents auto-read them. A status script tuned to one runtime's TUI **misclassifies** the others — read those panes directly and interpret their own indicators. Re-check which runtime is in each pane every session; backends get swapped.

---

## 4. Core Components

### 4.1 cmux Panes & Surfaces

cmux exposes **panes** (windows) each containing one or more **surfaces** (terminal views). The orchestrator addresses workers by surface ID.

```bash
CMUX=/Applications/cmux.app/Contents/Resources/bin/cmux

$CMUX list-panes                         # → pane:2 .. pane:N
$CMUX list-pane-surfaces --pane pane:5   # → surface:10  "Dev 2"
$CMUX read-screen --surface surface:10   # read a pane's visible screen
$CMUX read-screen --surface surface:10 --lines 24   # wider read (catch scrolled spinners)
$CMUX send --surface surface:10 "text"   # type text into a pane (no Enter)
$CMUX send-key --surface surface:10 enter   # press a key (enter, ctrl+c, …)
$CMUX notify --surface surface:10 "msg"  # optional toast
```

> **Surface IDs are NOT stable across sessions — re-map every restart.** Hardcoding them is the #1 cause of "I sent to the wrong pane."

### 4.2 Worktree Isolation — `scripts/new-worktree.sh`

```bash
#!/bin/bash
# Worker helper — spin up an isolated worktree on a fresh feat/<name> branch off
# the latest master. Workers MUST use this; committing on master is hook-blocked.
set -euo pipefail
name="${1:?usage: new-worktree.sh <short-task-name>}"
slug=$(printf '%s' "$name" | tr '[:upper:] ' '[:lower:]-' | tr -cd 'a-z0-9-' | sed 's/--*/-/g; s/^-//; s/-$//')
REPO="$(cd "$(dirname "$0")/.." && pwd)"
cd "$REPO"
git fetch -q origin master || true
wt="../<repo>-wt-$slug"
if git show-ref --quiet "refs/heads/feat/$slug"; then
  echo "branch feat/$slug already exists; reusing"; git worktree add "$wt" "feat/$slug" 2>/dev/null || true
else
  git worktree add -b "feat/$slug" "$wt" origin/master
fi
echo "✅ Worktree ready: $(cd "$wt" && pwd)  (branch feat/$slug)"
echo "   Work + commit on this branch only. The orchestrator merges & deploys automatically."
```

- Branches are always cut **from `origin/master`** (latest), not the local checkout — workers start current.
- Worktree dirs are siblings of the repo: `../<repo>-wt-<slug>`. **Worktrees have no `node_modules`** — workers must `bun install` / `npm install` once before running tests.

### 4.3 Single-Writer Master — Git Hooks

Two hooks live in the shared `.git/hooks/` and therefore apply to every worktree. They block all writes to `master` unless `KLAV_ORCHESTRATOR=1`.

**`.git/hooks/pre-commit`:**
```sh
#!/bin/sh
# ORCHESTRATOR ENFORCEMENT — Workers may NEVER commit on master. The orchestrator
# owns master and is the only writer (it sets KLAV_ORCHESTRATOR=1).
branch=$(git rev-parse --abbrev-ref HEAD 2>/dev/null)
if [ "$branch" = "master" ] && [ "$KLAV_ORCHESTRATOR" != "1" ]; then
  echo "✋ BLOCKED: you are on master. Run: bash scripts/new-worktree.sh <task>" >&2
  exit 1
fi
exit 0
```

**`.git/hooks/pre-push`:**
```sh
#!/bin/sh
# Reject any push to master from a worker. Only the orchestrator may push master.
[ "$KLAV_ORCHESTRATOR" = "1" ] && exit 0
while read local_ref local_sha remote_ref remote_sha; do
  case "$remote_ref" in
    refs/heads/master)
      echo "✋ BLOCKED: pushing to master is reserved for the orchestrator." >&2
      exit 1;;
  esac
done
exit 0
```

> Hooks are **not** copied on clone. After cloning, install them once (see Porting Checklist). They are the load-bearing safety mechanism — without them an agent can clobber master.

### 4.4 The Merge-Train — `scripts/merge-train.sh`

The integration engine. One pass = align to origin, merge every eligible `feat/*` branch `-X theirs`, run the integrity gate, stamp one version, push.

```bash
#!/bin/bash
# merge-train — the orchestrator's single-writer integration pass.
set -uo pipefail
export KLAV_ORCHESTRATOR=1          # the ONLY place this is set
REPO="$(cd "$(dirname "$0")/.." && pwd)"; cd "$REPO" || exit 1
log(){ echo "[$(date '+%F %T')] [merge-train] $*"; }

git fetch -q origin master 2>/dev/null
# Self-heal a WEDGED checkout (a 120s-killed mid-merge can leave unmerged files).
git merge --abort 2>/dev/null
git checkout -qf master 2>/dev/null || { git reset -q --hard; git clean -fdq; git checkout -qf master; }
git reset -q --hard origin/master 2>/dev/null   # single writer ⇒ align to origin
git clean -fdq 2>/dev/null

base_ver=$(sed -n 's/.*"version": *"\([0-9]*\.[0-9]*\.[0-9]*\)".*/\1/p' package.json | head -1)
[ -z "$base_ver" ] && base_ver="0.0.0"

EXCLUDE_RE='^feat/locked-research'   # branches to never auto-ship
QUIET_MIN=45                          # don't merge a branch whose last commit is younger (mid-burst)
now=$(date +%s)

# --- Post-merge integrity gate (CUSTOMIZE per project) -------------------
# WHY: we merge -X theirs. A branch on a STALE base silently wins its old copy
# of any file master fixed since — NOT a conflict, so it ships regressions.
# After each merge, re-check protected invariants vs the PRE-merge tree and
# REVERT just that branch if it regressed. Good branches in the cycle are kept.
DASH="path/to/critical-file.html"
PROTECTED_PATTERNS=('featureMarkerA' 'featureMarkerB')   # counts must never DROP
WIDGET_BUNDLE="path/to/built/bundle.js"                  # must stay valid JS
dash_count(){ [ -f "$DASH" ] && grep -c "$1" "$DASH" 2>/dev/null || echo 0; }

changed=0; merged=""
for b in $(git for-each-ref --format='%(refname:short)' refs/heads/ | grep -E '^feat/'); do
  echo "$b" | grep -qE "$EXCLUDE_RE" && continue
  ahead=$(git rev-list --count "master..$b" 2>/dev/null || echo 0)
  [ "${ahead:-0}" -eq 0 ] && continue
  ct=$(git log -1 --format=%ct "$b"); age=$(( now - ct ))
  [ "$age" -lt "$QUIET_MIN" ] && { log "skip $b (committed ${age}s ago, still hot)"; continue; }

  pre=$(git rev-parse HEAD)
  pre_counts=(); for p in "${PROTECTED_PATTERNS[@]}"; do pre_counts+=("$(dash_count "$p")"); done

  if git merge --no-edit -X theirs "$b" >/dev/null 2>&1; then
    why=""
    for i in "${!PROTECTED_PATTERNS[@]}"; do
      post=$(dash_count "${PROTECTED_PATTERNS[$i]}")
      [ "$post" -lt "${pre_counts[$i]}" ] && why="$why ${PROTECTED_PATTERNS[$i]}(${pre_counts[$i]}->$post)"
    done
    if command -v node >/dev/null && [ -f "$WIDGET_BUNDLE" ] && ! node --check "$WIDGET_BUNDLE" >/dev/null 2>&1; then
      why="$why bundle-syntax"
    fi
    if [ -n "$why" ]; then
      log "!!! INTEGRITY GATE BLOCKED $b — reverting (stale-base regressed:$why). Rebase needed."
      git reset -q --hard "$pre"
    else
      log "merged $b (+$ahead)"; changed=1; merged="$merged $b"
    fi
  else
    log "CONFLICT on $b — aborting that merge, skipping"; git merge --abort 2>/dev/null
  fi
done

[ "$changed" -eq 0 ] && { log "nothing to integrate"; exit 0; }

# Single monotonic version stamp across all manifests + docs.
maj=${base_ver%%.*}; rest=${base_ver#*.}; min=${rest%%.*}; pat=${rest##*.}
next="$maj.$min.$((pat+1))"
for f in package.json packages/*/package.json packages/*/manifest.json; do
  [ -f "$f" ] && sed -i '' "s/\"version\": *\"[0-9][0-9.]*\"/\"version\": \"$next\"/" "$f"
done
git add -A
git commit -q -m "orchestrator: integrate$merged → v$next"
if git push -q origin master 2>/dev/null; then
  log "pushed v$next ($(git rev-parse --short HEAD)) — integrated:$merged"
else
  log "PUSH FAILED (will retry next cycle)"
fi
```

**Four ideas make this safe:**

| Mechanism | Purpose |
|-----------|---------|
| `-X theirs` merge | Non-overlapping hunks always combine; overlapping hunks take the branch's version (no human conflict resolution) |
| **Quiet window** (`QUIET_MIN`) | Skip a branch whose last commit is <45s old — avoids merging mid-edit-burst |
| **Integrity gate** | After each merge, verify protected invariants (feature-marker counts didn't drop, built bundles still parse) vs the pre-merge tree; **revert just the offending branch** if a stale base regressed something |
| **Single version stamp** | One monotonic `patch+1` written to all manifests in the same integration commit — workers never bump versions |

> **`-X theirs` is the key trade-off.** It guarantees no human conflict resolution, but a worker on a **stale base** can silently revert a file `master` fixed since (not a conflict → ships). The integrity gate catches the known-critical cases; the broader mitigation is **lane discipline** (§6.3) — one owner per hot file per round.

### 4.5 The Merge-Loop Wrapper — `scripts/merge-loop.sh`

Runs the merge-train forever with a hard kill so one stuck git op can never freeze integration.

```bash
#!/bin/bash
# Minimal, hang-resistant merge loop — the SOLE runner of merge-train.
# Each merge-train runs in a child with a hard 120s kill.
REPO="/abs/path/to/repo"
LOG="$HOME/.config/orchestrator/merge-loop.log"
mkdir -p "$(dirname "$LOG")"
echo "[$(date '+%F %T')] merge-loop started (pid $$)" >> "$LOG"
while true; do
  ( bash "$REPO/scripts/merge-train.sh" ) >> "$LOG" 2>&1 &
  p=$!
  ( sleep 120 && kill -9 "$p" 2>/dev/null ) &   # watchdog
  k=$!
  wait "$p" 2>/dev/null
  kill "$k" 2>/dev/null
  sleep 25                                       # cadence
done
```

**Start it detached** (survives the launching session; do NOT use launchd on macOS if the repo is under `~/Downloads` — TCC blocks it):
```bash
python3 -c "import subprocess,os; subprocess.Popen(
  ['bash','/abs/path/to/repo/scripts/merge-loop.sh'],
  stdout=open(os.path.expanduser('~/.config/orchestrator/merge-loop.out'),'a'),
  stderr=subprocess.STDOUT, stdin=subprocess.DEVNULL, start_new_session=True)"
```

> **Ensure exactly ONE merge-loop.** `pgrep -f merge-loop.sh` should show one persistent pid (a transient second pid is the `merge-train.sh` child). Two persistent loops = double version stamps and racey pushes.

### 4.6 Autodeploy Bridge

The prod box owns deploy — the orchestrator never SSH-deploys per change. Pattern: a small loop/cron on prod polls `origin/master` and pulls on change, then restarts the service with a **health check + rollback**.

```
# on prod, every ~12s (or a */15 cron for static-only):
git -C /opt/app fetch origin master && \
  [ "$(git rev-parse HEAD)" != "$(git rev-parse origin/master)" ] && \
  git reset --hard origin/master && systemctl restart app && <healthcheck-or-rollback>
```

> Static assets (HTML/CSS) go live on pull with no restart. Code changes need the restart step. Decide per project whether autodeploy restarts on every push or only on a version change.

### 4.7 Worker Rules Files

Put the 5-rule worker contract in the runtime's instructions file at the **workspace root** so every worktree inherits it:
- `CLAUDE.md` (Claude Code)
- `AGENTS.md` (Codex)
- `GEMINI.md` (Gemini)

Minimum contents: never touch master · always `new-worktree.sh` · test before done · don't bump version · on block, abort & keep committing on your branch.

### 4.8 Status Board — `scripts/klav-status.sh`

A spinner-aware board so the orchestrator can classify each worker without eyeballing every pane. It reads a **wide** screen slice (spinners scroll above the input box) and does one confirming re-read to avoid mid-transition false IDLE.

```bash
#!/bin/bash
CMUX=/Applications/cmux.app/Contents/Resources/bin/cmux
for n in <surface-ids…>; do
  sid="surface:$n"
  scr="$("$CMUX" read-screen --surface "$sid" --lines 24 2>/dev/null)"
  if ! printf '%s' "$scr" | grep -qE '(…|\.\.\.)[[:space:]]*\(|esc to interrupt'; then
    sleep 0.6; scr="$("$CMUX" read-screen --surface "$sid" --lines 24 2>/dev/null)"  # confirm
  fi
  if   printf '%s' "$scr" | grep -qiE 'rate limit|try again in';                       then st="RATE-LTD"
  elif printf '%s' "$scr" | grep -qE '(…|\.\.\.)[[:space:]]*\(|esc to interrupt';        then st="WORKING"
  elif printf '%s' "$scr" | grep -qE 'Enter to select|Do you want to proceed|\(y/n\)';   then st="NEEDS-PICK"
  else
    pl="$(printf '%s' "$scr" | grep -E '^❯' | tail -1 | sed -E 's/^❯[[:space:]]*//')"
    if [ -n "$pl" ]; then st="PARKED"
    elif printf '%s' "$scr" | grep -qiE 'want me to|should i |go ahead\?|\?[[:space:]]*$'; then st="ASKED-Q"
    else st="IDLE"; fi
  fi
  printf '  %-6s %s\n' "Dev-$n" "$st"
done
echo "  merge-loop=[$(pgrep -f merge-loop.sh | tr '\n' ' ')]"
```

| Status | Meaning | Orchestrator action |
|--------|---------|---------------------|
| `WORKING` | live spinner present | leave alone |
| `IDLE` | empty prompt, no spinner | dispatch next task (confirm with a re-read) |
| `PARKED` | text sitting unsent in the input box | the worker drafted but didn't submit — press Enter |
| `NEEDS-PICK` | interactive menu | answer the prompt |
| `ASKED-Q` | finished a turn with a question | the human (or orchestrator) answers |
| `RATE-LTD` | usage/quota banner | reassign that task to a free runtime |

---

## 5. End-to-End Lifecycle

```
1. Orchestrator startup
   • map fleet: list-panes → list-pane-surfaces  (IDs change per session)
   • ensure ONE merge-loop running (start detached if absent)
   • read prod version, board state

2. Dispatch  (per idle worker)
   • send the task + worker rules into the pane
   • send-key enter to submit

3. Worker executes
   • bash scripts/new-worktree.sh <task>   → ../repo-wt-<task> on feat/<task>
   • cd in, install deps, build the change
   • run the test suite green; run relevant e2e
   • git add <files> && git commit   (on feat/<task>, NEVER master)
   • leave the branch; report "committed" in its pane

4. Merge-train (every ~25s, automatic)
   • align master to origin
   • for each feat/* with new commits, >QUIET_MIN old:
       merge -X theirs → integrity gate → keep or revert
   • if anything merged: stamp v(patch+1) across manifests, push origin master

5. Autodeploy (prod, automatic)
   • detect origin/master moved → pull → restart (health-rollback)
   • live within ~30s of the worker's last commit

6. Orchestrator status tick (recurring)
   • run status board; dispatch idle workers; unblock parked/conflicted; reassign rate-limited
```

---

## 6. Operational Procedures (Orchestrator Runbook)

### 6.1 Startup

```bash
# 1. Map the fleet (DO NOT trust last session's IDs)
CMUX=/Applications/cmux.app/Contents/Resources/bin/cmux
$CMUX list-panes
for p in <panes>; do $CMUX list-pane-surfaces --pane pane:$p; done

# 2. Ensure exactly one merge-loop
pgrep -fl merge-loop.sh           # zero → start detached (§4.5); ≥2 persistent → kill extras

# 3. Guard against a stray detached integrator (see Gotchas) — it can git reset --hard
pgrep -fl '<orchestrator-daemon>.py'   # kill if it reparented to init and spawns merge-train

# 4. Read prod state
ssh root@<prod> 'cd /opt/app; git rev-parse --short HEAD; grep -m1 -o "[0-9.]*" package.json'
```

### 6.2 Dispatch a task to a worker

Type the task **and** the worker rules into the pane, then submit. Multi-line input often needs an explicit Enter keystroke (a literal `\n` may just add a newline in the editor):

```bash
$CMUX send --surface surface:N "<full task + worker rules>"
$CMUX send-key --surface surface:N enter
# verify it started:
sleep 6; $CMUX read-screen --surface surface:N | tail -8
```

> **Never put backticks or `$(...)` in a `send` payload from bash** — they command-substitute on the orchestrator's own machine and mangle the dispatch. Single-quote the payload or avoid shell metacharacters.

### 6.3 Lane discipline (avoids stale-base regressions)

Only **one** worker edits a given hot file per round. `-X theirs` keeps non-overlapping hunks but drops overlapping ones, so two workers editing the same file silently lose work. Route concurrent tasks to **distinct files/areas** (a new module, test files, a different page). Assign hot shared files (a main dashboard, a server entrypoint, a built bundle) to a single owner at a time.

### 6.4 Unblock a stuck worker

| Symptom | Fix |
|---------|-----|
| Hook rejected a master commit | Tell the worker to `new-worktree.sh` and redo on a feat branch |
| Merge conflict against master | Worker runs `git merge --abort` and keeps committing on its branch; theirs-wins integrates |
| Integrity gate reverted its branch | Worker rebases onto `origin/master` (`git fetch && git rebase origin/master`) and re-commits |
| `PARKED` input | `send-key enter` to submit the drafted message |
| `RATE-LTD` | Reassign the task to an idle worker on another runtime |

### 6.5 Stop / restart

```bash
pkill -f merge-loop.sh           # stop integration (workers keep their branches)
# restart: start one detached merge-loop again (§4.5)
```

---

## 7. Dispatch Template (what the orchestrator sends a worker)

```
TASK: <one-paragraph goal + acceptance criteria>.

WORKER RULES: create your own worktree — bash scripts/new-worktree.sh <task> —
cd into ../<repo>-wt-<task> and work on that feat/ branch ONLY. Do NOT touch
master or version/CHANGELOG/manifest files (the orchestrator owns + stamps them).
Before done: install deps if the worktree has none, run <test command> green,
run the relevant e2e. Commit on your feat branch and STOP — the merge-train
integrates + deploys automatically. Report "committed <slug>" in this pane.
<any file-lane constraint, e.g. "only edit src/foo/*, do not touch dashboard.html">
```

For non-Claude runtimes, spell the rules out in full in the dispatch (they may not read `CLAUDE.md`), or ensure `AGENTS.md`/`GEMINI.md` exist at the root.

---

## 8. Gotchas & Known Limitations

| # | Gotcha | Mitigation |
|---|--------|-----------|
| 1 | **Surface IDs change every session** | Re-map with `list-pane-surfaces` on every startup; never hardcode |
| 2 | **`-X theirs` ships stale-base regressions** (not conflicts) | Integrity gate + lane discipline (§4.4, §6.3) |
| 3 | **Backticks/`$()` in a `send` payload** substitute on the orchestrator's host | Single-quote payloads; avoid shell metachars in dispatch text |
| 4 | **Worktrees have no `node_modules`** → import/test errors | Worker runs `bun install`/`npm ci` once before tests |
| 5 | **Multi-line `send` parks instead of submitting** | Always follow with `send-key enter` |
| 6 | **Two merge-loops** → double version stamps, racey pushes | `pgrep` check; keep exactly one |
| 7 | **A stray detached integrator daemon** (reparented to init) runs `git reset --hard` and silently wipes uncommitted edits / buries unpushed commits | Detect via a live `merge-train.sh` whose PPID is the rogue daemon; `pkill -9` it. Always `commit`+`push` orchestrator script edits in one step |
| 8 | **macOS launchd + TCC** can't read `~/Downloads` ("Operation not permitted") | Start daemons `Popen`-detached from the terminal session, not launchd |
| 9 | **Committed built bundles** served verbatim (no rebuild on prod) | `node --check` the bundle in the integrity gate; rebuild + commit a clean bundle |
| 10 | **Git hooks aren't cloned** | Install hooks as a setup step after clone (Porting Checklist) |
| 11 | **A killed mid-merge wedges the checkout** ("needs merge") | merge-train self-heals at the top (`merge --abort` / `reset --hard` / `clean`) |
| 12 | **Remote agent has read-only git creds** → authors but can't push (silent miss) | Grant the agent's identity write access, or have a write-capable local process publish its output |

---

## 9. Best Practices & Hard-Won Lessons

These are the lessons that turn the architecture from "works in a demo" into "runs a fleet unattended for days." Most were learned by getting bitten.

### 9.1 How to avoid merge conflicts (the whole point)

Conflicts are designed out, not resolved. Five compounding rules:

1. **One writer to the trunk.** Nothing but the merge-train writes `master`. Workers physically cannot (hook-enforced). 90% of multi-agent merge pain is two agents pushing the same branch — this removes it entirely.
2. **One worktree per worker.** Never let two agents share a working directory. Shared dirs cause *branch/commit contamination* — agent A's uncommitted edits get swept into agent B's commit. Isolated worktrees make each agent's tree private.
3. **One owner per hot file per round (lane discipline).** `-X theirs` silently drops *overlapping* hunks, so two agents editing the same file = lost work with no error. Decompose tasks by **lane** (distinct files/modules/areas), not by feature. If a shared file (main dashboard, server entrypoint, built bundle) must change, give it a single owner that round and route everyone else elsewhere.
4. **Workers rebase onto `origin/master` before finishing.** The dangerous case isn't a conflict — it's a worker built on a **stale base** whose old copy of a file `theirs`-wins over a fix master landed meanwhile. Ending with `git fetch origin master && git rebase origin/master` (then re-test) keeps every branch current so theirs-wins only ever applies to genuinely new work.
5. **Never `git add -A` in a possibly-shared context.** Stage explicit paths. Verify `git show --stat HEAD` contains only intended files before considering a task done.

> **Mental model:** treat `master` as an append-only output of a deterministic function over the `feat/*` branches. Workers produce *inputs* (small, current, lane-scoped branches); the merge-train is the *only* function that mutates the output.

### 9.2 Decompose tasks for parallelism

| Do | Don't |
|----|-------|
| Split into independent, single-lane tasks (no shared state) | Hand two agents tasks that both edit the same file |
| Keep each task small and shippable on its own | One giant task that blocks integration for an hour |
| Route by file ownership: A→`src/widget/*`, B→tests, C→`site/*` | Route by vague feature that spans the same files |
| Give one agent a "hot file" round, others new modules | Let 3 agents all touch the dashboard "a little" |

### 9.3 Worker discipline

- **Commit frequently** on the feat branch — the merge-train ships incrementally, so small commits go live faster and reduce stale-base risk.
- **Install deps first** in a fresh worktree (`bun install` / `npm ci`) — worktrees have no `node_modules`; skipping this produces phantom "module not found" test failures unrelated to the change.
- **Distinguish your failures from baseline failures.** Before claiming "tests green," run the suite on a clean checkout too — environment/credential/network test failures (403s, "network down") are not your regression. Compare counts; don't chase pre-existing red.
- **Report a verifiable signal** ("committed `<slug>` on `feat/<x>`") so the orchestrator and merge-train log can correlate.

### 9.4 Orchestrator discipline

- **Re-discover the fleet every session** — surfaces, runtimes, and rate-limit state all drift. Verify which runtime is in each pane (read the footer), not what you remember.
- **Exactly one merge-loop**, always. Check on startup and periodically.
- **Tick cadence matches activity:** ~60–90s while the fleet is busy, ~5 min when idle. Don't burn cycles polling a quiet fleet.
- **Confirm `IDLE` with a second read** before dispatching — a single read can catch a mid-transition and false-idle a working agent.
- **Don't auto-submit a worker's `PARKED` input while the human is actively steering it** — they may be mid-edit.
- **Correct stale beliefs proactively** — an agent that says "this isn't deployed" when prod is already on that commit needs a fact, not a retry.
- **Watch your own context.** The orchestrator is long-running; compact/clear before saturation or the supervision loop degrades.

### 9.5 Reliability patterns (steal these)

| Pattern | Why it matters |
|---------|----------------|
| **Watchdog kill on every git op** | One hung `git` (network stall, lock) must never freeze the integration loop. Wrap in a 120s `kill -9` child. |
| **Self-healing checkout** | A killed mid-merge leaves "needs merge" state. The loop resets/cleans at the top so it never needs a human to unstick it. |
| **Integrity gate + auto-revert** | Don't trust `theirs`-wins blindly. After each merge, assert protected invariants (feature-marker counts didn't drop, bundles still parse) and revert just the offending branch. |
| **Verify-after-write, not fire-and-forget** | A `git push` (or any publish) can 403 silently with read-only creds and *look* successful. Always re-read the remote to confirm the commit/state actually landed before reporting success. (This one cost days of "why didn't it publish" before it was caught.) |
| **`node --check` committed build artifacts** | If prod serves a committed bundle verbatim (pull, no rebuild), a broken minified bundle ships silently to every user. Syntax-check it in the gate. |
| **Event/hook state board over screen-scraping** | Scraping TUIs is brittle and runtime-specific. Where possible, have each agent emit working/idle/blocked to a shared file via hooks; the orchestrator reads facts, not pixels. |

### 9.6 Deploy safety

- **Health-check + rollback** on every restart — a bad push that crashes the service must auto-revert, not page a human.
- **Static vs code distinction:** static assets go live on pull (no restart); code changes need the restart step. If autodeploy pulls-without-restart for static, know that a code change pushed without a version bump can land un-restarted.
- **Stamp one version per integration** and key "is prod current?" off it — a single monotonic version is the cheapest possible deploy-truth signal.

### 9.7 Traps that waste hours

| Trap | Reality |
|------|---------|
| **Truncated IDs** (UUIDs, project keys) | A truncated ID often matches *nothing* and returns empty — looks like "no data" but is a lookup bug. Always use the full ID. |
| **Read-only bot/CI credentials** | The agent authors perfectly, then can't push — and may not surface the 403 loudly. Audit write access early; verify after push. |
| **Backticks/`$()` in pane sends** | Command-substitute on the *orchestrator's* host, silently mangling the dispatch (a stray substituted command can even run locally). |
| **Hardcoded surface/pane IDs** | Drift every session; re-map always. |
| **launchd/TCC on `~/Downloads`** | "Operation not permitted" reading the repo; run daemons detached from the terminal session instead. |
| **Trusting "deployed" without a fetch** | Beliefs about prod state go stale; re-read `origin/master` and the prod HEAD before asserting. |

---

## 10. Porting Checklist

- [ ] **Repo with a `master` (or `main`) trunk** and a GitHub (or equivalent) remote
- [ ] **Install the two hooks** — copy `pre-commit` + `pre-push` into `.git/hooks/`, `chmod +x`; verify a master commit is blocked and `KLAV_ORCHESTRATOR=1 git commit` is allowed
- [ ] **`scripts/new-worktree.sh`** — adjust the `../<repo>-wt-` prefix to your repo name
- [ ] **`scripts/merge-train.sh`** — set `EXCLUDE_RE`, `QUIET_MIN`, and **customize the integrity gate** (`DASH`, `PROTECTED_PATTERNS`, `WIDGET_BUNDLE`) to your project's critical files; set the manifest list for version stamping
- [ ] **`scripts/merge-loop.sh`** — set absolute `REPO` path + log location; start ONE detached instance
- [ ] **Autodeploy on prod** — poll `origin/master`, pull, restart with health-rollback (or `*/15` cron for static-only)
- [ ] **Worker rules** — write `CLAUDE.md` (+ `AGENTS.md`/`GEMINI.md` if mixing runtimes) at the workspace root with the 5-rule contract
- [ ] **Status board** — adapt `klav-status.sh` to your runtime's spinner signature + surface IDs
- [ ] **cmux (or tmux) panes** — one per worker; confirm the CLI path (`list-panes`/`read-screen`/`send`/`send-key`)
- [ ] **Orchestrator runbook** — codify startup (map fleet → one merge-loop → read prod), the status-tick, dispatch, and unblock procedures
- [ ] **Dry run** — dispatch one trivial task to one worker; confirm worktree → commit → merge-train integrates → version bumps → prod deploys end-to-end

---

## 11. Why This Beats a Branch-PR-Review Flow for Agents

| Property | PR-per-agent flow | This system |
|----------|-------------------|-------------|
| Merge conflicts | Frequent, human-resolved | Structurally impossible on master (single writer, theirs-wins) |
| Human in the loop per change | Yes (review + merge) | No (integrate + deploy are automatic) |
| Integration latency | Minutes–hours | ~25–30s |
| Versioning | Manual / CI per merge | One monotonic stamp per integration |
| Bad-merge protection | Reviewer judgement | Deterministic integrity gate + auto-revert |
| Stuck agents | Block the queue | Isolated; orchestrator unblocks or reassigns |

The cost is the `-X theirs` trade-off (stale-base regressions) — bounded by the integrity gate and lane discipline. For a fleet of autonomous agents shipping many small changes, that trade is heavily favorable.

---

## 12. Field Notes — Refinements from a Production Run

The following were learned running a 5-worker heterogeneous fleet (Claude Code · Codex · Gemini/Antigravity · a local model in opencode) through a ~21-ticket feature wave in one overnight session. They extend §4–§9 with the sharp edges that only show up under real load.

### 12.1 Make the type-check integrity gate BASELINE-AWARE (the single biggest fix)

A naive tsc gate — "run `tsc --noEmit` on the branch's changed files; block if any error is in a changed file" — **silently blocks every branch that touches a hot file** in a real codebase, because:

- The gate type-checks **isolated files**, not the whole project, so cross-file names (`import { foo } from './bar'`, types defined in sibling modules) resolve to *"Cannot find name"* false errors.
- Runtime-specific idioms (e.g. Bun accepting a `Uint8Array` as a `Response` body) are flagged by `tsc`'s stock lib even though they run fine.

Result observed: master's `server.ts` showed **18 tsc "errors"** that were all pre-existing noise, and the gate refused to merge *any* branch that edited `server.ts` — jamming a large fraction of the backlog with a misleading "stale-base regressed" log line.

**Fix — only block errors the branch *introduces*, not pre-existing ones.** Diff the error count of the changed files at `HEAD` (the merge result) against the same files on `origin/master`, checked in a throwaway worktree so sibling imports resolve identically:

```bash
# inside the integrity gate, after `git merge -X theirs "$b"`:
runner="bunx"; command -v bunx >/dev/null 2>&1 || runner="bun x"
TSC="--noEmit --pretty false --skipLibCheck --moduleResolution bundler --module esnext --target es2022 --lib es2022,dom"

# changed .ts files, EXCLUDING *.test.* (they import test-runner globals tsc can't resolve)
mapfile -t changed < <(git diff --name-only "$pre"..HEAD -- '*.ts' '*.tsx' | grep -vE '\.test\.[cm]?tsx?$')
[ "${#changed[@]}" -eq 0 ] && { : ; }   # nothing to check

$runner tsc $TSC "$shim" "${changed[@]}" >"$td/head.log" 2>&1 || true      # merged tree
git worktree add -q --detach "$td/base" origin/master 2>/dev/null && {
  base=(); for f in "${changed[@]}"; do [ -f "$td/base/$f" ] && base+=("$f"); done
  ( cd "$td/base" && $runner tsc $TSC "$shim" "${base[@]}" ) >"$td/base.log" 2>&1 || true
  git worktree remove --force "$td/base"
}
n_head=$(grep -c "error TS" "$td/head.log" || echo 0)
n_base=$(grep -c "error TS" "$td/base.log" || echo 0)
if [ "$n_head" -gt "$n_base" ]; then      # ONLY block a NET-NEW regression
  log "INTEGRITY GATE: $b increased tsc errors ($n_base -> $n_head) — reverting"
  git reset -q --hard "$pre"
fi
```

> **Principles that generalize to any linter/typechecker gate:** (1) compare against a **baseline**, never an absolute-zero-errors bar, in a codebase that isn't already clean; (2) exclude `*.test.*` from a *type* gate — test files import runner globals (`bun:test`, `node:*`) that isolated `tsc` can't resolve, and they're validated by the test run anyway; (3) resolve the baseline in a **real checkout** (throwaway worktree) so cross-file resolution matches the HEAD run and false positives cancel out. The merge-train reads its own script fresh each cycle, so shipping this fix is itself just a `feat/*` branch the train integrates.

### 12.2 Rate-limit levers beyond "reassign"

`RATE-LTD` doesn't have to mean parking the worker:

| Lever | When | How |
|-------|------|-----|
| **Model-swap within the pane** | The runtime bills **two model families to separate budgets** (e.g. Antigravity: Gemini *and* Claude budgets) | `/model` in that pane → pick the other family → resume the same task. Doubles effective runway; swap back when the first budget resets (often ~minutes). |
| **Probe at the reset ETA** | A backend showed an absolute reset time ("try again at 1:55") | The banner is **stale state from before the reset** — do NOT read it and assume still-locked. When the clock passes the ETA, **test-dispatch immediately** and read back; the only truth is whether it processes. Track each capped worker's ETA and probe then, don't wait to be told. |
| **Reassign** | No alternate budget, reset far off | Hand the task to an idle worker on another runtime. |

> A rate-limited pane often keeps showing its idle input placeholder *and* a live "Working (Ns)" indicator at the same time — the **timer is the truth**, the placeholder is just the always-rendered prompt line. Don't misclassify a working Codex pane as idle.

### 12.3 Context saturation — recovery differs per runtime

A long-running worker fills its context window and **jams** (new messages queue behind a full session; pure "thinking" with no tool calls). Recovery is runtime-specific:

| Runtime | Symptom | Recovery |
|---------|---------|----------|
| Claude Code | footer `ctx:100%`, `"Press up to edit queued messages"` | `Escape`×2 + `ctrl+u` to drop the queued input, then `/clear` **alone** + Enter; confirm the `ctx:%` disappears before re-dispatching. `/clear` wipes the conversation but **not** the worktree — uncommitted work survives. |
| Local model in opencode | frozen screen, `esc` won't interrupt, "running chat completion on N messages" | `/new` does **not** reliably clear opencode — you must **exit and relaunch the CLI** (`opencode --yolo`) for a truly fresh context. A control-plane `Ctrl+C` may not land; this often needs a human at the real terminal. |

> **Weak/local models: enforce one-task-per-context.** A small local model degrades badly as history grows (observed: wedged at ~400 accumulated messages on a *trivial* deletion). Give it a fresh session **between every ticket**, and route only mechanical, single-file work to it.

### 12.4 More gotchas (extends §8)

| # | Gotcha | Mitigation |
|---|--------|-----------|
| 13 | **Deleting a worktree a live worker is `cd`'d into** breaks its shell — every subsequent tool call fails with `FileSystem/NotFound` on the deleted path | During worktree cleanup, never remove a dir any pane currently occupies. Prune only *merged* branches' worktrees; check each isn't a live cwd first. |
| 14 | **Big multi-line dispatch pastes "collapse" and don't submit** (`"paste again to expand"`, a doubled `❯ ❯`, or an `N new messages` counter) | Press Enter again; if still staged, `Escape`×2 + `ctrl+u` and re-send a **shorter** payload. Always confirm a spinner started before moving on. |
| 15 | **Shipped tickets stay OPEN in the tracker** — the merge-train ships code but doesn't touch the issue tracker | Periodically reconcile: for each merged `feat/kla-<n>` branch, mark ticket `<n>` Done via the tracker API. Throttle bulk closes (~1.3s/call) to dodge 429s. A stale board hides real remaining work. |
| 16 | **Worktrees accumulate** (one per task, forever) — hundreds pile up | Sweep periodically: `git worktree remove` every **merged** branch's dir; for merged branches with *uncommitted* leftover edits, `git diff > save.patch` first, then force-remove. Keep only the main checkout + genuinely-unmerged branches + live workers' dirs. |
| 17 | **A stale-base branch that only edits a hot HTML file** can't be caught by a *type* gate (no `.ts` change) | Lane discipline (§6.3) is the primary defense; for critical HTML/bundle files, keep the feature-marker-count check from §4.4 alongside the tsc-baseline check. |

### 12.5 Orchestrator UX: a per-tick progress heatmap

When driving a multi-ticket wave, render a compact dot-map every status tick so the human sees velocity at a glance — `#` shipped, `*` in-flight, `·` queued — plus a rough ETA from observed throughput:

```
 WAVE (TICKET-146…166)                    # shipped   * in-flight   · queued
 146#  147#  148#  149#  150*  151*
 152#  153·  154*  155#  156#  157#
 158#  159·  160·  161#  162#  163·
 164·  165*  166·
 progress ▓▓▓▓▓▓▓▓▓░░░░░░░░░░░  9/21 shipped · ETA ~2h
```

Pair it with a one-line prod-truth check each tick (`prod HEAD == origin/master`, health `200`) so "is it actually live?" is never a guess. Keep a **light heartbeat** (a self-scheduled wake) so the loop survives even when no worker event fires.

### 12.6 The integrity gate is not enough — add a BOOT SMOKE (learned from two prod outages)

A single overnight run of ~30 tickets caused **two production outages**, both from bad code that the type/syntax gates *passed*. The fixes below make the pipeline genuinely hard to break — each was learned by getting bitten.

**Outage 1 — a syntax error slipped a count-based gate.** A `-X theirs` merge dropped a closing brace; the file wouldn't parse. The baseline-aware tsc gate (§12.1) compares *error counts* — but a fatal parse error makes tsc **bail early with FEWER errors** than the pre-existing baseline noise, so `n_head < n_base` and it shipped. Prod crash-looped.
> **Fix: hard-gate syntax-class errors independently of the count.** Syntax errors are `TS1xxx`; block on ANY net-new one regardless of total count — a file that won't parse must never merge, no matter how noisy the baseline.

**Outage 2 — a runtime boot-crash the gates can't see.** A branch parsed and type-checked fine but threw on boot (a migration referencing a column its schema never created: `no such column`). The type gate has no way to catch this. And critically: **a bad `master` defeats prod's health-rollback** — autodeploy reverts the prod *checkout* to last-good, then re-pulls the still-bad `master` next cycle and re-breaks. The rollback can't win while the trunk is poisoned.
> **Fix: a BOOT SMOKE before `git push`.** After merging + version-stamping, actually **start the server, wait for its ready-log, `curl` it for 200, then kill it** — if it doesn't come up, revert to `origin/master` and DON'T push. This catches syntax, import, *and* runtime boot-crashes. Known-good boots in ~4s locally (it can share the real DB — startup migrations are idempotent). This gate caught every subsequent boot-crash before it reached prod.
```bash
boot_smoke(){                      # runs post-merge, pre-push; return 1 ⇒ don't push
  local port=9187 lf pid ok=0
  # free the port ROBUSTLY — a stray process (even your own diagnostic) => EADDRINUSE,
  # a FALSE boot-fail that blocks EVERY branch. pkill-by-pattern is not enough:
  pkill -f "PORT=$port bun run server.ts" 2>/dev/null
  command -v lsof >/dev/null && lsof -ti:"$port" | xargs -r kill -9 2>/dev/null
  lf=$(mktemp); ( cd "$REPO/app" && PORT=$port your-run-cmd >"$lf" 2>&1 ) & pid=$!
  ( sleep 55 && kill -9 "$pid" 2>/dev/null ) &          # watchdog (no `timeout` on macOS)
  for i in $(seq 1 50); do
    kill -0 "$pid" 2>/dev/null || break                  # process died = boot crash
    grep -qE "READY_LOG_MARKER" "$lf" && { ok=1; break; }
    sleep 1
  done
  [ "$ok" = 1 ] && [ "$(curl -s -o /dev/null -w '%{http_code}' localhost:$port/)" = 200 ] || ok=0
  kill "$pid" 2>/dev/null
  [ "$ok" = 1 ] || { log "BOOT SMOKE FAILED"; tail -12 "$lf"; return 1; }
}
```

**Recovering a poisoned trunk (prod is crash-looping):** don't touch prod — fix `master`, let autodeploy redeploy. (1) `pkill` the merge-loop so it stops re-merging the bad branch; (2) roll `master` back to the last-good SHA (verify *that* boots first) and `git push --force-with-lease` as the orchestrator; (3) **quarantine** the bad branch (`git worktree remove --force` + rename `feat/*`→`wip/*`) so the restarted loop can't re-merge it; (4) autodeploy pulls the good `master` and prod recovers in ~50s; (5) restart the loop. Isolate *which* branch is bad by boot-testing each individually on top of master.

**Make crash-loops LOUD, not silent.** The service had `Restart=always RestartSec=3` with the default `StartLimitIntervalSec=10s` — at 3s/restart, 5 restarts take ~15s, so the limit *never trips* and it loops forever invisibly. Set `StartLimitIntervalSec=60 StartLimitBurst=5` so a genuine loop enters a **failed** state (a signal you can detect + rollback on) instead of thrashing silently. Add `MemoryMax`/`MemoryHigh` too, so a memory spike OOM-kills only the app's cgroup, not arbitrary system processes.

**Retire integrity markers a ticket is *supposed* to change.** The feature-marker gate (§4.4) blocked a legit ticket for dropping a protected marker count — but that ticket's whole job was to *refactor* that feature. A guard that fights intended work is worse than no guard: verify the change is good (boot it), then **retire that specific marker**. Guards protect invariants; a feature under active rework isn't an invariant.

**Meta-lesson: gates will have false-positives, and a false-positive that halts the pipeline is its own outage.** Budget for fixing the *gates* as fast as you fix the code — of the guards added this run, several needed same-night robustness fixes (the port-conflict false-fail, the `grep -c` returning `0\n0` and breaking an integer test, the over-eager marker guard). A gate you can't quickly tune becomes a bottleneck everyone routes around.

### 12.7 Fleet economics: multi-budget model-swaps and one-task-per-context locals

- **A single CLI pane can hold several independently-metered model budgets.** One agent runtime exposed Gemini, Claude (Sonnet/Opus), and an open model as *separate* quotas. When one hits "quota reached", `/model`-swap to another family and the *same task resumes* — multiplying runway instead of parking the pane. Cycle through the families on exhaustion before retiring a worker.
- **Weak/local models need one-task-per-context.** A small local model (in an opencode-style CLI) degraded badly as history grew and **wedged** at ~400 accumulated messages on a *trivial* task. Its `/new` didn't reliably clear — a true reset needed **exiting and relaunching the CLI**. Give such workers a fresh process per ticket and route only mechanical, single-file work to them.
- **Reconcile the tracker.** The merge-train ships code but never closes issues — periodically bulk-close tickets whose `feat/<id>` branch is merged (throttle API calls to dodge 429s). A stale board hides the real remaining work; one sweep closed ~90 already-shipped-but-open tickets.

---

<!-- Part of the ShipFactory feature spec library — https://github.com/vishalquantana/shipfactory -->

## About Us

We are [Quantana](https://quantana.com.au), an AI-first design and development agency working with Fortune 500s to build bespoke AI solutions and provide the audit and training needed to ensure success. [Click here to learn more](https://quantana.com.au).

## License

MIT
