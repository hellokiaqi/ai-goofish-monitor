# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

---

## 项目概述

AI Goofish Monitor 是基于 Playwright、FastAPI、Vue 3、SQLite 与 OpenAI 兼容多模态大模型的闲鱼（Goofish）多任务实时监控系统，支持多模态 AI 图文+卖家+历史价格评估、本地关键词规则过滤、多账号与代理池轮换、失败熔断保护以及 7 渠道消息推送。

---

## 核心架构与数据流

```text
Vue 3 SPA (web-ui/ -> 构建直接输出至根目录 dist/)
    │  HTTP (/api/*, /auth/status, /health) + WebSocket (/ws)
    ▼
FastAPI 主服务 (src/app.py)
    ├── API 路由层: src/api/routes/ (dashboard, tasks, accounts, results, logs, settings, prompts, login_state, websocket)
    ├── 业务服务层: src/services/
    │   ├── TaskService / TaskGenerationService / TaskGenerationRunner (任务 CRUD 与后台异步生成 AI 标准)
    │   ├── ProcessService (拉起与终止 spider_v2.py 子进程，管理 logs/<task>_<id>.log)
    │   ├── SchedulerService (APScheduler Cron 定时调度，时区 Asia/Shanghai)
    │   ├── ResultStorageService / ResultExportService / ResultBlacklistService (SQLite 结果查询、黑名单与 CSV 导出)
    │   ├── PriceHistoryService / DashboardService (SQLite 价格快照、趋势聚合与首页统计)
    │   └── NotificationService / NotificationConfigService (7 渠道通知推送与配置热更新)
    ├── 领域模型层: src/domain/ (Task, TaskGenerationJob, TaskRepository)
    └── 基础设施层: src/infrastructure/
        ├── config/      (settings.py Pydantic 配置, env_manager.py .env 读写)
        ├── persistence/ (sqlite_connection.py, sqlite_bootstrap.py, sqlite_task_repository.py, json_task_repository.py)
        └── external/    (ai_client.py, notification_clients/: ntfy, pushplus, bark, gotify, wecom_bot, telegram, webhook)
    │
    │  ProcessService 启动独立子进程: python -u spider_v2.py --task-name "<任务名>"
    ▼
爬虫与分析执行链 (spider_v2.py -> src/scraper.py)
    1. 默认从 SqliteTaskRepository 读取任务（仅传 --config <path> 时走 JSON 兼容模式）
    2. RotationPool 选择账号 (auto / fixed / rotate) 与代理池；FailureGuard 检查连续失败熔断状态
    3. Playwright 访问闲鱼搜索页应用价格/个人闲置/包邮/新发布/省市区三级筛选并翻页，写入 SQLite price_snapshots
    4. ItemAnalysisDispatcher 后台受控并发处理新商品：
       - SellerProfileCache 采集并缓存卖家主页信用、在售/已售商品与历史评价好评率
       - decision_mode=ai: src/ai_handler.py 下载商品图片至 images/task_images_<task_name>/ 并调用多模态模型分析（完后自动删图）
       - decision_mode=keyword: src/keyword_rule_engine.py 执行单组 OR 关键词匹配（含英数字词边界保护）
       - 结果写入 SQLite result_items，命中推荐时通过 NotificationService 推送通知
```

其他组件：
- `desktop_launcher.py`：本机或 PyInstaller 打包后的单文件启动入口（启动 FastAPI 并打开浏览器）。
- `chrome-extension/`：Manifest V3 扩展，用于从 `goofish.com` 导出 Cookie 与浏览器环境快照 JSON。

---

## 常用命令

```bash
# 后端开发（默认端口 SERVER_PORT=8000）
python -m src.app
uvicorn src.app:app --host 0.0.0.0 --port 8000 --reload

# 前端开发（代理 /api、/auth、/ws 到 http://127.0.0.1:8000）
cd web-ui && npm install && npm run dev

# 前端构建（直接输出到仓库根目录 dist/，无需手动拷贝）
cd web-ui && npm run build

# 一键环境检查、前端构建并启动后端
./start.sh

# CLI 运行爬虫
python spider_v2.py                                    # 运行 SQLite 中所有已启用任务
python spider_v2.py --task-name "Sony A7M4"            # 运行指定名称任务
python spider_v2.py --debug-limit 3                    # 调试模式：每任务最多处理 3 个新商品
python spider_v2.py --config config.json               # 显式读取 JSON 配置文件

# Docker 启动
docker compose up -d
docker compose -f docker-compose.dev.yaml up --build -d

# 测试与覆盖率
PYTEST_DISABLE_PLUGIN_AUTOLOAD=1 pytest
PYTEST_DISABLE_PLUGIN_AUTOLOAD=1 pytest tests/unit/test_utils.py::test_safe_get_nested_and_default
PYTEST_DISABLE_PLUGIN_AUTOLOAD=1 coverage run -m pytest && coverage report

# 真实流量冒烟测试（默认关闭，需有效 state/*.json 与 AI 配置）
./run_live_smoke.sh --without-generation
```

---

## 存储与开发注意事项

- **SQLite 为在线主存储**：默认文件 `data/app.sqlite3`（可通过 `APP_DATABASE_FILE` 覆盖），包含 `tasks`、`result_items`、`price_snapshots`、`result_blacklist_rules`、`app_metadata`。首次启动空库时 `sqlite_bootstrap.py` 会自动从旧版 `config.json`、`jsonl/`、`price_history/` 迁移一次数据。新增功能请直接走 SQLite 服务/仓储层，不要直接读写磁盘 `jsonl/` 或 `config.json`。
- **文件系统目录**：`.env`（配置）、`state/*.json` 与 `xianyu_state.json`（登录态）、`prompts/*.txt`（提示词模板与标准）、`logs/`（任务日志、AI 请求摘要、失败熔断状态）、`images/`（任务临时图片，结束自动清理）仍存储在文件系统中。
- **双端配置同步**：API 主进程使用 `src/infrastructure/config/settings.py` + `env_manager.py`，爬虫子进程同时读取 `src/config.py`；新增环境变量需同步维护这两处与 `.env.example`。
- **前端构建路径契约**：`web-ui/vite.config.ts` 配置了 `outDir: '../dist'`，不要改为 `web-ui/dist`，否则会破坏 `tests/test_frontend_build_paths.py` 与 Docker 构建。
- **AI 与通知扩展**：AI 请求统一复用 `src/services/ai_request_compat.py`（自动兼容 Chat Completions 与 Responses API）；通知渠道支持 `ntfy`、`pushplus`、`bark`、`gotify`、`wecom`、`telegram`、`webhook` 共 7 种。
- **测试规范**：离线测试位于 `tests/unit/`、`tests/integration/` 及根目录 `tests/test_*.py`，采用同步 `def test_*` 搭配 `asyncio.run()` 或 `TestClient`，并使用 `tmp_path` 隔离 SQLite 与文件读写。
