# Building effective AI agents

## 概述

Anthropic关于AI代理的研究强调简单性和可组合性优于复杂性。文档区分了两种架构方法：

**工作流（Workflows）：**
LLM和工具遵循预定义的代码路径，具有结构化编排

**代理（Agents）：**
LLM动态指导自己的流程和工具使用，具有更大的自主性

**核心哲学：**
> "LLM领域的成功不在于构建最复杂的系统，而在于构建适合你需求的正确系统。"

## 核心概念

### 工作流 vs 代理

#### 工作流特点
- 预定义的执行路径
- 程序化控制流
- 可预测的行为
- 明确的步骤

#### 代理特点
- 动态决策
- LLM驱动的导航
- 适应性行为
- 探索性方法

#### 选择标准

```
使用工作流当：
- 任务结构明确
- 步骤可预先定义
- 需要可预测性
- 成本敏感

使用代理当：
- 任务动态变化
- 需要适应性
- 探索性解决
- 自主性有价值
```

## 构建块

### 增强的LLM

**基础元素：**
通过以下能力增强的LLM：
- 检索（Retrieval）
- 工具（Tools）
- 记忆（Memory）

### 模型上下文协议（MCP）

**推荐：**
用于集成第三方工具

**优势：**
- 标准化接口
- 简化集成
- 自动认证
- 社区支持

## 五种工作流模式

### 1. Prompt Chaining（提示链）

#### 概念

**定义：**
将任务分解为顺序步骤，每个步骤由单独的LLM调用处理

#### 架构

```
步骤1 → LLM → 输出1
         ↓
步骤2 → LLM → 输出2
         ↓
步骤3 → LLM → 输出3
         ↓
     最终结果
```

#### 特点

**程序化门控：**
- 质量控制检查点
- 验证中间结果
- 条件分支

**示例流程：**
```python
# 步骤1：生成初稿
draft = llm.generate("Write a blog post about AI")

# 质量门控
if quality_check(draft) < threshold:
    draft = llm.generate("Improve this draft: " + draft)

# 步骤2：添加引用
with_citations = llm.generate("Add citations to: " + draft)

# 步骤3：最终润色
final = llm.generate("Polish this article: " + with_citations)
```

#### 适用场景

- 任务清晰分解为固定子任务
- 需要中间验证
- 步骤之间有依赖关系

#### 优势

- 可控性强
- 易于调试
- 可预测的成本

#### 劣势

- 缺乏灵活性
- 不适应意外情况

### 2. Routing（路由）

#### 概念

**定义：**
分类输入并将其定向到专门的下游处理器

#### 架构

```
输入 → 分类器LLM
         ├→ 技术问题 → 技术支持LLM
         ├→ 账单问题 → 账单LLM
         ├→ 一般查询 → 通用LLM
         └→ 升级 → 人工
```

#### 实现示例

```python
def route_query(query):
    # 分类
    category = classifier_llm.classify(query)

    # 路由到专门处理器
    if category == "technical":
        return technical_support_llm(query)
    elif category == "billing":
        return billing_llm(query)
    elif category == "escalation":
        return human_handoff(query)
    else:
        return general_llm(query)
```

#### 适用场景

- 明确的输入类别
- 每个类别需要专门处理
- 优化不同场景的性能

#### 优势

- 针对性优化
- 成本效率（小模型处理简单任务）
- 专业化能力

#### 劣势

- 需要良好的分类
- 可能错误路由

### 3. Parallelization（并行化）

#### 两种类型

##### Sectioning（分段）

**定义：**
同时执行独立子任务

**示例：**
```python
# 并行生成文档不同部分
import asyncio

async def generate_document():
    tasks = [
        llm.generate("Write introduction"),
        llm.generate("Write methodology"),
        llm.generate("Write results"),
        llm.generate("Write conclusion")
    ]

    sections = await asyncio.gather(*tasks)
    return combine_sections(sections)
```

##### Voting（投票）

**定义：**
运行相同任务多次以获得多样输出

**示例：**
```python
# 生成多个响应并选择最佳
responses = [
    llm.generate(prompt) for _ in range(5)
]

# 使用另一个LLM评估并选择最佳
best = evaluator_llm.select_best(responses)
```

#### 适用场景

**Sectioning：**
- 任务无相互依赖
- 可以独立完成

**Voting：**
- 需要高质量输出
- 多视角有价值
- 提高置信度

#### 优势

- 速度提升（sectioning）
- 质量提升（voting）
- 多样性

#### 劣势

- 成本增加
- 需要结果合并

### 4. Orchestrator-Workers（协调者-工作者）

#### 概念

**定义：**
中心LLM动态分解任务，委托给工作者，并综合结果

#### 架构

```
协调者LLM
├→ 分析任务
├→ 分解为子任务
├→ 分配给工作者
│   ├→ 工作者1
│   ├→ 工作者2
│   └→ 工作者N
└→ 综合结果
```

#### 编码场景示例

```python
class Orchestrator:
    def solve_issue(self, github_issue):
        # 分析问题
        analysis = self.llm.analyze(github_issue)

        # 动态创建工作者
        workers = []
        if analysis.needs_research:
            workers.append(ResearchWorker())
        if analysis.needs_coding:
            workers.append(CodingWorker())
        if analysis.needs_testing:
            workers.append(TestingWorker())

        # 执行并综合
        results = [w.execute() for w in workers]
        return self.llm.synthesize(results)
```

#### 适用场景

- 不可预测的任务
- 需要动态分解
- 复杂问题解决

#### 优势

- 高度灵活
- 适应性强
- 处理复杂性

#### 劣势

- 更高成本
- 更难调试
- 潜在的无限循环

### 5. Evaluator-Optimizer（评估者-优化者）

#### 概念

**定义：**
创建迭代反馈循环，一个LLM生成，另一个评估

#### 架构

```
生成器LLM → 输出
      ↓
评估器LLM → 反馈
      ↓
生成器LLM → 改进输出
      ↓
   [循环直到满意]
```

#### 类比

类似于迭代写作过程：
- 写作 → 审阅 → 修改 → 审阅 → ...

#### 实现示例

```python
def iterative_improve(initial_prompt, max_iterations=5):
    output = generator_llm.generate(initial_prompt)

    for i in range(max_iterations):
        # 评估当前输出
        feedback = evaluator_llm.evaluate(output)

        if feedback.score >= threshold:
            break

        # 基于反馈改进
        output = generator_llm.improve(output, feedback)

    return output
```

#### 适用场景

- 需要迭代改进
- 质量标准模糊
- 受益于多轮优化

#### 优势

- 持续改进
- 更高质量输出
- 捕获细微问题

#### 劣势

- 高token成本
- 可能过度优化
- 收敛不确定

## 代理实现

### 核心机制

**LLM-工具交互循环：**

```
1. LLM分析任务
2. 选择工具
3. 执行工具
4. 观察结果
5. 评估进度
6. 重复或完成
```

### 设计要求

#### 1. 清晰的工具集设计

**原则：**
```python
# 好的工具设计
{
    "name": "search_customer",
    "description": "Search for customers by name or email",
    "parameters": {
        "query": "Search term",
        "limit": "Max results (default: 10)"
    },
    "returns": "List of matching customers"
}
```

**关键问题：**
> "把自己放在模型的位置。基于描述和参数，如何使用这个工具是否明显？"

#### 2. 详细的文档

**包含：**
- 工具用途
- 参数说明
- 使用示例
- 边缘情况
- 错误处理

#### 3. 环境反馈

**每步评估进度：**
```python
while not task_complete:
    action = agent.decide_next_action()
    result = execute(action)

    # 关键：评估进度
    progress = agent.evaluate_progress(result)

    if progress.stuck:
        agent.try_different_approach()
    elif progress.complete:
        break
```

#### 4. 可选的人工检查点

**实现：**
```python
if agent.uncertainty > threshold:
    human_input = request_human_guidance()
    agent.incorporate_feedback(human_input)
```

#### 5. 停止条件

**明确定义：**
- 成功完成
- 最大迭代次数
- 检测到循环
- 不确定性过高
- 资源耗尽

### 实现建议

#### 框架选择

**可用框架：**
- LangGraph
- Bedrock AI Agent framework
- Rivet
- Vellum

**Anthropic建议：**
> 从直接使用LLM API开始

**原因：**
- 理解底层代码
- 避免抽象层遮蔽
- 更好的控制和调试
- 清晰的提示和响应

**渐进方法：**
```
1. 从API开始
2. 理解机制
3. 如需要再引入框架
4. 保持透明度
```

## 工具设计哲学

### 关键原则

#### 1. 充足的Token用于推理

**给模型空间思考：**
- 不要过度限制输出长度
- 允许解释推理
- 支持逐步思考

#### 2. 自然格式

**保持接近互联网文本：**
```
✓ 好：Markdown格式的响应
✗ 差：过度结构化的XML
```

#### 3. 最小化格式开销

**避免：**
- 行号计数
- 复杂字符串转义
- 嵌套引用

**示例：**
```python
# 简单直接
def edit_file(content):
    return content  # 直接返回内容

# vs 复杂
def edit_file_complex(content):
    lines = content.split('\n')
    numbered = [f"{i}: {line}" for i, line in enumerate(lines)]
    return json.dumps({"lines": numbered})  # 增加复杂性
```

#### 4. 详细的使用示例

**在工具定义中包含：**
```python
{
    "name": "calculate_discount",
    "description": "Calculate discount price",
    "examples": [
        {
            "input": {"price": 100, "discount": 0.2},
            "output": 80,
            "explanation": "20% off $100 = $80"
        }
    ]
}
```

#### 5. 边缘情况说明

```python
{
    "name": "divide",
    "description": "Divide two numbers",
    "edge_cases": {
        "division_by_zero": "Returns error",
        "negative_numbers": "Supported",
        "floats": "Supported with precision to 2 decimals"
    }
}
```

#### 6. Poka-yoke原则

**防错设计：**
- 使错误使用困难
- 使正确使用简单
- 提供清晰的默认值

**示例：**
```python
def send_email(
    to: str,
    subject: str,
    body: str,
    send_immediately: bool = False  # 默认False防止意外发送
):
    if not send_immediately:
        return preview_email(to, subject, body)
    else:
        return actually_send(to, subject, body)
```

### 测试和优化

#### Workbench测试

**流程：**
1. 在workbench创建工具
2. 广泛测试
3. 观察模型行为
4. 迭代改进
5. 部署到生产

**重要性：**
> 工具误解是常见的客户问题来源

## 主要优势

### 成本-性能权衡

**代理系统：**
- 交易延迟和费用
- 换取优越的任务性能

### 可扩展性

**自主性优势：**
- 在可信环境中扩展
- 减少人工干预
- 处理更多任务

### 可测量的结果

**验证机制：**

**代码解决方案：**
- 通过测试验证
- 客观评估

**客户支持：**
- 成功解决指标
- 满意度评分
- 响应时间

### 专业化优化

**路由优势：**
- 针对场景定制
- 不需要一刀切妥协
- 优化每个路径

## 挑战与约束

### 1. 代理成本

**高运营成本：**
- 更多LLM调用
- 更多token使用
- 更长执行时间

**风险：**
复合错误

### 2. 需要沙箱测试

**安全要求：**
- 隔离测试环境
- 适当的护栏
- 监控机制

### 3. 工具误解

**常见问题来源：**
- 不清晰的描述
- 缺少示例
- 复杂接口

**预防：**
充分的workbench测试

### 4. 框架复杂性

**问题：**
- 框架可能激励不必要的复杂性
- 抽象层遮蔽机制
- 调试困难

**解决：**
从API开始，理解基础

### 5. 错误假设

**客户问题：**
关于底层代码机制的不正确假设

**预防：**
- 清晰的文档
- 透明的实现
- 充分的测试

## 真实世界应用

### 1. 客户支持

**能力：**
- 对话界面
- 数据检索工具
- 退款处理
- 工单管理

**成功指标：**
基于使用的定价表明公司对代理有效性的信心

**价值：**
- 24/7可用性
- 一致的响应
- 可扩展性

### 2. 编码代理

**任务：**
自主解决GitHub问题

**验证：**
SWE-bench基准测试

**人工审查：**
确保与更广泛系统需求一致

**能力：**
- 理解代码库
- 识别问题
- 实施修复
- 运行测试

## 核心建议

### 三个基础原则

#### 1. 简单性（Simplicity）

**保持设计直接：**
- 不引入不必要的复杂性
- 从最简单的可行方案开始
- 仅在需要时增加复杂性

**示例：**
```python
# 简单
def process(input):
    return llm.generate(input)

# 不必要的复杂
def process_complex(input):
    router = Router()
    orchestrator = Orchestrator()
    workers = WorkerPool()
    # ... 过度设计
```

#### 2. 透明度（Transparency）

**显式显示：**
- 代理规划步骤
- 推理过程
- 决策逻辑

**实现：**
```python
class TransparentAgent:
    def act(self, task):
        # 显示思考过程
        plan = self.think(task)
        print(f"Plan: {plan}")

        # 显示行动
        for step in plan.steps:
            print(f"Executing: {step}")
            result = self.execute(step)
            print(f"Result: {result}")
```

#### 3. 工具文档和测试

**投入等同努力：**
代理-计算机界面 = 人机界面

**检查清单：**
- [ ] 清晰的工具描述
- [ ] 详细的参数说明
- [ ] 使用示例
- [ ] 边缘情况文档
- [ ] 充分的workbench测试
- [ ] 错误处理指导

### 总体建议

**起点：**
> 从简单提示和单个LLM调用开始

**扩展：**
仅当简单解决方案明显不足时才添加代理复杂性

**决策树：**
```
任务简单明确？
├─ Yes → 单个LLM调用
└─ No → 任务可分解为固定步骤？
    ├─ Yes → 工作流（提示链/路由）
    └─ No → 需要动态适应？
        ├─ Yes → 代理
        └─ No → 重新评估需求
```

## 实用建议

### 开始项目

1. **定义清晰目标**
2. **从最简单方案开始**
3. **测试和衡量**
4. **逐步增加复杂性**
5. **持续评估必要性**

### 选择模式

**评估标准：**
- 任务可预测性
- 所需自主性
- 成本约束
- 性能要求
- 调试需求

### 工具开发

1. **从用户视角设计**
2. **详细文档化**
3. **充分测试**
4. **迭代改进**
5. **监控实际使用**

### 调试和优化

**工作流：**
- 使用workbench测试
- 收集实际使用数据
- 分析失败模式
- 优化提示和工具
- 重新测试

## 关键要点

1. **简单优先**：从简单开始，仅在必要时增加复杂性
2. **正确系统**：不是最复杂的，而是最适合的
3. **工具设计关键**：投入与人机界面同等努力
4. **透明度重要**：显示代理的思考和行动
5. **充分测试**：workbench测试防止工具误解
6. **从API开始**：理解机制再考虑框架
7. **可测量结果**：建立清晰的成功指标
8. **迭代方法**：持续评估和改进

## 模式选择指南

### 任务特征分析

| 特征 | 推荐模式 |
|------|---------|
| 固定步骤序列 | Prompt Chaining |
| 明确输入类别 | Routing |
| 独立子任务 | Parallelization (Sectioning) |
| 需要多样性 | Parallelization (Voting) |
| 动态不可预测 | Orchestrator-Workers |
| 需要迭代改进 | Evaluator-Optimizer |
| 完全动态自主 | Agent |

### 复杂度梯度

```
简单 ──────────────────────────→ 复杂

单次LLM调用
    ↓
Prompt Chaining
    ↓
Routing
    ↓
Parallelization
    ↓
Orchestrator-Workers
    ↓
Evaluator-Optimizer
    ↓
Full Agent
```

## 结论

构建有效的AI代理需要在简单性和能力之间仔细平衡。通过理解不同的架构模式——从结构化工作流到自主代理——开发者可以选择最适合其特定需求的方法。

关键是避免过度设计的诱惑，从最简单的可行解决方案开始，只在有明确益处时才增加复杂性。通过优先考虑工具设计、保持透明度、充分测试，开发者可以构建既强大又可维护的AI系统。

记住：成功不在于构建最复杂的系统，而在于构建正确的系统——适合你的任务、约束和目标的系统。

---

*原文链接：https://www.anthropic.com/engineering/building-effective-agents*
