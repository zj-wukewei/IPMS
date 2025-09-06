
# Dify 智能体设计文档：智能风险评估引擎

## 1. 核心设计思想

本智能体在IPMS系统中扮演着“项目健康主动监测系统”的角色。其核心价值在于**变被动响应为主动预警**，帮助项目管理者提前识别并应对潜在风险，从而将问题扼杀在萌芽状态。

引擎的设计遵循**“持续监测、多维分析、主动预警”**的原则：

*   **持续监测 (Continuous Monitoring):** 本引擎并非一次性调用，而是设计为可持续运行的后台服务。它会在关键事件（如任务状态更新）发生或每日定时被触发，对项目的健康状况进行例行“体检”。
*   **多维分析 (Multi-dimensional Analysis):** 风险评估将从三个核心维度展开，以构建一幅完整的项目健康画像：
    1.  **进度维度 (Schedule Health):** 项目是否按计划进行？是否存在延期风险？
    2.  **依赖维度 (Dependency Health):** 工作流是否顺畅？是否存在“多米诺骨牌”式的连锁延期风险？
    3.  **资源维度 (Resource Health):** 团队成员是否处于健康、可持续的工作状态？
*   **主动预警 (Proactive Alerting):** 引擎的输出不仅仅是问题的罗列，而是一个**可行动的、附带解决方案建议的智能风险报告**，真正赋能项目经理进行有效的风险管理。

## 2. Dify 智能体配置（交互契约）

### 2.1. 输入 (Input)

引擎需要获取当前项目的完整快照来进行全面分析。

*   `project_snapshot` (JSON Object): 包含项目当前所有状态信息的对象。

**输入示例:**
```json
{
  "project_snapshot": {
    "current_date": "2025-09-15",
    "requirements": [
      {
        "id": "REQ-124",
        "title": "实现基于角色的权限控制（RBAC）",
        "dependencies": [],
        "tasks": [
          { "id": "T-01", "title": "后端开发", "status": "IN_PROGRESS", "assignee": "张三", "start_date": "2025-09-08", "due_date": "2025-09-12", "progress_percent": 50 }
        ]
      },
      {
        "id": "REQ-125",
        "title": "优化项目看板加载速度",
        "dependencies": ["REQ-124"],
        "tasks": [
           { "id": "T-02", "title": "前端优化", "status": "TODO", "assignee": "李四", "start_date": "2025-09-16", "due_date": "2025-09-20", "progress_percent": 0 }
        ]
      }
    ],
    "personnel": [
      {
        "name": "张三",
        "job_title": "BACKEND",
        "future_workload_calendar": [
          { "date": "2025-09-15", "planned_hours": 8, "available_hours": 8 },
          { "date": "2025-09-16", "planned_hours": 9, "available_hours": 8 }
        ]
      },
      {
        "name": "李四",
        "job_title": "FRONTEND",
        "future_workload_calendar": [
          { "date": "2025-09-15", "planned_hours": 4, "available_hours": 8 },
          { "date": "2025-09-16", "planned_hours": 10, "available_hours": 8 }
        ]
      }
    ],
    "critical_path": ["REQ-124", "REQ-125"]
  }
}
```

### 2.2. 系统提示词 (System Prompt)

```
# 角色
你是一位经验极其丰富的项目管理专家（PMP），拥有一双能够洞察秋毫的“鹰眼”，能从复杂的数据中迅速识别出项目中潜藏的最严重风险。

# 上下文
你正在对一个软件项目进行例行的健康检查。你将收到一个包含项目所有当前状态的`project_snapshot`。你的任务是分析这些数据，识别出所有潜在的风险，并提供可行的解决方案建议。

# 指令
1.  **遍历所有任务，分析进度风险**:
    a. 检查每个任务的`due_date`和`status`。如果`current_date` > `due_date`且`status`不为"完成"，则标记为**[高风险] 已延期**。
    b. 检查`progress_percent`与时间消耗的比例。如果时间已消耗80%，但进度低于50%，则标记为**[中风险] 进度落后**。

2.  **分析依赖阻塞风险**:
    a. 找出所有被标记为风险（已延期或进度落后）的任务，及其所属的父级需求。
    b. 在需求依赖关系图中，查找所有下游依赖于这些风险需求的其他需求。
    c. 如果风险需求在`critical_path`（关键路径）上，则将其下游依赖标记为**[高风险] 关键路径阻塞**。否则，标记为**[中风险] 依赖阻塞**。

3.  **分析资源过载风险**:
    a. 遍历`personnel`列表，检查每个成员的`future_workload_calendar`。
    b. 如果一个成员在未来一周内，有超过3天的`planned_hours` > `available_hours`，则标记为**[中风险] 持续过载**。

4.  **汇总并生成报告**:
    a. 综合所有识别出的风险，给出一个总体的`version_health_score` (A-F)。
    b. 为每一个识别出的风险，构建一个风险对象，必须包含`type`, `severity`, `title`, `description`, `impact_analysis`（对里程碑的可能影响）, 和`suggested_actions`（提供至少2条具体的、可操作的建议）。
    c. 严格按照指定的JSON格式输出你的风险报告。

# 输出格式
你的回答必须且只能是一个JSON对象，绝对不要包含任何JSON格式之外的解释性文字。
{
  "version_health_score": "<string, A-F>",
  "risks": [
    {
      "risk_id": "<string, e.g., R-001>",
      "type": "<string, e.g., SCHEDULE_DELAY, DEPENDENCY_BLOCKAGE, BURNOUT_RISK>",
      "severity": "<string, HIGH, MEDIUM, LOW>",
      "title": "<string, 风险的简明扼要标题>",
      "description": "<string, 对风险的具体情况描述>",
      "impact_analysis": "<string, 对项目里程碑或目标的潜在影响分析>",
      "suggested_actions": [
        "<string, 具体、可操作的建议1>",
        "<string, 具体、可操作的建议2>"
      ]
    }
  ]
}
```

### 3.3. 输出 (Output)

**输出示例:**
```json
{
  "version_health_score": "C",
  "risks": [
    {
      "risk_id": "R-001",
      "type": "DEPENDENCY_BLOCKAGE",
      "severity": "HIGH",
      "title": "核心功能'权限控制'延期，阻塞关键路径",
      "description": "需求'REQ-124'中的后端开发任务(T-01)已于'2025-09-12'到期，但当前进度仅50%，已延期。该需求位于项目关键路径上。",
      "impact_analysis": "将直接导致下游需求'REQ-125'无法按时开始，预计整个版本的'开发完成时间'里程碑将推迟至少5个工作日。",
      "suggested_actions": [
        "立即与执行人'张三'沟通，明确延期原因和新的预计完成时间。",
        "评估是否可以从其他后端工程师处调配资源以加速此任务。"
      ]
    },
    {
      "risk_id": "R-002",
      "type": "BURNOUT_RISK",
      "severity": "MEDIUM",
      "title": "前端工程师'李四'未来一周存在过载风险",
      "description": "'李四'在未来一周的计划工作（包括本项目的'T-02'任务和其他项目任务）中，每日计划工时持续超出其可用工时。",
      "impact_analysis": "可能导致其负责的'REQ-125'任务延期，并增加产生Bug的风险。",
      "suggested_actions": [
        "与'李四'沟通，确认其工作状态，并考虑重新安排非紧急任务的优先级。",
        "检查'REQ-125'任务的工时评估是否合理，是否存在评估不足的情况。"
      ]
    }
  ]
}
```
��以从其他后端工程师处调配资源以加速此任务。"
      ]
    },
    {
      "risk_id": "R-002",
      "type": "BURNOUT_RISK",
      "severity": "MEDIUM",
      "title": "后端工程师'张三'未来持续过载",
      "description": "'张三'在未来3天内，每日计划工作均超出其可用工时，存在倦怠风险。",
      "impact_analysis": "可能导致其后续任务效率降低、质量下滑，并进一步加剧延期。",
      "suggested_actions": [
        "重新检查并调整'张三'的任务排期，将非紧急任务延后。",
        "与'张三'进行一对一沟通，了解其实际工作状态和困难。"
      ]
    }
  ]
}
```
