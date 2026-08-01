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
- Re-run the agent's doctor/health check.
- Re-apply local patches (script), then verify the features they patch.
- If the update wiped pinned tool versions (security pins), re-pin and re-audit.
