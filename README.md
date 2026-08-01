# agent-fix-agent

A skill that turns any AI agent into a **repair technician for another AI agent**.

Claude Code fixing Hermes. Hermes fixing Claude Code. Either one fixing Codex, Gemini
CLI, or whatever agent you run locally. Updates that abort, services that respawn
mid-repair, 13 GB of invisible backup bloat, skills frozen by accidental edits,
duplicated providers in the model picker — this skill encodes the full maintenance
playbook, distilled from a real multi-week production saga.

## What's inside

```
SKILL.md                      # Hub: directives, maintenance loop, routing table
references/
  update-repair.md            # Updater failures, blocked processes, dependency phases
  disk-hygiene.md             # Backup consolidation, bloat patterns, safety rails
  skills-audit.md             # Frozen skills, CRLF false-positives, extract→restore
  config-audit.md             # Config rot lifecycle, profile drift, migrate warnings
  verification.md             # Health checks, honest reporting, upstream loop
```

The hub stays small; the agent loads only the reference the task needs
(progressive disclosure — no context saturation).

## Install

**Hermes Agent** — copy the folder into your skills directory, then `/reload-skills`:

```bash
cp -r agent-fix-agent ~/.hermes/skills/autonomous-ai-agents/agent-fix-agent
```

**Claude Code** — copy into your skills directory (project `.claude/skills/` or
user-level skills dir), or reference it from your CLAUDE.md.

Any agent that reads `SKILL.md`-style skills (frontmatter + markdown) can use it as-is.

## Principles

1. Never lose user data — back up first, prefer move over delete.
2. Diagnose before touching; root cause before fixes.
3. Backup → change → validate → restart → verify. Every time.
4. Ask before destructive actions; batch the questions.
5. Document what you learned — in the user's wiki and upstream.

## License

MIT — see [LICENSE](LICENSE).

---

*Nacida de mantener Hermes Agent con Claude Code (y pronto, viceversa).*
