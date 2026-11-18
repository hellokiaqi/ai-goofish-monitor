# AI Goofish Monitor Web UI

本目录为 AI Goofish Monitor 的前端单页应用（SPA），提供监控概览、任务调度配置、多账号登录态管理、结果分析、实时日志查看与系统设置界面，支持简体中文（`zh-CN`）与英文（`en-US`）双语切换。

---

## 技术栈

| 类别 | 依赖 / 工具 | 说明 |
| :--- | :--- | :--- |
| 核心框架 | Vue `^3.5` + TypeScript `~5.9` | 使用 `<script setup lang="ts">` 单文件组件 |
| 构建工具 | Vite `^7.2` + `vue-tsc` `^3.1` | 构建前执行严格 TypeScript 类型检查 |
| 路由与国际化 | `vue-router` `^4.6` + `vue-i18n` `^10.0` | HTML5 History 路由；语言偏好持久化于 `localStorage.app_locale` |
| UI 与样式 | Tailwind CSS `^3.4` + `reka-ui` `^2.7` + shadcn-vue | 无样式可访问性原语封装在 `src/components/ui/*`，搭配 `class-variance-authority` / `tailwind-merge` |
| 图标与工具库 | `lucide-vue-next`、`@vueuse/core` | 图标组件与组合式工具函数 |

---

## 页面路由与功能模块

路由定义在 `src/router/index.ts`，除 `/login` 外均通过前端路由守卫检查登录状态（`useAuth.ts`）：

| 路径 | 视图组件 | 核心功能 |
| :--- | :--- | :--- |
| `/login` | `LoginView.vue` | 提交用户名密码到 `POST /auth/status`；登录成功后保存本地会话并连接 `/ws` WebSocket |
| `/dashboard` | `DashboardView.vue` | 展示任务总数/运行中数量、扫描/推荐统计（区分 AI 与关键词推荐）、焦点任务日价格曲线（`PriceTrendChart.vue`）、最近动态与操作建议 |
| `/tasks` | `TasksView.vue` | 任务增删改查、手动启停、开关切换、AI 分析标准异步生成进度弹窗（`TaskGenerationDialog.vue`）、重新生成标准、省/市/区三级区域选择器（内置 `src/data/goofishRegions.json`）、Cron 预设与自定义配置 |
| `/accounts` | `AccountsView.vue` | 管理 `state/*.json` 闲鱼登录态快照（新增、编辑、删除、一键携带该账号跳转新建任务） |
| `/results` | `ResultsView.vue` | 按结果集浏览商品卡片、切换仅看 AI/关键词推荐或显示已隐藏商品、多字段排序、查看市场均价/中位数与趋势图、手动屏蔽单条商品、配置关键词或 `re:` 正则黑名单、导出 CSV、删除结果集 |
| `/logs` | `LogsView.vue` | 按任务实时增量拉取运行日志，支持向上滚动分页加载历史日志、自动刷新、自动滚底与清空日志 |
| `/settings` | `SettingsView.vue` | 包含 5 个标签页：`ai`（模型接口与连接测试）、`rotation`（账号与代理池轮换参数）、`notifications`（7 种通知渠道配置与单渠道测试推送）、`status`（爬虫运行状态与 `.env` 检查）、`prompts`（在线查看与编辑 `prompts/*.txt`） |

---

## 目录结构

```text
web-ui/
├── src/
│   ├── api/                 # 后端 REST API 请求封装（tasks/accounts/results/logs/settings/prompts/dashboard）
│   ├── components/
│   │   ├── layout/          # 侧边栏、顶栏、语言切换、概览页任务搜索
│   │   ├── results/         # 结果卡片、筛选栏、市场分析面板、价格趋势图表
│   │   ├── settings/        # 通知设置面板、账号与代理轮换设置面板
│   │   ├── tasks/           # 任务列表、表单、创建弹窗、AI 生成进度弹窗、三级区域选择器
│   │   └── ui/              # 基于 Reka UI + Tailwind 的基础组件库（shadcn-vue 风格）
│   ├── composables/         # 业务状态与交互逻辑（useTasks/useResults/useLogs/useSettings/useDashboard/useAuth 等）
│   ├── data/                # 内置闲鱼省/市/区三级区域快照（goofishRegions.json）
│   ├── i18n/                # 中英文文案与日期/数字格式化工具
│   ├── layouts/             # 主布局容器 MainLayout.vue
│   ├── lib/                 # HTTP 请求工具、Cron 辅助与样式合并函数
│   ├── router/              # Vue Router 路由配置与登录守卫
│   ├── services/            # WebSocket 自动重连与任务状态实时同步服务
│   ├── types/               # 任务、结果、概览页 TypeScript 类型声明
│   └── views/               # 路由级页面视图
├── Dockerfile               # 独立前端容器镜像构建文件（Node 22 构建 + Nginx 托管）
├── nginx.conf               # 独立部署时的 Nginx SPA 路由与反向代理配置
└── vite.config.ts           # Vite 构建输出路径与本地开发代理配置
```

---

## 本地开发与构建命令

环境要求：Node.js `20+` 与 `npm`。

```bash
cd web-ui

# 安装依赖
npm install

# 启动本地热更新开发服务器（默认 http://localhost:5173）
npm run dev

# 执行 vue-tsc 类型检查并构建生产静态资源
npm run build

# 本地预览生产构建结果
npm run preview
```

### 开发代理说明（`vite.config.ts`）

`npm run dev` 启动时，Vite 开发服务器会将以下请求转发到本地 FastAPI 后端（需先在仓库根目录启动 `python -m src.app` 或 `uvicorn src.app:app --port 8000`）：

- `/api` -> `http://127.0.0.1:8000`
- `/auth` -> `http://127.0.0.1:8000`
- `/ws` -> `ws://127.0.0.1:8000`（WebSocket 代理）

### 构建产物输出路径约定（重要）

`vite.config.ts` 已显式配置：

```ts
build: {
  outDir: path.resolve(__dirname, '../dist'),
  emptyOutDir: true,
}
```

运行 `npm run build` 时，**构建产物会直接输出到仓库根目录 `../dist/`**（而不是 `web-ui/dist/`），无需手动复制目录。

---

## 部署方式

1. **后端一体化托管（默认推荐）**
   - 根仓库 `Dockerfile`、`Dockerfile.release` 与 `start.sh` 均在构建阶段生成根目录 `dist/`，由 FastAPI（`src/app.py`）直接挂载 `/assets` 并通过 Catch-all 路由返回 `dist/index.html` 支持 History 路由。
2. **Nginx 独立容器托管（可选）**
   - 若需前后端分容器部署，可使用 `web-ui/Dockerfile` 构建前端镜像：Stage 1 使用 `node:22-alpine` 构建到容器 `/dist`，Stage 2 使用 `nginx:alpine` 托管静态文件并加载 `web-ui/nginx.conf`，监听 `80` 端口并将 `/api/`、`/auth/`、`/static/`、`/ws` 反向代理至 `http://app:8000`。
   - 注意不要随意改动 `vite.config.ts` 中的 `../dist` 输出路径，仓库根测试 `tests/test_frontend_build_paths.py` 会校验 `vite.config.ts`、`Dockerfile`、`web-ui/Dockerfile`、`.dockerignore` 与 `start.sh` 的路径一致性。
