# Browser MCP (sandboxed)

| | |
|---|---|
| id | `browser-mcp` |
| group | `ai-cli` |
| deps | `chrome`, `node-nvm`, `claude-code` |
| channel | npm (npx-delivered MCP servers) |

Chrome automation for AI agents with the browser's **kernel-level sandbox on**.

On a root-managed VPS Chrome refuses to start as root ([crbug.com/638180](https://crbug.com/638180)).
The usual workaround — `--no-sandbox` — turns off the browser's kernel protections, so a hostile
page can reach the root-owned host. This Component instead runs the whole MCP server as an
unprivileged account: the kernel provides user namespaces and Chrome's own setuid sandbox is
present, so Chrome keeps its real sandbox while the agent still drives it.

Covers the two official Claude Code browser plugins: `playwright` (`@playwright/mcp`) and
`chrome-devtools` (`chrome-devtools-mcp`).

## Guard (idempotency)

The run account name is a machine detail (this Component creates `browsermcp`, but an
existing machine may use another name), so it is read back from the launcher rather
than hardcoded.

```bash
RUN_USER="$(sed -n 's/^RUN_USER="\(.*\)"$/\1/p' /root/.claude/mcp-wrappers/playwright-mcp.sh 2>/dev/null)"
test -x /root/.claude/mcp-wrappers/playwright-mcp.sh &&
test -x /root/.claude/mcp-wrappers/chrome-devtools-mcp.sh &&
test -n "${RUN_USER}" &&
id "${RUN_USER}" >/dev/null 2>&1 &&
grep -q mcp-wrappers /root/.claude/plugins/cache/claude-plugins-official/playwright/*/.mcp.json 2>/dev/null &&
echo present
```

## Install

### 1. The unprivileged run account

```bash
# A dedicated account for the browser. It needs no login — the MCP servers only
# run as it. The name `ubuntu` is deliberately NOT reused: on most Ubuntu cloud
# images that is the vendor login user, and Rootify (ADR-0003) requires leaving
# that account untouched.
id -u browsermcp >/dev/null 2>&1 ||
  useradd -m -s /bin/bash -c "Unprivileged browser automation run user" browsermcp

# No password: nothing should ever log in as it. Set one only if you want to poke
# around by hand (`passwd browsermcp`) and never record it in the manifest.
passwd -l browsermcp >/dev/null 2>&1 || true
```

### 2. Launcher scripts

Both MCP servers are started through a launcher that drops privileges with `setpriv`. The
launcher stays root so the runtime identity can be re-pointed later without touching the
plugin config.

```bash
install -d -m 0755 /root/.claude/mcp-wrappers

cat > /root/.claude/mcp-wrappers/playwright-mcp.sh <<'PLAYWRIGHT_LAUNCHER_EOF'
#!/usr/bin/env bash
# Playwright MCP launcher.
#
# Chrome refuses to run as root (crbug.com/638180) and --no-sandbox would disable
# the browser's kernel-level protections. Instead the whole MCP server is started
# as an unprivileged run user: user namespaces are enabled on this host, so Chrome
# keeps its real sandbox. This launcher stays root so the runtime identity can be
# re-pointed without touching the plugin config.
#
# The run user cannot read /root/.npm, so it gets its own npm cache, and its own
# Xauthority copy so the browser can reach the Xvfb display.
set -euo pipefail

RUN_USER="browsermcp"
RUN_HOME="/home/${RUN_USER}"
NODE_BIN="$(ls -1d /root/.nvm/versions/node/*/bin 2>/dev/null | sort -V | tail -n1 || true)"
if [ -z "${NODE_BIN}" ]; then
	echo "playwright-mcp: no Node under /root/.nvm/versions/node - install the node-nvm component first" >&2
	exit 1
fi

OUT_DIR="${RUN_HOME}/playwright-out"
install -d -o "${RUN_USER}" -g "${RUN_USER}" -m 0755 "${OUT_DIR}"
cd "${RUN_HOME}"

# A display is only usable when the X socket exists and we hold its cookie;
# otherwise the browser runs headless.
env_args=(
	"HOME=${RUN_HOME}"
	"USER=${RUN_USER}"
	"LOGNAME=${RUN_USER}"
	"PATH=${NODE_BIN}:/usr/local/bin:/usr/bin:/bin"
	"npm_config_cache=${RUN_HOME}/.npm"
)
browser_args=()
if [ -n "${DISPLAY:-}" ] && [ -S "/tmp/.X11-unix/X${DISPLAY##*:}" ] && [ -r "${RUN_HOME}/.Xauthority" ]; then
	env_args+=("DISPLAY=${DISPLAY}" "XAUTHORITY=${RUN_HOME}/.Xauthority")
else
	browser_args+=(--headless)
	unset DISPLAY
fi

exec /usr/bin/setpriv --reuid="${RUN_USER}" --regid="${RUN_USER}" --init-groups \
	env "${env_args[@]}" \
	npx -y @playwright/mcp@latest "${browser_args[@]}" --output-dir "${OUT_DIR}" "$@"
PLAYWRIGHT_LAUNCHER_EOF

cat > /root/.claude/mcp-wrappers/chrome-devtools-mcp.sh <<'DEVTOOLS_LAUNCHER_EOF'
#!/usr/bin/env bash
# chrome-devtools-mcp launcher.
#
# Same idea as playwright-mcp.sh: the whole MCP server runs as an unprivileged
# run user, so the Chrome it spawns keeps its kernel-level sandbox instead of
# needing --no-sandbox. (Chrome refuses to run as root -- crbug.com/638180.)
#
# The run user cannot read /root/.npm, so it gets its own npm cache, and its own
# Xauthority copy so the browser can reach the Xvfb display.
set -euo pipefail

RUN_USER="browsermcp"
RUN_HOME="/home/${RUN_USER}"
NODE_BIN="$(ls -1d /root/.nvm/versions/node/*/bin 2>/dev/null | sort -V | tail -n1 || true)"
if [ -z "${NODE_BIN}" ]; then
	echo "chrome-devtools-mcp: no Node under /root/.nvm/versions/node - install the node-nvm component first" >&2
	exit 1
fi

install -d -o "${RUN_USER}" -g "${RUN_USER}" -m 0755 "${RUN_HOME}/.cache/chrome-devtools-mcp"

# The root mcp.json passes `--prefix ${PLUGIN_DATA}` so the package installs into
# the plugin's data dir. That dir is root-owned and the run user cannot write
# there, so drop the flag and let npx use the run user's own npm cache instead.
args=()
skip=0
for a in "$@"; do
	if [ "${skip}" -eq 1 ]; then
		skip=0
		continue
	fi
	case "${a}" in
	--prefix) skip=1 ;;
	--prefix=*) ;;
	*) args+=("${a}") ;;
	esac
done

# A display is only usable when the X socket exists and we hold its cookie;
# otherwise the browser runs headless.
env_args=(
	"HOME=${RUN_HOME}"
	"USER=${RUN_USER}"
	"LOGNAME=${RUN_USER}"
	"PATH=${NODE_BIN}:/usr/local/bin:/usr/bin:/bin"
	"npm_config_cache=${RUN_HOME}/.npm"
)
browser_args=()
if [ -n "${DISPLAY:-}" ] && [ -S "/tmp/.X11-unix/X${DISPLAY##*:}" ] && [ -r "${RUN_HOME}/.Xauthority" ]; then
	env_args+=("DISPLAY=${DISPLAY}" "XAUTHORITY=${RUN_HOME}/.Xauthority")
else
	browser_args+=(--headless)
	unset DISPLAY
fi

cd "${RUN_HOME}"

exec /usr/bin/setpriv --reuid="${RUN_USER}" --regid="${RUN_USER}" --init-groups \
	env "${env_args[@]}" \
	npx -y "${args[@]}" "${browser_args[@]}"
DEVTOOLS_LAUNCHER_EOF

chmod 0755 /root/.claude/mcp-wrappers/playwright-mcp.sh \
          /root/.claude/mcp-wrappers/chrome-devtools-mcp.sh
```

### 3. The repair skill

Both plugins auto-upgrade into a fresh version directory holding an unpatched config, which
silently re-breaks the browser. This skill is the one-command repair for that.

````bash
install -d -m 0755 /root/.claude/skills/browser-mcp-root-ns/scripts

cat > /root/.claude/skills/browser-mcp-root-ns/SKILL.md <<'SKILL_EOF'
---
name: browser-mcp-root-ns
description: 修复浏览器 MCP 插件（playwright / chrome-devtools）在本机（全程 root）起不来 Chrome 的问题 —— 报 "Running as root without --no-sandbox is not supported"、"Chromium sandboxing failed!" 或 "Protocol error (Target.setDiscoverTargets): Target closed"。做法是让整个 MCP 服务器以非特权用户身份运行，保留 Chrome 真正的内核级沙箱，而不是用 --no-sandbox 关掉防护。插件每次自动升级都会丢失修复，用本 skill 的脚本一键重打。当任何 playwright / chrome-devtools 浏览器工具报上述错误，或用户说「playwright 用不了 / chrome-devtools 用不了 / 浏览器打不开 / 起不来」时使用。
---

# 浏览器 MCP 在 root 环境下的修复

## 何时用

任何 playwright / chrome-devtools 浏览器工具返回以下任一错误：

- `Running as root without --no-sandbox is not supported`（crbug.com/638180）
- `Chromium sandboxing failed!`
- `Protocol error (Target.setDiscoverTargets): Target closed`
- `Chrome failed to start: ...`

## 一条命令修好

```bash
bash ~/.claude/skills/browser-mcp-root-ns/scripts/repair-browser-mcp.sh
```

脚本幂等：配置已是补丁状态就什么都不做。然后**必须让用户跑 `/reload-plugins`** ——
本会话里已在跑的 MCP 进程还是旧实例，改文件不会换掉它。

## 根因与正解

本机全程 root，Chrome 拒绝以 root 启动（[crbug.com/638180](https://crbug.com/638180)）。

**不要用 `--no-sandbox` 绕过。** 那是关掉浏览器的内核级防护：恶意网页能直接打到
root 权限的宿主上。本机 2026-10-19 之前正是这么绕的，已改掉。

正解：本机内核开启了 user namespace（`/proc/sys/user/max_user_namespaces` = 63656，
`unprivileged_userns_clone` = 1），`/opt/google/chrome/chrome_sandbox` 又是 setuid root。
所以只要**整个 MCP 服务器以非特权用户启动**，Chrome 就能用真正的沙箱跑起来。

要降权的是**整个 node 进程**。puppeteer 用绝对路径直连 `/opt/google/chrome/google-chrome`，
绕过 PATH wrapper，所以在 PATH 上做文章没用。

## 架构

```
Claude Code (root)
  ├─ /root/.claude/mcp-wrappers/playwright-mcp.sh        ← 改运行身份只动这两个文件
  │    └─ setpriv --reuid=<run user>
  │         └─ npx @playwright/mcp@latest                ← 以非特权用户
  │              └─ Chrome（沙箱开启，无 --no-sandbox）
  └─ /root/.claude/mcp-wrappers/chrome-devtools-mcp.sh
       └─ setpriv --reuid=<run user>
            └─ npx chrome-devtools-mcp@<版本>
                 └─ Chrome（沙箱开启，无 --no-sandbox）
```

## 哪份配置才生效（两个插件不一样）

| 插件 | 生效的文件 |
|---|---|
| `playwright` | 插件目录同级的 `.mcp.json`（`plugin.json` 里**没有**内嵌 `mcpServers`） |
| `chrome-devtools` | `.claude-plugin/plugin.json` 里的**内嵌 `mcpServers`** |

chrome-devtools 的根目录 `mcp.json` 是 agent-plugins.org 的跨工具规范，给 Gemini /
Cursor 等用，Claude Code 忽略它。**只改它对 Claude Code 无效** —— 2026-10-15 就踩过
这个坑。两份都改，免得换客户端时再踩一遍。

## 插件升级会让修复丢失（关键）

两个插件都会**自动升级**，每次升级生成一个**新的版本目录**（如 `1.9.0` → `1.11.0`，
或 `ea10b213686c` → 新的 hash），里面是**未打补丁的干净配置**，于是问题复发。

所以这不会"修一次就好"。每次报上面那些错，就重跑脚本。

## 排查：改完还是不生效

**别信「文件改了 = 配置生效」。** 用真实进程参数验证：

```bash
ps -eo user,args | grep "[p]laywright/mcp\|[c]hrome-devtools-mcp"
```

应该看到运行用户拥有该进程，且命令行里**没有** `--no-sandbox`。

再检查：

```bash
ls ~/.claude/plugins/cache/claude-plugins-official/chrome-devtools-mcp/   # 是否有多版本
ls ~/.claude/plugins/cache/claude-plugins-official/playwright/           # 是否有多版本
claude mcp list                                                            # 解析到哪份
```

如果进程参数已经带上了启动脚本、但 Chrome 仍起不来，那根因就不是 root 沙箱 ——
检查是不是有残留 Chrome 进程占着 profile、或 `/dev/shm` 太小。

## 验收

重载后用任意浏览器工具打开一个真实站点（如 `https://example.com`），应能拿到
Page URL 和 Page Title；再用 `ps -eo user,args | grep "[c]hrome"` 确认 chrome 进程
属运行用户且没有 `--no-sandbox`。
SKILL_EOF

cat > /root/.claude/skills/browser-mcp-root-ns/scripts/repair-browser-mcp.sh <<'REPAIR_EOF'
#!/usr/bin/env bash
# 重新给浏览器 MCP 插件打补丁。
#
# 插件每次自动升级都会生成新的版本目录，里面是未打补丁的干净配置，
# 于是修复丢失、Chrome 又起不来了。这个脚本把配置重新指向启动脚本，
# 并去掉 --no-sandbox。
#
# 幂等：已经是补丁状态就什么都不做。改完必须 /reload-plugins。
set -euo pipefail

WRAPPER_DIR="/root/.claude/mcp-wrappers"
CACHE="/root/.claude/plugins/cache/claude-plugins-official"
MARKET="/root/.claude/plugins/marketplaces/claude-plugins-official/external_plugins"

if [ ! -x "${WRAPPER_DIR}/playwright-mcp.sh" ] || [ ! -x "${WRAPPER_DIR}/chrome-devtools-mcp.sh" ]; then
	echo "启动脚本缺失 —— 重新安装 browser-mcp 组件（catalog/browser-mcp.md）" >&2
	exit 1
fi

patched=0

# playwright：生效的是插件目录同级的 .mcp.json
while IFS= read -r f; do
	[ -n "${f}" ] || continue
	grep -q "mcp-wrappers" "${f}" && continue
	python3 - "${f}" "${WRAPPER_DIR}/playwright-mcp.sh" <<'PY'
import json
import os
import sys

path, wrapper = sys.argv[1], sys.argv[2]
with open(path, encoding="utf-8") as handle:
    data = json.load(handle)
data["playwright"] = {"command": wrapper}
tmp = path + ".tmp"
with open(tmp, "w", encoding="utf-8") as handle:
    json.dump(data, handle, indent=2, ensure_ascii=False)
    handle.write("\n")
os.replace(tmp, path)
PY
	echo "patched: ${f}"
	patched=1
done < <(find "${CACHE}/playwright" "${MARKET}/playwright" -name ".mcp.json" -type f 2>/dev/null)

# chrome-devtools：任何声明了 mcpServers.chrome-devtools 的清单都要指向启动脚本
# 并去掉 --no-sandbox。各客户端读的清单不是同一个：
#   .claude-plugin/plugin.json  -> Claude Code
#   mcp.json                    -> agent-plugins.org 跨工具规范
#   gemini-extension.json       -> Gemini
#   .cursor-plugin/plugin.json  -> Cursor
#   .github/plugin/plugin.json  -> GitHub Copilot
# 插件源码树里的其它文件没有这个键，所以不会误伤。
while IFS= read -r f; do
	[ -n "${f}" ] || continue
	before="$(cat "${f}")"
	python3 - "${f}" "${WRAPPER_DIR}/chrome-devtools-mcp.sh" <<'PY'
import json
import os
import sys

path, wrapper = sys.argv[1], sys.argv[2]
with open(path, encoding="utf-8") as handle:
    data = json.load(handle)
server = data.get("mcpServers", {}).get("chrome-devtools")
if not isinstance(server, dict):
    raise SystemExit(0)
server["command"] = wrapper
args = server.get("args")
if isinstance(args, list):
    server["args"] = [a for a in args if a != "--chromeArg=--no-sandbox"]
tmp = path + ".tmp"
with open(tmp, "w", encoding="utf-8") as handle:
    json.dump(data, handle, indent=2, ensure_ascii=False)
    handle.write("\n")
os.replace(tmp, path)
PY
	if [ "${before}" != "$(cat "${f}")" ]; then
		echo "patched: ${f}"
		patched=1
	fi
done < <(grep -rl '"mcpServers"' --include="*.json" "${CACHE}/chrome-devtools-mcp" 2>/dev/null)

if [ "${patched}" -eq 0 ]; then
	echo "配置已是补丁状态，无需改动"
else
	echo "已重新打补丁"
fi

echo
echo "下一步 —— 必须让用户执行："
echo "    /reload-plugins"
echo
echo "本会话里已在跑的 MCP 进程还是旧实例，重载才会换成新的。"
echo "重载后用 claude mcp list 确认显示的是 mcp-wrappers 下的启动脚本，"
echo "并用 ps -eo user,args | grep '[c]hrome' 确认 chrome 进程属运行用户、命令行里没有 --no-sandbox。"
REPAIR_EOF

chmod 0755 /root/.claude/skills/browser-mcp-root-ns/scripts/repair-browser-mcp.sh
````

### 4. The Claude Code plugins

```bash
claude plugin install playwright@claude-plugins-official
claude plugin install chrome-devtools-mcp@claude-plugins-official
claude plugin enable chrome-devtools-mcp@claude-plugins-official || true
```

Both commands are idempotent (`already installed` / `already enabled` on a repeat run).

### 5. Point the plugin configs at the launchers

```bash
bash /root/.claude/skills/browser-mcp-root-ns/scripts/repair-browser-mcp.sh
```

### 6. Let the run account reach the display and npm

```bash
# Chrome needs the X cookie to talk to the display; /root/.Xauthority is 600 root,
# so the run account gets its own copy.
if [ -f /root/.Xauthority ]; then
  install -o browsermcp -g browsermcp -m 0600 /root/.Xauthority /home/browsermcp/.Xauthority
fi

# The MCP servers are delivered by npx, and the run account cannot read
# /root/.npm — it needs its own cache. Warm it so the first tool call does not
# wait on a registry download.
NODE_BIN="$(ls -1d /root/.nvm/versions/node/*/bin 2>/dev/null | sort -V | tail -n1)"
for pkg in @playwright/mcp@latest chrome-devtools-mcp@latest; do
  setpriv --reuid=browsermcp --regid=browsermcp --init-groups \
    env HOME=/home/browsermcp \
        PATH="${NODE_BIN}:/usr/local/bin:/usr/bin:/bin" \
        npm_config_cache=/home/browsermcp/.npm \
    npx -y "${pkg}" --version
done
```

### 7. Reload

Tell the user to run `/reload-plugins` — the MCP processes already running in this session are
still the old instances, and editing the config does not replace them.

## Verify

```bash
# 1) launchers and run account
RUN_USER="$(sed -n 's/^RUN_USER="\(.*\)"$/\1/p' /root/.claude/mcp-wrappers/playwright-mcp.sh 2>/dev/null)"
test -x /root/.claude/mcp-wrappers/playwright-mcp.sh &&
test -x /root/.claude/mcp-wrappers/chrome-devtools-mcp.sh &&
test -n "${RUN_USER}" &&
id "${RUN_USER}" >/dev/null 2>&1 || { echo "FAIL: launcher or run account missing"; exit 1; }

# 2) every launch manifest points at the launcher, and no --no-sandbox survives
python3 - <<'PY'
import glob
import json
import os
import sys

bad = []

for path in glob.glob("/root/.claude/plugins/cache/claude-plugins-official/playwright/*/.mcp.json"):
    with open(path, encoding="utf-8") as handle:
        server = json.load(handle).get("playwright", {})
    if "mcp-wrappers" not in server.get("command", ""):
        bad.append(path)

# os.walk, not glob: the client manifests live in dot-directories
# (.claude-plugin, .cursor-plugin, .github-plugin) that glob skips.
for root, _dirs, files in os.walk(
    "/root/.claude/plugins/cache/claude-plugins-official/chrome-devtools-mcp"
):
    for name in files:
        if not name.endswith(".json"):
            continue
        path = os.path.join(root, name)
        try:
            with open(path, encoding="utf-8") as handle:
                data = json.load(handle)
        except (OSError, ValueError):
            continue
        if not isinstance(data, dict):
            continue
        server = (data.get("mcpServers") or {}).get("chrome-devtools")
        if not isinstance(server, dict):
            continue
        if "mcp-wrappers" not in server.get("command", "") or any(
            "--no-sandbox" in arg for arg in server.get("args", [])
        ):
            bad.append(path)

if bad:
    print("FAIL: launch manifest not patched:")
    for path in bad:
        print("  " + path)
    sys.exit(1)
print("OK: every launch manifest points at the launcher, no --no-sandbox left")
PY

# 3) the real proof — Chrome starts as the unprivileged account with its sandbox on.
#    As root the same command dies with "Running as root without --no-sandbox is
#    not supported", so this is a real check. Chrome's dbus warnings are noise.
dom="$(setpriv --reuid="${RUN_USER}" --regid="${RUN_USER}" --init-groups \
  env HOME="/home/${RUN_USER}" \
  /opt/google/chrome/chrome --headless=new --disable-gpu --dump-dom about:blank 2>/dev/null)" ||
  { echo "FAIL: Chrome did not start as the unprivileged account"; exit 1; }
printf '%s\n' "${dom}" | grep -q "<html" || { echo "FAIL: no DOM from Chrome"; exit 1; }
echo "OK: Chrome started as ${RUN_USER} with its sandbox on"
```

Step 3 dumps the DOM on success. As root the same command dies with
`Running as root without --no-sandbox is not supported`, so it is a real check, not a formality.

After `/reload-plugins`, `claude mcp list` must show the launchers:

```
plugin:playwright:playwright: /root/.claude/mcp-wrappers/playwright-mcp.sh - ✔ Connected
plugin:chrome-devtools-mcp:chrome-devtools: /root/.claude/mcp-wrappers/chrome-devtools-mcp.sh ... - ✔ Connected
```

And a live tool call must show the browser running as the unprivileged account, with no
`--no-sandbox` in its command line:

```bash
ps -eo user,args | grep "[c]hrome" | grep -v "^root" ||
  echo "(no browser running — expected before the first browser tool call)"
```

## Known Pitfalls

- **Do not reach for `--no-sandbox`.** It is what the `chrome` component's desktop wrapper uses
  for a human-facing browser in a root desktop session — a different case, documented there.
  For an agent-driven browser it disables the kernel protections that keep a hostile page away
  from the root-owned host. Run the server as an unprivileged account instead.
- **The two plugins read different config files.** `playwright` is configured by the `.mcp.json`
  sitting next to the plugin manifest (its `plugin.json` has no embedded `mcpServers`);
  `chrome-devtools` is configured by the embedded `mcpServers` in `.claude-plugin/plugin.json`.
  Editing only one of them changes nothing. For `chrome-devtools` the root-level `mcp.json` is
  the agent-plugins.org cross-tool spec for Gemini/Cursor — Claude Code ignores it — but the
  repair script patches it too, so switching clients later does not land on a broken config.
- **A plugin upgrade silently reverts the fix.** Each upgrade creates a new version directory
  (`1.9.0` → `1.11.0`, or a fresh hash for `playwright`) with a clean, unpatched config, so the
  browser starts failing again. Re-run
  `bash ~/.claude/skills/browser-mcp-root-ns/scripts/repair-browser-mcp.sh`, then
  `/reload-plugins`. This is not a one-time fix.
- **The run account cannot read `/root/.npm`.** Both servers are delivered by `npx`, so give the
  account its own cache (`npm_config_cache=/home/browsermcp/.npm`) and warm it during install.
  Otherwise the first tool call after every start blocks on a registry download.
- **Chrome needs the X cookie.** `/root/.Xauthority` is mode 600 root, so the run account needs
  its own copy and `XAUTHORITY` must point at it; otherwise Chrome reports
  `Authorization required, but no authorization protocol specified`. With no display at all the
  launchers fall back to `--headless` automatically — check the X socket and the cookie before
  assuming the display is fine.
- **Do not hardcode the Node path.** `/root/.nvm/versions/node/<version>/bin` changes on every
  Node upgrade; resolve it with
  `ls -1d /root/.nvm/versions/node/*/bin | sort -V | tail -n1`.
- **The workspace belongs to root.** `playwright` resolves explicitly named output files against
  the workspace root, which the run account cannot write to — hence `--output-dir` pointing at
  `/home/browsermcp/playwright-out`.
- **`--prefix ${PLUGIN_DATA}` is dropped by the launcher.** The cross-tool `mcp.json` asks npx to
  install into the plugin data directory, which is root-owned; the launcher strips the flag and
  the pair behind it so npx uses the run account's own cache instead.
- **The `ubuntu` name is not free on a cloud image.** Most Ubuntu VPS images ship a vendor
  `ubuntu` login user. Reusing it as the browser account would put the agent's browsing inside
  the fallback login's home directory; Rootify (ADR-0003) requires leaving that account
  untouched, so this Component creates a dedicated one instead.
- **The account is for the browser, not for the agent.** Keep it passwordless and unprivileged.
  Give it a password only for hand-inspection, and never record credentials in the manifest
  (ADR-0002).
