# codex-profile

[![English](https://img.shields.io/badge/lang-English-lightgrey)](README.md)
[![繁體中文](https://img.shields.io/badge/lang-繁體中文-blue)](README.zh-TW.md)

透過 `CODEX_HOME` 隔離切換多個 Codex CLI 帳號。每個 profile 各自保存自己的 `auth.json` 跟 sessions;`config.toml`(MCP servers、features)預設跨所有 profile 共用。

## 安裝

```bash
mkdir -p ~/.local/bin
mv codex-profile ~/.local/bin/
chmod +x ~/.local/bin/codex-profile

# 讓每個新 shell 自動套用當前 active profile(建議)
echo 'eval "$(codex-profile shell-init)"' >> ~/.zshrc
exec zsh
```

## 指令列表

### Profile 管理

| 指令 | 作用 |
|---|---|
| `list` (alias: `ls`) | 列出所有 profile,active 標 `*`,顯示登入狀態 |
| `current` (alias: `active`) | 印出目前 active 的 profile 名稱 |
| `add <name>` | 建立空 profile。第一次跑 `codex` 時會觸發 OAuth |
| `import-current [name]` | 把現有 `~/.codex` 收編成 profile(預設名稱 `default`)。首次 import 時會把它的 `config.toml` 升格為 shared |
| `switch <name>` (alias: `use`) | 切到指定 profile。**不會碰 auth**,不會重登 |
| `remove <name>` (alias: `rm`) | 刪除 profile 跟所有 credential(要打字確認名稱) |

### 使用 active profile

| 指令 | 作用 |
|---|---|
| `env` | 印出 `export CODEX_HOME=...`,搭配 `eval "$(codex-profile env)"` 套用到當前 shell |
| `run -- <cmd> [args]` (alias: `exec`) | 用 active profile 的 `CODEX_HOME` 跑 `<cmd>` |
| `shell-init` (alias: `init`) | 印出可加到 `~/.zshrc` / `~/.bashrc` 的 snippet,讓每個新 shell 自動套用 |

### Shared config (`config.toml`)

`config.toml`(MCP servers、`[features]`、model 預設)從每個 profile symlink 到 `~/.codex-profiles/_shared/config.toml`。改一次,所有 profile 立刻生效。

| 指令 | 作用 |
|---|---|
| `config status` | 顯示 shared config 路徑,跟每個 profile 的 link 狀態(linked / local / missing / broken) |
| `config path` | 印出 shared `config.toml` 的路徑 |
| `config edit` | 用 `$EDITOR` 開 shared `config.toml` |
| `config link <profile> [--force]` | 把指定 profile 的 `config.toml` 重新 link 到 shared。`--force` 才會覆蓋已存在的本地檔 |
| `config unlink <profile>` | 把指定 profile 從 shared 拆開(變成獨立的私有檔) |
| `config relink-all [--force]` | 把所有 profile 重新 link 到 shared |
| `config help` | 顯示 `config` 子指令的說明 |

### 其他

| 指令 | 作用 |
|---|---|
| `help` (alias: `-h`, `--help`) | 顯示總體用法 |

## 使用範例

### 1. 從零開始

**你已經有登入過的 `~/.codex`:**

```bash
# 收編成命名的 profile
codex-profile import-current work
codex-profile list
# *  work    logged in
```

**你還沒有任何 Codex 設定:**

```bash
codex-profile add work
codex-profile switch work
codex                       # 跳瀏覽器 OAuth
```

### 2. 登入 / 切換 profile

```bash
# 新增第二個 profile(空的 -> 第一次跑 codex 會 OAuth)
codex-profile add personal
codex-profile switch personal
codex                       # OAuth 登第二個 ChatGPT 帳號

# 之後切回去 — 不會重登,每個 profile 的 auth 各自被 cache
codex-profile switch work
codex
```

> ⚠️ **切換前一定要關掉所有 codex process。** 否則會撞到
> `refresh token was already used`(Codex CLI 的 OAuth race condition bug):
> ```bash
> pkill -f codex; sleep 2
> codex-profile switch <name>
> ```

### 3. Import 現有的 config

跑 `import-current` 時,既有的 `config.toml` 會自動升格成 shared,以後新增的 profile 直接繼承:

```bash
codex-profile import-current work
# -> 偵測到 ~/.codex/config.toml,搬到 ~/.codex-profiles/_shared/config.toml
# -> ~/.codex-profiles/work/config.toml 變成指向 shared 的 symlink

codex-profile add personal
# personal/config.toml 也是指向 shared 的 symlink
# -> 你所有的 MCP server 在 'personal' 也馬上可用
```

### 4. 編輯 shared config

```bash
codex-profile config edit
# 在 $EDITOR 開 ~/.codex-profiles/_shared/config.toml
# 存檔後 -> 立刻套用到所有 profile(不用 relink)

codex-profile config status
# Shared config: /home/you/.codex-profiles/_shared/config.toml
#   42 lines, 3 MCP server(s)
#
# Per-profile config.toml:
#   work        linked    -> shared
#   personal    linked    -> shared
```

某個 profile 需要不同設定(罕見,但留個逃生口):

```bash
codex-profile config unlink personal       # personal/config.toml 變成獨立檔
$EDITOR ~/.codex-profiles/personal/config.toml   # 獨立編輯

# 後悔了想跟回 shared:
codex-profile config link personal --force
```

### 5. 確認 auth token

```bash
# 當前 profile 的 auth.json 在哪?
codex-profile current
ls -la "$CODEX_HOME/auth.json"

# 看 token 結構(敏感欄位 mask 掉)
python3 <<'EOF'
import json, os
d = json.load(open(f"{os.environ['CODEX_HOME']}/auth.json"))
print("auth_mode:", "chatgpt" if d.get("tokens") else "apikey")
t = d.get("tokens", {})
print("account_id:", t.get("account_id"))
print("last_refresh:", d.get("last_refresh"))
print("access_token len:", len(t.get("access_token", "")))
print("refresh_token len:", len(t.get("refresh_token", "")))
EOF

# 比對兩個 profile — 如果 account_id 一樣,代表它們共用同一條 OAuth chain(這樣會壞)
for p in ~/.codex-profiles/*/auth.json; do
    echo "=== $p ==="
    python3 -c "import json; print(json.load(open('$p'))['tokens']['account_id'])"
done
```

如果 token 卡在 refresh race 狀態,通常的修法:

```bash
pkill -f codex; sleep 2
codex     # 90% 機率不用重登就能拿到能用的新 token
```

只有上面失敗時才需要完整重登:

```bash
rm "$CODEX_HOME/auth.json"
codex     # 對這個 profile 做完整 OAuth 重登
```

## Shared vs Private

| 檔案 | 範圍 |
|---|---|
| `config.toml` | **Shared**,透過 symlink |
| `auth.json` | 每個 profile 獨立 |
| `sessions/`、`history.jsonl`、`log/` | 每個 profile 獨立 |

## 目錄結構

```
~/.codex-profiles/
├── _shared/
│   └── config.toml              # 真正的 shared config 檔
├── .active                      # 純文字檔:當前 active profile 名稱
├── work/
│   ├── auth.json
│   ├── config.toml -> ../_shared/config.toml
│   ├── sessions/
│   └── ...
└── personal/
    ├── auth.json
    ├── config.toml -> ../_shared/config.toml
    └── ...
```

## 環境變數

| 變數 | 用途 | 預設值 |
|---|---|---|
| `CODEX_HOME` | Codex CLI 讀取所有 state 的位置,由 `codex-profile` 設定 | `~/.codex` |
| `CODEX_PROFILES_DIR` | `codex-profile` 存放 profile 的位置 | `~/.codex-profiles` |
| `EDITOR` / `VISUAL` | `config edit` 用的編輯器 | `vi` |

## 移除

```bash
# 從 ~/.zshrc / ~/.bashrc 移除 shell-init 那行,然後:
unset CODEX_HOME
exec zsh

rm -rf ~/.codex-profiles   # 可選,清掉所有 profile
rm ~/.local/bin/codex-profile
```

`CODEX_HOME` unset 之後,codex 會回到預設的 `~/.codex`。
