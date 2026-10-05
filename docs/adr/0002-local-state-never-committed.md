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

The entire `machines/` tree is gitignored. Manifests (with their optional
`credentials:` block), profiles and notes live only on the user's machine.
Upstream ships `templates/machine/` examples instead. Machine-specific lessons
stay in the user's local `notes.md`; only generally-useful pitfall fixes flow
back upstream (CONTRIBUTING.md).

## Consequences

- `git pull` never conflicts; upstream and personal state are physically separated.
- Credentials never enter git history anywhere.
- Update delivery is conversational: `git pull` + update-check mode diffs the catalog changelog against the local manifest.
- Users must back up `machines/` themselves — it is not in git.
