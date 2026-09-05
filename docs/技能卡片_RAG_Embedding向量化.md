# 技能卡片：RAG Embedding 向量化

## 它属于哪一环

AI 应用流程位置：

```text
RAG 资料准备链路：document -> chunk -> embedding -> vector store
RAG 用户提问链路：question -> embedding -> retrieve -> prompt -> answer
```

上游输入：

```text
资料准备链路：chunk 文本
用户提问链路：question 用户问题
```

下游输出：

```text
一组数字向量 vector
```

一句话定位：

```text
embedding 是把文本转换成“可计算语义相似度”的数字向量。
```

---

## 它解决什么问题

计算机不能直接理解两段文字“意思像不像”。

比如：

```text
问题：公司赚钱能力变差了吗？
chunk A：毛利率下降可能说明成本压力或产品结构变化。
chunk B：公司总部位于上海。
```

人一眼知道 A 更相关，但程序如果只靠关键词，可能很难稳定判断。因为“赚钱能力变差”和“毛利率下降”不是同一组字，却在语义上有关。

embedding 解决的就是这个问题：

```text
把文字变成向量后，
系统可以用数学方式计算“语义距离”或“语义相似度”。
```

如果没有 embedding：

- 只能靠关键词匹配，容易漏掉同义表达。
- 很难找出“说法不同但意思相关”的片段。
- RAG 只能停留在简单搜索，难以处理复杂问题。

---

## 它的原理

核心概念：

- 向量：一组数字，例如 `[0.12, -0.03, 0.88, ...]`。
- 语义空间：相似含义的文本，在向量空间里的位置更接近。
- 相似度：用数学公式比较两个向量是否接近。
- embedding 模型：专门把文本转换成向量的模型。

底层机制可以这样理解：

```text
文本
-> embedding 模型
-> 高维向量
-> 向量之间可以计算相似度
```

它不是把每个字简单编码成数字，而是把整段文本的语义压缩成一个向量表示。

直觉例子：

```text
“收入增长”
“营业收入提升”
“销售规模扩大”
```

这三句话字面不同，但语义接近。好的 embedding 会让它们在向量空间里距离比较近。

而：

```text
“收入增长”
“服务器连接超时”
```

语义差很远，对应向量距离通常也更远。

---

## 它的核心逻辑

资料准备时：

```text
chunk
-> embedding model
-> chunk vector
-> vector store
```

用户提问时：

```text
question
-> embedding model
-> question vector
-> retrieve
```

关键节点职责：

| 节点 | 接收什么 | 它做什么 | 输出什么 | 为什么需要它 |
|---|---|---|---|---|
| chunk | 一段知识片段 | 提供要被索引的文本 | chunk text | 每个知识单元都要变成可检索表示 |
| embedding model | chunk 或 question | 把文本转换成向量 | vector | 让文本可以参与相似度计算 |
| chunk vector | chunk 的向量 | 表示知识片段的语义位置 | 可存储向量 | 后续写入 vector store |
| question vector | 用户问题的向量 | 表示问题的语义位置 | 查询向量 | 后续用来找相近 chunk |
| similarity | 两个向量 | 计算距离或相似度 | 分数 score | 判断哪个 chunk 更相关 |

---

## 为什么 chunk 和 question 都要 embedding

RAG 检索时，本质是在比较：

```text
question vector 和 chunk vector 谁更接近
```

所以两边都必须进入同一个向量空间。

资料准备阶段：

```text
chunk -> embedding -> chunk vector -> 存入 vector store
```

用户提问阶段：

```text
question -> embedding -> question vector -> 去 vector store 查询
```

如果只给 chunk 做 embedding，问题没有向量，就没法比较。

如果只给 question 做 embedding，知识库里没有 chunk vector，也没法检索。

---

## 相似度怎么理解

常见理解方式：

```text
两个向量越接近，说明两段文本语义越接近。
```

实际系统里常见计算方式有：

- cosine similarity：看两个向量方向是否接近。
- dot product：看两个向量的匹配程度。
- distance：看两个向量距离远近。

对初学者来说，不需要先陷入公式，先记住：

```text
embedding 让文本相似度可以被计算。
retrieve 根据相似度分数选出 topK chunk。
```

---

## 它的边界

embedding 负责：

- 把文本变成向量。
- 让文本可以用相似度进行比较。
- 给 retrieve 提供可计算基础。

embedding 不负责：

- 不负责切分文档。
- 不负责保存向量。
- 不负责最终选择 Prompt 怎么写。
- 不负责生成答案。
- 不保证检索一定正确。

上游依赖：

```text
chunk 质量、文本清洗质量、embedding 模型选择
```

下游影响：

```text
embedding 质量 -> retrieve 命中质量 -> prompt 上下文质量 -> answer 质量
```

特别要注意：

```text
embedding 不是答案；
embedding 是让检索系统能找到相关资料的语义坐标。
```

---

## 常见错误

错误 1：以为 embedding 会生成答案。

实际：

```text
embedding 只生成向量，不生成自然语言答案。
```

错误 2：以为 embedding 等于关键词搜索。

实际：

```text
embedding 更偏语义相似度，可以命中字面不同但意思相关的内容。
```

错误 3：chunk 太差，却怪 embedding 不准。

实际：

```text
如果 chunk 被切碎、噪声多、缺少上下文，embedding 得到的语义表示也会变差。
```

错误 4：资料准备和用户提问用了不同 embedding 模型。

实际：

```text
chunk vector 和 question vector 最好来自同一个 embedding 模型，否则相似度比较可能失真。
```

错误 5：只看 topK 分数，不看命中的原文。

实际：

```text
RAG 调试一定要看 retrieved chunk 的原文、来源和分数。
```

---

## 在项目里的落点

当前 MiniAINotes 的 RAG v0 还没有真正接入 embedding 模型，而是先用关键词检索模拟 retrieve。

当前阶段：

```text
本地知识片段
-> 关键词匹配
-> score
-> retrieved_contexts
```

后续升级为真正 RAG 时：

```text
本地知识片段 / 文档 chunk
-> embedding model
-> vector
-> vector store
-> question embedding
-> similarity search
-> topK retrieved_contexts
```

iOS 侧关注：

- 不需要计算 embedding。
- 只展示后端返回的 `retrieved_contexts`。
- 如果后端返回 score，可以展示或用于调试。

后端侧关注：

- 调用 embedding 模型。
- 保存 chunk vector。
- 查询 question vector。
- 返回 topK chunks 和 metadata。

AI 侧关注：

- embedding 不进入用户可见答案。
- retrieve 出来的上下文会进入 Prompt。
- 模型回答时应该被要求基于 retrieved_context。

---

## 复习时要会回答

1. embedding 为什么能帮助 RAG 检索？
2. chunk embedding 和 question embedding 分别发生在哪条链路？
3. embedding 输出的 vector 后续交给谁？
4. 为什么 chunk 和 question 最好用同一个 embedding 模型？
5. embedding、retrieve、answer 三者有什么区别？

---

## 本轮流程

理解 RAG 中的 `embedding` 节点。

## 核心思想

embedding 把文字变成语义向量，让系统可以用数学方式比较文本之间的语义相似度。

## 它在 AI 应用开发流程中的位置

- 上游：chunk 或 question
- 当前环节：embedding 向量化
- 下游：vector store / retrieve / prompt / answer

## 需要关注的技能知识点

- iOS：理解客户端一般不计算 embedding，只消费后端返回的检索上下文。
- 后端：掌握 embedding 模型调用、向量保存、查询向量生成。
- AI 应用：理解语义相似度如何让 RAG 找到相关资料。
- 工程化：调试 RAG 时要记录 query、命中 chunk、score 和来源。

## 下一步建议重点学习

1. vector store：向量和原文怎么存，为什么能快速检索。
2. retrieve：topK 相似度检索如何决定哪些 chunk 进入 Prompt。
3. prompt：如何把 retrieved_context 组织成模型可用的上下文。

