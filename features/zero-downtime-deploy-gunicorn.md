# Zero-Downtime Deployments (Gunicorn Graceful Reload)

## Feature Specification for Porting

This document describes how to make deploys of a Python **ASGI** app (FastAPI/Starlette on **Uvicorn**) behind **nginx** on **systemd** cause **~0 downtime** — no dropped requests, no 502s during a release. It replaces the usual "swap files + `systemctl restart`" (which briefly kills the app) with a **graceful reload** where new workers spawn and old ones drain with overlap.

It is written as a drop-in addition to a file-based deploy script (`deploy.sh` that rsyncs a build to a server and promotes it under a lock), and is **opt-in** behind a flag so nothing changes for existing deploys until you turn it on.

---

## 1. Overview

**The problem.** A deploy applies new code, then runs `systemctl restart <service>`. With plain Uvicorn (`uvicorn main:app --workers 2`) this kills **all** workers at once; nginx returns **502 Bad Gateway** for the ~5–15 seconds until the app reboots and passes health. Anyone using the app during that window hits an error.

**The fix.** Run the app under **Gunicorn with Uvicorn workers** and use Gunicorn's **graceful reload** (`SIGHUP`):
- On `HUP`, Gunicorn **spawns a new generation of workers** (which import the new code), and only **after they're up** does it retire the old ones — so there is always a live worker behind nginx. **The socket/port never goes down.**
- The deploy calls `systemctl reload` (which sends `HUP`) instead of `systemctl restart`.
- **nginx is never touched** — it keeps proxying to the same `127.0.0.1:<port>` throughout. (Important if your nginx config is hand-edited / drifted from the repo.)

**Cost.** A **one-time** switch from Uvicorn to Gunicorn requires a single restart (do it at low traffic). Every deploy after that is a graceful reload with no downtime.

**Layman's version.** Instead of *closing the shop to retrain the staff*, you bring the **new shift** in, let them start serving, and the **old shift** finishes their current customers and clocks out. For a moment both shifts work at once, so the door is never locked.

---

## 2. How It Works

```
                       ┌─────────┐
   visitors ──────────▶│  nginx  │  proxy_pass 127.0.0.1:5999   (UNCHANGED throughout)
                       └────┬────┘
                            │ always a live worker behind here
                            ▼
        ┌───────────────────────────────────────────────┐
        │  Gunicorn master (PID tracked by systemd)      │
        │                                                │
        │   BEFORE reload:   [worker A1] [worker A2]     │  ← serving old code
        │                                                │
        │   systemctl reload → SIGHUP to master          │
        │                                                │
        │   DURING reload:   [A1][A2]  +  [B1][B2]        │  ← overlap: both serve
        │                     (old drain)  (new boot new code)
        │                                                │
        │   AFTER reload:              [worker B1][B2]    │  ← old retired, new code
        └───────────────────────────────────────────────┘

   No moment where zero workers are listening on :5999  →  no 502s.
```

Key property: **`--preload` must be OFF.** Without preload, each worker imports the app itself, so freshly-spawned workers on `HUP` pick up the new code. With `--preload`, the app is imported once in the master and `HUP` would **not** load new code.

---

## 3. The One-Time Setup — systemd drop-in

Do **not** rewrite the whole unit. Use a systemd **drop-in** that overrides only `ExecStart` and adds `ExecReload`, preserving the base unit's `User`, `EnvironmentFile`, `WorkingDirectory`, etc.

Baseline unit (example) runs:
```ini
ExecStart=/opt/app/venv/bin/python3 -m uvicorn main:app --host 127.0.0.1 --port 5999 --workers 2
```

Create `/etc/systemd/system/<service>.d/10-zero-downtime.conf`:
```ini
[Service]
# reset the base ExecStart, then run gunicorn + uvicorn workers.
# NO --preload → SIGHUP reloads app code. Base unit User/EnvironmentFile are kept.
ExecStart=
ExecStart=/opt/app/venv/bin/gunicorn main:app -k uvicorn.workers.UvicornWorker -w 2 -b 127.0.0.1:5999 --timeout 120 --graceful-timeout 30
ExecReload=/bin/kill -s HUP $MAINPID
KillMode=mixed
TimeoutStopSec=35
```
Then:
```bash
/opt/app/venv/bin/python3 -m pip install "gunicorn>=21"   # see Gotcha 2
systemctl daemon-reload
systemctl restart <service>        # the ONE-TIME restart — run at 0 users
```

Notes:
- `Type=simple` + Gunicorn is correct: the Gunicorn **master stays in the foreground**, so systemd tracks it as `$MAINPID` and `ExecReload=kill -HUP $MAINPID` reloads it.
- `--graceful-timeout 30` gives draining workers up to 30s to finish in-flight requests.
- Make the switch **self-healing**: if Gunicorn doesn't pass health after the restart, remove the drop-in, `daemon-reload`, and restart back onto Uvicorn.

---

## 4. Deploy-Script Integration (`deploy.sh`)

Add an **opt-in flag** and two changes. Default deploys stay byte-identical.

**a) Flag + default**
```bash
# with the other arg defaults:
ZDT="${ZERO_DOWNTIME:-0}"
# in the arg-parse loop:
-z|--zero-downtime) ZDT=1 ;;
```

**b) Idempotent one-time setup** (run on a real apply, before staging; a no-op once configured):
```bash
ensure_zdt_setup() {
  $SSH APP_DIR="$APP_DIR" SERVICE="$SERVICE" bash -s <<'ZDTSETUP'
set -euo pipefail
DROPIN_DIR="/etc/systemd/system/${SERVICE}.d"; DROPIN="$DROPIN_DIR/10-zero-downtime.conf"
GUNI="$APP_DIR/venv/bin/gunicorn"
if [ -f "$DROPIN" ] && [ -x "$GUNI" ] && systemctl is-active --quiet "$SERVICE"; then
  echo "[zdt] already configured"; exit 0; fi
"$APP_DIR/venv/bin/python3" -m pip install --quiet "gunicorn>=21" || { echo "[zdt] pip failed"; exit 1; }
mkdir -p "$DROPIN_DIR"
cat > "$DROPIN" <<UNIT
[Service]
ExecStart=
ExecStart=$GUNI main:app -k uvicorn.workers.UvicornWorker -w 2 -b 127.0.0.1:5999 --timeout 120 --graceful-timeout 30
ExecReload=/bin/kill -s HUP \$MAINPID
KillMode=mixed
TimeoutStopSec=35
UNIT
systemctl daemon-reload
systemctl restart "$SERVICE"        # one-time switch
ok=0; for i in $(seq 1 15); do sleep 2;
  [ "$(curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:5999/api/health)" = "200" ] && { ok=1; break; }; done
if [ "$ok" != "1" ]; then           # self-heal: revert to uvicorn
  rm -f "$DROPIN"; systemctl daemon-reload; systemctl restart "$SERVICE"; exit 1; fi
echo "[zdt] zero-downtime service ready"
ZDTSETUP
}
# call it on a real apply only:
[ "$ZDT" = "1" ] && [ "$APPLY" = "1" ] && ensure_zdt_setup
```

**c) Promote: reload instead of restart** (inside the server-side promote step, after files are staged + import-smoked):
```bash
if [ "$ZDT" = "1" ]; then
  echo "[promote] graceful reload (SIGHUP — overlapping workers)…"
  systemctl reload "$SERVICE" || { echo "[promote] reload failed — restart"; systemctl restart "$SERVICE"; }
else
  systemctl restart "$SERVICE"
fi
# then the existing health-check + rollback loop (unchanged)
```

Usage: `deploy.sh prod --apply -z` (first run does the one-time switch; later runs are graceful).

---

## 5. Safety / Rollback (keep your existing guards)

The zero-downtime path composes with a standard promote's safety net:
1. **Import-smoke** before swapping traffic: `python3 -c "import main"` on the staged code — a syntax/import error aborts and rolls back the files, no reload happens.
2. **Health-gate**: after the reload, poll `/api/health`; if it never returns 200, restore the backed-up files and `restart`.
3. **Setup self-heal**: if the one-time Gunicorn switch is unhealthy, the drop-in is removed and the service reverts to Uvicorn automatically.

Net effect: a broken build **reverts itself** instead of taking the site down — the opposite of the old "restart and hope."

---

## 6. Requirements / Assumptions

- App is **ASGI** (FastAPI/Starlette) served by **Uvicorn** today; `uvicorn.workers.UvicornWorker` is importable.
- Process manager is **systemd**; you can add a drop-in and `systemctl reload/restart`.
- **nginx** (or any reverse proxy) sits in front on a **fixed local port** and stays pointed there.
- App **loads its own env** (e.g. `load_dotenv`) or the base unit's `EnvironmentFile` is preserved by the drop-in.
- App startup is **idempotent** across workers (e.g. `CREATE TABLE IF NOT EXISTS` boot DDL) — multiple workers already boot concurrently under `--workers 2`, so graceful reload adds no new concurrency risk.

---

## 7. Gotchas (learned during the live rollout)

1. **`--preload` breaks code reload.** With preload, the app is imported in the master; `HUP` respawns workers but they inherit the preloaded module → old code. Leave it off.
2. **The venv's `pip` wrapper can have a broken shebang** (relocated venv): `bin/pip: cannot execute: required file not found`. Always install via `venv/bin/python3 -m pip …`, not `venv/bin/pip …`.
3. **Bash `set -u` + a variable glued to a multibyte char.** `"…on $TARGET…"` (an ellipsis right after `$TARGET`) is parsed as the variable `TARGET…` → `unbound variable`. Brace-delimit: `${TARGET}`.
4. **`Type=notify` would break this.** Gunicorn doesn't `sd_notify` by default; the drop-in assumes `Type=simple`. If the base unit is `notify`, add `Type=simple` to the drop-in or run Gunicorn with an sd-notify shim.
5. **Don't touch nginx** if it's hand-edited/drifted from your repo — this design deliberately avoids nginx changes (no blue-green port flip), so a drifted proxy config is a non-issue.
6. **Blue-green is the alternative** for true zero-downtime *without* switching app servers: run the new version on a second port, health-check it, flip the nginx upstream, `nginx -s reload`, stop the old. It's more robust but requires an nginx `upstream` change — only worth it if you can't use Gunicorn or need instance-level isolation.

---

## 8. Porting Checklist

- [ ] Confirm app is Uvicorn/ASGI on systemd behind a reverse proxy on a fixed port.
- [ ] Add the `-z/--zero-downtime` flag + `ensure_zdt_setup` (drop-in installer, self-healing) to your deploy script.
- [ ] Switch the promote step to `systemctl reload` (with `restart` fallback) when the flag is set.
- [ ] Keep import-smoke + health-gate + file rollback around the reload.
- [ ] Run the **first** `-z` deploy at low traffic (the one-time Uvicorn→Gunicorn switch does a single restart).
- [ ] Verify: `systemctl show <svc> -p ExecStart` shows gunicorn; a subsequent deploy logs "graceful reload" and health stays 200 with no 502s.

---

## 9. Related

- Pair this with an **active-user deploy guard** (warn/abort a deploy when real users are online) for defense-in-depth — though with zero-downtime reload the guard becomes a courtesy rather than a necessity.
- **Divergence warning:** if you deploy from a `dev` branch, ensure `dev` isn't missing hotfixes that live only on `main`/prod (merge `main` into `dev` first) — otherwise a full-tree deploy can *revert* prod fixes regardless of how graceful the reload is.
