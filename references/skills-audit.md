# Skills / Plugins Audit

Auditing an agent's skills (or plugins/extensions) for duplicates, corruption, and the
frozen-updates problem — and separating user knowledge from managed components.

## The central tension

Agents ship **managed** skills (bundled/hub-installed, updated by the agent) and users
accumulate **customizations**. The moment a user edits a managed skill, most agents
freeze its updates ("user-modified, kept as-is") — silently trading future improvements
for the edit. Months later the skill is an outdated fork whose custom parts nobody
remembers.

**Target state:** every managed skill tracks upstream; every customization lives where
updates can't touch it.

## Audit procedure

1. List modified/frozen skills (`skills list-modified` or the agent's equivalent),
   across **all profiles** — profiles hold independent copies.
2. For each, diff installed copy vs stock source. **Normalize line endings first**:
   compare bytes after `\r\n → \n`. In practice a large share of "modified" skills
   differ ONLY in line endings (an editor or agent rewrote them) — zero real changes.
3. Classify every real difference:
   - **Regression**: the local copy is an *older* stock version (updates froze, upstream
     moved on). The "customization" is actually missing upstream fixes.
   - **Genuine knowledge**: hard-won pitfalls, working configurations, environment
     notes. Worth keeping — elsewhere.
   - **Misplaced artifacts**: model weights, templates, working files inside the skill
     dir. Belongs in project/asset folders (and bloats every skill snapshot — see
     `disk-hygiene.md`).
   - **Junk**: `__pycache__`, editor droppings.
4. Check for **name collisions** between hub-installed and custom skills (same name in
   two scopes = nondeterministic loading). Rename the custom one.

## Extract → restore pattern

For each frozen skill with real custom content:

1. **Back up** the whole modified skill to the backup location.
2. **Extract knowledge** into the user's wiki/knowledge base (respect that KB's own
   schema, linter, and index — orient first, then write, then lint).
3. **Extract behavior** into a **companion skill**: a separate, user-owned skill (e.g.
   `<tool>-<platform>-launcher`) that activates on the same triggers and points to the
   wiki and asset folders. Companion skills never get overwritten by updates.
4. **Move artifacts** (templates, model files) to the folders where the consuming
   application actually loads them.
5. **Restore stock** (`skills reset <name> --restore` or equivalent) and confirm the
   modified-list is empty in every profile.

After this, updates flow again and nothing was lost — it was *relocated*.

## Hash/update-loop gotchas

- Perpetual "update available" on a skill that never changes: usually a **hash parity
  bug** (path separators or sort order differing between hasher implementations across
  OSes). Verify by computing both sides' hashes; report upstream with a minimal repro.
- Bundled-skill seeding into per-profile copies is by design in multi-profile agents —
  identical copies across profiles are NOT duplicates to clean.
