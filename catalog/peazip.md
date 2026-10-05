# PeaZip

| | |
|---|---|
| id | `peazip` |
| group | apps |
| deps | none (right-click integration needs `xfce-desktop`'s Thunar) |
| channel | github-api (asset names carry the version — no stable latest URL) |

PeaZip archive manager — open/extract 200+ formats, create 7Z/ZIP/TAR/PEA etc. Installed as the official **GTK2** deb (avoids dragging a Qt6 runtime onto a GTK desktop) plus **Thunar custom actions** so it lives in the right-click menu.

## Guard (idempotency)

```bash
dpkg -s peazip >/dev/null 2>&1 && grep -q peazip /root/.config/Thunar/uca.xml 2>/dev/null && echo present
```

## Install

```bash
export DEBIAN_FRONTEND=noninteractive
# Asset filenames embed the version (peazip_11.3.0.LINUX.GTK2-1_amd64.deb) — resolve via API, never hardcode
DEB_URL=$(curl -fsSL --max-time 20 https://api.github.com/repos/peazip/PeaZip/releases/latest \
  | grep -oP '"browser_download_url":\s*"\K[^"]*\.LINUX\.GTK2[^"]*amd64\.deb')

cd /tmp
curl --fail --location --retry 3 -o peazip.deb "$DEB_URL"
apt-get install -y ./peazip.deb   # pulls libgtk2.0-0t64 if missing
rm -f peazip.deb

# Thunar right-click actions (fresh profile — if uca.xml already exists, merge inside <actions> instead)
mkdir -p /root/.config/Thunar
cat > /root/.config/Thunar/uca.xml <<'XML'
<?xml version="1.0" encoding="UTF-8"?>
<actions>
<action>
	<icon>package-x-generic</icon>
	<name>PeaZip: Extract Here</name>
	<unique-id>0000000000000001-1</unique-id>
	<command>peazip -ext2here %F</command>
	<description>Extract the selected archives in place</description>
	<patterns>*.7z;*.zip;*.zipx;*.rar;*.tar;*.gz;*.bz2;*.xz;*.zst;*.tgz;*.tbz2;*.txz;*.tzst;*.br;*.lz;*.lzma;*.ace;*.arc;*.arj;*.cab;*.iso;*.wim;*.lha;*.lzh;*.pea;*.zpaq;*.rpm;*.deb;*.jar;*.apk;*.xar;*.001</patterns>
	<startup-notify/>
	<other-files/>
</action>
<action>
	<icon>package-x-generic</icon>
	<name>PeaZip: Extract To...</name>
	<unique-id>0000000000000001-2</unique-id>
	<command>peazip -ext2main %F</command>
	<description>Extract via the PeaZip dialog (choose path, password...)</description>
	<patterns>*.7z;*.zip;*.zipx;*.rar;*.tar;*.gz;*.bz2;*.xz;*.zst;*.tgz;*.tbz2;*.txz;*.tzst;*.br;*.lz;*.lzma;*.ace;*.arc;*.arj;*.cab;*.iso;*.wim;*.lha;*.lzh;*.pea;*.zpaq;*.rpm;*.deb;*.jar;*.apk;*.xar;*.001</patterns>
	<startup-notify/>
	<other-files/>
</action>
<action>
	<icon>package-x-generic</icon>
	<name>PeaZip: Open in PeaZip</name>
	<unique-id>0000000000000001-3</unique-id>
	<command>peazip -ext2browse %F</command>
	<description>Browse / manage this archive in PeaZip</description>
	<patterns>*</patterns>
	<startup-notify/>
	<directories/>
	<audio-files/>
	<image-files/>
	<other-files/>
	<text-files/>
	<video-files/>
</action>
<action>
	<icon>package-x-generic</icon>
	<name>PeaZip: Add to ZIP</name>
	<unique-id>0000000000000001-4</unique-id>
	<command>peazip -add2zip %F</command>
	<description>Create a .zip from the selection</description>
	<patterns>*</patterns>
	<startup-notify/>
	<directories/>
	<audio-files/>
	<image-files/>
	<other-files/>
	<text-files/>
	<video-files/>
</action>
<action>
	<icon>package-x-generic</icon>
	<name>PeaZip: Add to 7Z</name>
	<unique-id>0000000000000001-5</unique-id>
	<command>peazip -add27z %F</command>
	<description>Create a .7z from the selection</description>
	<patterns>*</patterns>
	<startup-notify/>
	<directories/>
	<audio-files/>
	<image-files/>
	<other-files/>
	<text-files/>
	<video-files/>
</action>
</actions>
XML
```

## Verify

```bash
dpkg -s peazip | grep -q 'Status: install ok installed' && command -v peazip && grep -q 'PeaZip' /root/.config/Thunar/uca.xml && echo ok
```

## Known Pitfalls

- **Thunar caches custom actions**: an already-running Thunar won't show the new entries until it restarts — close all Thunar windows (`pkill Thunar` as last resort) or relogin. Fresh logins are unaffected.
- uca.xml semantics: an empty element (`<other-files/>`, `<directories/>`) means "show for this type"; a *missing* element means "don't". Comments or text inside do **not** count as present.
- If `uca.xml` already exists with user-defined actions, merge the `<action>` blocks inside `<actions>` — do not overwrite the file wholesale.
- Pick the **GTK2** asset on GTK desktops; the Qt6 deb works too but pulls the whole Qt6 stack (~200 MB).
- Thunar patterns are case-sensitive; uppercase extensions (`*.ZIP`) won't match the lowercase entries (rare on Linux).
- CLI flags used here (`-ext2here`, `-ext2main`, `-ext2browse`, `-add2zip`, `-add27z`) are from PeaZip's official command-line page — all open a GUI window, so they only make sense inside a session (RDP/console), never headless.
- Optional 32-bit-only backends (notably FreeARC) need ia32 libs — skip unless a 32-bit format is actually needed.

## Manual follow-ups

- none
