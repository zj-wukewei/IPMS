
# IPMS 数据库模式设计文档

| 文档版本 | V3.1 |
| :--- | :--- |
| 创建日期 | 2025年9月5日 |
| 更新日期 | 2025年9月5日 |
| 状态 | 修正版 (恢复所有章节的完整内容) |

---

## 1. 核心设计原则与选型

*   **数据库类型:** 关系型数据库 (e.g., PostgreSQL, MySQL)。
*   **数据一致性:** 数据的引用完整性**由应用层负责**，数据库层面**不设外键约束**。本文档中的枚举值注释是应用层实现的**推荐约定**。
*   **软删除:** 所有核心实体表均包含 `is_deleted` 字段，所有查询默认应过滤 `is_deleted = false` 的记录。
*   **主键:** 所有主键统一使用 `bigint` 类型。

---

## 2. 核心实体表 (Core Entity Tables)

### 2.1. 用户与权限

#### `users`
*   **用途:** 存储核心认证信息，保障系统安全。

| 列名 | 数据类型 | 约束 | 描述 |
| :--- | :--- | :--- | :--- |
| `id` | bigint | Primary Key | 用户的唯一标识符 |
| `email` | VARCHAR(255) | NOT NULL, UNIQUE | 用户邮箱，用于登录 |
| `password_hash` | VARCHAR(255) | NOT NULL | 加密后的密码哈希 |
| `last_login_at` | TIMESTAMPTZ | NULL | 最后登录时间 |
| `last_failure_at` | TIMESTAMPTZ | NULL | 最后一次登录失败时间 |
| `login_failure_count` | INT | NOT NULL, DEFAULT 0 | 连续登录失败次数 |
| `locked_until` | TIMESTAMPTZ | NULL | 账户被锁定的截止时间 |
| `is_deleted` | BOOLEAN | NOT NULL, DEFAULT FALSE | 是否已删除 |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | 创建时间 |
| `updated_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | 最后更新时间 |

#### `user_profiles`
*   **用途:** 存储用户的个人及专业档案信息。

| 列名 | 数据类型 | 约束 | 描述 |
| :--- | :--- | :--- | :--- |
| `user_id` | bigint | Primary Key | 用户ID (关联 `users.id`) |
| `full_name` | VARCHAR(100) | NOT NULL | 真实姓名 |
| `job_title` | VARCHAR(50) | NOT NULL | 职位。枚举: `PRODUCT`, `FRONTEND`, `BACKEND`, `TESTING`, `UI`, `PROJECT` |
| `job_level` | INT | NOT NULL | 职级。枚举: `1`:实习生, `2`:初级, `3`:中级, `4`:高级, `5`:专家, `6`:总监 |
| `department` | VARCHAR(100) | NOT NULL | 部门 |
| `team` | VARCHAR(100) | NULL | 团队 |
| `hire_date` | DATE | NOT NULL | 入职日期 |
| `phone_number` | VARCHAR(20) | NULL | 电话号码 |
| `avatar` | VARCHAR(500) | NULL | 头像URL |
| `biography` | TEXT | NULL | 个人简介 |
| `is_active` | BOOLEAN | NOT NULL, DEFAULT TRUE | 账户是否激活 |
| `time_zone` | VARCHAR(50) | NOT NULL, DEFAULT 'UTC' | 时区 |
| `work_location` | VARCHAR(200) | NULL | 办公地点 |
| `working_hours_per_day` | DECIMAL(4, 2) | NOT NULL, DEFAULT 8.00 | 每日标准工作时长 |
| `is_deleted` | BOOLEAN | NOT NULL, DEFAULT FALSE | 是否已删除 |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | |

#### `user_skills`
*   **用途:** 建立详细、可查询的用户技能库。

| 列名 | 数据类型 | 约束 | 描述 |
| :--- | :--- | :--- | :--- |
| `id` | bigint | Primary Key | 技能条目的唯一ID |
| `user_id` | bigint | NOT NULL | 用户ID (关联 `users.id`) |
| `skill_name` | VARCHAR(100) | NOT NULL | 技能名称 (e.g., "React", "SQL") |
| `level` | VARCHAR(20) | NOT NULL | 技能等级 (e.g., "Beginner", "Expert") |
| `years_of_experience` | DECIMAL(4, 1) | NOT NULL | 使用年限 |
| `last_used_date` | DATE | NULL | 最后使用日期 |
| `certifications` | JSONB | NULL | 认证信息 (JSON Array) |
| `description` | TEXT | NULL | 描述 |
| `is_deleted` | BOOLEAN | NOT NULL, DEFAULT FALSE | 是否已删除 |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | |

### 2.2. 项目管理

#### `projects`
*   **用途:** 存储项目信息。

| 列名 | 数据类型 | 约束 | 描述 |
| :--- | :--- | :--- | :--- |
| `id` | bigint | Primary Key | 项目的唯一标识符 |
| `name` | VARCHAR(255) | NOT NULL | 项目名称 |
| `description` | TEXT | | 项目的详细描述 |
| `project_manager_id` | bigint | NOT NULL | 项目经理ID (关联 `users.id`) |
| `status` | VARCHAR(50) | NOT NULL | 项目状态。枚举: `PLANNING`, `IN_PROGRESS`, `PAUSED`, `COMPLETED`, `CANCELLED` |
| `start_date` | DATE | | 项目计划开始日期 |
| `end_date` | DATE | | 项目计划结束日期 |
| `is_deleted` | BOOLEAN | NOT NULL, DEFAULT FALSE | 是否已删除 |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | |

#### `versions`
*   **用途:** 作为需求的集合，定义了一个交付单元。具体的交付计划由`schedules`表管理。

| 列名 | 数据类型 | 约束 | 描述 |
| :--- | :--- | :--- | :--- |
| `id` | bigint | Primary Key | 版本的唯一标识符 |
| `project_id` | bigint | NOT NULL | 所属项目ID (关联 `projects.id`) |
| `version_number` | VARCHAR(50) | NOT NULL | 版本号 (e.g., v1.4.1) |
| `description` | TEXT | | 版本的核心价值主张或功能摘要 |
| `version_type` | VARCHAR(50) | NOT NULL | 版本类型。枚举: `MAJOR`, `MINOR`, `PATCH`, `HOTFIX` |
| `status` | VARCHAR(50) | NOT NULL | 版本状态。枚举: `PLANNING`, `DEVELOPING`, `TESTING`, `RELEASING`, `RELEASED`, `ARCHIVED` |
| `is_deleted` | BOOLEAN | NOT NULL, DEFAULT FALSE | 是否已删除 |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | |

#### `schedules`
*   **用途:** 存储一个版本的具体排期计划。一个版本可对应多个排期方案。

| 列名 | 数据类型 | 约束 | 描述 |
| :--- | :--- | :--- | :--- |
| `id` | bigint | Primary Key | 排期方案的唯一标识符 |
| `version_id` | bigint | NOT NULL | 关联的版本ID (关联 `versions.id`) |
| `schedule_mode` | VARCHAR(50) | NOT NULL | 排期模式。枚举: `RESOURCE_BASED`, `DEADLINE_BASED` |
| `planned_start_date` | DATE | NOT NULL | 计划开始日期 |
| `planned_end_date` | DATE | NOT NULL | 计划结束日期 |
| `actual_start_date` | DATE | NULL | 实际开始日期 |
| `actual_end_date` | DATE | NULL | 实际结束日期 |
| `status` | VARCHAR(50) | NOT NULL, DEFAULT 'DRAFT' | 状态。枚举: `DRAFT`, `ACTIVE`, `COMPLETED`, `CANCELLED` |
| `risk_level` | VARCHAR(50) | NOT NULL, DEFAULT 'LOW' | 风险等级。枚举: `LOW`, `MEDIUM`, `HIGH`, `CRITICAL` |
| `risk_notes` | TEXT | NULL | 风险说明 |
| `ai_analysis_result` | JSONB | NULL | AI分析的原始结果或摘要 |
| `is_deleted` | BOOLEAN | NOT NULL, DEFAULT FALSE | 是否已删除 |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | |

#### `milestones`
*   **用途:** 存储与某次排期关联的关键里程碑节点。

| 列名 | 数据类型 | 约束 | 描述 |
| :--- | :--- | :--- | :--- |
| `id` | bigint | Primary Key | 里程碑的唯一标识符 |
| `schedule_id` | bigint | NOT NULL | 关联的排期ID (关联 `schedules.id`) |
| `type` | VARCHAR(100) | NOT NULL | 里程碑类型。枚举: `原型完成时间`, `UI完成时间`, `开发完成时间`, `联调时间`, `测试时间`, `发布时间` |
| `planned_date` | DATE | NOT NULL | 计划日期 |
| `actual_date` | DATE | NULL | 实际日期 |
| `description` | TEXT | NULL | 描述 |
| `status` | VARCHAR(50) | NOT NULL, DEFAULT 'PENDING' | 状态。枚举: `PENDING`, `IN_PROGRESS`, `COMPLETED`, `AT_RISK` |
| `completion_percentage` | INT | NOT NULL, DEFAULT 0 | 完成百分比(0-100) |
| `assignee_id` | bigint | NULL | 负责人ID (关联 `users.id`) |
| `notes` | TEXT | NULL | 备注 |
| `is_critical_path` | BOOLEAN | NOT NULL, DEFAULT FALSE | 是否在关键路径上 |
| `risk_level` | VARCHAR(50) | NOT NULL, DEFAULT 'LOW' | 风险等级。枚举: `LOW`, `MEDIUM`, `HIGH` |
| `is_deleted` | BOOLEAN | NOT NULL, DEFAULT FALSE | 是否已删除 |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | |

#### `requirements`
*   **用途:** 存储需求信息。

| 列名 | 数据类型 | 约束 | 描述 |
| :--- | :--- | :--- | :--- |
| `id` | bigint | Primary Key | 需求的唯一标识符 |
| `version_id` | bigint | NOT NULL | 所属版本ID (关联 `versions.id`) |
| `title` | VARCHAR(255) | NOT NULL | 需求标题 |
| `description` | TEXT | | 需求的详细描述 |
| `priority` | INT | NOT NULL, DEFAULT 0 | 需求的优先级 (数值越大越高) |
| `type` | VARCHAR(50) | NOT NULL | 需求类型。枚举: `FEATURE`, `ENHANCEMENT`, `BUG_FIX`, `TECHNICAL`, `INTEGRATION` |
| `estimated_hours` | JSONB | | AI评估的工时，格式: `{"total": 110, "breakdown": [...]}` |
| `is_deleted` | BOOLEAN | NOT NULL, DEFAULT FALSE | 是否已删除 |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | |

#### `tasks`
*   **用途:** 存储任务信息。

| 列名 | 数据类型 | 约束 | 描述 |
| :--- | :--- | :--- | :--- |
| `id` | bigint | Primary Key | 任务的唯一标识符 |
| `requirement_id` | bigint | NOT NULL | 所属需求ID (关联 `requirements.id`) |
| `assignee_id` | bigint | NULL | 任务的执行人ID (关联 `users.id`) |
| `title` | VARCHAR(255) | NOT NULL | 任务标题 |
| `type` | VARCHAR(50) | NOT NULL | 任务类型。枚举: `DEVELOPMENT`, `TESTING`, `DESIGN`, `ANALYSIS`, `DOCUMENTATION`, `BUG_FIX`, `CODE_REVIEW`, `DEPLOYMENT`, `RESEARCH`, `MEETING` |
| `status` | VARCHAR(50) | NOT NULL | 任务状态。枚举: `TODO`, `IN_PROGRESS`, `IN_REVIEW`, `DONE`, `CLOSED` |
| `start_date` | DATE | | 计划开始日期 |
| `due_date` | DATE | | 计划完成日期 |
| `actual_hours_spent` | DECIMAL(10, 2) | NULL | 实际花费的工时 |
| `is_deleted` | BOOLEAN | NOT NULL, DEFAULT FALSE | 是否已删除 |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | |

---

## 3. 关联与连接表 (Join & Association Tables)

#### `project_members`
*   **用途:** 连接 `users` 和 `projects` (多对多关系)。

| 列名 | 数据类型 | 约束 | 描述 |
| :--- | :--- | :--- | :--- |
| `user_id` | bigint | PK | 成员ID (关联 `users.id`) |
| `project_id` | bigint | PK | 项目ID (关联 `projects.id`) |
| `role` | VARCHAR(50) | NOT NULL | 成员在项目中的角色。枚举: `PROJECT_MANAGER`, `TECH_LEAD`, `DEVELOPER`, `TESTER`, `DESIGNER`, `MEMBER` |
| `daily_available_hours` | DECIMAL(4, 2) | NOT NULL | 该成员承诺每天投入本项目的工时 |

#### `requirement_dependencies`
*   **用途:** 需求的自我连接，定义依赖关系 (多对多)。

| 列名 | 数据类型 | 约束 | 描述 |
| :--- | :--- | :--- | :--- |
| `requirement_id` | bigint | PK | 当前需求ID (关联 `requirements.id`) |
| `depends_on_id` | bigint | PK | 前置需求ID (关联 `requirements.id`) |

---

## 4. 待议题 (Open Questions)

1.  **AI历史数据:** 如何高效地存储和检索用于AI输入的`historical_data`？是实时查询聚合，还是建立专门的AI特征表？
