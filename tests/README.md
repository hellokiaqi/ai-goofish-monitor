# 测试说明

本项目使用 `pytest` 组织自动化测试，分为**离线快速测试**（单元测试 + 集成测试 + 仓库级契约测试）与**真实流量冒烟测试**（Live Smoke）两层。默认执行 `pytest` 时只运行离线测试，不访问真实闲鱼、AI 接口或通知服务。

---

## 环境准备

测试依赖（`pytest`、`pytest-asyncio`、`coverage`）包含在根目录 `requirements.txt` 中（注：`requirements-runtime.txt` 仅含生产运行依赖，不含测试包）：

```bash
pip install -r requirements.txt
```

建议在本地运行 `pytest` 时加上 `PYTEST_DISABLE_PLUGIN_AUTOLOAD=1`，避免宿主环境中全局安装的第三方 pytest 插件干扰同步测试执行。

---

## 测试目录结构

```text
tests/
├── conftest.py                               # 共享 fixtures（临时 SQLite、FastAPI TestClient、FakeProcessService/Scheduler）
├── test_failure_guard.py                     # 任务连续失败熔断器与 Cookie 更新自动恢复测试
├── test_frontend_build_paths.py              # 前端构建输出路径跨 Vite / Dockerfile / start.sh 一致性契约测试
├── fixtures/                                 # 离线测试使用的真实结构脱敏样例数据
│   ├── config.sample.json                    # 示例任务 JSON 配置
│   ├── search_results.json                   # 闲鱼搜索接口返回样例
│   ├── user_head.json                        # 卖家个人主页头部摘要样例
│   ├── user_items.json                       # 卖家在售/已售商品列表样例
│   ├── ratings.json                          # 卖家评价列表样例
│   └── state.sample.json                     # 登录态快照样例
├── unit/                                     # 模块级单元测试（全部离线 Mock）
│   ├── test_ai_client.py                     # AIClient 初始化、重试与 NO_PROXY IPv6 清理
│   ├── test_ai_handler_analysis.py           # 商品多模态分析流程与校验
│   ├── test_ai_handler_downloads.py          # 商品图片并发下载与清理
│   ├── test_ai_request_compat.py             # Chat Completions / Responses API 互转与降级参数构建
│   ├── test_ai_response_parser.py            # JSON 提取、代码块剥离与 reasoning_content 回退
│   ├── test_app_lifespan.py                  # FastAPI 启动/关闭生命周期与状态重置
│   ├── test_cron_utils.py                    # 5 段 / 6 段 Cron 与 @daily 等别名校验
│   ├── test_domain_task.py                   # Task / TaskCreate / TaskUpdate 领域校验与规则归一化
│   ├── test_env_manager.py                   # .env 读写、转义与键删除
│   ├── test_item_analysis_dispatcher.py      # 后台并发分析分发器（AI / 关键词 / 跳过 AI）
│   ├── test_keyword_rule_engine.py           # 关键词 OR 匹配与纯英数字词边界匹配
│   ├── test_notification_service.py          # 7 种通知渠道校验、脱敏与发送逻辑
│   ├── test_price_history_service.py         # SQLite 价格快照聚合、日趋势与性价比评分
│   ├── test_process_service.py               # spider_v2.py 子进程启停、日志句柄与重排
│   ├── test_prompt_utils.py                  # AI 分析标准生成与 <think> 块剥离
│   ├── test_result_blacklist_service.py      # 结果关键词与 re: 正则黑名单匹配
│   ├── test_scraper_browser_channel.py       # 本地 Chrome/Edge 与 Docker Chromium 通道解析
│   ├── test_scraper_extra_headers.py         # 浏览器快照请求头安全过滤
│   ├── test_scraper_region_filter.py         # 搜索页省/市/区三级筛选交互逻辑
│   ├── test_search_pagination.py             # 搜索翻页监听与超时重试
│   ├── test_seller_profile_cache.py          # 卖家画像 TTL 缓存与并发请求合并
│   ├── test_task_log_cleanup_service.py      # 过期任务运行日志自动清理
│   └── test_utils.py                         # 嵌套字典读取、链接归一化与 SQLite 结果写入封装
├── integration/                              # 跨层集成测试（基于临时 SQLite 与 TestClient）
│   ├── test_api_dashboard.py                 # /api/dashboard/summary 统计聚合接口
│   ├── test_api_results.py                   # /api/results 查询、筛选、状态修改、黑名单与 CSV 导出
│   ├── test_api_settings.py                  # /api/settings 通知、AI、轮换与系统状态接口
│   ├── test_api_tasks.py                     # /api/tasks CRUD、启停、异步 AI 生成 Job 与 Cron 校验
│   ├── test_cli_spider.py                    # spider_v2.py CLI 参数、Prompt 拼接与关键词模式
│   └── test_pipeline_parse.py                # 搜索 -> 卖家主页 -> 评价好评率完整解析链路
└── live/                                     # 真实流量端到端冒烟测试（默认跳过）
    ├── conftest.py                           # RUN_LIVE_TESTS=1 门控与独立临时工作区 uvicorn 进程管理
    ├── _support.py                           # 环境变量加载、通知屏蔽、空闲端口分配与健康检查等待
    └── test_live_smoke.py                    # 健康检查、真实 AI 连通性、真实闲鱼抓取落库与 AI 生成任务验证
```

---

## 常用测试命令

### 1. 运行全部离线测试（日常开发必跑）

```bash
PYTEST_DISABLE_PLUGIN_AUTOLOAD=1 pytest
```

### 2. 运行指定目录、文件或单个测试函数

```bash
# 仅跑单元测试或集成测试
PYTEST_DISABLE_PLUGIN_AUTOLOAD=1 pytest tests/unit
PYTEST_DISABLE_PLUGIN_AUTOLOAD=1 pytest tests/integration

# 指定单个测试文件
PYTEST_DISABLE_PLUGIN_AUTOLOAD=1 pytest tests/integration/test_api_tasks.py

# 指定单个测试函数
PYTEST_DISABLE_PLUGIN_AUTOLOAD=1 pytest tests/unit/test_utils.py::test_safe_get_nested_and_default
```

### 3. 代码覆盖率统计

`pyproject.toml` 已将覆盖率统计目录指向 `src/`：

```bash
PYTEST_DISABLE_PLUGIN_AUTOLOAD=1 coverage run -m pytest
coverage report
coverage html   # 输出到 htmlcov/index.html
```

---

## Live Smoke（真实流量冒烟测试）

`tests/live/` 用于在真实浏览器、真实闲鱼登录态与真实大模型接口下验证完整抓取链路。`pyproject.toml` 注册了两个 marker：

- `live`：需要真实凭据与外部网络服务的冒烟测试（未设置 `RUN_LIVE_TESTS=1` 时自动 skip）。
- `live_slow`：耗时较长的真实 AI 分析标准生成测试（由 `LIVE_ENABLE_TASK_GENERATION=1` 控制）。

### 一键脚本运行（推荐）

```bash
# 默认执行任务 CRUD 前置测试 + 全部 live 测试（含 AI 任务生成）
./run_live_smoke.sh

# 跳过较慢的 AI 任务生成用例，仅验证真实抓取与商品 AI 分析
./run_live_smoke.sh --without-generation

# 自定义关键词、账号文件与抓取上限
./run_live_smoke.sh --keyword "MacBook Air M1" --account-file state/acc_1.json --debug-limit 1

# 仅打印将要执行的命令与配置，不实际运行
./run_live_smoke.sh --dry-run
```

### 手动通过环境变量运行

```bash
RUN_LIVE_TESTS=1 \
PYTEST_DISABLE_PLUGIN_AUTOLOAD=1 \
LIVE_TEST_ACCOUNT_STATE_FILE=/abs/path/to/state/acc_1.json \
LIVE_TEST_KEYWORD="MacBook Pro M2" \
LIVE_ENABLE_TASK_GENERATION=0 \
pytest tests/live -m "live and not live_slow" -v
```

### Live 测试环境变量一览

| 变量名 | 默认值 | 说明 |
| :--- | :--- | :--- |
| `RUN_LIVE_TESTS` | 未设置（关闭） | 必须显式设为 `1` 才会执行 `tests/live` |
| `LIVE_TEST_ACCOUNT_STATE_FILE` | 自动取 `state/*.json` 第一个 | 真实闲鱼登录态 JSON 路径 |
| `LIVE_TEST_KEYWORD` | `MacBook Pro M2` | 冒烟测试搜索关键词 |
| `LIVE_TEST_TASK_NAME` | `Live Smoke Task` | 冒烟测试创建的任务名称 |
| `LIVE_EXPECT_MIN_ITEMS` | `1` | 期望至少落库的结果条数，达到后自动发停止信号 |
| `LIVE_TEST_DEBUG_LIMIT` | `1` | 透传为 `SPIDER_DEBUG_LIMIT`，限制单次最多抓取分析的新商品数 |
| `LIVE_TIMEOUT_SECONDS` | `180` | 单个异步阶段的轮询超时时间（秒） |
| `LIVE_ENABLE_TASK_GENERATION` | Python 直接跑默认 `false`；`run_live_smoke.sh` 默认 `1` | 是否开启 `test_live_ai_task_generation_job` |

**隔离机制说明**：`live_server` fixture 会在系统临时目录下创建独立工作区与独立数据库 `data/live.sqlite3`，自动寻找空闲本地端口启动真实 `uvicorn`，并清空常见通知渠道环境变量，避免污染仓库根目录数据。若测试失败，临时工作区目录会被保留并在终端打印路径，方便查看 `live-app.log` 与 `logs/*_<id>.log`。

---

## 编写新测试的约定

1. **文件与函数命名**：单元测试放入 `tests/unit/test_*.py`，API/CLI/跨模块链路放入 `tests/integration/test_*.py`，测试函数以 `test_` 开头。
2. **同步调用异步代码**：离线测试函数统一写成普通同步函数 `def test_xxx(...)`，内部使用 `asyncio.run(...)` 或 FastAPI `TestClient` 驱动异步逻辑，不依赖 `pytest-asyncio` 自动模式。
3. **严格隔离存储与副作用**：涉及 SQLite、`.env`、`logs/`、`prompts/` 读写的测试必须使用 `tmp_path`（例如传入 `db_path`、设置 `APP_DATABASE_FILE` 或 `monkeypatch.chdir(tmp_path)`），禁止读写仓库根目录真实 `data/app.sqlite3` 或 `.env`。
4. **离线可重复**：单元与集成测试中涉及 Playwright、OpenAI SDK、`requests` 通知推送等外部 I/O 一律 Mock，需要真实响应结构时复用或补充 `tests/fixtures/*.json`。
