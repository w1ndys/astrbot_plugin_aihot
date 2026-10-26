# 需求实施计划

- [ ] 1. 实体层：固定值与数据形状
  - [ ] 1.1 编写 `entity/constants.py`
    - 常量：`PLUGIN_NAME`、`API_ORIGIN`、`SNAPSHOT_PATH`、`CHANGES_PATH`、`ITEMS_PATH`、`GROUP_MESSAGE_TYPE`
    - 上限与默认值：`SUMMARY_LIMIT=120`、`MESSAGE_CHAR_LIMIT=1500`、`MAX_ITEMS_DEFAULT=10`、`HISTORY_LIMIT=200`、`MAX_RETRY=3`、`MAX_CHANGES_CALLS=10`、`MIN_POLL_SECONDS=60`、`POLL_SECONDS_DEFAULT=900`、`COMMAND_LIMIT_DEFAULT=5`、`COMMAND_LIMIT_MAX=10`
    - `CATEGORY_LABELS` 六个 slug 到中文的映射（设计「实体层」；需求 5.5、10.1）
  - [ ] 1.2 编写 `entity/record.py`
    - `ItemRecord`、`ChangeEntry`、`GroupRow`、`SyncState`、`PendingRow`、`FetchResult`、`CollectResult`、`DeliverResult`、`ActionResult` 九个 dataclass
    - 字段与设计「实体层」表一致；不 import `astrbot`、不访问数据库（需求 2.6、3.1–3.3、7.1）
  - [ ] 1.3 为实体写单元测试
    - 断言 `CATEGORY_LABELS` 覆盖六个 slug、`FetchResult` 默认值、`ChangeEntry` 的 `item` 与 `item_id` 二选一约束

- [ ] 2. 数据层：连接与建表
  - [ ] 2.1 编写 `data/db.py`
    - `connect(path)` 返回标准库 `sqlite3.Connection`，开启 `row_factory`
    - `create_table(conn, sql)` 与 `init_schema(conn)`，建 `monitor_group`、`pushed_item`、`push_history`、`sync_state`、`pending_push` 五张表与 `idx_push_history_time` 索引
    - DDL 照抄设计「Data Models」，`sync_state` 用 `CHECK (id = 1)` 保证单行（设计「数据层」）
  - [ ] 2.2 为建表写单元测试
    - 临时目录建库两次，断言幂等；断言五张表与索引存在

- [ ] 3. 数据层：同步状态与已推记录
  - [ ] 3.1 编写 `data/sync_store.py`
    - `load()` 返回 `SyncState`，无行时返回全空默认值
    - `save_cursor(cursor, etag)`、`clear()`、`note_poll(result, error)`：写库成功后才改内存快照（需求 2.6、11.3）
  - [ ] 3.2 编写 `data/item_store.py`
    - `exists(item_id)`、`save_pushed(item)`、`mark_updated(item)`、`mark_removed(item_id)`、`count()`
    - 已推记录永久保留，不提供删除方法（需求 3.1–3.3、12.2；设计 P1）
  - [ ] 3.3 为两个 store 写单元测试
    - 断言 `save_pushed` 重复写入幂等、`mark_removed` 只改标记、`clear()` 后 `load()` 为空

- [ ] 4. 数据层：群、历史、待重试
  - [ ] 4.1 编写 `data/group_store.py`
    - 启动加载全表到内存；`list_all()`、`get()`、`insert()`、`save()`、`delete()`、`enabled_rows()` 走快照
    - `save_umo()`、`note_push_ok()`、`note_push_fail()`、`mark_notice_sent()`：先落库再改内存（需求 6.1、6.4、7.1、7.5）
  - [ ] 4.2 编写 `data/history_store.py` 与 `data/pending_store.py`
    - `append()`、`trim()`（只留最近 `HISTORY_LIMIT=200` 条）、`recent(limit=200)` 按时间倒序（需求 9.5、12.1）
    - `enqueue()`、`for_group()`、`bump_attempts()`、`delete()`、`count()`，主键 `(group_id, item_id)`（需求 7.1、7.2）
  - [ ] 4.3 为群、历史、待重试写单元测试
    - 历史写 250 条后 `trim()` 只剩 200 且保留最新；`bump_attempts` 累加到 3 后 `delete` 生效；重复 `enqueue` 不产生第二行

- [ ] 5. 数据层：AIHOT 客户端
  - [ ] 5.1 编写 `data/client.py`
    - `snapshot(limit=1)`、`changes(cursor, etag)`、`latest(limit)` 三个方法与 `FetchResult` 返回
    - `aiohttp` 在函数内运行时导入；统一带 `Accept-Encoding: gzip` 与配置 `user_agent`；`changes` 带 `If-None-Match`（需求 11.1、11.3、11.5）
    - 超时按 `http_timeout_seconds`；`proxy` 非空时只走该代理；429 读 `Retry-After` 写入 `FetchResult.retry_after`（需求 10.3、10.4、11.2）
    - 网络失败返回 `status=0` 与错误类型字符串，不抛异常（设计「错误处理」）
  - [ ] 5.2 用夹具响应为客户端写单元测试
    - `tests/fixtures/` 放本次核对得到的真实响应结构；断言 304、409（`code=snapshot_required`）、200 三种 `FetchResult` 与 `retry_after` 解析

- [ ] 6. 业务层：配置读取
  - [ ] 6.1 编写 `business/settings.py`
    - `poll_seconds`（小于 60 按 60）、`command_limit`（夹在 1..10）、`categories`、`min_score`、`keywords_block`、`keywords_allow`、`max_items`、`notice_enabled`、`proxy`、`timeout_seconds`、`user_agent`
    - 缺失或类型不符返回默认值（需求 10.1–10.4）
  - [ ] 6.2 为配置回落写单元测试
    - 空 dict 全取默认；`poll_seconds=30` 得 60；`command_limit=50` 得 10；字符串型 `categories` 被丢弃

- [ ] 7. 业务层：过滤
  - [ ] 7.1 编写 `business/filter.py`
    - `build_rules(config) -> FilterRules` 与 `should_push(item, rules) -> bool`
    - 判定顺序：分数 → 黑名单 → 白名单 → 分类；分类空列表全放行；`category=None` 走同一规则（需求 4.1–4.7；设计 P7）
  - [ ] 7.2 为过滤写单元测试
    - 覆盖分类空/非空、黑名单优先于白名单、白名单非空收敛、`min_score` 边界相等、`category=None`

- [ ] 8. 业务层：渲染
  - [ ] 8.1 编写 `business/render.py`
    - `category_label`（未映射与 `None` 返回「其他」）、`trim_summary`（截 120 字）、`render_item`（序号标题 / 摘要 / 分类·来源 / 双链接）
    - `render_push(items, total)`：0 条返回空列表；超过 `max_items` 取前 N 条并追加「另有 M 条未推送」；交 `split_messages` 拆分（需求 5.1–5.6）
    - `split_messages`：按条目边界累计到 1500 字换段，段首加「第 x/y 段」；`render_digest`、`render_notice`（需求 5.4、6.5、8.1）
  - [ ] 8.2 为渲染写单元测试
    - 摘要截断、`summary` 为空省行、`links.original` 为空省行、12 条候选只出 10 条并注明剩余、超长正文分段且每段不超 1500 字（设计 P5）

- [ ] 9. 业务层：增量收集
  - [ ] 9.1 编写 `business/sync.py` 的 `collect()`
    - 游标为空 → `snapshot(limit=1)` 写游标、返回空候选（需求 1.1–1.4）
    - 304 → 空候选且保留游标；409 → `clear()` 后返回空候选；200 → 逐条处理 `changes[]`（需求 2.1–2.6）
    - `upsert` 未见过入候选并 `save_pushed`，已见过 `mark_updated`；`remove` → `mark_removed`，两类都不入候选（需求 3.1–3.3）
    - `hasMore` 循环到 10 次上限后保留原游标退出（需求 2.3、2.4；设计 P2）
  - [ ] 9.2 为 `collect()` 写单元测试
    - 假客户端返回 304 / 409 / 单批 upsert / 多批 hasMore / 达到上限五种序列，断言候选、游标与 `save_pushed` 调用次数
  - [ ] 9.3 写属性测试：同一条目只进候选一次
    - 对应设计 P1：随机生成含重复 id 的增量序列，断言候选去重且 `pushed_item` 行数不增

- [ ] 10. 业务层：推送编排
  - [ ] 10.1 编写 `business/push.py`
    - `umo_for(group_id, platform_id)`、`rewrite_umo(umo, platform_id)`（需求 6.2、6.3、6.4）
    - `deliver(groups, pending, history, texts, sender, platform_id, now)`：开关关跳过；通知先发且 `notice_sent` 置 1；待重试先补推；本轮内容失败入队（需求 6.5、6.6、7.1–7.6）
    - 不 import `astrbot`，发送经 `sender` 回调（设计「业务层」）
  - [ ] 10.2 为 `deliver()` 写单元测试
    - 假 sender 断言 UMO 重写并回写、通知只发一次、成功清空 `fail_count`、失败入队、`attempts` 达 3 后删除并记日志、关闭群不发（设计 P4、P6）
  - [ ] 10.3 写属性测试：开关关闭的群不产生任何发送
    - 对应设计 P4：随机群开关组合，断言 sender 只收到开启群的 UMO

- [ ] 11. 业务层：查询指令
  - [ ] 11.1 编写 `business/command.py`
    - `latest_digest(client, limit) -> (正文, 错误)`；条数夹在 1..10；失败返回空正文与错误串；不写游标、不写任何表（需求 8.1–8.4）
  - [ ] 11.2 为指令摘要写单元测试
    - 假客户端成功与失败两条路径；断言超限条数被夹到 10

- [ ] 12. 检查点 - 确保所有测试通过
  - 确保所有测试通过,如有疑问请询问用户
  - 运行 `python3 -m unittest discover -s tests -t .`，确认未安装 `astrbot` 与 `aiohttp` 时业务层用例仍可执行

- [ ] 13. 入口层：插件骨架与轮询任务
  - [ ] 13.1 编写 `main.py` 的插件类与轮询
    - `AihotPlugin(Star)`：`__init__` 建库、建 store、注册 WebUI、注册指令、起 `asyncio.create_task`；`terminate()` 取消任务（需求 6.1、10.1）
    - `_tick()`：`collect` → 无候选且无待重试则只更新 `last_poll_at` → 否则 `render_push` → `deliver` → 写游标与 ETag（设计「入口层」；需求 7.6）
    - `_sender()` 用 `MessageChain` 封装 `context.send_message`；`_platform_id()` 优先配置值，否则取 OneBot 实例 `meta().id`（需求 6.2、6.4、11.4）
    - 日志经 `astrbot.api.logger`，带插件名前缀且不含代理凭据（需求 12.3、12.4）
  - [ ] 13.2 编写 WebUI 群接口
    - `GET/POST /astrbot_plugin_aihot/groups`、`POST /groups/update`、`POST /groups/delete`；请求体校验群号为数字、平台 ID 不含冒号；新增默认开启（需求 6.1、9.1、9.2）
  - [ ] 13.3 编写状态、测试推送、历史接口
    - `GET /status` 返回游标摘要、上次轮询、上一轮结果、待重试条数；`POST /test-push` 向指定群发固定样例并返回成功或错误；`GET /history` 返回最近 200 条（需求 9.3–9.5）
  - [ ] 13.4 编写配置读写接口与指令 handler
    - `GET/PUT /settings` 只读写非密钥字段，保存调 `self.config.save_config()`（需求 9.6、9.7）
    - `@filter.command("aihot")` handler 调 `business/command.py`，失败回固定提示（需求 8.1–8.5）

- [ ] 14. 配置清单与依赖
  - [ ] 14.1 编写 `_conf_schema.json` 与 `requirements.txt`、`__init__.py`
    - 十一个字段与默认值照需求 10.1，每项带 `hint`（需求 9.7、10.1）
    - `requirements.txt` 只写 `aiohttp`；`__init__.py` 只标明插件包（设计「配置」）
  - [ ] 14.2 为 `_conf_schema.json` 写结构校验用例
    - 断言默认值与需求 10.1 一致、字段名与 `business/settings.py` 读取的键一致（防漂移）

- [ ] 15. 前端：监控群页
  - [ ] 15.1 搭建 `dashboard/` 工程并接 bridge
    - 复用 `astrbot_plugin_codex_reset` 的 `vite.config.ts`、`bridge.ts`、`App.tsx`、`StyleProvider layer` 方案；Vite 打成单份 IIFE，HTML 用 `<script defer>`（设计「前端」）
    - 页内 `Tabs` 四个页签：监控群 / 监控状态 / 推送记录 / 全局配置（需求 9.1）
  - [ ] 15.2 实现监控群 Tab
    - 表格列：群号、备注、开关、UMO 是否存在、`last_push_at`、`fail_count`、`last_error`；行内保存；上方新增群（默认开启）与删除；行内测试推送按钮（需求 9.1、9.2、9.4）
    - `fail_count > 0` 的行标红（需求 7.4）
  - [ ] 15.3 实现监控状态 Tab 与推送记录 Tab
    - 状态：游标摘要、上次轮询时间、上一轮结果、待重试条数（需求 9.3）
    - 记录：倒序 200 条，含群号、标题、时间、结果（需求 9.5）
  - [ ] 15.4 实现全局配置 Tab
    - 手写非密钥字段表单，打开时读 schema 当前值，保存走 `PUT /settings`（需求 9.6、9.7）
  - [ ] 15.5 构建产物落到 `views/monitor/`
    - `cd dashboard && npm install && npm run build`，产出 `views/monitor/index.html` 与 `assets/index.js`、`assets/style.css`；确认无 `type="module"`、无第二份 JS chunk、无 CDN 引用

- [ ] 16. 检查点 - 确保所有测试通过
  - 确保所有测试通过,如有疑问请询问用户
  - 复跑 `python3 -m unittest discover -s tests -t .`，并确认 `views/monitor/` 产物已提交、`metadata.yaml` 与 README 描述一致

## 需求覆盖核对

| 需求 | 对应任务 |
|---|---|
| 1 同步基线 | 5.1、9.1 |
| 2 增量拉取 | 5.1、9.1、9.2 |
| 3 条目处理 | 3.2、9.1、9.3 |
| 4 过滤 | 7.1、7.2 |
| 5 消息格式 | 8.1、8.2 |
| 6 群与目标 | 4.1、10.1、13.1、13.2、15.2 |
| 7 失败重试 | 4.1、4.2、10.1、10.2、13.1、15.2 |
| 8 查询指令 | 11.1、13.4 |
| 9 页面 | 13.2–13.4、15.1–15.4 |
| 10 配置默认值 | 1.1、6.1、14.1 |
| 11 请求纪律 | 5.1、13.1 |
| 12 留存与日志 | 3.2、4.2、13.1 |
