# uv (Python)

| | |
|---|---|
| id | `uv` |
| group | runtime |
| deps | none |
| channel | official-script (astral.sh, always latest) |

Astral's uv / uvx — fast Python package and tool manager. Also the carrier for the `tavly-cli` component.

## Guard (idempotency)

```bash
test -x /root/.local/bin/uv && echo present
```

## Install

```bash
curl --fail --location --retry 3 -LsSf https://astral.sh/uv/install.sh | sh
```

Installs to `/root/.local/bin/uv` (the installer also prints a PATH line for `.bashrc`; irrelevant over non-interactive SSH).

## Verify

```bash
/root/.local/bin/uv --version && /root/.local/bin/uvx --version
```

## Known Pitfalls

- Absolute path (or `export PATH=/root/.local/bin:$PATH`) in every remote command — `.bashrc` PATH lines don't apply non-interactively.
