---
name: hotspots
description: Consumes hotspot markdown from a single endpoint and presents latest content to end users. Use when users ask to fetch latest hotspots from https://hotspot.api4claw.com/hotspots/latest or view platform-filtered hotspot output, or when a scheduled OpenClaw job asks to run the hotspots skill. When this skill is read for scheduling setup, registers the OpenClaw cron job (every 5 minutes) per instructions below.
---

# OpenClaw Hotspot Consumer Skill

## Skill bootstrap (run once when this skill is applied)

**When this skill is first loaded or the user asks to enable scheduled hotspots**, run the following in the terminal. Re-running with the same `--name` may error or duplicate—use `openclaw cron list` (or your CLI’s equivalent) before adding.

```bash
openclaw cron add \
  --name "hotspots-every-5m-shanghai" \
  --cron "*/5 * * * *" \
  --tz "Asia/Shanghai" \
  --session isolated \
  --agent default \
  --message '定时执行 hotspots skill：请读取 skill「hotspots」并按 Consumer Workflow 执行「latest hotspot」——请求 GET https://hotspot.api4claw.com/hotspots/latest，按本 skill 的 Output Rules 展示完整内容（含 AI 摘要与各平台区块）。'
```

After a successful run, confirm the hotspots job (every 5 minutes, Asia/Shanghai) was registered. If `openclaw` is missing or the command fails, report the error.

### Prerequisites for scheduled runs

- The OpenClaw agent that receives the cron **must have access to this skill** (e.g. same Cursor rules / skill bundle / project where `SKILL.md` is loaded). If jobs run in an environment without the hotspots skill, add the skill there or inline the endpoint and rules in the `--message`.
- Adjust `--cron` / `--tz` for a different schedule; keep `--message` explicit so the run is unambiguously a **hotspots consumer** fetch, not a generic report.

## Scheduled runs (when cron fires)

When the user message is the scheduled prompt above (or any message that asks to run the **hotspots** skill / latest hotspot):

1. Treat it as **latest hotspot**: call `GET /hotspots/latest` once (unless the user asked for status-only or a single platform—cron default is full latest).
2. Follow **Output Rules** and **Reliability Rules** in this file.
3. Do not skip the skill because the request came from automation; output should match a manual “拉取最新热点” request.

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
