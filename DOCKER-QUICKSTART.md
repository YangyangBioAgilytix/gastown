# Gas Town — Docker Quickstart (automated)

**For an AI agent or a human re-spinning this service.** Everything here is scriptable. The
irreducibly-manual bits (Docker Desktop, Claude login, the rig URL) live in
[`DOCKER-MANUAL-SETUP.md`](DOCKER-MANUAL-SETUP.md) — do those **once** before/after this.

> This is a **machine-local** runbook (contains this host's paths/identity). Safe to gitignore.

## TL;DR

1. Ensure the one-time prerequisites in `DOCKER-MANUAL-SETUP.md` are met (Docker Desktop running, WSL integration on).
2. Run the script below **inside your WSL2 Ubuntu shell**.
3. Do the 3 interactive steps in `DOCKER-MANUAL-SETUP.md` (Claude login → `gt rig add` → `gt mayor attach`).

## The script (run inside WSL2 Ubuntu-24.04)

Save as `spinup.sh`, then `bash spinup.sh`. Idempotent — safe to re-run.

```bash
#!/usr/bin/env bash
set -euo pipefail

# ---- edit if replicating elsewhere; pre-filled for this machine ----
REPO_DIR="/mnt/i/BioAgilytix/gastown"   # gastown source (Windows I:\ seen from WSL)
HQ_DIR="/home/liuman/gt-hq"             # HQ on Linux fs (fast I/O) -> mounts to /gt
GIT_USER="YangyangBioAgilytix"
GIT_EMAIL="yliu5054@gmail.com"
DASHBOARD_PORT="8091"                   # host port -> container 8080 (8080 is taken here)
# -------------------------------------------------------------------

cd "$REPO_DIR"

# 0. Fail fast if the daemon isn't reachable inside WSL.
docker version --format '{{.Server.Version}}' >/dev/null 2>&1 || {
  echo "Docker daemon not reachable in WSL. Start Docker Desktop and enable WSL integration."
  echo "See DOCKER-MANUAL-SETUP.md."; exit 1; }

# 1. CRITICAL: strip CRLF from the entrypoint. System-wide core.autocrlf=true re-adds it
#    on every git checkout, which makes tini fail: 'exec /app/docker-entrypoint.sh: No such file'.
sed -i 's/\r$//' docker-entrypoint.sh

# 2. Write .env (LF endings).
cat > .env <<EOF
GIT_USER=$GIT_USER
GIT_EMAIL=$GIT_EMAIL
FOLDER=$HQ_DIR
DASHBOARD_PORT=$DASHBOARD_PORT
EOF

# 3. HQ dir must exist and be empty (first run) or an existing HQ.
mkdir -p "$HQ_DIR"

# 4. Build + start.
docker compose build
docker compose up -d

# 5. Wait for the entrypoint's `gt install` to finish.
echo "Waiting for HQ init..."
for i in $(seq 1 150); do
  if docker compose logs gastown 2>&1 | grep -qi "HQ created successfully"; then
    echo "HQ ready"; break; fi
  if [ "$(docker inspect -f '{{.State.Running}}' gastown-sandbox 2>/dev/null)" != "true" ]; then
    echo "Container exited early:"; docker compose logs --tail=40 gastown; exit 1; fi
  sleep 2
done

# 6. In-container bootstrap: enable, shell integration, start services, self-heal.
docker compose exec -T gastown bash -lc '
  gt enable &&
  gt shell install &&
  gt up --restore &&
  gt doctor --fix
'

echo ""
echo "OK: base service is up (gt doctor should read all-pass)."
echo "Next: DOCKER-MANUAL-SETUP.md -> Claude login, gt rig add <url>, gt mayor attach."
```

## Expected end state

- Container `gastown-sandbox` **Up**, ports `0.0.0.0:8091->8080/tcp`.
- `gt doctor` → **93 passed / 0 failed** (a few auto-fixes on first run are normal).
- Services: Dolt, Daemon, Deacon, Mayor. tmux sessions `hq-mayor`, `hq-deacon`, `hq-boot`.

## Lifecycle (from `$REPO_DIR` in WSL)

```bash
docker compose stop      # pause, keep everything
docker compose start     # resume
docker compose down      # remove container, keep volumes + HQ
docker compose down -v   # also drop named volumes (host HQ folder survives)
```

## If something breaks

| Symptom | Cause / fix |
|---|---|
| `exec /app/docker-entrypoint.sh: No such file or directory` | CRLF crept back in. Re-run step 1 (`sed`) and `docker compose build`. |
| `Bind for 0.0.0.0:8091 failed: port is already allocated` | Another process/container holds the port. Change `DASHBOARD_PORT` and re-run. |
| `docker: command could not be found in this WSL 2 distro` | WSL integration off — see `DOCKER-MANUAL-SETUP.md`. |
| Daemon not reachable | Start Docker Desktop; wait until it reports "running". |
