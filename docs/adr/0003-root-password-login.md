# ADR 0003 — Deployments standardize on root password SSH login

Date: 2026-10-05
Status: accepted

## Context

Vendors ship VPSes with varying login users (root, ubuntu, admin, ...). The
project's goal is that every machine ends up in one predictable shape, so a
user always knows how to get in and every runbook/doc can assume one layout
(`/root`, root-owned dotfiles, one home to configure).

Alternatives: operate via the vendor user + sudo (two shapes, dotfiles split
across homes), or key-only root (requires the user to already have SSH keys —
a real barrier for the target audience).

## Decision

Deployments *Rootify*: root's password is set to the credential the user
already provided, `PermitRootLogin yes` is enabled via a drop-in, sshd is
reloaded (never restarted), and the vendor user is left untouched as a
fallback. Everything installs into `/root`.

This is a deliberate trade of convenience against exposure: root + password on
a public IP is what SSH scanners hammer all day. Mitigations shipped with the
catalog: the `fail2ban` component (sshd jail), which rate-limits password
brute-force attempts.

## Consequences

- One mental model: "the credential I told my AI is the root login."
- The RDP password is the same root password (xrdp authenticates against system users).
- Vendor user remains as the recovery path if rootify goes wrong.
