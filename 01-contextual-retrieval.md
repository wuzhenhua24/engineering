# Introducing Contextual Retrieval

## 概述

**上下文检索（Contextual Retrieval）**是Anthropic改进检索增强生成（RAG）系统信息检索的方法。传统RAG系统存在的问题是"在编码信息时移除了上下文"，导致检索失败。该技术通过在嵌入和索引之前为块添加特定于块的解释性上下文来解决这个问题。

## 核心问题

### 传统RAG的局限

**问题：**
传统RAG系统在对文档进行分块时丢失上下文

**示例：**
```
原始文档：
"公司在2023年第三季度收入增长了25%。这一增长主要归功于新产品线的推出。"

分块后：
Chunk 1: "公司在2023年第三季度收入增长了25%。"
Chunk 2: "这一增长主要归功于新产品线的推出。"

问题：Chunk 2失去了"增长"指的是什么的上下文
```

**后果：**
- 检索失败
- 相关信息未被找到
- 回答质量下降

## 解决方案：上下文检索

### 核心思想

**在嵌入前添加上下文：**
```
原始Chunk: "这一增长主要归功于新产品线的推出。"

添加上下文后:
"本文档讨论公司2023年第三季度的财务表现。这一增长（指25%的收入增长）主要归功于新产品线的推出。"
```

### 两种子技术

#### 1. 上下文嵌入（Contextual Embeddings）

**方法：**
在将文本块转换为向量嵌入之前添加上下文

**流程：**
```
文档 → 分块 → 为每个块生成上下文 → 添加上下文 → 嵌入 → 索引
```

#### 2. 上下文BM25（Contextual BM25）

**方法：**
在创建基于关键词的搜索索引之前添加上下文

**BM25简介：**
- Best Matching 25
- 基于词频的排序算法
- 传统信息检索技术
- 补充语义搜索

## 技术实现

### 预处理工作流

#### 1. 文档分块

```python
def chunk_document(document, chunk_size=800):
    """将文档分割为固定大小的块"""
    chunks = []
    for i in range(0, len(document), chunk_size):
        chunks.append(document[i:i + chunk_size])
    return chunks
```

#### 2. 生成块特定上下文

**使用Claude自动生成：**

```python
def generate_context(document, chunk):
    """使用Claude为块生成上下文"""
    prompt = f"""
    <document>
    {document}
    </document>

    Here is the chunk we want to situate within the whole document:
    <chunk>
    {chunk}
    </chunk>

    Please give a short succinct context to situate this chunk
    within the overall document for the purposes of improving
    search retrieval of the chunk. Answer only with the succinct
    context and nothing else.
    """

    context = claude.generate(
        model="claude-3-haiku-20240307",
        prompt=prompt,
        max_tokens=100  # 50-100 tokens
    )

    return context
```

#### 3. 添加上下文到块

```python
def add_context_to_chunk(chunk, context):
    """将生成的上下文添加到块前"""
    return f"{context}\n\n{chunk}"
```

#### 4. 嵌入和索引

```python
def process_document(document):
    # 分块
    chunks = chunk_document(document)

    # 为每个块生成上下文
    contextualized_chunks = []
    for chunk in chunks:
        context = generate_context(document, chunk)
        contextualized_chunk = add_context_to_chunk(chunk, context)
        contextualized_chunks.append(contextualized_chunk)

    # 嵌入
    embeddings = [embed(chunk) for chunk in contextualized_chunks]

    # 索引
    index.add(embeddings, chunks)  # 存储原始块，但使用上下文化版本的嵌入
```

### 提示缓存优化

**成本效率：**
使用Claude的提示缓存功能

**机制：**
```python
# 缓存整个文档
cache_prompt = {
    "system": [
        {
            "type": "text",
            "text": document,
            "cache_control": {"type": "ephemeral"}
        }
    ]
}

# 为每个块重用缓存的文档
for chunk in chunks:
    context = claude.generate(
        cached_prompt=cache_prompt,
        user_message=f"Contextualize this chunk: {chunk}"
    )
```

**成本：**
$1.02 每百万文档token

### 运行时检索

**双重检索：**
```python
def retrieve(query, top_k=20):
    # 1. 语义搜索（使用上下文化的嵌入）
    semantic_results = semantic_search(query, top_k)

    # 2. 词法搜索（使用上下文化的BM25）
    lexical_results = bm25_search(query, top_k)

    # 3. 组合结果
    combined = combine_results(semantic_results, lexical_results)

    # 4. 可选：重排序
    if use_reranking:
        combined = rerank(query, combined)

    return combined[:top_k]
```

## 性能结果

### 检索失败率降低

**上下文嵌入单独：**
- 检索失败减少：**35%**
- 从基线到改进的显著提升

**上下文嵌入 + 上下文BM25：**
- 检索失败减少：**49%**
- 组合方法效果最佳

**添加重排序：**
- 检索失败减少：**67%**
- 从5.7%降至1.9%
- 最佳整体性能

### 数值对比

```
方法                          失败率    改进
────────────────────────────────────────────
基线（标准RAG）                5.7%      -
+ 上下文嵌入                   3.7%     -35%
+ 上下文嵌入 + 上下文BM25      2.9%     -49%
+ 以上 + 重排序                1.9%     -67%
```

### 跨领域表现

在多个知识域测试表现一致：
- 技术文档
- 法律文件
- 科学论文
- 业务报告

## 实施考虑

### 1. 块大小和边界

**影响检索性能：**

**策略：**
```python
# 固定大小分块
chunks = fixed_size_chunking(document, size=800)

# 句子边界分块
chunks = sentence_boundary_chunking(document)

# 段落边界分块
chunks = paragraph_chunking(document)

# 语义分块
chunks = semantic_chunking(document)
```

**建议：**
- 测试不同分块策略
- 根据文档类型调整
- 平衡块大小和上下文完整性

### 2. 嵌入模型选择

**测试结果：**
- **Gemini**：表现最佳
- **Voyage**：表现最佳
- 其他模型：可接受

**建议：**
- 在你的数据上测试不同模型
- 考虑成本-性能权衡
- 可能需要特定领域模型

### 3. 自定义提示

**领域特定变体：**

```python
# 技术文档
tech_prompt = """
Provide technical context for this code chunk,
including: function purpose, dependencies, and
related components.
"""

# 法律文档
legal_prompt = """
Provide legal context for this clause,
including: document type, relevant sections,
and legal implications.
"""

# 科学论文
science_prompt = """
Provide scientific context for this passage,
including: research topic, methodology section,
and key findings.
"""
```

**可能改进结果：**
根据特定领域优化

### 4. 块数量

**测试结果：**
```
块数量      检索质量
──────────────────
5个         基线
10个        中等
20个        最佳
```

**建议：**
20个块优于5个或10个

### 5. 评估重要性

**必须运行评估：**
验证在你的特定用例上的改进

**评估指标：**
- 检索准确率
- 召回率
- F1分数
- 用户满意度

**方法：**
```python
def evaluate_retrieval(queries, ground_truth):
    results = []
    for query, expected in zip(queries, ground_truth):
        retrieved = retrieve(query)
        precision = calculate_precision(retrieved, expected)
        recall = calculate_recall(retrieved, expected)
        results.append((precision, recall))
    return results
```

## 替代方法评估

### 其他上下文增强方法

**Anthropic测试的方法：**

#### 1. 通用文档摘要

**方法：**
为每个块添加整个文档的摘要

**表现：**
有限的改进

**原因：**
摘要过于通用，缺少块特定细节

#### 2. 假设文档嵌入

**方法：**
为块生成假设问题，然后嵌入

**表现：**
低于上下文检索

#### 3. 基于摘要的索引

**方法：**
索引块摘要而不是原始内容

**表现：**
表现不佳

### 为何上下文检索更优

**特异性：**
- 块特定上下文
- 保留细节
- 准确定位

**灵活性：**
- 适应不同文档类型
- 可自定义提示
- 支持多语言

## 使用建议

### 小型知识库（<200K tokens）

**建议：**
```python
# 简单方法：将所有内容放入提示
def answer_query(query, knowledge_base):
    # 使用提示缓存降低成本
    response = claude.generate(
        system=f"""
        <knowledge_base cache=true>
        {knowledge_base}
        </knowledge_base>
        """,
        user=query
    )
    return response
```

**优势：**
- 最简单
- 无需检索
- 完整上下文

### 大型知识库（>200K tokens）

**建议：**
结合上下文检索 + 上下文BM25 + 重排序

**实现：**
```python
class ContextualRAG:
    def __init__(self):
        self.embeddings = ContextualEmbeddings()
        self.bm25 = ContextualBM25()
        self.reranker = Reranker()

    def retrieve(self, query, top_k=20):
        # 双重检索
        semantic = self.embeddings.search(query, top_k)
        lexical = self.bm25.search(query, top_k)

        # 组合
        combined = self.combine(semantic, lexical)

        # 重排序
        reranked = self.reranker.rerank(query, combined)

        return reranked[:top_k]

    def answer(self, query):
        # 检索相关块
        chunks = self.retrieve(query)

        # 生成回答
        response = claude.generate(
            system=f"Use these chunks: {chunks}",
            user=query
        )
        return response
```

### 成本-延迟权衡

**考虑因素：**

| 组件 | 成本 | 延迟 | 质量提升 |
|------|------|------|---------|
| 上下文嵌入 | 低 | 低 | +35% |
| + 上下文BM25 | 低 | 低 | +49% |
| + 重排序 | 中 | 中 | +67% |

**建议：**
- 低预算/低延迟：上下文嵌入
- 平衡：上下文嵌入 + BM25
- 最高质量：全部组合

## 技术深入

### 上下文生成示例

**输入：**
```
文档：《2023年年度报告》
块："收入增长25%"
```

**生成的上下文：**
```
本文档是XYZ公司2023年度财务报告。
以下数据涉及第三季度的财务表现，
特别是与去年同期相比的收入增长情况。
```

**最终上下文化的块：**
```
本文档是XYZ公司2023年度财务报告。
以下数据涉及第三季度的财务表现，
特别是与去年同期相比的收入增长情况。

收入增长25%
```

### BM25工作原理

**基本原理：**
```
BM25分数 = Σ IDF(qi) × (f(qi, D) × (k1 + 1)) /
                      (f(qi, D) + k1 × (1 - b + b × |D| / avgdl))

其中：
- qi: 查询中的词项
- f(qi, D): 词项在文档中的频率
- |D|: 文档长度
- avgdl: 平均文档长度
- k1, b: 调优参数
```

**上下文BM25：**
将上下文也纳入词频计算

### 重排序机制

**两阶段检索：**

**第一阶段：**
```python
# 快速检索大量候选（如100个）
candidates = initial_retrieve(query, top_k=100)
```

**第二阶段：**
```python
# 使用更复杂模型精确排序
reranked = reranker.score(query, candidates)
top_results = reranked[:20]
```

**重排序模型：**
- Cohere Rerank
- 交叉编码器
- 更大的嵌入模型

## 实施检查清单

### 准备阶段
- [ ] 评估知识库大小
- [ ] 选择分块策略
- [ ] 确定嵌入模型
- [ ] 准备评估数据集

### 开发阶段
- [ ] 实现文档分块
- [ ] 集成Claude用于上下文生成
- [ ] 配置提示缓存
- [ ] 实现上下文嵌入
- [ ] 实现上下文BM25（可选）
- [ ] 集成重排序（可选）

### 测试阶段
- [ ] 在测试数据上运行评估
- [ ] 测试不同块大小
- [ ] 测试不同嵌入模型
- [ ] 测试不同提示变体
- [ ] 比较成本和性能

### 优化阶段
- [ ] 调优块大小和边界
- [ ] 优化上下文生成提示
- [ ] 调整检索参数
- [ ] 平衡成本和质量

### 部署阶段
- [ ] 监控检索质量
- [ ] 跟踪成本
- [ ] 收集用户反馈
- [ ] 持续优化

## 最佳实践

### 1. 从简单开始

**进展路径：**
```
1. 基础RAG（建立基线）
2. + 上下文嵌入（35%改进）
3. + 上下文BM25（49%改进）
4. + 重排序（67%改进）
```

### 2. 测量一切

**关键指标：**
```python
metrics = {
    'retrieval_accuracy': measure_accuracy(),
    'token_cost': calculate_cost(),
    'latency': measure_latency(),
    'user_satisfaction': survey_users()
}
```

### 3. 领域适配

**自定义：**
- 提示模板
- 块大小
- 嵌入模型
- 重排序策略

### 4. 成本优化

**策略：**
```python
# 使用提示缓存
use_prompt_caching = True

# 批处理上下文生成
batch_size = 100

# 选择合适的Claude模型
model = "claude-3-haiku-20240307"  # 成本效率高
```

### 5. 持续改进

**循环：**
```
部署 → 监控 → 分析 → 优化 → 部署
```

## 关键要点

1. **上下文丢失问题**：传统RAG在分块时丢失上下文
2. **自动化解决**：使用Claude自动生成块特定上下文
3. **显著改进**：检索失败减少35-67%
4. **组合效果最佳**：上下文嵌入 + BM25 + 重排序
5. **成本效益**：提示缓存使成本降至$1.02/百万tokens
6. **简单优先**：小知识库直接使用提示即可
7. **必须评估**：在你的数据上验证改进
8. **持续优化**：根据实际使用调整策略

## 技术优势

### 1. 自动化

**无需手动注释：**
- Claude自动生成上下文
- 减少人工工作
- 一致性高

### 2. 可扩展性

**适用于：**
- 任何规模的知识库
- 多种文档类型
- 不同语言

### 3. 灵活性

**可定制：**
- 提示模板
- 上下文长度（50-100 tokens）
- 分块策略

### 4. 成本效率

**提示缓存：**
- $1.02/百万文档tokens
- 可负担的大规模部署

## 与其他技术的关系

### RAG系统组件

```
完整的上下文RAG系统
├── 文档处理
│   ├── 加载
│   ├── 清理
│   └── 分块
├── 上下文增强（本文重点）
│   ├── 上下文生成
│   └── 上下文添加
├── 索引
│   ├── 上下文嵌入
│   └── 上下文BM25
├── 检索
│   ├── 语义搜索
│   ├── 词法搜索
│   ├── 混合检索
│   └── 重排序
└── 生成
    └── Claude生成回答
```

## 未来方向

### 潜在改进

**1. 动态上下文长度**
- 根据块复杂度调整
- 简单块：短上下文
- 复杂块：长上下文

**2. 多级上下文**
- 文档级
- 章节级
- 块级

**3. 跨文档上下文**
- 考虑相关文档
- 知识图谱集成

**4. 自适应提示**
- 根据文档类型
- 根据查询类型
- 动态调整

## 实际应用场景

### 1. 企业知识库

**用例：**
- 内部文档搜索
- 政策和程序查询
- 技术文档检索

**好处：**
- 提高员工效率
- 准确信息访问
- 减少重复问题

### 2. 客户支持

**用例：**
- 产品文档搜索
- 常见问题解答
- 故障排除指南

**好处：**
- 更快的问题解决
- 一致的回答质量
- 减少支持成本

### 3. 法律和合规

**用例：**
- 合同条款检索
- 法规遵从性检查
- 判例法研究

**好处：**
- 准确的法律信息
- 降低合规风险
- 提高研究效率

### 4. 研究和开发

**用例：**
- 科学文献搜索
- 实验数据检索
- 专利分析

**好处：**
- 加速研究进程
- 发现相关工作
- 避免重复研究

## 结论

上下文检索通过在嵌入和索引之前为文档块添加特定于块的解释性上下文，显著改进了RAG系统的信息检索能力。通过使用Claude自动生成上下文，结合上下文嵌入、上下文BM25和重排序，可以将检索失败率降低高达67%。

该方法的关键优势在于：自动化（使用Claude生成上下文）、成本效益（通过提示缓存）、显著的性能提升，以及跨不同知识域的一致表现。

对于小型知识库（<200K tokens），最简单的方法是直接将所有内容放入提示并使用缓存。对于更大的知识库，结合上下文检索技术可以在保持成本可控的同时大幅提升检索质量。

关键是要在自己的数据上进行评估，根据具体需求在成本、延迟和质量之间做出权衡，并持续监控和优化系统性能。

---

*原文链接：https://www.anthropic.com/engineering/contextual-retrieval*
