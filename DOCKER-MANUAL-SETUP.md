# Gas Town — Docker Manual Setup (things only a human can provide)

Companion to [`DOCKER-QUICKSTART.md`](DOCKER-QUICKSTART.md). Everything below either needs a
GUI, an interactive login, or a decision. Do the **one-time prerequisites** once per machine;
do the **interactive steps** after each fresh `up`.

---

## A. One-time prerequisites (per machine)

1. **Docker Desktop** installed and **running** (WSL2 backend). Start the app; wait until it
   reports "running" (`docker version` inside WSL must return a `Server:` version).

2. **WSL2 with an Ubuntu distro** (this machine: `Ubuntu-24.04`, user `liuman`).

3. **Enable WSL integration for that distro** (this is what bit us — it was off):
   - Docker Desktop → **Settings → Resources → WSL Integration** → toggle **Ubuntu-24.04** on → **Apply & Restart**.
   - Equivalent without the GUI: add the distro to `integratedWslDistros` in
     `%APPDATA%\Docker\settings.json` (quit Docker Desktop first, edit, relaunch):
     ```json
     "integratedWslDistros": [ "Ubuntu-24.04" ],
     ```
   - Verify: `wsl -d Ubuntu-24.04 -- docker version` shows a **Server** section.
   - *Already enabled on this machine — you shouldn't need to redo it.*

4. **(Recommended) Prevent the CRLF recurrence permanently.** System-wide
   `core.autocrlf=true` re-adds CRLF to `*.sh` on every checkout, re-breaking the container
   entrypoint. Add a repo `.gitattributes` so shell scripts stay LF:
   ```
   *.sh text eol=lf
   ```
   Then renormalize once: `git add --renormalize . && git commit -m "Force LF on shell scripts"`.
   Until this is committed, the Quickstart's `sed` step is the safety net.

---

## B. Interactive steps (after `DOCKER-QUICKSTART.md` brings the base service up)

Open a shell into the running container (from any WSL2 Ubuntu terminal):

```bash
docker exec -it gastown-sandbox zsh
```

### 1. Log in to Claude (subscription)

Inside the container:

```bash
claude
```
Follow the prompt (or type `/login`) → choose your **Claude subscription account** → it prints
a URL + code → open the URL in your Windows browser → authorize → paste the code back.
Persists in the `agent-home` volume (**one-time**). Then `exit` the `claude` REPL.

> Alternative (not chosen here): set `ANTHROPIC_API_KEY` in `.env` for per-token API billing
> instead of the subscription.

### 2. Add your project repo as a rig  ← **YOU MUST PROVIDE THE URL**

```bash
# public repo:
gt rig add <name> <git-url>          # name = letters/digits/underscores only (no hyphens)

# private repo: authenticate first, then add:
gh auth login
gt rig add <name> <git-url>
```

*Fill in:* repo URL = `__________________________`  ·  rig name = `__________`  ·  private? `Y/N`

### 3. Enter the Mayor and start working

```bash
gt mayor attach
```

### (Optional) Web dashboard

Inside the container: `gt dashboard` → then open `http://localhost:8091` on Windows.
Nothing serves on 8091 until `gt dashboard` is running.

---

## C. Values used for this deployment (fill in for a new machine)

| Input | This machine | Notes |
|---|---|---|
| WSL distro / user | `Ubuntu-24.04` / `liuman` | Run compose here, not PowerShell |
| Repo dir (from WSL) | `/mnt/i/BioAgilytix/gastown` | gastown source |
| HQ dir (`FOLDER`→`/gt`) | `/home/liuman/gt-hq` | Linux fs; empty or existing HQ |
| `GIT_USER` / `GIT_EMAIL` | `YangyangBioAgilytix` / `yliu5054@gmail.com` | git + Dolt commit identity |
| Dashboard host port | `8091` | 8080 is taken by another container here |
| Claude auth | Subscription login | Interactive `/login` |
| Rig repo URL | **(pending — you provide)** | + `gh auth login` if private |
