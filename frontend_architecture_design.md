
# IPMS 前端架构设计文档

| 文档版本 | V1.0 |
| :--- | :--- |
| 创建日期 | 2025年9月5日 |
| 状态 | 已确认 |

---

## 1. 核心设计原则

本前端架构旨在充分利用我们选定的现代化技术栈（Next.js, React, TailwindCSS, shadcn/ui, Zustand, TanStack Query）的优势，构建一个高性能、可维护、开发者体验优秀的前端应用。

*   **约定优于配置 (Convention over Configuration):** 严格遵循Next.js App Router的路由和文件结构约定。
*   **组件化 (Component-Driven):** 将UI拆分为独立的、可复用的组件。通过`shadcn/ui`提供的模式，我们获得了对组件样式的完全控制权。
*   **关注点分离 (Separation of Concerns):** 将UI渲染、业务逻辑、状态管理、API调用等清晰地分离到不同的模块和目录中。
*   **服务端状态与客户端状态分离:** 明确区分并使用最合适的工具来管理两种不同类型的状态。**TanStack Query** 负责管理来自后端API的服务端状态（缓存、同步、失效），而 **Zustand** 负责管理纯粹的、全局性的客户端状态（如UI主题、用户认证状态等）。

## 2. 项目模块结构

我们将采用业界推荐的 `src` 目录结构，以更好地组织源代码。

```
ipms-frontend/
├── .next/                      # Next.js 编译产物
├── public/                     # 静态资源 (图片, 字体等)
│
├── src/
│   ├── app/                    # --- Next.js App Router 核心 ---
│   │   ├── (auth)/             # 认证相关页面路由组 (e.g., 登录, 注册)
│   │   │   ├── login/
│   │   │   │   └── page.tsx
│   │   │   └── layout.tsx
│   │   │
│   │   ├── (main)/             # 登录后主应用路由组
│   │   │   ├── projects/
│   │   │   │   ├── [projectId]/
│   │   │   │   │   ├── versions/
│   │   │   │   │   │   └── [versionId]/
│   │   │   │   │   │       └── page.tsx      # 版本详情页
│   │   │   │   │   └── page.tsx          # 项目详情页
│   │   │   │   └── page.tsx              # 项目列表页
│   │   │   └── layout.tsx              # 主应用布局 (包含导航栏、侧边栏)
│   │   │
│   │   ├── api/                  # Next.js API 路由 (用于简单的BFF或webhook)
│   │   │   └── ...
│   │   │
│   │   ├── globals.css           # 全局样式与TailwindCSS指令
│   │   ├── layout.tsx            # 根布局
│   │   └── page.tsx              # 首页 (Landing Page)
│   │
│   ├── components/               # --- React 组件 ---
│   │   ├── ui/                   # shadcn/ui 组件 (由CLI自动生成和管理)
│   │   │   ├── button.tsx
│   │   │   └── card.tsx
│   │   │
│   │   ├── common/               # 通用的、跨业务的组件 (e.g., PageHeader, DataGrid)
│   │   ├── features/             # 特定业务功能的组合组件 (e.g., CreateRequirementForm, ProjectKanbanBoard)
│   │   └── layout/               # 布局相关组件 (e.g., Navbar, Sidebar, Footer)
│   │
│   ├── hooks/                    # --- 自定义 Hooks ---
│   │   ├── queries/              # 封装 TanStack Query 的自定义Hooks (e.g., useProjects.ts, useUser.ts)
│   │   └── use-local-storage.ts  # 其他通用自定义Hooks
│   │
│   ├── lib/                      # --- 库与配置 ---
│   │   ├── axios.ts              # Axios 实例的全局配置 (拦截器等)
│   │   ├── query-client.ts       # TanStack Query 客户端实例配置
│   │   └── utils.ts              # 通用工具函数 (shadcn/ui 默认创建)
│   │
│   ├── services/                 # --- API 服务层 ---
│   │   ├── authService.ts        # 封装所有认证相关的API调用
│   │   ├── projectService.ts     # 封装所有项目相关的API调用
│   │   └── types.ts              # TypeScript 类型定义 (API请求和响应)
│   │
│   └── stores/                   # --- Zustand 状态管理 ---
│       ├── authStore.ts          # 管理用户认证状态
│       └── uiStore.ts            # 管理全局UI状态 (e.g., 侧边栏是否折叠)
│
├── .eslintrc.json
├── next.config.mjs
├── package.json
├── postcss.config.js
├── tailwind.config.ts
└── tsconfig.json
```

## 3. 核心目录职责详解

*   **`src/app`:**
    *   **职责:** 定义应用的路由结构、页面和布局。使用**路由组 `(group)`** 来组织不同部分的页面（如认证页面和主应用页面），使其可以拥有不同的布局而**不影响URL结构**。
    *   **数据获取:** 在页面组件（`page.tsx`）中，通过调用`hooks/queries`中的自定义Hooks来获取和管理服务端数据。

*   **`src/components`:**
    *   **职责:** 存放所有UI组件。
    *   `ui/`: `shadcn/ui` 的“半成品”组件，可直接修改。
    *   `common/`: 基于`ui/`组件构建的、具有一定业务含义但可在多处复用的组件。
    *   `features/`: 复杂的、与特定业务功能强相关的组件，通常会在这里组合`common/`组件并处理复杂的内部状态。

*   **`src/services`:**
    *   **职责:** **API调用的唯一出口**。封装所有与后端API的HTTP通信。每个`service`文件对应后端的一个业务领域（如`ProjectContext`）。这些函数内部使用在`lib/axios.ts`中配置好的Axios实例。

*   **`src/hooks/queries`:**
    *   **职责:** **TanStack Query 的核心封装层**。为每个API请求创建一个对应的自定义Hook（如`useGetProjects`）。组件中**永远不应该直接调用`services`**，而应该调用这里的Hook。
    *   **好处:** 这样做将数据获取、缓存、加载状态、错误处理等逻辑与UI组件完全解耦，极大提升了代码的可维护性和健壮性。

*   **`src/stores`:**
    *   **职责:** **Zustand 的全局客户端状态管理**。只存放那些与UI交互强相关、需要跨组件共享、但又**不属于服务端持久化**的状态。

## 4. 核心数据流示例：展示项目列表

1.  **页面加载:** 用户访问 `/projects`。Next.js 渲染 `src/app/(main)/projects/page.tsx` 组件。
2.  **调用Hook:** 在 `page.tsx` 组件内部，调用 `useGetProjects()` Hook (来自 `src/hooks/queries/useProjects.ts`)。
3.  **执行查询:** `useGetProjects` Hook 内部使用 TanStack Query 的 `useQuery`。它的 `queryFn` 会调用 `projectService.getProjects()` (来自 `src/services/projectService.ts`)。
4.  **发起API请求:** `projectService.getProjects()` 使用配置好的 Axios 实例，向后端 `GET /api/v1/projects` 发起HTTP请求。
5.  **数据返回与缓存:** TanStack Query 接收到数据后，会将其缓存起来，并管理好加载状态（`isLoading`）、错误状态（`isError`）和最终数据（`data`）。
6.  **UI渲染:** `page.tsx` 组件从 `useGetProjects` Hook 中获取到 `isLoading`, `data` 等状态，并据此渲染UI（显示加载动画或项目列表）。
7.  **自动刷新:** 当用户切换浏览器标签页再切回来时，TanStack Query 会在后台自动重新获取数据，确保用户看到的是最新信息。
