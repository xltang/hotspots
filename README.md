# Hotspots Skill

`hotspots` 是一个 OpenClaw Consumer Skill，用于从热点接口拉取 JSON 数据并生成结构化输出。

## 功能概览

- 拉取接口：`GET /hotspots/latest?userId=$USER_ID&&timestamp=$TIME_STEMP`
- 数据格式：仅支持 JSON（source 列表 + items）
- Top 区块：按模型预估点击率排序（最多 10 条）
- 分组展示：按 `source_name` 输出标题列表
- 定时任务：默认每天上午 `9:30`（`Asia/Shanghai`）

## 核心文件

- `SKILL.md`: Skill 主规范（安装、调度、调用、输出、可靠性、安全）

## 安装与初始化（按 SKILL.md）

1. 生成并持久化本地 `userID`（若已存在则复用）：
   - 路径：`~/.openclaw/hotspots/user_id`
2. 注册定时任务：
   - 任务名：`hotspots-scheduled-shanghai`
   - Cron：`30 9 * * *`
   - 时区：`Asia/Shanghai`
3. 定时消息会触发 skill 执行最新热点拉取与输出。

## 请求规范

每次请求前需生成分钟级时间戳：

- 示例：`TIME_STEMP="$(TZ=Asia/Shanghai date +%Y-%m-%dT%H:%M)"`

调用 URL：

- `https://hotspot.api4claw.com/hotspots/latest?userId=$USER_ID&&timestamp=$TIME_STEMP`

## 输出规范

输出顺序：

1. Top（最多 10 条）：标题 + AI 摘要（不显示 `hotness` / 热度 / 点击率数值）
2. 按 `source_name` 分组展示标题
3. 不显示 `fetched_at` 和 `data_date`

## 可靠性与约束

- 请求失败：返回明确失败原因
- JSON 非法：明确报格式错误并停止
- 数据异常项：跳过并继续，必要时报告跳过数量
- 禁止访问其它热点接口，仅允许：
  - `/hotspots/latest?userId=$USER_ID&&timestamp=$TIME_STEMP`

## 维护建议

- 若修改调度频率，更新 `SKILL.md` 中 `--cron` 和默认周期说明
- 若接口参数变更，同步更新：
  - `description`
  - `--message`
  - `Scheduled runs`
  - `Endpoints`
  - `Consumer Workflow`
  - `Reliability/Security` 规则
