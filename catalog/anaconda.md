# Anaconda

| | |
|---|---|
| id | `anaconda` |
| group | runtime |
| deps | none |
| channel | ai-lookup (agent resolves the current release at install time) |

Full Anaconda distribution into `/root/anaconda3`. The agent looks up the **current** installer and its SHA256 from `repo.anaconda.com` at install time — no pinned versions (deliberate change from the predecessor project's fixed-checksum approach; we stay latest *and* verify integrity).

## Guard (idempotency)

```bash
test -x /root/anaconda3/bin/conda && echo present
test -d /root/anaconda3 && ! test -x /root/anaconda3/bin/conda && echo BROKEN  # dir without conda: report, do not reinstall over it
```

## Install

```bash
cd /tmp
# 1. Resolve the newest Linux x86_64 installer from the public archive listing
ARCHIVE_HTML=$(curl --fail --location --retry 3 https://repo.anaconda.com/archive/)
INSTALLER=$(echo "$ARCHIVE_HTML" | grep -oP 'Anaconda3-[0-9]{4}\.[0-9]{2}(-[0-9]+)?-Linux-x86_64\.sh' | sort -V | tail -1)

# 2. Download installer + its published checksum, verify
curl --fail --location --retry 3 -O "https://repo.anaconda.com/archive/$INSTALLER"
curl --fail --location --retry 3 -O "https://repo.anaconda.com/archive/${INSTALLER}.sha256"
sha256sum -c "${INSTALLER}.sha256"

# 3. Silent install + shell integration
bash "$INSTALLER" -b -p /root/anaconda3
/root/anaconda3/bin/conda init bash
rm -f "$INSTALLER" "${INSTALLER}.sha256"
```

## Verify

```bash
/root/anaconda3/bin/conda --version
```

## Known Pitfalls

- ~500 MB download — on slow links budget several minutes; retry with `--retry` rather than switching mirrors.
- `-b` (batch) installs **without** running `conda init` — step 3 does it explicitly; the resulting `.bashrc` block only loads in interactive shells (absolute path `/root/anaconda3/bin/conda` over SSH).
- If `.sha256` is missing for the newest file (rare, mid-publish), fall back one version — never skip verification.
