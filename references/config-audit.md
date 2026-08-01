# Config Audit

Finding and fixing duplicated, obsolete, and drifted configuration in an agent's
config files — across every profile it has.

## The lifecycle that creates config rot

1. User configures a capability **manually** (custom provider, script, workaround).
2. A later agent version ships the same capability **natively**; the user adopts it.
3. The manual config stays behind — now redundant, sometimes conflicting (duplicate
   entries in pickers, shadowed keys, double registrations).

Every "the UI shows X twice" report is this lifecycle until proven otherwise.

## Audit procedure

1. **Reproduce the symptom** and identify the duplicated identity (e.g. a provider
   appearing twice in a model selector — note both display names exactly).
2. Grep all config files for that identity: main config + **every profile's** config
   (`profiles/*/config.yaml`) + env files. Rot copies itself when profiles are cloned.
3. Determine which entry is native and which is residual: the residual one is usually
   in a `custom_*` section, references localhost URLs, and/or reuses the native
   provider's key (`provider_key` collision = the actual duplication mechanism).
4. **Backup → edit → validate**: copy configs to the backup location; remove the
   residual block (leave an explicit empty list `[]` rather than deleting the key);
   parse-validate every edited file (YAML/JSON load); run the agent's
   `config check`; restart the service; verify the symptom is gone in the UI/CLI.

## Version drift across profiles

- Compare the config schema version marker (`_config_version` or equivalent) between
  the default config and every profile. Profiles silently fall behind.
- Run the agent's `config migrate` on each lagging profile.
- **Read the migrate warnings** — they are a free obsolescence detector. Typical finds:
  references to renamed/removed subsystems (a toolset that became a built-in command),
  integrations never installed, deprecated keys. Clean each one with the same
  backup→edit→validate cycle.

## Env-file hygiene

- Env files accumulate keys for providers no longer used and duplicated base-URL
  overrides. Flag them; let the user confirm removals (keys are credentials).
- When adding shared variables (e.g. a knowledge-base path), add them to the default
  profile AND every named profile — and verify with a grep afterwards.
- **Before inventing a path variable, find where the current processes actually
  write.** Check cron/scheduled jobs' working directories and the running services'
  configs. Setting a path variable that redirects a live pipeline to a new empty
  location is a self-inflicted outage; timing matters (fix before the next scheduled
  run).

## Verification checklist (after any config change)

- [ ] All edited files parse (yaml/json load clean)
- [ ] Agent's own `config check` passes on default + every profile
- [ ] Schema versions aligned across profiles
- [ ] Zero warnings referencing unknown/removed subsystems
- [ ] Service restarted; symptom confirmed gone where it was seen (UI and CLI)
- [ ] Backups of pre-change configs stored in the single backup location
