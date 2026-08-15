# Update & Updater Repair

Fixing a target agent whose update mechanism fails, hangs, or leaves the install broken.

## Triage order

1. **Read the updater's own output/log first.** Most updaters log to a file
   (`logs/update.log` or similar) even when the UI only shows a generic dialog.
2. **Identify which phase failed**: fetch (git/network) → code apply → language deps
   (pip/npm/cargo) → asset builds (web UI, desktop) → service restart. Fix only the
   failed phase; don't reinstall the world.
3. **Check upstream before deep-diving**: search the project's issues/PRs for the exact
   error. A failure that started right after an update is often a known regression —
   reporting it (or finding the open fix PR) beats patching blind.

## Processes blocking the update

Symptoms: "another process is using this installation", "Access denied" on package
files, updater aborts listing PIDs.

- Enumerate holders: find processes whose command line references the agent's
  install/venv path (`Get-CimInstance Win32_Process` on Windows; `lsof`/`fuser` on Unix).
- Stop the agent's **service form** first (gateway/daemon), then GUI, then strays.
- **Find the auto-restarter**: scheduled tasks, systemd units, login items, startup
  scripts. A service with `RestartCount=999` will respawn mid-update and re-block it.
  Pause the restarter, update, re-enable.
- Watch for **spawn paths the updater's own discovery misses** (e.g. processes started
  with `--replace` flags, or shim-wrapped children where the parent exe differs). The
  updater may pause "all" its services and still get blocked by one of these.
- If a blocker is unkillable by owner *and* admin (state Unknown, 0% CPU): it's
  kernel-wedged. Reboot. Do not spend hours on it.

## The updater must never run from the binary it replaces

The classic Windows trap: the agent's CLI is a console-script shim / zipapp trampoline
(`Scripts\agent.exe`), and `agent update` launched from it must *rewrite that same exe*.

- **Error semantics tell you the holder**: "os error 5 / Access denied" on delete = a
  *running image* (delete blocked, rename allowed). "os error 32 / sharing violation" =
  some process holds an *open handle without FILE_SHARE_DELETE* (rename ALSO fails —
  quarantine/rename dances die with PermissionError).
- A zipapp-style shim is both at once: the trampoline maps the image AND the child
  interpreter keeps/reopens a read handle on the exe for lazy imports mid-install.
  Result: intermittent error-32 failures that correlate with machine load, not with
  anything the user did. Slow/busy machines reproduce; fast laptops don't.
- **The fix is always the same**: relaunch the update via the interpreter —
  `venv\Scripts\python.exe -m <cli_module> update` (or the equivalent `node script.js`,
  never the packaged shim). Check every entry path: user shell, GUI update button,
  handoff scripts, self-repair. A traceback showing `...\agent.exe\__main__.py` is the
  smoking gun that an invocation still uses the shim.
- GUI updaters often "verify the exe is unlocked, then run the update *from that exe*"
  — the preflight passes and the update still dies. Patch the handoff to use the
  interpreter and add the patch to the idempotent re-apply script.
- Self-lock *preflights* have their own failure mode: if any module in the CLI's
  import chain loads a native extension at module scope (a crypto lib pulled in by a
  plugin registry, say), the guard trips on EVERY update and defers/refuses forever.
  Trace with `python -X importtime -c "import <cli_module>"`, then make the heavy
  import lazy (move it inside the functions that use it) and upstream that fix.
- When upstream adopts one of your local fixes (pins, handoff patches), **retire the
  local version the same day** — a redundant re-pin or re-patch becomes *drift* that
  newer updaters detect and fight, turning yesterday's medicine into today's disease.
  Replace retired patches with a cheap regression check that alerts if upstream reverts.

## Interrupted self-repair: markers and orphan locks

Updaters leave breadcrumbs (`.update-incomplete` + a `.lock` claimed with the healer's
PID). Know the lifecycle:

- Marker present + lock whose PID is **dead** = orphaned: every launch *silently skips*
  healing until a stale-lock timeout (often 1 h). Check the PID inside the lockfile;
  delete only the orphan lock, then trigger healing via a normal command and let it
  **finish**.
- **Never truncate a live updater/healer through pipeline operators that close the
  pipe** (`Select-Object -First`, `head`): they kill the child mid-install and create
  the orphaned state above. Use `-Last`, a file redirect, or a background task.
- Trivial commands (`--version`) may exit before the healing hook runs; trigger healing
  with a real subcommand (e.g. the service status command).

## Dependency-phase failures

- **Engine/version gates** (e.g. `EBADENGINE: requires node >=X`): the runtime, not the
  agent, is outdated. Upgrade the runtime (verify checksum of the download), then re-run
  only the dependency step. Portable runtime installs can be upgraded in place: rename
  the running binary (processes keep it), copy the new one in, replace only the package
  manager's own module directory — user-installed global packages survive.
- **Native-extension corruption** (missing RECORD/METADATA after a failed upgrade):
  delete the package's directories from site-packages entirely, then clean-install the
  pinned version. Root cause is almost always "upgraded while the agent was running".
- **Package-manager override quirks**: some managers silently ignore changed dependency
  overrides while old artifacts exist. If an override "doesn't take", remove lockfile
  *and* all module directories (including workspaces), then reinstall.
- **Skipped-step traps**: updaters cache "nothing changed" markers (lockfile hashes,
  build stamps). After you fix a failed phase manually, the updater may skip it forever.
  Run the underlying commands yourself with the same flags the updater uses (read its
  source to find them).

## Local patches vs updates

- Updater autostash: local changes to the agent's checkout get stashed on update and
  may be dropped or conflict on restore. Treat all local patches as temporary.
- Keep an **idempotent re-apply script** for necessary local patches; run it after
  every update. Retire it the moment the fix lands upstream.
- If history diverged (upstream force-push): `git fetch` + `git reset --hard
  origin/<default-branch>` is the fix — after confirming no unpushed local commits
  worth saving.

## After the update

- Restart the service form deliberately; verify PID and health endpoint/status command.
- **Restarting the GUI can kill the service**: desktop shells that manage a backend may
  kill process trees on startup/shutdown. After any GUI restart, re-check the service.
- Re-run the agent's doctor/health check.
- Re-apply local patches (script), then verify the features they patch.
- If the update wiped pinned tool versions (security pins), re-pin and re-audit.
