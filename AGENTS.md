# AGENTS.md

本文件面向参与本仓库开发的工程师与 Coding Agent，记录基于当前真实源码的模块职责、启动/测试命令、存储边界与改动注意事项。

---

## 1. 架构与进程模型

本项目由 **FastAPI 主服务进程**、**按任务拉起的 `spider_v2.py` 爬虫子进程** 与 **Vue 3 SPA 前端** 组成：

```text
Vue 3 Web UI (web-ui/ -> 构建输出至根目录 dist/)
    │  HTTP (/api/*, /auth/status, /health) + WebSocket (/ws)
    ▼
FastAPI 主进程 (src/app.py)
    ├── API 层: src/api/routes/{dashboard,tasks,accounts,results,logs,settings,prompts,login_state,websocket}.py
    ├── 服务层: src/services/ (TaskService, ProcessService, SchedulerService, TaskGenerationService,
    │          NotificationService, ResultStorageService, PriceHistoryService, DashboardService 等)
    ├── 领域层: src/domain/models/{task,task_generation}.py, src/domain/repositories/task_repository.py
    └── 基础设施层: src/infrastructure/
        ├── config/      (settings.py Pydantic 配置, env_manager.py .env 持久化读写)
        ├── persistence/ (sqlite_connection.py, sqlite_bootstrap.py, sqlite_task_repository.py, json_task_repository.py)
        └── external/    (ai_client.py, notification_clients/{ntfy,pushplus,bark,gotify,wecom_bot,telegram,webhook}_client.py)
    │
    │  ProcessService 通过 asyncio.create_subprocess_exec 拉起独立子进程
    ▼
爬虫子进程 (spider_v2.py --task-name "<任务名>")
    ├── 读取任务: 默认从 SqliteTaskRepository 读取；仅显式传 --config <path> 时读取 JSON
    ├── 组装 Prompt: AI 模式下将 prompts/<keyword>_criteria.txt 嵌入 prompts/base_prompt.txt 的 {{CRITERIA_SECTION}}
    └── 执行抓取 (src/scraper.py: scrape_xianyu)
        ├── 账号/代理池与熔断: src/rotation.py (RotationPool), src/failure_guard.py (FailureGuard)
        ├── 页面解析与翻页: src/parsers.py, src/services/search_pagination.py
        ├── 价格快照记录: src/services/price_history_service.py -> 写入 SQLite price_snapshots
        └── 后台商品分析分发: src/services/item_analysis_dispatcher.py (ItemAnalysisDispatcher)
            ├── 卖家画像采集与缓存: scrape_user_profile + src/services/seller_profile_cache.py
            ├── AI 模式: src/ai_handler.py (并发下载图片至 images/task_images_<task_name>/ -> 多模态 AI 分析 -> 清理临时图)
            ├── 关键词模式: src/keyword_rule_engine.py (单组 OR 匹配，英数字完整词边界保护)
            ├── 结果落库: src/utils.py:save_to_jsonl -> ResultStorageService -> 写入 SQLite result_items
            └── 推荐推送: NotificationService 并发推送至所有已启用通知渠道
```

其他入口：
- `desktop_launcher.py`：切换工作目录至项目根（或 PyInstaller `_MEIPASS`），自动打开浏览器并启动 `uvicorn`（监听 `127.0.0.1:<SERVER_PORT>`）。
- `chrome-extension/`：Manifest V3 浏览器扩展，从 `goofish.com` 导出 Cookie 与浏览器环境指纹 JSON。

---

## 2. 开发、构建与测试命令

```bash
# 后端启动（默认端口 SERVER_PORT=8000）
python -m src.app
# 或开发热重载
uvicorn src.app:app --host 0.0.0.0 --port 8000 --reload

# 前端开发（Vite 代理 /api、/auth、/ws 到 127.0.0.1:8000）
cd web-ui && npm install && npm run dev

# 前端类型检查与生产构建（注意：产物直接输出到仓库根目录 dist/，非 web-ui/dist/）
cd web-ui && npm run build

# 本地一键检查环境、安装依赖、构建前端并启动后端
./start.sh

# 爬虫 CLI 直接运行
python spider_v2.py                                    # 运行 SQLite 中所有已启用任务
python spider_v2.py --task-name "Sony A7M4"            # 运行单个任务
python spider_v2.py --debug-limit 3                    # 每个任务最多处理 3 个新商品
python spider_v2.py --config config.json               # 显式使用 JSON 任务文件兼容模式

# Docker 部署与本地镜像构建
docker compose up -d                                   # 使用官方镜像启动
docker compose -f docker-compose.dev.yaml up --build -d  # 基于本地源码构建启动

# 离线单元与集成测试（推荐禁用第三方 pytest 插件自动加载）
PYTEST_DISABLE_PLUGIN_AUTOLOAD=1 pytest
PYTEST_DISABLE_PLUGIN_AUTOLOAD=1 pytest tests/unit/test_utils.py::test_safe_get_nested_and_default
PYTEST_DISABLE_PLUGIN_AUTOLOAD=1 coverage run -m pytest && coverage report

# 真实流量冒烟测试（需有效 state/*.json 与 .env 中的 AI 配置）
./run_live_smoke.sh --without-generation
```

---

## 3. 数据存储与配置约定

1. **SQLite 是唯一在线主存储**
   - 默认路径 `data/app.sqlite3`，可通过环境变量 `APP_DATABASE_FILE` 覆盖（Docker Compose 设为 `/app/data/app.sqlite3`）。
   - 核心表：`app_metadata`、`tasks`、`result_items`、`price_snapshots`、`result_blacklist_rules`（定义见 `src/infrastructure/persistence/sqlite_connection.py`）。
   - `src/infrastructure/persistence/sqlite_bootstrap.py` 会在首次建库且表为空时，自动从旧版 `config.json`、`jsonl/*.jsonl`、`price_history/*_history.jsonl` 执行一次性导入。
   - **注意**：`result_items` 表保留了 `result_filename`（形如 `<keyword>_full_data.jsonl`）作为按关键词分组的逻辑键，且 `src/utils.py:save_to_jsonl` 为兼容旧函数名保留了原名，但底层实际读写均为 SQLite，**不要在新增业务代码中直接扫描或写入磁盘 `jsonl/` 目录或 `config.json`**。
2. **文件系统目录职责**
   - `.env`：运行时环境变量配置；Web UI 修改设置通过 `src/infrastructure/config/env_manager.py` 写回 `.env` 并调用 `reload_settings()`。
   - `state/`：多账号登录态目录（默认 `ACCOUNT_STATE_DIR=state`），每个账号对应 `<name>.json`；根目录 `xianyu_state.json` 为单账号兼容路径。
   - `prompts/`：`base_prompt.txt` 为通用分析模板，`*_criteria.txt` 为各任务分析标准（AI 创建任务时参考 `prompts/macbook_criteria.txt` 异步生成）。
   - `logs/`：`ProcessService` 将子进程输出写入 `logs/<sanitized_task_name>_<task_id>.log`（启动时自动清理超过 `TASK_LOG_RETENTION_DAYS` 天的日志）；`logs/ai/` 存放最近 1 天的 AI 请求摘要；`logs/task-failure-guard.json` 存放失败熔断状态。
   - `images/`：任务运行期临时图片目录 `images/task_images_<task_name>/`，单品分析完成及任务结束时自动清理。

---

## 4. 修改代码时的关键注意事项

- **保持两套配置读取同步**：FastAPI 主进程主要通过 `src/infrastructure/config/settings.py`（Pydantic）与 `env_manager` 读取配置，而爬虫子进程 `spider_v2.py` / `src/scraper.py` / `src/ai_handler.py` 仍部分引用 `src/config.py`。新增或修改环境变量时，请同时核对这两处及 `.env.example`。
- **不要破坏前端构建路径契约**：`web-ui/vite.config.ts` 的 `build.outDir` 指向 `../dist`，`Dockerfile`、`Dockerfile.release`、`web-ui/Dockerfile`、`.dockerignore` 与 `start.sh` 均依赖该路径，`tests/test_frontend_build_paths.py` 会对此做静态断言。
- **AI 接口兼容性封装**：所有新增大模型调用应复用 `src/services/ai_request_compat.py` 与 `src/services/ai_response_parser.py`，保留 Chat Completions API 与 Responses API 自动回退、`json_object` / `temperature` 不支持时自动降级重试、`reasoning_content` 提取及 `NO_PROXY` IPv6 清理逻辑。
- **扩展通知渠道的完整改动点**：新增推送渠道需同步更新 `src/infrastructure/external/notification_clients/`（客户端实现 + `factory.py`）、`src/infrastructure/config/settings.py`（`NotificationSettings`）、`src/services/notification_config_service.py`（字段映射、校验与敏感字段脱敏）、`src/api/routes/settings.py`（请求模型）、`.env.example` 以及前端 `web-ui/src/components/settings/NotificationSettingsPanel.vue` 与 `src/i18n/messages/*`。
- **测试隔离与同步约定**：新增测试放入 `tests/unit/` 或 `tests/integration/`，使用普通同步 `def test_*` 函数配合 `asyncio.run()` 或 `TestClient`，所有文件/数据库操作必须落在 `tmp_path` 内。提交改动前请确保 `PYTEST_DISABLE_PLUGIN_AUTOLOAD=1 pytest` 与 `cd web-ui && npm run build` 通过。
- **安全边界**：在文件读写路由（如 `prompts.py`、`results.py`、`accounts.py`）中必须保留文件名正则或 `Path.resolve().relative_to(...)` 路径穿越校验；切勿提交真实 `.env`、`xianyu_state.json` 或 `state/*.json`。
