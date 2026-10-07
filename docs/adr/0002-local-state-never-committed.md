# ADR 0002 — Machine state is local-only and gitignored

Date: 2026-10-05
Status: accepted

## Context

Every user maintains several machines, each with a manifest (component
selection + installed record), credentials, a profile and notes. The repo is
public and shared; users must be able to `git pull` upstream updates forever.

Earlier drafts kept `machines/` committed to the (then private) repository.
That works for one private owner — and is exactly how the predecessor project
stored credentials — but in a public, multi-user repo it means merge conflicts
on every pull, users leaking their IPs/passwords into git history, and issue
reports about machine-specific quirks that do not generalize.

## Decision

Local State under `machines/<alias>/` is gitignored. Manifests (with their optional
`credentials:` block), profiles and notes live only on the user's machine.
Upstream tracks only the shared `manifest.yaml`, `profile.md` and `notes.md`
templates in `machines/templates/`, so cloning also creates `machines/`.
All other files under `machines/` remain ignored, including extra files inside
the template directories. Copy templates to `machines/<alias>/` before filling
in any machine details; shared templates must never contain real credentials.
`templates` is reserved and cannot be used as a Machine Alias. Machine-specific lessons
stay in the user's local `notes.md`; only generally-useful pitfall fixes flow
back upstream (CONTRIBUTING.md).

## Consequences

- Local State does not conflict with `git pull`; upstream templates and personal state live in separate directories.
- Credentials never enter git history anywhere.
- Update delivery is conversational: `git pull` + update-check mode diffs the catalog changelog against the local manifest.
- Users must back up their `machines/<alias>/` directories themselves — Local State is not in git.
