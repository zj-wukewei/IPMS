
# IPMS 后端架构设计文档

| 文档版本 | V1.0 |
| :--- | :--- |
| 创建日期 | 2025年9月5日 |
| 状态 | 已确认 |

---

## 1. 核心架构思想

IPMS后端采用**增强型单体 (Enhanced Monolith)**架构模式，基于**领域驱动设计 (Domain-Driven Design, DDD)**思想进行构建。此架构旨在项目初期保证高开发效率和简便的部署流程，同时通过清晰的模块化设计，为系统未来的平滑演进和微服务化改造奠定坚实基础。

核心原则包括：

*   **关注点分离:** 严格分离业务逻辑、应用流程和技术实现。
*   **依赖倒置:** 核心领域层不依赖任何技术细节，技术细节依赖于领域层定义的抽象。
*   **限界上下文:** 通过限界上下文（Bounded Contexts）作为模块划分的最高准则，确保业务边界的清晰。

## 2. 项目模块结构 (Gradle 多模块)

项目采用Gradle多模块结构，根项目`ipms-backend`作为协调者，所有代码均在子模块中。

```
ipms-backend/
├── build.gradle.kts                # 根构建文件
├── settings.gradle.kts             # 模块定义文件
│
├── ipms-app/                       # 启动模块
│   └── src/main/kotlin/com/ipms/Application.kt
│
├── ipms-domain/                    # 领域层模块 (业务核心)
│   └── src/main/kotlin/com/ipms/domain/
│       ├── project/                # -> Project Context
│       └── identityaccess/         # -> Identity & Access Context
│
├── ipms-application/               # 应用层模块 (业务流程编排)
│   └── src/main/kotlin/com/ipms/application/
│       ├── project/                # -> Project Context
│       ├── identityaccess/         # -> Identity & Access Context
│       └── aiorchestration/        # -> AI Orchestration Context
│
├── ipms-infrastructure/            # 基础设施层模块 (技术实现)
│   └── src/main/kotlin/com/ipms/infrastructure/
│       ├── persistence/
│       ├── aiclient/
│       ├── messaging/
│       └── security/
│
└── ipms-shared/                    # 共享模块 (通用工具与配置)
    └── src/main/kotlin/com/ipms/shared/
        ├── common/
        ├── config/
        ├── exception/
        └── web/
```

## 3. 模块职责与依赖关系

### `ipms-domain` (领域层)
*   **职责:** **项目的业务核心**。包含所有聚合根、实体、值对象、领域服务和仓储接口。此模块是纯粹的业务逻辑，不包含任何外部框架（如Spring, JPA）的依赖。
*   **依赖:** `ipms-shared`

### `ipms-application` (应用层)
*   **职责:** 编排业务用例（Use Cases），协调领域对象和基础设施服务来完成一个完整的业务流程。它定义了系统的应用服务接口。
*   **依赖:** `ipms-domain`, `ipms-shared`

### `ipms-infrastructure` (基础设施层)
*   **职责:** 提供所有技术细节的具体实现。包括JPA仓储实现、Redis消息发布器、Dify API客户端、Sa-Token安全配置等。它实现了`ipms-domain`和`ipms-application`中定义的接口。
*   **依赖:** `ipms-application`, `ipms-domain`, `ipms-shared`

### `ipms-shared` (共享模块)
*   **职责:** 存放不属于任何特定业务领域，但被多个模块共同需要的通用代码，如统一响应体、全局异常处理器、通用工具类、自定义异常基类等。
*   **依赖:** (无) - 它是最基础的模块。

### `ipms-app` (启动模块)
*   **职责:** 唯一的职责是打包和启动整个Spring Boot应用，将所有模块组合在一起运行。
*   **依赖:** `ipms-infrastructure`, `ipms-application`, `ipms-domain`, `ipms-shared`

## 4. 限界上下文 (Bounded Contexts) 代码归属

*   **`ProjectContext` (项目上下文):** 负责项目管理的核心业务。
    *   **领域模型:** `ipms-domain/src/.../project/`
    *   **应用服务:** `ipms-application/src/.../project/`
    *   **技术实现:** `ipms-infrastructure/src/.../persistence/project/`

*   **`IdentityAccessContext` (身份与访问上下文):** 负责用户、权限、安全。
    *   **领域模型:** `ipms-domain/src/.../identityaccess/`
    *   **应用服务:** `ipms-application/src/.../identityaccess/`
    *   **技术实现:**
        *   数据库: `ipms-infrastructure/src/.../persistence/identityaccess/`
        *   安全框架: `ipms-infrastructure/src/.../security/satoken/`

*   **`AIOrchestrationContext` (AI编排上下文):** 负责与外部AI服务的所有技术交互。
    *   **领域模型:** (无)
    *   **应用服务:** `ipms-application/src/.../aiorchestration/`
    *   **技术实现:**
        *   Dify客户端: `ipms-infrastructure/src/.../aiclient/dify/`
        *   消息队列: `ipms-infrastructure/src/.../messaging/redis/`
