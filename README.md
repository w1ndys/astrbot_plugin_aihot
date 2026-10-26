# AIHOT 精选监控

AstrBot 插件 `astrbot_plugin_aihot` `0.1.0`。只读请求 [aihot.news](https://aihot.news) 的公开 API，把 AIHOT 新入选的**精选**内容合并成一条消息，推送到 WebUI 里开启的群。

推送目标在插件页面增删改查，不使用群口令。群内另有只读指令 `aihot` 查当前最新精选。

> 当前状态：设计阶段。需求见 `specs/2026-10-26-aihot-monitor/requirements.md`，技术设计见同目录 `design.md`，尚无插件实现代码。

## 会做什么

- 第一次运行只请求一次 `selected/snapshot`（`fields=default`、`limit=1`）拿同步游标，不推装机前的旧内容。
- 之后每 900 秒（可配）请求一次 `selected/changes?cursor=…`，走官方增量接口，只处理新增、修改、撤选三类变化。
- 新增的条目按分类、最低分、关键词黑白名单过滤后合并成一条消息；同一轮超过 10 条只推前 10 条并注明剩余。
- 修改和撤选只在本地更新，不在群里说话。
- 每群第一次推送前先发一条「已开启监控」通知，带上 `aihot` 指令用法，只发一次。
- 推送失败进待重试队列，下一轮先补推，累计 3 轮仍失败则丢弃并记日志；页面把持续失败的群标红，开关保持原值。

## 用到的接口

全部匿名只读，不需要 API Key，版本 2.0.0，路径统一 `/api/v1`。

| 路径 | 用途 | 用到的字段 |
|---|---|---|
| `GET /api/v1/selected/snapshot?fields=default&limit=1` | 首次建同步基线 | `cursor`、`asOf` |
| `GET /api/v1/selected/changes?cursor=…` | 增量主路径 | `cursor`、`hasMore`、`changes[].op`、`changes[].item`（`id`、`title`、`summary`、`category`、`score`、`source.name`、`links.aihot`、`links.original`）、`changes[].id` |
| `GET /api/v1/items?mode=selected&limit=N` | 群内 `aihot` 指令 | `items[]` 同上 |

接口纪律：带 `Accept-Encoding: gzip` 与可识别的 `User-Agent`；增量请求带 `If-None-Match`，304 直接当「无变化」；`429` 按 `Retry-After` 等待；`409 snapshot_required` 时清游标重建基线。单 IP 约 60 次/分钟封顶。

## 配置项

| 配置 | 默认 | 说明 |
|---|---|---|
| `platform_id` | 空 | OneBot 平台实例 ID，不能含冒号。留空时等群先说话记下会话 |
| `poll_seconds` | 900 | 轮询间隔，低于 60 按 60 执行 |
| `command_limit` | 5 | `aihot` 指令返回条数，上限 10 |
| `categories` | `[]` | 分类多选，空表示全推 |
| `min_score` | 0 | 最低分阈值 |
| `keywords_block` | `[]` | 关键词黑名单，命中即丢 |
| `keywords_allow` | `[]` | 关键词白名单，非空时只放行命中项 |
| `notice_enabled` | `true` | 是否发开启通知 |
| `http_timeout_seconds` | 40 | 单次请求超时 |
| `proxy` | 空 | 留空跟随 AstrBot 全局代理 |
| `user_agent` | `astrbot_plugin_aihot/0.1 (github.com/w1ndys)` | 请求 UA |

分类中文名：`ai-models` 模型发布、`ai-products` 产品更新、`industry` 行业动态、`paper` 论文研究、`tip` 教程实践、`opinion` 大佬观点，其余与 `null` 记为「其他」。

## 插件页面

源码在 `dashboard/`，React 19 + antd 6 + Vite 打成单份 IIFE，产物放 `views/monitor/`，宿主不需要 Node。

- 监控群：群号、备注、开关、会话是否已记录、最后推送时间、连续失败次数；行内改完保存，上方加新群（默认开启）。
- 监控状态：当前游标摘要、上次轮询时间与结果、待重试条数。
- 测试推送：选一个群发一条固定样例，验证平台实例 ID 与会话串。
- 推送记录：最近 200 条，倒序。
- 全局配置：手写非密钥字段，读写 `_conf_schema.json` 当前值。

## 安装

放到 `AstrBot/data/plugins/astrbot_plugin_aihot`，在 WebUI 重载。依赖见 `requirements.txt`（仅 `aiohttp`）。

## 数据

业务库在 `data/plugin_data/astrbot_plugin_aihot/aihot.db`：`monitor_group`、`pushed_item`、`push_history`、`sync_state`、`pending_push`。已推 id 永久保留作去重依据，推送记录只留最近 200 条。

## 署名

推送内容附站内链接 `https://aihot.news/items/<id>` 与原文链接；内容来自 AIHOT 的聚合摘要，原文版权归各来源所有。使用边界见 [aihot.news/terms](https://aihot.news/terms)。
