# Contributing

Thanks for improving vps2desktop!

## What belongs upstream, and what doesn't

| Finding | Where it goes |
|---|---|
| A pitfall any user would hit (upstream URL changed, an apt package rename, a flag that broke, a new workaround) | The component's **Known Pitfalls** section in `catalog/<id>.md` |
| A new generally-useful component | A new `catalog/<id>.md` + `INDEX.yaml` entry + `CHANGELOG.md` entry |
| A bug in an install/verify command | Fix in the component doc |
| Something that only happened on *your* machine (one provider's quirk, your network) | Your local `machines/<alias>/notes.md` — not the repo |

## Rules

1. The repo is **English only**, including examples.
2. **No real IPs, hostnames, credentials, or machine names** in any committed file. Use placeholders (`<host>`, `203.0.113.10`).
3. One component per PR. If you touch a component doc, update `catalog/INDEX.yaml` and append a `CHANGELOG.md` entry in the same PR.
4. Install commands must be idempotent (guarded), latest-channel (no pinned versions), and must come with a working *Verify*.
5. Never automate login states or API keys — that boundary is deliberate.

## Suggested PR flow

1. Fork → branch → edit `catalog/<id>.md` (+ `INDEX.yaml` + `CHANGELOG.md`).
2. If you can, test your change on a disposable VPS via the SKILL.md protocol.
3. Open the PR describing which component and which pitfall/feature it addresses.
