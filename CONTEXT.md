# CONTEXT.md — Glossary

The vocabulary of vps2desktop. Use these terms exactly; when a term gains a new meaning, update this file in the same change.

- **Component** — a single selectable installable unit in the catalog, identified by a stable id (e.g. `xfce-desktop`, `claude-code`). One component = one file in `catalog/`.

- **Catalog** — the upstream, machine-agnostic collection of component docs (`catalog/`), the registry (`INDEX.yaml`) and the `CHANGELOG.md`. The catalog is the single source of truth for how to install and verify a component.

- **Preset** — a named bundle of components offered by the checklist (`Minimal`, `Remote Desktop`, `AI Toolkit`, `Full`) as a starting point before fine-tuning.

- **Manifest** — the per-machine file `machines/<alias>/manifest.yaml`: the selected components, optional connection credentials, and the agent-maintained `installed:` record (component id, version, date) plus `last_sync`. Local state; never committed.

- **Machine Profile** — the per-machine `machines/<alias>/profile.md`: facts about one VPS (provider, IP, Hostname, Machine ID, OS, what is installed, follow-ups). Identity fields help determine whether the agent is already running on the target VPS. Written by the agent after verifying the target. Local state; never committed.

- **Local State** — user-specific files under `machines/<alias>/`: manifests, profiles, credentials, notes. Gitignored by design; survives upstream `git pull` without conflict. Shared templates under `machines/templates/` are tracked and are not Local State.

- **Machine Alias** — the short user-chosen name identifying one VPS (the `<alias>` in `machines/<alias>/`). Aliases are unique per user; they never appear in the catalog. `templates` is reserved for shared templates.

- **Rootify** — the normalization step that turns whatever login the vendor shipped (root, ubuntu, admin, ...) into the standard end-state: root password SSH login, vendor user left untouched.

- **Update Check** — the SKILL.md mode that diffs `catalog/CHANGELOG.md` entries newer than a manifest's `last_sync` against its `installed:` list, then offers the user new or updated components.

- **Known Pitfall** — a catalog-level, machine-agnostic gotcha documented inside a component doc (e.g. "npm ≥ 12 blocks postinstall scripts"). Contrast with machine-specific quirks, which live only in the machine's local `notes.md`.

- **Verify** — the command(s) in each component doc that prove the component works on this machine. An install without a passing verify is not recorded in the manifest.
