---
title: "Claude Code 切换模型供应商"
date: 2026-05-13
description: Claude Code 快速切换 ANTHROPIC_BASE_URL 等配置的几种方案与自动化脚本
tags:
  - Agent
  - Claude
  - 工具
---

# Claude Code 切换模型供应商

## 问题

每次切换模型供应商都需要手动编辑 `~/.claude/settings.json` 中的 `ANTHROPIC_BASE_URL`、`ANTHROPIC_AUTH_TOKEN`、`ANTHROPIC_MODEL` 和 `model` 字段，非常不便。

## 几种解决方案

| 方案 | 原理 | 适合场景 |
|------|------|----------|
| 一、`/model` 命令 | Claude Code 内置，交互式切换 | 临时切换、快速体验 |
| 二、Shell 别名 | 在 `~/.zshrc` 中预设多组环境变量别名，启动时选择 | 供应商固定、配置不多 |
| 三、切换脚本 | 独立脚本管理供应商配置表，一键切换 | 供应商较多、需要增删管理 |

---

## 方案一：/model 命令

- 要求，该 base_url的供应商支持多个模型或者多个版本

## 方案二：Shell 别名

在 `~/.zshrc` 中为每个供应商设置一个别名，每个别名通过环境变量覆盖对应的配置项。启动时选择对应的别名即可。

编辑 `~/.zshrc`，添加：

```bash
# Claude Code - SiliconFlow GLM-5.1
alias claude-sf='ANTHROPIC_BASE_URL="https://api.siliconflow.cn/" ANTHROPIC_AUTH_TOKEN="sk-xxx" ANTHROPIC_MODEL="Pro/zai-org/GLM-5.1" claude'

# Claude Code - DeepSeek
alias claude-ds='ANTHROPIC_BASE_URL="https://api.deepseek.com/anthropic" ANTHROPIC_AUTH_TOKEN="sk-yyy" ANTHROPIC_MODEL="deepseek-v4-flash" claude'

# Claude Code - OpenRouter
alias claude-or='ANTHROPIC_BASE_URL="https://openrouter.ai/api/v1/" ANTHROPIC_AUTH_TOKEN="sk-zzz" ANTHROPIC_MODEL="anthropic/claude-sonnet-4-20250514" claude'
```

保存后执行 `source ~/.zshrc`，然后：

- `claude-sf` — 用 SiliconFlow + GLM-5.1 启动
- `claude-ds` — 用 DeepSeek 启动
- `claude-or` — 用 OpenRouter 启动

**优点：** 零依赖，无需额外脚本。
**缺点：** 增删供应商需手动编辑 `.zshrc`；供应商多了别名单调难记。

---

## 方案三：切换脚本

由两个文件组成：

```
~/.claude/
├── settings.json      # Claude Code 主配置（脚本自动修改）
└── providers.json     # 供应商配置表（用户维护，脚本读取）

~/.local/bin/
└── claude-switch      # 切换脚本
```

### providers.json

存储所有供应商的预设配置，脚本读取此文件进行切换：

```json
{
  "siliconflow-glm": {
    "ANTHROPIC_BASE_URL": "https://api.siliconflow.cn/",
    "ANTHROPIC_AUTH_TOKEN": "sk-xxx",
    "ANTHROPIC_MODEL": "Pro/zai-org/GLM-5.1",
    "model": "Pro/zai-org/GLM-5.1"
  },
  "deepseek": {
    "ANTHROPIC_BASE_URL": "https://api.deepseek.com/anthropic",
    "ANTHROPIC_AUTH_TOKEN": "sk-yyy",
    "ANTHROPIC_MODEL": "deepseek-v4-flash",
    "model": "deepseek-v4-flash"
  }
}
```

### claude-switch 脚本源码

将以下内容保存到 `~/.local/bin/claude-switch` 并执行 `chmod +x ~/.local/bin/claude-switch`：

```bash
#!/bin/bash
set -euo pipefail

CLAUDE_DIR="$HOME/.claude"
SETTINGS="$CLAUDE_DIR/settings.json"
PROVIDERS="$CLAUDE_DIR/providers.json"

RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
CYAN='\033[0;36m'
BOLD='\033[1m'
NC='\033[0m'

die() { echo -e "${RED}Error:${NC} $*" >&2; exit 1; }

check_deps() {
  command -v jq &>/dev/null || die "jq is required. Install with: brew install jq"
  [[ -f "$SETTINGS" ]] || die "Settings file not found: $SETTINGS"
  [[ -f "$PROVIDERS" ]] || die "Providers file not found: $PROVIDERS"
}

get_current() {
  local base_url model
  base_url=$(jq -r '.env.ANTHROPIC_BASE_URL // ""' "$SETTINGS")
  model=$(jq -r '.env.ANTHROPIC_MODEL // ""' "$SETTINGS")

  local best_match=""
  local best_score=0

  while IFS=$'\t' read -r name murl mkey mmodel; do
    local score=0
    [[ "$murl" == "$base_url" ]] && ((score++))
    [[ "$mmodel" == "$model" ]] && ((score++))
    if ((score > best_score)); then
      best_score=$score
      best_match="$name"
    fi
  done < <(jq -r 'to_entries[] | [.key, .value.ANTHROPIC_BASE_URL // "", .value.ANTHROPIC_AUTH_TOKEN // "", .value.ANTHROPIC_MODEL // ""] | @tsv' "$PROVIDERS")

  echo "$best_match"
}

cmd_list() {
  local current
  current=$(get_current)

  echo -e "${BOLD}Available providers:${NC}"
  echo ""

  local names=()
  while IFS=$'\t' read -r name murl mmodel; do
    names+=("$name")
  done < <(jq -r 'to_entries[] | [.key, .value.ANTHROPIC_BASE_URL // "", .value.ANTHROPIC_MODEL // ""] | @tsv' "$PROVIDERS")

  if ((${#names[@]} == 0)); then
    echo -e "  ${YELLOW}No providers configured.${NC}"
    echo -e "  Run ${CYAN}claude-switch --add <name>${NC} to add one."
    return
  fi

  for name in "${names[@]}"; do
    local url model
    url=$(jq -r ".[\"$name\"].ANTHROPIC_BASE_URL // \"\"" "$PROVIDERS")
    model=$(jq -r ".[\"$name\"].ANTHROPIC_MODEL // \"\"" "$PROVIDERS")

    if [[ "$name" == "$current" ]]; then
      echo -e "  ${GREEN}* ${BOLD}${name}${NC}  ${CYAN}${model}${NC}  ${YELLOW}${url}${NC}"
    else
      echo -e "    ${name}  ${CYAN}${model}${NC}  ${YELLOW}${url}${NC}"
    fi
  done

  echo ""
  echo -e "  Current: ${GREEN}${current:-none}${NC}"
}

cmd_switch() {
  local name="$1"

  jq -e --arg n "$name" 'has($n)' "$PROVIDERS" &>/dev/null \
    || die "Provider '$name' not found. Run 'claude-switch' to see available providers."

  local base_url auth_token env_model model
  base_url=$(jq -r ".[\"$name\"].ANTHROPIC_BASE_URL" "$PROVIDERS")
  auth_token=$(jq -r ".[\"$name\"].ANTHROPIC_AUTH_TOKEN" "$PROVIDERS")
  env_model=$(jq -r ".[\"$name\"].ANTHROPIC_MODEL" "$PROVIDERS")
  model=$(jq -r ".[\"$name\"].model // .[\"$name\"].ANTHROPIC_MODEL" "$PROVIDERS")

  local tmp
  tmp=$(mktemp)
  jq --arg url "$base_url" \
     --arg token "$auth_token" \
     --arg emodel "$env_model" \
     --arg model "$model" \
     '.env.ANTHROPIC_BASE_URL = $url |
      .env.ANTHROPIC_AUTH_TOKEN = $token |
      .env.ANTHROPIC_MODEL = $emodel |
      .model = $model' \
     "$SETTINGS" > "$tmp" && mv "$tmp" "$SETTINGS"

  echo -e "${GREEN}Switched to ${BOLD}${name}${NC} (${CYAN}${env_model}${NC})"
  echo -e "${YELLOW}Restart Claude Code to apply changes.${NC}"
}

cmd_add() {
  local name="$1"

  jq -e --arg n "$name" 'has($n)' "$PROVIDERS" &>/dev/null \
    && die "Provider '$name' already exists. Use a different name or remove it first with --remove."

  echo -e "${BOLD}Adding provider: ${CYAN}${name}${NC}"
  read -rp "  ANTHROPIC_BASE_URL: " base_url
  read -rp "  ANTHROPIC_AUTH_TOKEN: " auth_token
  read -rp "  ANTHROPIC_MODEL: " env_model
  read -rp "  model (press Enter to use ANTHROPIC_MODEL): " model
  model="${model:-$env_model}"

  local tmp
  tmp=$(mktemp)
  jq --arg n "$name" \
     --arg url "$base_url" \
     --arg token "$auth_token" \
     --arg emodel "$env_model" \
     --arg m "$model" \
     '.[$n] = {"ANTHROPIC_BASE_URL": $url, "ANTHROPIC_AUTH_TOKEN": $token, "ANTHROPIC_MODEL": $emodel, "model": $m}' \
     "$PROVIDERS" > "$tmp" && mv "$tmp" "$PROVIDERS"

  echo -e "${GREEN}Provider '${name}' added.${NC}"
}

cmd_remove() {
  local name="$1"

  jq -e --arg n "$name" 'has($n)' "$PROVIDERS" &>/dev/null \
    || die "Provider '$name' not found."

  local current
  current=$(get_current)
  [[ "$name" == "$current" ]] && echo -e "${YELLOW}Warning: You are removing the currently active provider.${NC}"

  local tmp
  tmp=$(mktemp)
  jq --arg n "$name" 'del(.[$n])' "$PROVIDERS" > "$tmp" && mv "$tmp" "$PROVIDERS"

  echo -e "${GREEN}Provider '${name}' removed.${NC}"
}

cmd_current() {
  local current
  current=$(get_current)

  if [[ -z "$current" ]]; then
    echo -e "${YELLOW}No matching provider found in current settings.${NC}"
    echo -e "  Current config: $(jq -r '.env.ANTHROPIC_MODEL // "unknown"' "$SETTINGS") @ $(jq -r '.env.ANTHROPIC_BASE_URL // "unknown"' "$SETTINGS")"
  else
    local url model
    url=$(jq -r ".[\"$current\"].ANTHROPIC_BASE_URL" "$PROVIDERS")
    model=$(jq -r ".[\"$current\"].ANTHROPIC_MODEL" "$PROVIDERS")
    echo -e "Current: ${GREEN}${BOLD}${current}${NC}  ${CYAN}${model}${NC}  ${YELLOW}${url}${NC}"
  fi
}

# --- Main ---
check_deps

case "${1:-}" in
  --current|-c)
    cmd_current
    ;;
  --add|-a)
    [[ -z "${2:-}" ]] && die "Usage: claude-switch --add <name>"
    cmd_add "$2"
    ;;
  --remove|-r)
    [[ -z "${2:-}" ]] && die "Usage: claude-switch --remove <name>"
    cmd_remove "$2"
    ;;
  --help|-h)
    echo "Usage: claude-switch [OPTIONS] [NAME]"
    echo ""
    echo "Options:"
    echo "  --current, -c       Show current provider"
    echo "  --add, -a <name>    Add a new provider"
    echo "  --remove, -r <name> Remove a provider"
    echo "  --help, -h          Show this help"
    echo ""
    echo "Without arguments, lists all providers."
    echo "With a name argument, switches to that provider."
    ;;
  "")
    cmd_list
    ;;
  *)
    cmd_switch "$1"
    ;;
esac
```

### 将~/.local/bin/目录添加到系统的 PATH 环境变量中

- 此举的作用是，执行脚本时就不需要找绝对路径（~/.local/bin/claude-switch）下的脚本来执行了，而是在终端直接输入 claude-switch就可以识别到该脚本并执行

```bash
# 将下面这行命令加到 ~/.zshrc 文件中
# User's bash script load location
export PATH="$HOME/.local/bin:$PATH"

# 最后记得让改动的变量生效
source ~/.zshrc
```

### 用法速查

| 命令 | 作用 |
|------|------|
| `claude-switch` | 列出所有供应商（当前激活的标 `*`） |
| `claude-switch <name>` | 切换到指定供应商 |
| `claude-switch --current` | 查看当前供应商 |
| `claude-switch --add <name>` | 交互式添加新供应商 |
| `claude-switch --remove <name>` | 删除供应商 |
| `claude-switch --help` | 帮助 |

### 依赖

- [jq](https://jqlang.org/) — JSON 处理（macOS: `brew install jq`）
- `~/.local/bin` 在 PATH 中

切换后**重启 Claude Code** 生效。
