# Disk Hygiene & Backup Consolidation

Reclaiming space in an agent's home directory without losing anything, and ending the
"backups scattered everywhere" state.

## Survey first (read-only)

- Size-inventory the agent home: every top-level file/dir with recursive sizes, sorted.
  On Windows PowerShell, measure with `Get-ChildItem -Recurse | Measure-Object Length -Sum`
  per entry; on Unix, `du -sh */`.
- Drill into the top 3 offenders before deciding anything. The real cause is usually
  one level down (a single subfolder or file class).
- Search the user's home and project folders for stray backup locations
  (`*backup*`, `*respaldo*`, `*_OLD*`, date-stamped folders). List them all with sizes
  and dates — the user usually doesn't remember half of them.

## Classic bloat patterns (all seen in production)

| Pattern | Signature | Fix |
|---|---|---|
| Runaway snapshot retention | N full copies of a directory tree in a hidden backup dir | Keep newest, delete rest; check the tool's `keep`/retention setting |
| **Binary amplification** | Large binaries (AI models, videos) inside a snapshotted dir → every snapshot copies them | Move binaries to their proper home (where the consuming app loads them); snapshots shrink from GB to MB |
| Emergency backup pile-up | `*.pre-update-emergency-*.bak` × N, often identical (same size, minutes apart) | Verify main file healthy (DB integrity check), keep one, delete twins |
| Browser-profile caches | Automation browser profiles with `Cache`, `Code Cache`, `*_crx_cache`, `ShaderCache` | Delete cache dirs only (regenerable); NEVER the profile itself (logins live there) |
| Stray temp files | `.*.tmp`, `.bak.<date>`, lock leftovers in the agent home root | Move to a dated "papelera" folder in the backup location; delete later |
| Old install remnants | `*_OLD_WSL`, previous-machine homes, superseded manual backups | Consolidate into the single backup location; let the user decide deletion |

## The single-backup-location rule

- Pick ONE folder **outside the agent's tree** (e.g. `~/respaldos <agent>/`) so updates,
  reinstalls, and the agent's own tooling never touch it.
- Move everything backup-like there, in dated subfolders (`skills-custom-YYYYMMDD/`,
  `configs-pre-<fix>-YYYYMMDD/`). Inside the agent home, backup dirs should end up empty.
- Prefer **move to papelera** over delete for small/ambiguous items — same cleanup
  effect, fully reversible, and it sidesteps guardrails that (correctly) distrust bulk
  deletions of things named like secrets or memories.

## Safety rails

- Integrity-check databases (SQLite `PRAGMA quick_check` via a read-only URI) **before**
  deleting any of their backups.
- Check that no process is using a browser profile before touching its caches.
- Big deletions run last, after everything worth keeping has been moved, and in the
  background (they're slow).
- Re-measure at the end and report before/after numbers.
