---
name: hotspots
description: Consumes latest hotspot content from https://hotspot.api4claw.com/hotspots/latest?userId=$USER_ID and presents it to end users. Response format is JSON array (sources with items). Ranks Top items by model-estimated click-through rate (not raw hotness). On skill install or first apply, registers an OpenClaw cron job every 5 minutes (Asia/Shanghai). Use when users ask for latest hotspots, status, source-grouped titles, top items by estimated CTR, or when a cron message triggers this skill.
---

# OpenClaw Hotspot Consumer Skill

## Installation：安装本 skill 时自动注册定时任务

**Mandatory：** 用户将本 skill 安装到项目或首次在本对话/会话中启用本 skill 时，**必须立即**完成定时任务注册（不要等用户再说「帮我设定时」）。

1. 先在 OpenClaw 本地生成并持久化 `userID`（若已存在则复用，不重复生成）：

```bash
mkdir -p ~/.openclaw/hotspots
if [ ! -s ~/.openclaw/hotspots/user_id ]; then
  if command -v uuidgen >/dev/null 2>&1; then
    uuidgen | tr 'A-Z' 'a-z' > ~/.openclaw/hotspots/user_id
  elif command -v python3 >/dev/null 2>&1; then
    python3 -c "import uuid; print(uuid.uuid4())" > ~/.openclaw/hotspots/user_id
  else
    # Last resort fallback: timestamp + random suffix
    printf "uid-%s-%s\n" "$(date +%s)" "$(LC_ALL=C tr -dc 'a-z0-9' </dev/urandom | head -c 8)" > ~/.openclaw/hotspots/user_id
  fi
fi
USER_ID="$(cat ~/.openclaw/hotspots/user_id)"
```

2. 在终端执行 `openclaw cron list`（或当前 OpenClaw CLI 等价命令），查找是否已有名为 `hotspots-scheduled-shanghai` 的任务。
3. **若已存在**：向用户简短确认「定时任务已存在」，并回显本地 `userID`（`$USER_ID`）。
4. **若不存在**：执行下面的 `openclaw cron add`（仅注册一次）。

```bash
openclaw cron add \
  --name "hotspots-scheduled-shanghai" \
  --cron "*/5 * * * *" \
  --tz "Asia/Shanghai" \
  --session isolated \
  --agent default \
  --announce \
  --channel "当前会话的channel（自动替换）" \
  --to "当前会话的用户ID（自动替换）" \
  --message '定时执行 hotspots skill：请读取 skill「hotspots」并按 Consumer Workflow 执行「latest hotspot」——请求 GET https://hotspot.api4claw.com/hotspots/latest?userId=$USER_ID；按 JSON 解析：Top 区块按预估点击率排序（不展示热度和内部排序逻辑），再按 source_name 分组展示标题。'
```

- 默认周期 `*/5 * * * *`（每 5 分钟一次，时区见 `--tz`）；若用户要求更低频，可改为 `*/15 * * * *`、`*/30 * * * *` 或 `0 * * * *`（每小时整点）。修改时保留同一 `--name` 前先 `cron remove` 再 `add`（以 CLI 实际语法为准）。
- 注册成功后向用户确认：**已自动设置定时任务**（周期、时区、任务名、`userID=$USER_ID`）。

若 `openclaw` 未安装或命令失败：**不要静默跳过**——说明原因，并给出用户可手动执行的同一条 `cron add` 命令。

### Prerequisites for scheduled runs

- 接收 cron 的 OpenClaw agent **须能加载本 skill**（同一项目中的 `SKILL.md` / 规则包）。若定时环境无本 skill，须在 `--message` 中内联完整 URL 与 Output Rules 要点。
- 修改 `--cron` / `--tz` 时保持 `--message` 明确为 **hotspots consumer** 拉取，避免与其它定时报告混淆。

## Scheduled runs（定时触发时）

当消息为上述 `--message` 内容，或明确要求执行 **hotspots** / 最新热点：

1. 视为 **latest hotspot**：对 `GET https://hotspot.api4claw.com/hotspots/latest?userId=$USER_ID` 请求一次（除非用户只要 status 或单平台；定时默认输出完整最新内容），并按 JSON 结构处理。
2. 遵循本文件 **Output Rules** 与 **Reliability Rules**。
3. 不因来自自动化而省略步骤；输出应与用户手动「拉取最新热点」一致。

## Scope

This skill is only for Consumer behavior.

Use this skill when users ask to:

- read latest hotspot data from `/hotspots/latest?userId=$USER_ID` (primary: JSON)
- get top items ranked by estimated click-through rate (with AI summaries)
- view titles grouped by `source_name`
- check hotspot service status

Do not include Publisher generation logic or Server upload/storage internals in responses.

## Endpoints

Base URL:

- `https://hotspot.api4claw.com`

Only endpoint:

- `GET /hotspots/latest?userId=$USER_ID`: fetch the latest hotspot payload as JSON array. Each element is a source block (e.g. `source`, `source_name`, `fetched_at`, `items[]`), and each item usually includes `title`, `content`, `link`, `hotness`. Do **not** use `hotness` for display or ranking; use model-estimated CTR for Top ordering only.

## Consumer Workflow

For each user intent:

- `latest hotspot`:
  1. Call `GET /hotspots/latest?userId=$USER_ID`.
  2. If body is JSON array, parse and flatten `items[]` across all sources.
  3. Build **Top (max 10)** list: for every item, assign an **estimated click-through rate (预估点击率)** internally from `title` + `content` only (ranking signal only). Sort all items by this estimate descending, take up to 10. Do **not** sort by JSON `hotness` and do **not** expose `hotness` or 热度 in output.
  4. For each Top item, write concise AI summary from `title` + `content` (1-2 lines each, no fabrication).
  5. After Top section, group display by `source_name`, showing item titles under each source (still no `hotness`/热度).
- `status`: call `GET /hotspots/latest?userId=$USER_ID`, then report reachable/unreachable and basic stats from JSON (source count, total item count). Do NOT show `fetched_at` or `data_date`.
- `source filter`: filter by `source`/`source_name`.

JSON grouping targets (if present):

- `xwlb` / `新闻联播`
- `weibo` / `微博`
- `zhihu` / `知乎`
- `qqMorningPost` / `腾讯早报`

## Configuration

Required:

- `HOTSPOT_BASE_URL`: set to `https://hotspot.api4claw.com`.

Recommended defaults:

- request timeout around `6000 ms`
- clear error text for timeout/network/HTTP failures

## Output Rules

When showing hotspot content, use this order:

1. **Top（最多 10 条，按预估点击率）**:
   - Flatten JSON `items[]`, estimate CTR per item from `title` + `content`, sort by estimate desc, take up to 10.
   - For each entry, show **title + AI summary** (from `title/content`). Do **not** show `hotness`, numeric hotness, or the word 热度, or the word 点击率.
   - Optional: prefix with rank `1.` … `10.` only; do not print numeric CTR unless the user explicitly asks for a quantitative estimate.
2. **By source_name**:
   - Group all items by `source_name`.
   - Under each group, list item titles in original order (or by estimated CTR within that group if the user asks for ranking).
3. **Metadata**:
   - Do NOT show `fetched_at` or `data_date` in output.
4. **Completeness checks**:
   - If some source has empty `items`, keep the source header and mark it as empty.

When showing status:

- reachable or unreachable
- endpoint used: `/hotspots/latest?userId=$USER_ID`
- Do NOT show `fetched_at` or `data_date`

## Reliability Rules

- If `/hotspots/latest?userId=$USER_ID` fails, return explicit failure reason.
- Do not fabricate content when server is unreachable.
- In JSON mode, skip malformed items safely and continue with valid items; report skipped count briefly if non-zero.
- If response is not valid JSON, report format mismatch explicitly and stop.
- Return explicit degraded reason and a next action.
- Keep responses concise and user-facing.

## Security Rules

- Do not expose tokens or secret headers in output.
- Do not call any hotspot endpoint except `/hotspots/latest?userId=$USER_ID`.
