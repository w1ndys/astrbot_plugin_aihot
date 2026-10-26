# astrbot_plugin_aihot

AstrBot 插件。用中文交流。

## 编码契约

本仓库开发默认启用 `human-coding-contract`（人话编码契约 / 土法编码 / 分层交付），skill 取全局安装的那份，不在本仓另存副本。

- 缺背景就停，问用户。不要猜接口字段，字段以 `https://aihot.news/openapi-v1.json` 当前定义为准。
- 设计顺序：实体 → 数据 → 业务 → 入口。一刀一个逻辑目标。
- 写完本刀、跑检查、**不要提交**。停一次，同一条消息给拟议提交、审阅路径（函数粒度，按层）、本次完成、检查、全部进度、当前下一步。不要在对话里贴完整 git diff。用户回「继续」才提交刚审过的改动。
- 等审时不要偷做下一刀。总体做完就停。
- 入口层只调业务层，不要在 handler 里直接查库。
- 函数（不含空行和注释）≤ 50 行。普通语法，不要多余抽象 / DI / utils 藏逻辑。
- 每个文件、每个函数、每个 `if` 都要有短中文注释（判断目的和后果）。
- 提交前跑已有检查：`python3 -m unittest discover -s tests -t .`。本仓还没有 ruff 配置时，不要假装跑过 ruff。
- 不引入 `requests`，网络用 `aiohttp`，并在函数内运行时导入，保证单测不装网络库也能跑。
- 密钥、token、密码不得进代码、注释、日志、diff 展示和提交信息。
- 未经用户明确允许不要 push。

## 分层

| 层 | 目录 | 职责 |
|---|---|---|
| 实体 | `entity/` | 固定值（接口地址、上限、分类中文名）与数据形状 |
| 数据 | `data/` | SQLite、公开接口读取；连库建表放 `data/db.py` |
| 业务 | `business/` | 增量处理、过滤、文案拼装、重试与推送编排 |
| 入口 | `main.py`、`views/` | 轮询任务、WebUI 接口、`aihot` 查询指令 |

推送目标只在 WebUI 维护，不使用群口令。

## 文档

- `specs/2026-10-26-aihot-monitor/requirements.md`：需求（EARS）
- `specs/2026-10-26-aihot-monitor/design.md`：技术设计
- `specs/2026-10-26-aihot-monitor/tasklist.md`：实施计划（拆解阶段产出）
