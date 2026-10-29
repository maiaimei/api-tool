# Web API 调用工具（类 Postman Web版）功能需求文档

> **文档版本**：v1.0  
> **创建日期**：2026-10-29  
> **目标读者**：前端/后端开发工程师  
> **项目目标**：实现一个现代化的本地运行 Web API 调用工具，支持多项目管理、嵌套文件夹、拖拽排序、环境变量、请求链依赖等核心功能。

---

## 一、技术选型

| 层级 | 技术 | 说明 |
|------|------|------|
| 包管理器 | **pnpm** | 快速、节省磁盘空间、严格的依赖管理 |
| 后端 | Node.js + Express + TypeScript | 轻量、跨平台、本地运行 |
| 前端 | Vue 3 + Vite + TypeScript | 现代化、响应式 |
| UI 组件 | Element Plus | 成熟的企业级组件库 |
| 拖拽 | VueDraggableNext（基于 Sortable.js） | 支持嵌套列表拖拽排序 |
| 数据存储 | SQLite（本地文件数据库） | 无需额外安装数据库服务 |
| HTTP 请求 | Axios（前端）+ node-fetch（后端代理） | 前端发请求到后端代理，后端转发实际 HTTP 请求 |

---

## 二、功能清单

| 功能模块 | 功能描述 | 优先级 |
|----------|----------|--------|
| **基础架构** | Web 版 API 调用工具 | P0 |
| | 本地安装 Node.js + pnpm 后直接运行（`pnpm install && pnpm dev`） | P0 |
| **项目管理** | 多项目管理（创建/重命名/删除/切换） | P0 |
| **目录结构** | 嵌套文件夹（无限层级） | P0 |
| | 文件夹/请求拖拽排序（跨文件夹移动） | P0 |
| | 拖拽后实时自动保存 | P0 |
| **请求编辑** | 新建/编辑/复制/删除请求 | P0 |
| | 支持 HTTP Method（GET/POST/PUT/DELETE/PATCH/HEAD/OPTIONS） | P0 |
| | URL 输入（支持变量 `{{变量名}}`） | P0 |
| | Query 参数（Key-Value 表格，支持启用/禁用） | P0 |
| | Headers（Key-Value 表格，支持启用/禁用） | P0 |
| | Body（none/form-data/x-www-form-urlencoded/raw/json） | P0 |
| **cURL** | 导入 cURL 命令自动解析填充表单 | P1 |
| | 导出当前请求为 cURL 命令 | P1 |
| **布局** | 左中右三栏布局 | P0 |
| | 面板拖拽调整宽度（可折叠） | P0 |
| | 各区域独立滚动条 | P0 |
| **环境变量** | 环境变量管理（多环境：开发/测试/生产等） | P0 |
| | 请求 URL/参数/Body 中使用 `{{变量名}}` 自动替换 | P0 |
| | 变量作用域：全局 > 环境 > 集合 | P1 |
| **请求依赖** | 请求链依赖（用其他请求的响应数据作为变量） | P1 |
| | 支持从响应 JSON 中提取字段（如 `$.data.token`） | P1 |
| | 支持前置脚本（Pre-request Script）设置变量 | P2 |

---

## 三、项目结构

```
api-tool/
├── package.json
├── tsconfig.json
├── vite.config.ts
├── server/
│   ├── src/
│   │   ├── index.ts              # 服务入口
│   │   ├── db/
│   │   │   └── database.ts       # SQLite 数据库初始化 & CRUD
│   │   ├── routes/
│   │   │   ├── project.ts        # 项目管理 API
│   │   │   ├── request.ts        # 请求管理 API
│   │   │   ├── env.ts            # 环境变量 API
│   │   │   └── proxy.ts          # HTTP 请求代理执行
│   │   └── types/
│   │       └── index.ts          # TypeScript 类型定义
│   └── data/
│       └── api-tool.db           # SQLite 数据库文件（运行时自动创建）
├── src/
│   ├── main.ts
│   ├── App.vue
│   ├── components/
│   │   ├── LeftPanel.vue         # 左侧 API 树形管理区
│   │   ├── MiddlePanel.vue       # 中间请求编辑区
│   │   ├── RightPanel.vue        # 右侧响应展示区
│   │   ├── RequestForm.vue       # 请求表单（Method/URL/Params/Body）
│   │   ├── EnvManager.vue        # 环境变量管理弹窗/面板
│   │   └── ResizablePanel.vue    # 可拖拽调整宽度的面板容器
│   ├── stores/
│   │   └── apiStore.ts           # Pinia 状态管理
│   └── utils/
│       ├── curl.ts               # cURL 导入/导出解析
│       └── variable.ts           # 变量替换引擎
```

---

## 四、数据库设计（SQLite）

### 4.1 projects 表

| 字段 | 类型 | 说明 |
|------|------|------|
| id | INTEGER PRIMARY KEY AUTOINCREMENT | 项目ID |
| name | TEXT NOT NULL | 项目名称 |
| created_at | DATETIME DEFAULT CURRENT_TIMESTAMP | 创建时间 |
| updated_at | DATETIME DEFAULT CURRENT_TIMESTAMP | 更新时间 |

### 4.2 folders 表

| 字段 | 类型 | 说明 |
|------|------|------|
| id | INTEGER PRIMARY KEY AUTOINCREMENT | 文件夹ID |
| project_id | INTEGER NOT NULL | 所属项目ID |
| parent_id | INTEGER | 父文件夹ID（NULL 表示根级） |
| name | TEXT NOT NULL | 文件夹名称 |
| sort_order | INTEGER DEFAULT 0 | 排序权重 |
| created_at | DATETIME DEFAULT CURRENT_TIMESTAMP | 创建时间 |

### 4.3 requests 表

| 字段 | 类型 | 说明 |
|------|------|------|
| id | INTEGER PRIMARY KEY AUTOINCREMENT | 请求ID |
| project_id | INTEGER NOT NULL | 所属项目ID |
| folder_id | INTEGER | 所属文件夹ID（NULL 表示根级） |
| name | TEXT NOT NULL | 请求名称 |
| method | TEXT DEFAULT 'GET' | HTTP 方法 |
| url | TEXT | 请求URL |
| headers | TEXT | JSON 字符串存储 Headers 数组 |
| params | TEXT | JSON 字符串存储 Query 参数 |
| body_type | TEXT DEFAULT 'none' | Body 类型 |
| body_content | TEXT | Body 内容 |
| sort_order | INTEGER DEFAULT 0 | 排序权重 |
| created_at | DATETIME DEFAULT CURRENT_TIMESTAMP | 创建时间 |
| updated_at | DATETIME DEFAULT CURRENT_TIMESTAMP | 更新时间 |

### 4.4 environments 表

| 字段 | 类型 | 说明 |
|------|------|------|
| id | INTEGER PRIMARY KEY AUTOINCREMENT | 环境ID |
| project_id | INTEGER NOT NULL | 所属项目ID |
| name | TEXT NOT NULL | 环境名称（如：开发、测试、生产） |
| variables | TEXT | JSON 字符串存储变量键值对数组 |
| is_active | INTEGER DEFAULT 0 | 是否为当前激活环境 |

### 4.5 request_dependencies 表

| 字段 | 类型 | 说明 |
|------|------|------|
| id | INTEGER PRIMARY KEY AUTOINCREMENT | 依赖ID |
| project_id | INTEGER NOT NULL | 所属项目ID |
| source_request_id | INTEGER NOT NULL | 源请求ID（被依赖的请求） |
| target_request_id | INTEGER NOT NULL | 目标请求ID（使用依赖数据的请求） |
| extract_path | TEXT | JSONPath 提取表达式（如 `$.data.token`） |
| variable_name | TEXT | 提取后存储的变量名 |

---

## 五、API 接口设计

### 5.1 项目管理

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/projects` | 获取所有项目列表 |
| POST | `/api/projects` | 创建项目 `{ name: string }` |
| PUT | `/api/projects/:id` | 更新项目名称 |
| DELETE | `/api/projects/:id` | 删除项目（级联删除文件夹、请求、环境） |

### 5.2 文件夹管理

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/projects/:projectId/folders` | 获取项目下所有文件夹（树形结构） |
| POST | `/api/folders` | 创建文件夹 `{ project_id, parent_id, name }` |
| PUT | `/api/folders/:id` | 重命名文件夹 |
| PUT | `/api/folders/reorder` | 批量更新排序 `{ items: [{id, parent_id, sort_order}] }` |
| DELETE | `/api/folders/:id` | 删除文件夹（级联移动子项） |

### 5.3 请求管理

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/projects/:projectId/requests` | 获取项目下所有请求（含文件夹归属） |
| GET | `/api/requests/:id` | 获取单个请求详情 |
| POST | `/api/requests` | 创建请求 `{ project_id, folder_id, name, method, url, ... }` |
| PUT | `/api/requests/:id` | 更新请求 |
| PUT | `/api/requests/reorder` | 批量更新排序（同文件夹拖拽） |
| DELETE | `/api/requests/:id` | 删除请求 |
| POST | `/api/requests/:id/duplicate` | 复制请求 |

### 5.4 环境变量

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/projects/:projectId/envs` | 获取项目所有环境 |
| POST | `/api/envs` | 创建环境 `{ project_id, name, variables }` |
| PUT | `/api/envs/:id` | 更新环境（变量列表） |
| PUT | `/api/envs/:id/activate` | 激活指定环境 |
| DELETE | `/api/envs/:id` | 删除环境 |

### 5.5 请求代理执行

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/api/proxy` | 后端代理执行 HTTP 请求 |
| | Body: `{ method, url, headers, body, env_vars }` | |
| | Response: `{ status, statusText, headers, data, duration }` | |

### 5.6 cURL 转换

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/api/curl/import` | 解析 cURL 命令 → 请求对象 |
| POST | `/api/curl/export` | 请求对象 → cURL 命令字符串 |

---

## 六、前端组件详细需求

### 6.1 整体布局（App.vue）

- 采用 **左-中-右三栏布局**，使用 `ResizablePanel` 组件包裹
- 左栏默认宽度 280px，中栏默认 420px，右栏自适应剩余空间
- 面板分隔线支持鼠标拖拽调整宽度
- 各面板内部独立 `overflow-y: auto` 滚动
- 左栏可折叠为图标模式（收起后显示项目切换器）

### 6.2 左栏 — LeftPanel.vue（API 树形管理区）

- 顶部：项目选择器（下拉切换项目）+ 新建请求/文件夹按钮
- 树形结构展示：文件夹（可展开/折叠）+ 请求（叶子节点）
- 使用 `VueDraggableNext` 实现：
  - 文件夹之间拖拽排序
  - 请求在文件夹之间移动
  - 请求在同一文件夹内排序
  - 拖拽结束立即调用 `reorder` API 保存
- 右键菜单：重命名 / 复制 / 删除 / 新建子文件夹
- 点击请求节点 → 中栏加载请求详情

### 6.3 中栏 — MiddlePanel.vue（请求编辑区）

- 请求名称（可编辑）
- HTTP Method 选择器（下拉）
- URL 输入框（支持 `{{变量}}` 高亮提示）
- 发送按钮（Send）+ 保存按钮
- Tabs 切换：
  - **Params**：Query 参数表格（Key/Value/启用开关/删除）
  - **Headers**：Headers 表格（同上）
  - **Body**：类型选择（none / form-data / x-www / raw / JSON）+ 对应编辑区
  - **Pre-request**：前置脚本编辑器（可选，P2）
  - **Tests**：测试脚本（可选，P2）

### 6.4 右栏 — RightPanel.vue（响应展示区）

- 响应状态行：状态码（彩色标签）+ 响应时间 + 响应大小
- Tabs 切换：
  - **Body**：JSON 高亮格式化展示（可折叠）/ 其他格式纯文本
  - **Headers**：响应头表格
  - **Cookies**：Cookie 列表

### 6.5 环境变量管理 — EnvManager.vue

- 弹窗/抽屉形式打开
- 环境列表（左侧）+ 变量表格（右侧）
- 支持添加/编辑/删除环境变量（Key/Value/启用开关）
- 切换当前激活环境
- 在请求编辑区 URL/参数中，输入 `{{` 弹出变量自动补全提示

### 6.6 可拖拽面板 — ResizablePanel.vue

- 接收 `minWidth` / `maxWidth` props
- 鼠标按下分隔线 → 监听 mousemove 动态调整宽度 → mouseup 结束
- 支持双击分隔线重置为默认宽度

---

## 七、核心业务逻辑

### 7.1 变量替换引擎（utils/variable.ts）

```
输入：原始字符串 "https://api.com/{{version}}/users/{{userId}}"
环境：{ version: "v1", userId: "123" }
前置变量：{ token: "abc" }
输出："https://api.com/v1/users/123"
```

- 正则匹配 `{{变量名}}` 模式
- 按优先级查找：前置脚本变量 > 环境变量 > 全局变量
- 未找到变量时保留原始 `{{变量名}}` 或替换为空（可配置）

### 7.2 请求依赖（Request Chain）

1. 用户在请求 A 的 **Tests** 或响应后处理中定义提取规则：
   - 变量名：`auth_token`
   - 提取路径：`$.data.access_token`（JSONPath）
2. 执行请求 A 后，自动提取响应数据存入临时变量池
3. 执行请求 B 时，变量替换引擎自动使用临时变量池中的数据
4. 依赖关系持久化到 `request_dependencies` 表

### 7.3 cURL 解析（utils/curl.ts）

- 导入：解析 cURL 命令字符串 → 提取 method / url / headers / data → 填充到请求表单
- 导出：将当前请求配置序列化为标准 cURL 命令（支持 `-X` / `-H` / `-d` 等参数）

### 7.4 后端代理（routes/proxy.ts）

- 接收前端传来的请求配置
- 执行变量替换（服务端二次确认）
- 使用 `node-fetch` 发送实际 HTTP 请求
- 记录响应时间
- 返回结构化响应给前端
- 处理超时、重定向、HTTPS 证书等边界情况

---

## 八、环境准备与启动指南（新手入门）

> ⚠️ 本项目使用 **pnpm** 作为包管理器，请确保已安装。

### 8.1 环境要求

| 依赖 | 最低版本 | 说明 |
|------|----------|------|
| Node.js | >= 18.x | 建议安装 LTS 版本 |
| pnpm | >= 8.x | 包管理器 |

### 8.2 安装 pnpm（如未安装）

```bash
# 方式一：使用 npm 安装 pnpm（全局）
npm install -g pnpm

# 方式二：使用 Node.js 自带脚本安装
curl -fsSL https://get.pnpm.io/install.sh | sh -

# 方式三：macOS 使用 Homebrew
brew install pnpm

# 验证安装
pnpm --version
```

### 8.3 获取项目

```bash
# 克隆仓库
git clone <项目仓库地址>
cd api-tool
```

### 8.4 安装依赖

```bash
# 在项目根目录执行（会自动安装前端 + 后端所有依赖）
pnpm install
```

> 💡 pnpm 使用内容寻址存储，依赖安装速度比 npm 更快，且不会重复下载相同版本的包。

### 8.5 启动开发环境

```bash
# 同时启动前端（Vite）和后端（Express）开发服务器
pnpm dev
```

启动成功后终端会显示类似输出：
```
  ➜  Frontend: http://localhost:5173
  ➜  Backend:  http://localhost:3000
```

打开浏览器访问：**http://localhost:5173**

### 8.6 常用 pnpm 脚本

```bash
# 启动开发环境（前后端同时）
pnpm dev

# 仅启动前端
pnpm dev:client

# 仅启动后端
pnpm dev:server

# 构建前端生产包
pnpm build

# 预览生产构建
pnpm preview

# 代码检查
pnpm lint

# 代码格式化
pnpm format
```

### 8.7 常见问题

| 问题 | 解决方案 |
|------|----------|
| `pnpm: command not found` | 未安装 pnpm，执行 `npm install -g pnpm` |
| 端口被占用 | 修改 `vite.config.ts` 或 `server/src/index.ts` 中的端口号 |
| SQLite 数据库文件不存在 | 首次启动后端服务时自动在 `server/data/` 下创建 |
| 依赖安装失败 | 尝试 `pnpm store prune` 清理缓存后重新 `pnpm install` |

---

## 九、非功能需求

| 项目 | 要求 |
|------|------|
| 性能 | 拖拽排序响应 < 100ms；请求执行显示 Loading 状态 |
| 数据持久化 | 所有操作实时保存到 SQLite，刷新页面不丢失 |
| 错误处理 | API 错误统一返回 `{ code, message }` 格式；前端 Element Plus Message 提示 |
| 代码规范 | ESLint + Prettier；commit 规范 |
| 跨域 | 后端代理模式，前端不直接请求目标 API（避免 CORS） |
| 本地运行 | `pnpm install && pnpm dev` 一键启动（Vite 前端 + Express 后端） |

---

### 变更记录

| 版本 | 日期 | 变更内容 |
|------|------|----------|
| v1.0 | 2026-10-29 | 初始版本，包管理器由 npm 迁移至 pnpm，新增新手入门启动指南 |