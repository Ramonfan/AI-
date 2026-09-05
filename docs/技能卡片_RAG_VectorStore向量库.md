# 技能卡片：RAG Vector Store 向量库

## 它属于哪一环

AI 应用流程位置：

```text
RAG 资料准备链路：document -> chunk -> embedding -> vector store
RAG 用户提问链路：question -> embedding -> retrieve -> prompt -> answer
```

上游输入：

```text
chunk 原文
chunk embedding 向量
metadata 来源信息
```

下游输出：

```text
可被 retrieve 查询的知识索引
```

一句话定位：

```text
vector store 是保存“文本块 + 向量 + 来源信息”，并支持按语义相似度检索的数据库或索引系统。
```

---

## 它解决什么问题

embedding 会把文本变成向量，但向量本身只是内存里的一组数字。

如果没有 vector store，会有几个问题：

- chunk vector 没地方长期保存。
- 用户每次提问都要重新计算和扫描全部向量。
- 无法根据问题快速找出最相似的 topK chunk。
- 无法把检索结果追溯回原文、标题、页码或文档来源。

vector store 解决的是：

```text
把已经向量化的知识组织成可查询资产，
让在线提问时可以快速从大量 chunk 中找出相关内容。
```

---

## 它的原理

核心概念：

- vector：embedding 模型生成的一组数字。
- payload / metadata：和向量绑定的业务信息，例如原文、标题、文件名、页码、章节。
- index：为了快速相似度搜索建立的数据结构。
- similarity search：根据问题向量查找最接近的 chunk 向量。
- topK：返回最相关的前 K 条结果。

底层机制可以这样理解：

```text
资料准备时：
chunk text
-> embedding
-> chunk vector
-> vector store 保存 vector + text + metadata

用户提问时：
question
-> question embedding
-> question vector
-> vector store 相似度搜索
-> topK chunks
```

vector store 不是只存向量。真正有用的是它同时保存：

```text
向量：用于相似度计算
原文：用于塞进 Prompt
metadata：用于展示引用和调试
```

---

## 它的核心逻辑

写入阶段：

```text
chunk
-> embedding
-> vector + metadata
-> upsert 到 vector store
```

查询阶段：

```text
question
-> embedding
-> query vector
-> similarity search
-> topK retrieved_context
```

关键节点职责：

| 节点 | 接收什么 | 它做什么 | 输出什么 | 为什么需要它 |
|---|---|---|---|---|
| chunk text | 一段知识片段 | 提供可进入 Prompt 的原文 | text | 检索命中后要把原文交给模型 |
| chunk vector | chunk 的 embedding | 表示 chunk 的语义位置 | vector | 用于和 question vector 做相似度比较 |
| metadata | 文档名、标题、页码、章节 | 描述 chunk 来源 | source info | 让回答可引用、可追溯、可调试 |
| vector index | 大量 chunk vector | 建立快速查询结构 | searchable index | 避免每次暴力扫描全部向量 |
| similarity search | question vector | 查找最接近的 chunk vector | topK chunks | 把相关知识取出来进入 Prompt |

---

## 和普通数据库有什么区别

普通数据库更擅长精确匹配：

```sql
WHERE ticker = 'AAPL'
WHERE report_year = 2024
```

vector store 更擅长语义相似：

```text
问题：“公司赚钱质量怎么样？”
命中：“经营现金流和净利润背离，可能说明利润质量需要进一步确认。”
```

也就是说：

```text
普通数据库回答：字段是否等于某个值。
向量库回答：哪段文本和这个问题意思更接近。
```

两者不是替代关系。真实 AI 应用里经常一起用：

```text
业务数据库：存用户、订单、财报指标等结构化数据
vector store：存文档片段、知识片段、语义索引
```

---

## Vector Store 里通常存什么

一条典型记录可以理解为：

```json
{
  "id": "chunk_001",
  "vector": [0.12, -0.03, 0.88],
  "text": "毛利率下降通常需要拆分价格、成本、产品结构和产能利用率。",
  "metadata": {
    "source": "financial_analysis_basics.md",
    "title": "毛利率与成本压力",
    "section": "盈利质量",
    "page": 3
  }
}
```

其中：

- `id`：方便更新、删除和定位。
- `vector`：用于相似度检索。
- `text`：命中后进入 Prompt。
- `metadata`：用于引用来源、过滤和调试。

---

## 查询时发生了什么

用户问：

```text
这家公司利润质量怎么样？
```

系统会做：

```text
question
-> embedding model
-> question vector
-> vector store search
-> 返回 topK chunks
```

返回结果可能是：

```text
1. 经营现金流和净利润背离，可能说明利润质量需要进一步确认。
2. 毛利率下降需要拆分价格、成本、产品结构和产能利用率。
3. 应收账款快速增长可能影响现金回收质量。
```

这些结果再进入 Prompt：

```text
请基于以下参考知识回答用户问题：
context 1...
context 2...
context 3...

用户问题：这家公司利润质量怎么样？
```

---

## 它的边界

vector store 负责：

- 保存向量、原文和 metadata。
- 支持相似度检索。
- 返回 topK 相关 chunk。
- 支持按 metadata 过滤结果。

vector store 不负责：

- 不负责把文档切成 chunk。
- 不负责生成 embedding。
- 不负责判断答案是否正确。
- 不负责组织 Prompt。
- 不负责生成最终自然语言回答。

上游依赖：

```text
chunk 质量、embedding 质量、metadata 是否完整
```

下游影响：

```text
vector store 检索质量 -> retrieved_context 质量 -> prompt 质量 -> answer 质量
```

特别要注意：

```text
vector store 不是知识本身；
它是让知识可以被语义检索的存储和索引层。
```

---

## 常见错误

错误 1：只存 vector，不存原文。

问题：

```text
检索命中了向量，但没有文本可以塞进 Prompt。
```

错误 2：不存 metadata。

问题：

```text
回答无法展示引用来源，也无法调试为什么命中。
```

错误 3：把 vector store 当成普通数据库。

问题：

```text
向量库适合语义相似度，不适合替代所有结构化查询。
```

错误 4：topK 越大越好。

问题：

```text
取太多 chunk 会让 Prompt 变长，噪声变多，模型反而更容易跑偏。
```

错误 5：不做更新策略。

问题：

```text
文档更新了，但旧 chunk 和旧 vector 还在库里，回答会引用过期知识。
```

---

## 在项目里的落点

当前 MiniAINotes 的 RAG v0 还没有接入真正的 vector store。

当前阶段是：

```text
本地知识片段
-> 关键词打分
-> top3 retrieved_contexts
```

后续升级时会变成：

```text
知识片段 / 文档 chunk
-> embedding
-> vector store upsert
-> question embedding
-> vector search
-> topK retrieved_contexts
```

iOS 侧关注：

- iOS 不直接访问 vector store。
- iOS 只展示后端返回的 `retrieved_contexts`。
- 引用标题、来源、分数可以帮助用户判断可信度。

后端侧关注：

- 选择 vector store。
- 设计 chunk id 和 metadata。
- 写入向量和原文。
- 查询 topK。
- 控制过滤条件和返回数量。

AI 侧关注：

- vector store 的检索结果不是最终答案。
- retrieved_context 会进入 Prompt。
- 模型必须被约束为基于 retrieved_context 回答。

---

## 复习时要会回答

1. vector store 存的是不是只有向量？
2. 为什么要同时保存 text 和 metadata？
3. vector store 和普通数据库的区别是什么？
4. 用户 question embedding 后，vector store 如何参与 retrieve？
5. topK 太大或太小分别有什么问题？

---

## 本轮流程

理解 RAG 中的 `vector store` 节点。

## 核心思想

vector store 是 RAG 的语义索引层。它把 chunk、embedding 向量和来源信息组织起来，让系统能根据用户问题快速找回相关知识。

## 它在 AI 应用开发流程中的位置

- 上游：chunk -> embedding
- 当前环节：vector store 存储和索引
- 下游：question embedding -> retrieve -> prompt -> answer

## 需要关注的技能知识点

- iOS：理解客户端只消费 retrieved_context，不直接查询向量库。
- 后端：掌握向量写入、metadata 设计、topK 查询和更新策略。
- AI 应用：理解向量库负责找资料，不负责生成答案。
- 工程化：需要验证命中原文、score、source，避免黑盒检索。

## 下一步建议重点学习

1. retrieve：question vector 如何从 vector store 找到 topK chunks。
2. topK 和 score：如何判断命中结果是否可靠。
3. prompt：如何把 retrieved_context 组织成高质量模型输入。

