# Verification & Health Checks

How to prove the target agent is healthy — before claiming any repair is done.

## The verification stack (run top to bottom)

1. **Agent's own doctor**: `doctor` / `status` / `config check` commands. These are the
   target agent's self-diagnostics; a repair isn't done until they're green.
2. **Database integrity**: open every state database read-only and check it —
   SQLite: `sqlite3.connect("file:<path>?mode=ro", uri=True)` then
   `PRAGMA quick_check`. Also sanity-count core tables (sessions, messages) and compare
   with expectations ("497 sessions" is an answer; "it opens" is not).
3. **Service health**: the daemon/gateway is running with a fresh PID after your
   changes, its status command reports connected, and it survives a restart cycle.
4. **Dependency audit**: the agent's supply-chain/security audit (or `pip-audit`/
   `npm audit`). Distinguish *fixable now* / *blocked by a dependency cap* / *no
   patched version exists* — report each class honestly.
5. **Domain-specific linters**: if the agent maintains a knowledge base, kanban, or
   plugin registry with its own validator, run it from the correct working directory.
6. **The original symptom**: reproduce the exact thing the user reported. Green checks
   with the symptom still present = not fixed.

## Reporting results

- Lead with the verdict: what was broken, what it is now ("doctor: all checks passed,
  0 vulnerabilities across 136 components, 497 sessions intact").
- Before/after numbers for anything quantitative (disk GB, warning counts, versions).
- State explicitly what was NOT fixed and why (upstream bug pending, blocked pin),
  with links to the upstream issue/PR.
- List every backup created and where it lives.
- End with the user's action items (restart an app to refresh its UI, delete a
  papelera folder later, re-run a script after the next update).

## Verification pitfalls

- **Piped exit codes**: `build 2>&1 | tail` returns tail's status. Capture the real
  exit code or judge by output content.
- **Cached UI state**: model pickers and dashboards may render stale config until the
  app restarts. Verify at the source (config files + service) AND tell the user to
  restart the UI before judging.
- **Console encodings** (Windows cp1252): scripts printing Unicode (✅) crash with
  `UnicodeEncodeError` and *look* like real failures. Reconfigure stdout with
  `errors="replace"` in maintenance scripts.
- **Non-interactive prompts**: CLI confirmation prompts read EOF in automation and
  self-cancel. Find the `--yes` flag; if output is empty, check you didn't swallow it
  with over-filtering.
- **Interrupted-then-resumed checks**: if you fixed phase 2 of a pipeline manually,
  re-run phases 3..N yourself — the orchestrator may consider them done.

## Upstream reporting (close the loop)

When the root cause is an agent bug, not local damage:

- Search existing issues first; comment/+1 with your repro instead of duplicating.
- File with: exact versions + commit, OS, minimal repro steps, logs, and the
  workaround you used. Offer the fix as a PR when you have one (with a regression test).
- Track the issue/PR; when it merges, retire your local patches and re-apply scripts.
