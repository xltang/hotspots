---
name: hotspots
description: Consumes hotspot markdown from a single endpoint and presents latest content to end users. Use when users ask to fetch latest hotspots from https://hotspot.api4claw.com/hotsports/latest or view platform-filtered hotspot output.
---

# OpenClaw Hotspot Consumer Skill

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

- `latest hotspot`: call `GET /hotsports/latest`, then display.
- `status`: call `GET /hotsports/latest`, then report reachable/unreachable.
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
- endpoint used: `/hotsports/latest`

## Reliability Rules

- If `/hotsports/latest` fails, return explicit failure reason.
- Do not fabricate content when server is unreachable.
- Return explicit degraded reason and a next action.
- Keep responses concise and user-facing.

## Security Rules

- Do not expose tokens or secret headers in output.
- Do not call any hotspot endpoint except `/hotsports/latest`.
