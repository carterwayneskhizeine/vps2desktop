# ADR 0001 — The AI agent executes commands directly; there is no installer script library

Date: 2026-10-05
Status: accepted

## Context

vps2desktop could ship a classic installer (one big `setup.sh`, or a library of
per-component shell scripts). The predecessor project had exactly that: two
parallel ~170-line setup scripts with modules, plan/apply flags, and their own
error handling.

## Decision

There is no installer script. `catalog/<id>.md` component docs are the single
source of truth: each carries install commands (guarded, idempotent) and verify
commands, and the deploying AI agent (`SKILL.md` protocol) executes them over
SSH, handling credentials, preflight, rootification, failures and reporting.

## Consequences

- One representation of the procedure (docs), not two (docs + scripts drifting apart).
- Agents can adapt: retry a flaky mirror, pick the right absolute PATH, read errors and consult the *Known Pitfalls* section, skip what a guard already proves present.
- Hard requirements on the protocol instead: one multiplexed SSH connection (fail2ban), non-interactive PATH discipline, stop-on-failure semantics.
- The trade-off: you need an AI agent to deploy. Humans can still copy-paste commands from catalog docs manually when they want to.
