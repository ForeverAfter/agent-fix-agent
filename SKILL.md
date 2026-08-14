---
name: agent-fix-agent
description: "Maintain, diagnose, and repair another AI agent's installation — updates, disk hygiene, skills, configs — without losing user data."
version: 0.2.0
author: ForeverAfter (Jorge Cordero) + Claude Code
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [agent-maintenance, diagnostics, repair, configuration, updates, cross-agent]
    category: autonomous-ai-agents
    related_skills: [hermes-agent, codex]
---

# Agent Fix Agent

One AI agent maintains and repairs another AI agent's installation: Claude Code fixing
Hermes, Hermes fixing Claude Code, either fixing Codex, Gemini CLI, or any locally
installed agent. Born from a real multi-week maintenance saga (broken updaters, zombie
processes, 13 GB of hidden backup bloat, frozen skills, duplicated providers) — every
procedure here was used in production at least once.

**The core insight:** an agent's installation is a living system — updater + services +
skills + configs + state databases — and it degrades in predictable ways. A *second*
agent is the ideal repair technician: it can inspect files the broken agent can't reach
while running, kill the broken agent's processes, and verify from outside.

## When This Skill Activates

- The user asks to fix, repair, audit, or maintain another agent's installation
- Another agent's updater fails, its service won't start, or its UI shows errors
- The user asks for a "health check", "mantenimiento", or cleanup of an agent
- Disk usage of an agent's home directory grows unexpectedly
- An agent's skills/plugins/configs behave inconsistently across profiles

## Prime Directives (non-negotiable)

1. **Never lose user data.** Sessions, memories, configs, and API keys survive every
   operation. When in doubt, back up first and prefer *move* over *delete*.
2. **Diagnose before touching.** Reproduce or observe the failure, read logs, and state
   a root-cause hypothesis before changing anything.
3. **Backup → change → validate → restart → verify.** Every config or file change
   follows this cycle. No exceptions.
4. **Ask before destructive or irreversible actions** (deleting data, resetting
   customized components, publishing anything). Batch the questions; don't drip them.
5. **The patient keeps running.** Whenever possible, repair without extended downtime;
   stop services deliberately, never leave them half-dead.
6. **Document what you learned.** Fixed something non-obvious? File it in the user's
   knowledge base / wiki and, if it's an upstream bug, report it upstream.

## The Maintenance Loop

```
1. INTAKE     What is broken or requested? Capture exact errors/screenshots.
2. SURVEY     Health checks, versions, logs, disk inventory (read-only).
3. DIAGNOSE   Root cause per finding. Distinguish upstream bugs from local damage.
4. PLAN       Ordered fix list; mark destructive steps; get user approval.
5. EXECUTE    Apply with the backup→change→validate cycle, one concern at a time.
6. VERIFY     Re-run the checks that failed; confirm services and UI healthy.
7. DOCUMENT   Wiki/knowledge base entry + upstream issue/PR when applicable.
```

## Routing Table — load the reference for the task

| Task at hand | Load |
|---|---|
| Updater fails, locked files, processes blocking, version conflicts | `references/update-repair.md` |
| Disk bloat, runaway backups, caches, misplaced binaries | `references/disk-hygiene.md` |
| Skills/plugins duplicated, frozen updates, custom-vs-stock conflicts | `references/skills-audit.md` |
| Config duplicates, obsolete entries, version drift across profiles | `references/config-audit.md` |
| Health checks, DB integrity, security audit, final verification | `references/verification.md` |

Load only what the task needs — this hub plus one reference is usually enough.

## Hard-Won Invariants (apply everywhere)

- **Windows locks native extensions**: never let a package manager touch packages with
  compiled extensions (.pyd/.node) while the target agent runs — it corrupts the
  package metadata. Stop the agent first.
- **Auto-restarting services fight you**: gateways/daemons installed as scheduled tasks
  or login items respawn after you kill them. Find and pause the restarter, not just
  the process.
- **Updates wipe local patches**: any local modification to the agent's own code is
  temporary. Keep an idempotent re-apply script, or upstream the fix.
- **"Modified" flags lie**: line-ending conversion (CRLF/LF) makes files look edited
  when they aren't. Diff with newline normalization before believing any
  "user-modified" marker.
- **Exit codes lie in pipelines**: `cmd | tail` reports tail's exit code. Judge build
  and install steps by their output, not the pipeline's status.
- **A truly unkillable process means reboot**: if the owner + admin cannot kill a
  process (state Unknown, 0 CPU), it is kernel-wedged. Stop burning time; reboot.
- **Custom content never lives inside managed components**: knowledge goes to the
  user's wiki/KB, machine-specific behavior goes to companion skills, working
  artifacts go to project folders. Managed (bundled) components stay stock.
- **An updater must never run from the binary it replaces**: shim/trampoline exes hold
  handles on themselves (error 32 on Windows). Always relaunch via the interpreter
  (`python -m <module>`) — in every entry path, including the GUI's handoff script.
- **Error 5 vs error 32 name the culprit**: 5 = running image (rename works, delete
  doesn't); 32 = open handle without share-delete (rename fails too). Diagnose from the
  number before hunting processes.
- **Never truncate a live installer's output** with pipe-closing operators
  (`Select-Object -First`, `head`) — the broken pipe kills the child mid-install and
  leaves orphaned recovery markers/locks that silently block self-healing.
