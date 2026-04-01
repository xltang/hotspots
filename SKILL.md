---
name: hotspots
description: Consumes latest hotspot content from https://hotspot.api4claw.com/hotspots/latest (response may be markdown text or image/binary) and presents it to end users. On skill install or first apply, registers an OpenClaw cron job every 5 minutes (Asia/Shanghai). Use when users ask for latest hotspots, status, platform-filtered output, or when a cron message triggers this skill.
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
  --cron "*/5 * * * *" \
  --tz "Asia/Shanghai" \
  --session isolated \
  --agent default \
  --announce \
  --channel last \
  --to "当前会话的用户ID（自动替换）" \
  --message '定时执行 hotspots skill：请读取 skill「hotspots」并按 Consumer Workflow 执行「latest hotspot」——请求 GET https://hotspot.api4claw.com/hotspots/latest；响应可能是 markdown 或图片，按本 skill Output Rules 完整展示。'
```

- 默认周期 `*/5 * * * *`（每 5 分钟一次）；若用户要求更低频，可改为 `*/15 * * * *` 或 `*/30 * * * *`，修改时保留同一 `--name` 前先 `cron remove` 再 `add`（以 CLI 实际语法为准）。
- 注册成功后向用户确认：**已自动设置定时任务**（周期、时区、任务名）。

若 `openclaw` 未安装或命令失败：**不要静默跳过**——说明原因，并给出用户可手动执行的同一条 `cron add` 命令。

### Prerequisites for scheduled runs

- 接收 cron 的 OpenClaw agent **须能加载本 skill**（同一项目中的 `SKILL.md` / 规则包）。若定时环境无本 skill，须在 `--message` 中内联完整 URL 与 Output Rules 要点。
- 修改 `--cron` / `--tz` 时保持 `--message` 明确为 **hotspots consumer** 拉取，避免与其它定时报告混淆。

## Scheduled runs（定时触发时）

当消息为上述 `--message` 内容，或明确要求执行 **hotspots** / 最新热点：

1. 视为 **latest hotspot**：对 `GET https://hotspot.api4claw.com/hotspots/latest` 请求一次（除非用户只要 status 或单平台；定时默认输出完整最新内容）。按 `Content-Type` 处理 markdown 或图片。
2. 遵循本文件 **Output Rules** 与 **Reliability Rules**。
3. 不因来自自动化而省略步骤；输出应与用户手动「拉取最新热点」一致。

## Scope

This skill is only for Consumer behavior.

Use this skill when users ask to:

- read or view latest hotspot content (markdown and/or image from `/hotspots/latest`)
- check hotspot service status
- view one platform section when the payload is markdown (section slicing does not apply to image-only responses)

Do not include Publisher generation logic or Server upload/storage internals in responses.

## Endpoints

Base URL:

- `https://hotspot.api4claw.com`

Only endpoint:

- `GET /hotspots/latest`: fetch the latest hotspot **payload**. The body may be **markdown text** (e.g. `Content-Type: text/markdown` or `text/plain`) or **image content** (e.g. `image/png`, `image/jpeg`, `image/webp`, `image/gif`). Treat according to the response `Content-Type` and body.

## Consumer Workflow

For each user intent:

- `latest hotspot`: call `GET /hotspots/latest`, inspect `Content-Type` and body, then display:
  - **Markdown (or plain text treated as markdown)**: render as usual; apply platform filter and Output Rules below.
  - **Image**: 
    1. Save the image to a temporary file (e.g., `/tmp/hotspot_latest.png` or `workspace/hotspot_latest.png`)
    2. If executing in a cron job or automated context where the target channel and user are known (e.g., from cron job's `--channel` and `--to` parameters), use the `openclaw message send` command to embed the image inline:
       ```bash
       openclaw message send \
         --channel <current_channel> \
         --target <current_user_id> \
         --media <image_file_path> \
         --message "热点截图 | 更新时间: {update_time} | 尺寸: {width}×{height}"
       ```
    3. If in an interactive session where channel/user context is not predefined, describe the image and offer to send it via the current channel.
    4. Do not pretend missing text sections exist.
- `status`: call `GET /hotspots/latest`, then report reachable/unreachable (same as today; status does not depend on markdown vs image).
- `platform filter`: only when the fetched body is **markdown** (or text with recognizable section headers): fetch first, then locally slice by section keyword. If the response is **image-only**, state that platform filtering does not apply and show the image (or describe accessibility constraints if images cannot be shown).

### Implementation Notes for Image Display

When the skill is triggered by a cron job, the following context is typically available:
- The cron job's `--channel` parameter (e.g., `feishu`, `telegram`, etc.)
- The cron job's `--to` parameter (target user/group ID)
- The agent has access to the workspace filesystem for saving temporary image files

For proper inline image display, the skill should:
1. Determine the current channel and target from context (cron parameters or session metadata)
2. Save the received image data to a local file
3. Use the appropriate channel-specific method to send the image:
   - **Feishu**: Use `openclaw message send --channel feishu --media <file>`
   - **Telegram**: Use `openclaw message send --channel telegram --media <file>`
   - Other channels as supported by OpenClaw

Platform filter targets (markdown sections only):

- `weibo`
- `zhihu`
- `xiaohongshu`
- `cctv`

When returning **filtered markdown** output, always keep the AI summary block. Image responses have no section slice; preserve the full image.

## Configuration

Required:

- `HOTSPOT_BASE_URL`: set to `https://hotspot.api4claw.com`.

Recommended defaults:

- request timeout around `6000 ms`
- clear error text for timeout/network/HTTP failures

## Output Rules

When showing hotspot content, branch on format:

**If the response is markdown (or text with the same structure):**

1. show update time if available (from body headers or markdown metadata if present)
2. show AI summary block first when present in the body
3. show platform sections in stable order:
   - Weibo
   - Zhihu
   - Xiaohongshu
   - CCTV
4. if partial/missing, explicitly list unavailable sections

**If the response is an image:**

1. **Save the image**: Write the image data to a local file (e.g., `workspace/hotspot_latest.png`, `workspace/hotspot_latest.jpg`)
2. **Send inline image**: Use OpenClaw's message sending capability to embed the image in the response:
   - For cron/scheduled runs: Use `openclaw message send` with the channel and target from the cron job configuration
   - For interactive sessions: Use the current session's channel and user context
3. **Provide metadata**: Include image information:
   - Update time (from `Last-Modified`, `Date`, or response headers)
   - Image dimensions (if available or can be determined)
   - File size and format
4. **Do not invent content**: Do not invent markdown sections or an AI summary block that are not in the response
5. **Multiple images**: If multiple images are returned by the API (e.g. multipart or JSON with URLs), show each in a consistent order

**Important for cron jobs**: When this skill is triggered by a scheduled cron job, the agent should:
- Save the image to the workspace (ensuring it persists for the session)
- Use the same channel and target as configured in the cron job
- Include a descriptive message with the image (e.g., "热点截图 | 更新时间: {timestamp}")

When showing status:

- reachable or unreachable
- latest update time (if any)
- endpoint used: `/hotspots/latest`

## Reliability Rules

- If `/hotspots/latest` fails, return explicit failure reason.
- Do not fabricate content when server is unreachable.
- For **image** responses: if decode or display fails, say so and offer the raw URL or content-type if available; do not substitute with unrelated markdown.
- Return explicit degraded reason and a next action.
- Keep responses concise and user-facing.

## Security Rules

- Do not expose tokens or secret headers in output.
- Do not call any hotspot endpoint except `/hotspots/latest`.
