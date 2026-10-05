# Anaconda

| | |
|---|---|
| id | `anaconda` |
| group | runtime |
| deps | none |
| channel | ai-lookup (agent resolves the current release at install time) |

Full Anaconda distribution into `/root/anaconda3`. The agent looks up the **current** installer from `repo.anaconda.com` at install time — no pinned versions. Integrity relies on TLS from the official domain: upstream stopped publishing `.sha256` sidecar files (verified 2026-10-06, none in the archive listing anymore), so the checksum step was removed (user decision).

## Guard (idempotency)

```bash
test -x /root/anaconda3/bin/conda && echo present
test -d /root/anaconda3 && ! test -x /root/anaconda3/bin/conda && echo BROKEN  # dir without conda: report, do not reinstall over it
```

## Install

```bash
cd /tmp
# 1. Resolve the newest Linux x86_64 installer from the public archive listing.
#    Each filename appears twice in the listing HTML — sort -u (not plain -V) or
#    "second newest" resolves to the same file.
ARCHIVE_HTML=$(curl --fail --location --retry 3 https://repo.anaconda.com/archive/)
INSTALLER=$(echo "$ARCHIVE_HTML" | grep -oP 'Anaconda3-[0-9]{4}\.[0-9]{2}(-[0-9]+)?-Linux-x86_64\.sh' | sort -uV | tail -1)

# 2. Download (TLS to the official domain; no .sha256 exists anymore — see pitfalls)
curl --fail --location --retry 3 -O "https://repo.anaconda.com/archive/$INSTALLER"

# 3. Silent install + shell integration
bash "$INSTALLER" -b -p /root/anaconda3
/root/anaconda3/bin/conda init bash
rm -f "$INSTALLER"
```

## Verify

```bash
/root/anaconda3/bin/conda --version
```

## Known Pitfalls

- ~500 MB–1.2 GB download — on slow links budget several minutes; retry with `--retry` rather than switching mirrors.
- `-b` (batch) installs **without** running `conda init` — step 3 does it explicitly; the resulting `.bashrc` block only loads in interactive shells (absolute path `/root/anaconda3/bin/conda` over SSH).
- Upstream no longer publishes `.sha256` checksum files in the archive (removed sometime before 2026-10-06; the old fallback-one-version advice is obsolete). Integrity rests on HTTPS to `repo.anaconda.com` only; if checksums return in any form, reinstate verification.
