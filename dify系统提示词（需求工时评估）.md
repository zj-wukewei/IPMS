
# Dify 智能体设计文档：AI工时评估引擎

## 1. 核心设计思想

本智能体的核心目标是为一个产品需求（Requirement）提供一个精准的、按职位细分的“基准工时”评估。

我们采用一个先进的 **“两阶段协作（Two-Stage Collaboration）”** 设计模式，该模式极大地提升了评估的准确性和可解释性：

1.  **第一阶段 (任务分解):** 在调用本引擎之前，系统首先调用 **“需求-任务自动分解引擎”**，将一个抽象的需求转化为一个具体的、结构化的任务清单。
2.  **第二阶段 (汇总评估):** 本引擎接收原始需求信息和第一阶段生成的**任务清单**作为输入。它通过对每一个具体任务进行“微观”评估，然后按职位（JobTitle）进行汇总，最终得出一个“宏观”的、自下而上的精准工时评估报告。

这种设计将复杂的“需求级”模糊评估，拆解成了简单的“任务级”清晰评估的加总，结果更可靠，过程更透明。

## 2. 交互流程

1.  **用户提交需求:** 用户在系统中创建一个新的需求。
2.  **系统后台调用引擎一:** 系统自动调用“需求-任务自动分解引擎”，获得该需求的`tasks`数组。
3.  **系统后台调用引擎二 (本引擎):** 系统立即将原始需求信息和刚刚生成的`tasks`数组，一同作为输入，调用本“AI工时评估引擎”。
4.  **返回最终结果:** 本引擎输出的结构化JSON工时报告被系统接收，并存储为该需求的“预估工时”属性，最终呈现给用户。

对用户而言，整个过程是无缝且自动的。

## 3. Dify 智能体配置（交互契约）

### 3.1. 输入 (Input)

建议使用JSON作为输入格式，并定义以下变量：

*   `project_context` (JSON)
*   `versions` (JSON)
*   `requirement` (JSON)
*   `tasks` (JSON Array) - **关键输入**
*   `historical_data` (JSON Array)

**输入示例:**
```json
{
  "project_context": {
    "name": "Intelligent Project Management System (IPMS)",
    "description": "一个Web应用，使用AI来自动化软件开发团队的任务创建、工时估算和风险预测。",
    "tech_stack": ["React", "Node.js", "PostgreSQL", "RESTful API"]
  },
  "versions": [
    { "version_number": "v1.0", "description": "核心功能：项目、需求和任务管理。" },
    { "version_number": "v1.1", "description": "引入AI能力：工时评估引擎。" }
  ],
  "requirement": {
    "id": "REQ-124",
    "type": "FEATURE",
    "title": "实现基于角色的权限控制（RBAC）",
    "description": "为了系统安全，需要引入权限管理。项目经理可以修改项目内的一切信息，而普通成员只能查看和修改自己被分配到的任务。"
  },
  "tasks": [
    { "title": "架构设计：权限系统与现有用户模块集成方案", "type": "DESIGN" },
    { "title": "后端开发：数据库模型扩展", "type": "DEVELOPMENT" },
    { "title": "后端开发：权限校验中间件", "type": "DEVELOPMENT" },
    { "title": "后端开发：核心业务逻辑改造", "type": "DEVELOPMENT" },
    { "title": "前端开发：根据用户角色动态显示UI元素", "type": "DEVELOPMENT" },
    { "title": "集成测试：跨角色权限场景验证", "type": "TESTING" }
  ],
  "historical_data": [
    {
      "requirement_title": "实现用户邮箱密码登录功能",
      "actual_hours": {"total": 80, "breakdown": [{"job_title": "BACKEND", "hours": 50}, {"job_title": "FRONTEND", "hours": 30}]}
    }
  ]
}
```

### 3.2. 系统提示词 (System Prompt)

```
# 角色
你是一位顶级的软件工程估算专家（Estimation Expert），你像一名外科医生一样，通过分析详细的任务清单来精准计算项目工作量。

# 上下文
你正在为IPMS系统工作，核心任务是为一个新的“需求”评估其所需的“基准工时”。
- “基准工时”定义为：一个“中级工程师”（经验系数1.0）完成此工作所需的小时数。
- 你的评估结果必须是一个按不同职位（JobTitle）细分的工时组合。
- **最关键的输入**是`tasks`数组，这是已经由另一个AI为你分解好的任务清单。你的主要工作就是逐一评估这些任务。

# 指令
1.  **建立总体认知**: 快速阅读`project_context`, `versions`, 和 `requirement`，理解这个需求的总体目标和背景。
2.  **逐一评估任务 (Core Logic)**: 遍历`tasks`数组中的每一个任务。对于每个任务：
    a.  **判断归属**: 根据任务的`title`和`type`，确定它主要属于哪个职位（`BACKEND`, `FRONTEND`, `TESTING`, `DESIGN`等）的工作。
    b.  **估算工时**: 结合需求描述和你的技术经验，为这个**单个任务**估算一个“基准工时”。
    c.  **参考历史数据**: 在估算时，可以参考`historical_data`中相似功能的工时数据来进行校准。
3.  **汇总工时**: 完成所有任务的评估后，按职位（JobTitle）将工时进行分组求和。
4.  **生成报告**: 计算总工时，并生成最终的评估报告。在`reasoning`字段中，必须清晰地列出主要任务及其估算工时，以展示你的计算过程。
5.  **输出结果**: 严格按照指定的JSON格式输出。

# 输出格式
你的回答必须且只能是一个JSON对象，绝对不要包含任何JSON格式之外的解释性文字。
{
  "total_base_hours": <number>,
  "breakdown": [
    {
      "job_title": "<string, e.g., 'BACKEND', 'FRONTEND'>",
      "base_hours": <number>
    }
  ],
  "reasoning": "<string, 必须包含主要任务的工时分解说明>",
  "confidence_score": <number, 0.0-1.0之间>
}
```

### 3.3. 输出 (Output)

智能体的输出应配置为解析JSON。

**输出示例:**
```json
{
  "total_base_hours": 112,
  "breakdown": [
    {
      "job_title": "DESIGN",
      "base_hours": 8
    },
    {
      "job_title": "BACKEND",
      "base_hours": 64
    },
    {
      "job_title": "FRONTEND",
      "base_hours": 24
    },
    {
      "job_title": "TESTING",
      "base_hours": 16
    }
  ],
  "reasoning": "工时评估基于以下任务分解：'架构设计' (DESIGN: 8小时)。后端工作量主要由三部分构成：'数据库模型扩展' (16小时), '权限校验中间件' (32小时), '核心业务逻辑改造' (16小时)，合计64小时。前端工作量为'动态UI元素' (24小时)。'集成测试' (TESTING: 16小时)。总计112基准小时。评估参考了历史'登录功能'的复杂度。",
  "confidence_score": 0.90
}
```
