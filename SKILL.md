---
name: hotspots
description: Consumes hotspot markdown from https://hotspot.api4claw.com/hotspots/latest and presents it to end users. On skill install or first apply, registers an OpenClaw cron job so hotspots run on a schedule without manual setup. Use when users ask for latest hotspots, status, platform-filtered output, or when a cron message triggers this skill.
---

# OpenClaw Hotspot Consumer Skill

## Installation：安装本 skill 时自动注册定时任务

**Mandatory：** 用户将本 skill 安装到项目或首次在本对话/会话中启用本 skill 时，**必须立即**完成定时任务注册（不要等用户再说「帮我设定时」）。

1. 在终端执行 `openclaw cron list`（或当前 OpenClaw CLI 等价命令），查找是否已有名为 `hotspots-scheduled-shanghai` 的任务。
2. **若已存在**：向用户简短确认「定时任务已存在」，跳过添加。
3. **若不存在**：执行下面的 `openclaw cron add`（仅注册一次）。

```bash
openclaw cron add \
  --name "hotspots-scheduled-shanghai" \
  --cron "*/30 * * * *" \
  --tz "Asia/Shanghai" \
  --session isolated \
  --agent default \
  --message '定时执行 hotspots skill：请读取 skill「hotspots」并按 Consumer Workflow 执行「latest hotspot」——请求 GET https://hotspot.api4claw.com/hotspots/latest，按本 skill 的 Output Rules 展示完整内容（含 AI 摘要与各平台区块）。'
```

- 默认周期 `*/30 * * * *` 与热点内容约 30 分钟更新节奏一致；若用户要求更高频，可改为 `*/5 * * * *` 并保留同一 `--name` 前先 `cron remove` 再 `add`（以 CLI 实际语法为准）。
- 注册成功后向用户确认：**已自动设置定时任务**（周期、时区、任务名）。

若 `openclaw` 未安装或命令失败：**不要静默跳过**——说明原因，并给出用户可手动执行的同一条 `cron add` 命令。

### Prerequisites for scheduled runs

- 接收 cron 的 OpenClaw agent **须能加载本 skill**（同一项目中的 `SKILL.md` / 规则包）。若定时环境无本 skill，须在 `--message` 中内联完整 URL 与 Output Rules 要点。
- 修改 `--cron` / `--tz` 时保持 `--message` 明确为 **hotspots consumer** 拉取，避免与其它定时报告混淆。

## Scheduled runs（定时触发时）

当消息为上述 `--message` 内容，或明确要求执行 **hotspots** / 最新热点：

1. 视为 **latest hotspot**：对 `GET https://hotspot.api4claw.com/hotspots/latest` 请求一次（除非用户只要 status 或单平台；定时默认输出完整最新内容）。
2. 遵循本文件 **Output Rules** 与 **Reliability Rules**。
3. 不因来自自动化而省略步骤；输出应与用户手动「拉取最新热点」一致。

## Scope

This skill is only for Consumer behavior.

Use this skill when users ask to:

- read latest hotspot markdown
- check hotspot service status
- view one platform section from hotspot markdown

Do not include Publisher generation logic or Server upload/storage internals in responses.

## Endpoints

Base URL:

- `https://hotspot.api4claw.com`

Only endpoint:

- `GET /hotspots/latest`: fetch latest markdown content.

## Consumer Workflow

For each user intent:

- `latest hotspot`: call `GET /hotspots/latest`, then display.
- `status`: call `GET /hotspots/latest`, then report reachable/unreachable.
- `platform filter`: fetch markdown first, then locally slice by section keyword.

Platform filter targets:

- `weibo`
- `zhihu`
- `xiaohongshu`
- `cctv`

Always keep the AI summary block when returning filtered output.

## Configuration

Required:

- `HOTSPOT_BASE_URL`: set to `https://hotspot.api4claw.com`.

Recommended defaults:

- request timeout around `6000 ms`
- clear error text for timeout/network/HTTP failures

## Output Rules

When showing hotspot content:

1. show update time if available
2. show AI summary block first
3. show platform sections in stable order:
   - Weibo
   - Zhihu
   - Xiaohongshu
   - CCTV
4. if partial/missing, explicitly list unavailable sections

When showing status:

- reachable or unreachable
- latest update time (if any)
- endpoint used: `/hotspots/latest`

## Reliability Rules

- If `/hotspots/latest` fails, return explicit failure reason.
- Do not fabricate content when server is unreachable.
- Return explicit degraded reason and a next action.
- Keep responses concise and user-facing.

## Security Rules

- Do not expose tokens or secret headers in output.
- Do not call any hotspot endpoint except `/hotspots/latest`.
