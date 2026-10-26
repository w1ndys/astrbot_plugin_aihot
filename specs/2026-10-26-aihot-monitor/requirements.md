# Requirements Document

## Introduction

`astrbot_plugin_aihot` 是 AstrBot 的一个独立插件。插件只读请求 AIHOT 的公开接口，跟踪 AIHOT「精选」条目的增量变化，把新入选的条目合并成一条消息，推送到运维者在 WebUI 里开启的群。

插件不抓取网页、不保存第三方正文、不提供对外接口；推送内容只有标题、AI 摘要、分类、来源名与两个链接。

本文档覆盖已与用户确认的 v1 范围：只做精选增量。日报、周报、月报、热点榜、全部动态与 Codex 重置监控不在 v1 范围。

## Glossary

- **系统**：`astrbot_plugin_aihot` 插件本体。
- **精选**：AIHOT 站点中被编辑策展选中的条目，接口口径为 `selected=true`，通过 `/api/v1/selected/*` 读写。
- **同步游标**：`/api/v1/selected/snapshot` 或 `/api/v1/selected/changes` 响应里的 `cursor` 字符串，代表已同步到的时间点。
- **增量条目**：`/api/v1/selected/changes` 响应 `changes[]` 数组里的元素，形状为 `{op:"upsert", changedAt, item}` 或 `{op:"remove", changedAt, id}`。
- **已推记录**：本地 `pushed_item` 表里的一行，代表某个条目已经进入过推送候选。
- **推送目标**：群表里一行开启的记录，对应一个平台会话串（unified_msg_origin，以下简称 UMO）。
- **UMO**：AstrBot 的会话唯一标识，形如 `平台实例ID:GroupMessage:群号`。
- **待重试队列**：`pending_push` 表，保存某个群在本轮未发送成功的推送内容。
- **轮次**：一次完整的「拉增量 → 过滤 → 合并 → 推送 → 推进游标」处理。
- **平台实例 ID**：AstrBot 平台适配器实例的 `meta().id`，用于拼接 UMO 前缀。
- **分类 slug**：接口 `category` 字段取值 `ai-models`、`ai-products`、`industry`、`paper`、`tip`、`opinion`，另有 `null`。
- **本轮候选**：通过去重与过滤、等待合并成一条消息的条目集合。

## Requirements

### Requirement 1

**User Story:** AS 运维者, I want 插件第一次运行时只建立同步基线, so that 群里不会突然出现装机前的旧内容。

#### Acceptance Criteria

1. WHEN 插件加载且本地同步游标为空, 系统 SHALL 请求 `/api/v1/selected/snapshot`，参数 `fields=default`、`limit=1`。
2. WHEN 该快照请求返回 200, 系统 SHALL 把响应 `cursor` 与 `asOf` 写入同步状态，并把响应 `items` 丢弃、不写入已推记录。
3. IF 该快照请求失败, 系统 SHALL 保持同步游标为空，并在 `poll_seconds` 之后重试。
4. WHILE 同步游标为空, 系统 SHALL 每轮只重试快照接口，不请求增量接口。

### Requirement 2

**User Story:** AS 运维者, I want 插件按官方增量接口取变化, so that 插件只处理真正变化的条目，且能拿到修改与撤选。

#### Acceptance Criteria

1. WHEN 本地同步游标非空, 系统 SHALL 请求 `/api/v1/selected/changes`，参数 `cursor=<同步游标>`，并携带 `If-None-Match` 头。
2. WHEN 增量接口返回 304, 系统 SHALL 结束本轮且不发送任何消息，并保留原同步游标。
3. WHEN 增量响应 `hasMore` 为 `true`, 系统 SHALL 用响应 `cursor` 继续请求增量接口，直到 `hasMore` 为 `false`。
4. IF 单轮增量请求次数达到 10 次且 `hasMore` 仍为 `true`, 系统 SHALL 保留本轮开始前的同步游标，并在下一轮用该游标重新请求，已推记录保证同一条目只进入候选一次。
5. IF 增量接口返回 409 且响应 `code` 为 `snapshot_required`, 系统 SHALL 清空同步游标与 ETag，并按 Requirement 1 重建基线。
6. WHEN 增量响应返回 200 且 `hasMore` 为 `false`, 系统 SHALL 把最后一次响应 `cursor` 记为待提交游标，并在本轮所有群处理完成后写库。

### Requirement 3

**User Story:** AS 群成员, I want 群里只出现真正新增的精选, so that 改稿和撤稿不会造成重复刷屏。

#### Acceptance Criteria

1. WHEN `changes[]` 元素的 `op` 为 `upsert` 且 `item.id` 不在已推记录中, 系统 SHALL 把该 `item` 加入本轮候选。
2. WHEN `changes[]` 元素的 `op` 为 `upsert` 且 `item.id` 已在已推记录中, 系统 SHALL 只更新已推记录的 `title`、`category`、`score`，并把该条目标记为「已修改」，不加入本轮候选。
3. WHEN `changes[]` 元素的 `op` 为 `remove`, 系统 SHALL 把该 `id` 在已推记录中标记为已撤选，不加入本轮候选。
4. WHEN 本轮候选条数为 0, 系统 SHALL 结束本轮且不发送任何消息、不写入推送记录。
5. WHEN 本轮候选条数超过 `max_items`（默认 10）, 系统 SHALL 只取前 10 条推送，并在消息末尾注明未推送条数。

### Requirement 4

**User Story:** AS 运维者, I want 用一套全局规则筛掉不需要的内容, so that 群里只看到关心的方向。

#### Acceptance Criteria

1. WHEN 配置 `categories` 为空列表, 系统 SHALL 对全部 `category` 取值放行。
2. WHEN 配置 `categories` 非空, 系统 SHALL 只放行 `category` 在列表内的条目。
3. IF 条目的 `title` 或 `summary` 命中 `keywords_block` 中任一关键词, 系统 SHALL 丢弃该条目。
4. WHEN 配置 `keywords_allow` 非空, 系统 SHALL 只放行 `title` 或 `summary` 命中 `keywords_allow` 中任一关键词的条目。
5. WHEN 条目 `score` 低于配置 `min_score`, 系统 SHALL 丢弃该条目。
6. IF 条目 `category` 为 `null`, 系统 SHALL 按 Requirement 4.1 与 4.2 的放行规则判定，并在消息模板中显示为「其他」。
7. WHEN 同一条目同时命中 `keywords_allow` 与 `keywords_block`, 系统 SHALL 丢弃该条目。

### Requirement 5

**User Story:** AS 群成员, I want 一条消息读完本轮内容, so that 群里不被碎消息刷屏。

#### Acceptance Criteria

1. WHEN 本轮候选至少 1 条, 系统 SHALL 生成一条以 `AIHOT 新精选 · N 条` 开头的消息，`N` 为本轮候选条数。
2. WHEN 渲染单个条目, 系统 SHALL 依次输出序号与标题、摘要行（截断至 120 字）、`分类中文名 · 来源名` 行、站内链接行、原文链接行。
3. WHEN 条目 `summary` 为 `null` 或空串, 系统 SHALL 省略摘要行，并保持其余行顺序不变。
4. WHEN 渲染后的消息正文超过 1500 字, 系统 SHALL 按条目边界拆分为多条消息，并在每段首行追加 `第 x/y 段`。
5. IF 条目 `category` 不在中文映射表内, 系统 SHALL 用「其他」作为分类中文名。
6. WHEN 条目 `links.original` 为空, 系统 SHALL 只输出站内链接行。

### Requirement 6

**User Story:** AS 运维者, I want 在页面里维护推送群, so that 不用在群里发口令就能决定推到哪。

#### Acceptance Criteria

1. WHEN 运维者在页面新增群并保存, 系统 SHALL 把群号、备注、开关写入群表，新增记录的开关默认取值为开启。
2. WHEN 群记录的 UMO 为空且配置 `platform_id` 非空, 系统 SHALL 用 `<platform_id>:GroupMessage:<群号>` 作为推送目标。
3. WHEN 群记录的 UMO 为空且配置 `platform_id` 为空, 系统 SHALL 跳过该群并记录一条缺少会话的错误。
4. WHEN 已记录 UMO 的前缀与当前平台实例的 `meta().id` 不一致, 系统 SHALL 在发送前把 UMO 前缀重写为当前平台实例 `meta().id`，并把重写后的值写回群表。
5. WHEN 某个群 `notice_sent` 为 0 且配置 `notice_enabled` 为 `true`, 系统 SHALL 在该群本轮内容推送之前发送一条开启通知，内容包含 `aihot` 指令用法，并把 `notice_sent` 置为 1。
6. WHILE 群记录开关为关闭, 系统 SHALL 跳过该群的开启通知与内容推送。

### Requirement 7

**User Story:** AS 运维者, I want 推送失败能自己补上, so that 平台短暂异常不会让内容凭空消失。

#### Acceptance Criteria

1. WHEN 某个群在本轮发送失败, 系统 SHALL 把该群本轮内容写入待重试队列，并把该群 `last_error` 更新为错误类型、`fail_count` 加 1。
2. WHEN 待重试队列中某条记录 `attempts` 达到 3 且本轮仍发送失败, 系统 SHALL 删除该记录并写一条丢弃日志。
3. WHEN 新一轮开始且待重试队列非空, 系统 SHALL 先补推待重试内容，再处理本轮候选。
4. WHEN 群 `fail_count` 大于 0, 系统 SHALL 在页面把该群标记为异常并展示连续失败次数，群开关保持原值。
5. WHEN 某群推送成功, 系统 SHALL 把该群 `last_push_at` 更新为本轮时间、`fail_count` 归零、`last_error` 清空。
6. WHEN 一轮内所有群处理完成, 系统 SHALL 写入 Requirement 2.6 的待提交游标与 ETag。

### Requirement 8

**User Story:** AS 群成员, I want 用一条指令看当前最新精选, so that 不用等下一次推送。

#### Acceptance Criteria

1. WHEN 群成员发送 `aihot` 指令, 系统 SHALL 回复当前最新精选，条数取 `command_limit`（默认 5）。
2. WHEN `command_limit` 大于 10, 系统 SHALL 按 10 条返回。
3. IF 接口请求失败或超时, 系统 SHALL 回复一条固定失败提示，本轮不修改同步游标、ETag 与已推记录。
4. WHILE 处理 `aihot` 指令, 系统 SHALL 保持只读，不写入群表、推送记录与待重试队列。
5. WHEN 指令由群成员触发, 系统 SHALL 按 AstrBot 全局唤醒策略响应，指令名为 `aihot`。

### Requirement 9

**User Story:** AS 运维者, I want 一个页面看完群、状态和推送记录, so that 排查一次就够了。

#### Acceptance Criteria

1. WHEN 运维者打开监控群页, 系统 SHALL 展示每群的群号、备注、开关、UMO 是否存在、`last_push_at`、`fail_count` 与 `last_error`。
2. WHEN 运维者保存群改动, 系统 SHALL 写入群表并返回更新后的行数据。
3. WHEN 运维者打开监控状态页, 系统 SHALL 展示当前游标摘要、上次轮询时间、上一轮结果、待重试条数。
4. WHEN 运维者点击测试推送并选定群, 系统 SHALL 向该群发送一条固定样例消息，并返回成功或错误摘要。
5. WHEN 运维者打开推送记录页, 系统 SHALL 按时间倒序展示最近 200 条记录，含群号、标题、时间、结果。
6. WHEN 运维者修改全局配置页字段并保存, 系统 SHALL 把非密钥字段写回插件配置并落盘。
7. WHEN 运维者打开全局配置页, 系统 SHALL 展示配置项当前值而非默认值。

### Requirement 10

**User Story:** AS 运维者, I want 插件开箱即可跑, so that 只填平台实例 ID 就能用。

#### Acceptance Criteria

1. WHEN 插件加载且配置项缺失, 系统 SHALL 使用默认值：`poll_seconds=900`、`command_limit=5`、`categories=[]`、`min_score=0`、`keywords_block=[]`、`keywords_allow=[]`、`max_items=10`、`notice_enabled=true`、`http_timeout_seconds=40`。
2. WHEN 配置 `poll_seconds` 小于 60, 系统 SHALL 按 60 秒执行。
3. WHEN 配置 `proxy` 非空, 系统 SHALL 让全部 HTTP 请求走该代理。
4. WHEN 配置 `proxy` 为空, 系统 SHALL 沿用 AstrBot 全局代理设置。

### Requirement 11

**User Story:** AS 接口提供方, I want 插件按公开规则请求, so that 不触发限流也不浪费带宽。

#### Acceptance Criteria

1. WHEN 发起任何 HTTP 请求, 系统 SHALL 携带配置的 `user_agent` 与 `Accept-Encoding: gzip`。
2. WHEN 响应状态为 429, 系统 SHALL 读取 `Retry-After` 头，并把下一轮等待时长设为 `Retry-After` 与 `poll_seconds` 两者中的较大值。
3. WHEN 响应包含 `ETag`, 系统 SHALL 保存该值用于下一次条件请求。
4. WHILE 上一轮尚未结束, 系统 SHALL 跳过本轮调度，不并发发起同一接口的请求。
5. WHEN 单次请求耗时超过 `http_timeout_seconds`, 系统 SHALL 取消该请求并按轮询错误处理。

### Requirement 12

**User Story:** AS 运维者, I want 数据不会无限膨胀、日志能定位问题, so that 长期跑不用管。

#### Acceptance Criteria

1. WHEN 推送记录行数超过 200, 系统 SHALL 删除最旧的记录，保留最近 200 条。
2. WHEN 已推记录写入, 系统 SHALL 永久保留该行，作为去重依据。
3. WHEN 插件发生异常或完成关键动作, 系统 SHALL 通过 `astrbot.api.logger` 输出带插件名前缀的日志。
4. WHEN 输出日志, 系统 SHALL 省略代理地址中的凭据、`user_agent` 之外的请求头与任何密钥值。
