
# Dify 智能体设计文档：最佳人员匹配引擎

## 1. 核心设计思想

本智能体的核心目标是为一个具体任务，从项目团队中推荐出**综合最优**的执行人选。它旨在超越简单的“技能匹配”，成为一个能够平衡**专业能力、工作效率、团队负载**乃至**成员成长**等多个维度的智能决策辅助工具。

为实现此目标，我们采用 **“层层筛选，综合评分 (Filter then Score & Rank)”** 的核心设计模式：

1.  **硬性资格筛选 (Hard Filtering):** 首先，通过一组不可违背的“硬性规则”，快速排除所有不合格的人选，缩小候选人范围。这保证了推荐的底线质量。
2.  **智能评分排序 (Smart Scoring & Ranking):** 其次，对通过筛选的候选人，使用一个可配置的**加权评分模型**，为每位候选人计算一个“综合匹配分”。分数越高，代表在当前情境下，该人选的综合“性价比”越高。

最终，引擎会输出一个带详细理由的、按分数排序的推荐列表，将最终决策权交给项目经理，同时使其决策过程有据可依。

## 2. 匹配逻辑详解

### 2.1. 第一阶段：硬性资格筛选

候选人必须同时满足以下所有条件，才能进入评分阶段：

1.  **职位匹配 (Job Title Match):**
    *   **规则:** 任务的 `type` 必须与候选人的 `JobTitle` (职位) 兼容。
    *   **映射示例:**
        *   `DESIGN` 任务 -> `UI设计师`
        *   `DEVELOPMENT` 任务 -> `前端工程师`, `后端工程师`
        *   `TESTING` 任务 -> `测试工程师`
    *   **目的:** 确保专业的人做专业的事。

2.  **可用性检查 (Availability Check):**
    *   **规则:** 候选人在该项目中的 `每日可用工时` 必须大于 0。
    *   **目的:** 排除正在休假或已无本项目工时的成员，确保推荐的有效性。

### 2.2. 第二阶段：智能评分排序

对通过筛选的候选人，我们会计算一个“综合匹配分”，公式如下：

`综合匹配分 = (权重A * 负载得分) + (权重B * 效率得分)`

*   **因子A：当前负载 (Current Workload)**
    *   **指标:** `current_workload_percent` (成员当前总负载百分比)。
    *   **逻辑:** 负载越低，得分越高。鼓励任务分配给相对空闲的成员，促进团队负载均衡。
    *   **计算:** `负载得分 = (100 - current_workload_percent)`

*   **因子B：工作效率 (Efficiency)**
    *   **指标:** `experienceMultiplier` (经验系数，来自其职级 `JobLevel`)。
    *   **逻辑:** 经验系数越低（效率越高），得分越高。在同等负载下，优先推荐更高效的成员。
    *   **计算:** `效率得分 = (2.0 - experienceMultiplier) * 50` (乘以50是为了让其与负载得分在同一数量级)

*   **权重 (Weights):**
    *   `权重A` 和 `权重B` 是可配置的系统参数，代表了团队的管理导向。
    *   **推荐初始值:** `权重A = 1.5` (负载均衡优先), `权重B = 1.0` (效率为辅)。
    *   这个设计使得系统可以灵活适配不同阶段的需求（如冲刺阶段可调高效率权重）。

## 3. Dify 智能体配置（交互契约）

### 3.1. 输入 (Input)

*   `task` (JSON): 需要被分配的任务。
*   `available_personnel` (JSON Array): 经过第一阶段硬性筛选后的候选人列表。

**输入示例:**
```json
{
  "task": {
    "title": "前端开发：根据用户角色动态显示UI元素",
    "type": "DEVELOPMENT"
  },
  "available_personnel": [
    {
      "name": "张三",
      "job_title": "FRONTEND",
      "job_level": "SENIOR",
      "experience_multiplier": 0.8,
      "current_workload_percent": 90
    },
    {
      "name": "李四",
      "job_title": "FRONTEND",
      "job_level": "INTERMEDIATE",
      "experience_multiplier": 1.0,
      "current_workload_percent": 30
    },
    {
      "name": "赵六",
      "job_title": "FRONTEND",
      "job_level": "JUNIOR",
      "experience_multiplier": 1.3,
      "current_workload_percent": 20
    }
  ]
}
```

### 3.2. 系统提示词 (System Prompt)

```
# 角色
你是一位高效的AI资源协调官，擅长在团队中为任务找到最合适的执行人。

# 上下文
你的任务是为一个给定的`task`，从`available_personnel`候选人列表中，推荐最合适的前3位人选。
- “最合适”是一个综合概念，需要平衡“当前负载”和“工作效率”。
- `current_workload_percent`: 成员当前有多忙，越低越好。
- `experience_multiplier`: 成员的经验系数，越低代表效率越高，越好。

# 指令
1.  **计算匹配分**: 遍历`available_personnel`列表中的每一位候选人。为每个人计算一个“综合匹配分”。
2.  **评分公式**: 请严格使用以下加权公式进行计算：
    `匹配分 = (1.5 * (100 - current_workload_percent)) + (1.0 * ((2.0 - experience_multiplier) * 50))`
3.  **排序**: 根据计算出的“匹配分”，从高到低对候选人进行排序。
4.  **生成推荐理由**: 为排名前3的每一位候选人，生成一句简明扼要的`reasoning`，必须清晰地体现出他/她的主要优势（是负载低，还是效率高）。
5.  **输出结果**: 严格按照指定的JSON格式输出你的推荐报告。

# 输出格式
你的回答必须且只能是一个JSON对象，绝对不要包含任何其他文字。
{
  "recommendations": [
    {
      "name": "<string>",
      "match_score": <number>,
      "reasoning": "<string, 解释推荐的关键原因>"
    }
  ]
}
```

### 3.3. 输出 (Output)

**输出示例:**
```json
{
  "recommendations": [
    {
      "name": "李四",
      "match_score": 155.0,
      "reasoning": "中级工程师，经验匹配。当前负载仅30%，非常可用，是立即开始此任务的理想人选。"
    },
    {
      "name": "赵六",
      "match_score": 155.0,
      "reasoning": "初级工程师，经验匹配。当前负载仅20%，可用性极高，适合在非核心任务中培养新人。"
    },
    {
      "name": "张三",
      "match_score": 75.0,
      "reasoning": "高级工程师，效率最高。但当前负载已达90%，建议仅在任务紧急或需要攻坚时考虑。"
    }
  ]
}
```
