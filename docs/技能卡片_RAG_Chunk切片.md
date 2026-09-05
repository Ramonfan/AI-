# 技能卡片：RAG Chunk 切片

## 它属于哪一环

AI 应用流程位置：

```text
RAG 资料准备链路：document -> chunk -> embedding -> vector store
```

上游输入：

```text
document：原始文档，例如 PDF、网页、Markdown、公告、财报、课程笔记
```

下游输出：

```text
chunks：多个较小、语义相对完整、可以被 embedding 和检索的文本块
```

一句话定位：

```text
chunk 是把“不可直接检索和使用的长文档”变成“可检索知识单元”的过程。
```

---

## 它解决什么问题

大模型和检索系统都不适合直接处理整篇长文档。

如果不切片，会有几个问题：

- 文档太长，不能全部塞进 Prompt。
- 文档里只有一小段和问题相关，但整篇文档会引入大量噪声。
- embedding 一整篇长文档时，语义会被平均，细节信号会变弱。
- retrieve 很难精确命中真正相关的内容。

所以 chunk 的目标是：

```text
把长文档切成一组小而完整的知识单元，
让每个知识单元都可以独立参与 embedding、检索和 Prompt 拼接。
```

---

## 它的原理

核心概念：

- 文档是原始知识。
- chunk 是知识单元。
- embedding 是 chunk 的语义坐标。
- vector store 保存 chunk 和对应语义坐标。
- retrieve 根据问题找到相关 chunk。

底层机制：

```text
原始文档通常很长
-> 先按一定规则切成多个 chunk
-> 每个 chunk 单独生成 embedding
-> 每个 embedding 和 chunk 原文一起写入 vector store
-> 用户提问时，系统检索的是 chunk，不是整篇文档
```

为什么能解决问题：

```text
因为用户的问题通常只和文档中的一部分内容相关。
chunk 让系统可以只取出相关片段，
而不是把整篇文档都交给模型。
```

和直接把全文塞进 Prompt 的区别：

```text
全文 Prompt：简单，但贵、慢、容易超上下文，也容易引入噪声。
chunk + retrieve：多一步索引，但更精准、更可扩展、更适合大量资料。
```

---

## 它的核心逻辑

```text
document
-> 清洗文本
-> 按规则切分
-> 给每个 chunk 记录元数据
-> 交给 embedding
```

关键节点职责：

| 节点 | 接收什么 | 它做什么 | 输出什么 | 为什么需要它 |
|---|---|---|---|---|
| document | 原始资料 | 提供完整知识来源 | 长文本 | RAG 的知识起点 |
| text cleanup | 文档文本 | 去掉无意义符号、重复页眉页脚、乱码 | 干净文本 | 避免噪声进入检索 |
| chunking rule | 干净文本 | 按标题、段落、长度、语义边界切分 | 多个候选片段 | 控制每个知识单元的大小 |
| chunk | 一段文本片段 | 承载一个相对完整的知识点 | 可 embedding 的文本块 | 后续检索的最小知识单元 |
| metadata | 来源信息 | 记录标题、文档名、页码、章节、位置 | 可追溯信息 | 回答时可以展示引用来源 |

---

## 常见切片方式

### 1. 固定长度切片

按字符数或 token 数切。

```text
每 500 tokens 切一段
```

优点：

- 实现简单。
- 每段长度比较稳定。
- 适合作为早期版本。

缺点：

- 可能把一个完整语义切断。
- 标题和正文可能被拆开。

### 2. 按段落切片

按自然段、空行、Markdown 段落切。

优点：

- 语义比固定长度更自然。
- 对 Markdown、文章、课程笔记比较友好。

缺点：

- 有些段落太长，有些段落太短。
- 需要再做长度合并或拆分。

### 3. 按标题层级切片

按 Markdown 标题、章节、小节切。

优点：

- 保留文档结构。
- 适合技术文档、课程笔记、财报章节。

缺点：

- 如果小节过长，仍然需要二次切分。

### 4. 语义切片

根据语义变化来切，比如一个主题结束后再切。

优点：

- 语义完整性最好。
- 检索质量通常更好。

缺点：

- 实现更复杂。
- 可能需要模型或专门算法辅助。

---

## Chunk 大小怎么理解

chunk 太小：

```text
优点：命中更精准
问题：上下文不足，模型看到片段后可能不知道前因后果
```

chunk 太大：

```text
优点：上下文更完整
问题：检索不够精准，容易把无关内容一起带进 Prompt
```

比较实用的理解：

```text
chunk 不是越小越好，也不是越大越好。
它应该刚好能表达一个完整知识点。
```

举例：

```text
坏 chunk：
“毛利率下降主要因为”

好 chunk：
“毛利率下降通常需要拆分价格、成本、产品结构和产能利用率。如果收入增长但毛利率下降，可能说明规模扩张没有转化为利润质量。”
```

---

## Overlap 是什么

Overlap 指相邻 chunk 之间保留一小段重复内容。

例子：

```text
chunk 1：A B C D
chunk 2：D E F G
```

这里 `D` 就是 overlap。

为什么需要 overlap：

- 防止重要信息刚好被切在边界。
- 让相邻 chunk 保留上下文连续性。
- 提高检索时命中完整语义的概率。

代价：

- 会增加存储量。
- 会增加 embedding 成本。
- 检索结果里可能出现重复片段。

---

## 它的边界

chunk 负责：

- 把长文档变成可检索文本块。
- 尽量保留语义完整性。
- 携带必要的来源元数据。

chunk 不负责：

- 不负责理解用户问题。
- 不负责计算语义相似度。
- 不负责生成回答。
- 不负责判断最终答案是否正确。

上游依赖：

```text
文档解析质量、文本清洗质量、文档结构是否清楚
```

下游影响：

```text
chunk 质量 -> embedding 质量 -> retrieve 命中质量 -> prompt 上下文质量 -> answer 质量
```

这条影响链非常重要：

```text
切片切坏了，后面 embedding、retrieve、answer 都会跟着变差。
```

---

## 在项目里的落点

当前 MiniAINotes 的 RAG v0 还没有真正引入 embedding 和 vector store，而是先用本地知识片段 + 关键词检索。

所以当前项目里的“chunk”可以理解为：

```text
后端内置的一条条财报阅读知识片段
```

对应后端概念：

```text
knowledge snippet
-> source_id
-> title
-> text
-> score
```

后续升级为真正 RAG 时，会变成：

```text
文档解析
-> chunk 切片
-> embedding
-> vector store
-> retrieve topK chunks
-> prompt 注入 retrieved_context
```

iOS 侧关注：

- 不需要知道 chunk 怎么切。
- 只需要展示后端返回的 `retrieved_contexts`。
- 如果有来源、标题、分数，可以展示给用户增强可信度。

后端侧关注：

- chunk 切片策略。
- chunk 元数据设计。
- embedding 生成。
- vector store 写入。
- 检索 topK。

AI 侧关注：

- Prompt 里如何放入 retrieved_context。
- 如何要求模型基于上下文回答。
- 如何避免模型忽略检索内容自由发挥。

---

## 常见错误

错误 1：chunk 太大。

表现：

```text
检索命中了一个大段落，但里面只有一小部分相关，Prompt 里噪声很多。
```

错误 2：chunk 太小。

表现：

```text
检索命中了碎片，但模型缺少上下文，回答容易断章取义。
```

错误 3：没有 metadata。

表现：

```text
回答里无法告诉用户引用来自哪里，也无法调试为什么命中。
```

错误 4：切片破坏语义边界。

表现：

```text
标题和正文分离，定义和解释分离，风险描述只剩半句。
```

错误 5：只看切片数量，不看检索效果。

表现：

```text
知识库看起来很多，但用户一问，retrieve 命中的不是最相关内容。
```

---

## 复习时要会回答

1. 为什么 RAG 不能直接把整篇文档都丢给模型？
2. `chunk` 在 `document -> chunk -> embedding -> vector store` 里负责什么？
3. chunk 太大和太小分别会造成什么问题？
4. 为什么 chunk 要保留 metadata？
5. chunk 质量会怎样影响 embedding、retrieve 和 answer？

---

## 本轮流程

理解 RAG 资料准备链路中的 `chunk` 节点。

## 核心思想

chunk 是 RAG 的知识单元设计。它把长文档拆成可以被 embedding、检索和 Prompt 使用的小块。

## 它在 AI 应用开发流程中的位置

- 上游：document 原始文档
- 当前环节：chunk 切片
- 下游：embedding -> vector store -> retrieve -> prompt -> answer

## 需要关注的技能知识点

- iOS：理解 iOS 展示的是 retrieved_context，不处理切片策略。
- 后端：掌握文本清洗、切片策略、metadata 设计。
- AI 应用：理解 chunk 质量会影响检索上下文，进而影响模型答案。
- 工程化：先用可解释的本地片段验证流程，再升级向量库。

## 下一步建议重点学习

1. embedding：为什么文本可以变成向量。
2. vector store：向量库怎么保存和检索 chunk。
3. retrieve：topK 相似度检索如何决定哪些上下文进入 Prompt。

