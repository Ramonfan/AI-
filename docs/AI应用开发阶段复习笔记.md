# AI 应用开发阶段复习笔记

这份笔记用于复习 Mini AI Notes 已经学过的 AI 应用开发知识。它不是按代码文件展开，而是按“你以后做一个 AI App 时脑子里应该先出现什么结构”来整理。

配套知识图谱：

```text
docs/ai_application_learning_knowledge_graph.html
```

---

## 0. 一句话总览

我们目前学到的核心是：

```text
AI 应用不是客户端直接问模型，
而是客户端提出业务请求，
后端拿到可信数据，
再把事实、上下文、规则和 Prompt 组织好，
交给模型生成结果，
最后用稳定的数据契约返回给客户端展示。
```

更短一点：

```text
业务问题 -> 客户端 -> 后端 -> 数据 Provider -> 数据契约 -> AI 编排 -> 模型 -> 流式/结构化返回 -> 客户端状态展示
```

---

## 1. 先做 App 状态，再接 AI

AI App 首先是一个 App。第 1、2 课的重点不是 AI，而是把用户体验跑通。

你要先能回答：

- 用户输入在哪里？
- 请求中怎么显示 loading？
- 成功结果怎么展示？
- 失败结果怎么提示？
- Mock 数据能不能替代真实服务跑完整流程？

这一阶段学到的基本模型：

```text
View
-> ViewModel
-> Service 协议
-> Mock Service
-> Result State
-> SwiftUI 刷新
```

复习时重点看：

- SwiftUI 状态绑定
- ViewModel 如何隔离业务状态
- 为什么先 Mock 后真实 API

---

## 2. iOS 不直接调用模型 API

第 3 课开始进入真实 AI 应用架构。

最重要的安全原则：

```text
iOS 客户端不保存 OpenAI API Key。
```

正确链路：

```text
iOS
-> 自己的 FastAPI 后端
-> 后端读取 .env 里的 OPENAI_API_KEY
-> 后端调用 OpenAI
-> 后端把结果返回给 iOS
```

为什么这么做：

- API Key 放 iOS 会被逆向或抓包拿到。
- 后端可以统一做鉴权、限流、日志、降级。
- 后端可以替换模型供应商，iOS 不需要频繁改。

复习时重点看：

- Backend Proxy 是什么
- `.env` / 环境变量的作用
- ChatGPT Plus 和 OpenAI API 额度是分开的

---

## 3. Prompt 不是一句话，而是输入协议

Prompt 不只是“让模型总结一下”。

在工程里，Prompt 更像一份输入协议：

```text
instructions：告诉模型扮演什么角色、遵守什么规则
input：把用户输入或业务事实交给模型
schema：约束模型输出结构
max_output_tokens：限制输出长度
timeout：控制请求等待时间
```

你要记住：

```text
Prompt 的目标不是让模型自由发挥，
而是让模型在业务边界内稳定输出。
```

复习时重点看：

- `instructions` 和 `input` 的区别
- 为什么要限制输出长度
- 为什么 AI 层失败不能拖垮整个业务

---

## 4. Structured Outputs：让 AI 结果变成稳定数据

第 4、5 课的核心是结构化输出。

没有结构化输出时：

```text
模型返回一段自然语言
-> iOS 很难稳定解析
-> UI 不知道标题、摘要、关键词在哪里
```

有结构化输出时：

```text
OpenAI JSON Schema
-> 后端 Pydantic 校验
-> JSON Response
-> iOS Codable 解码
-> SwiftUI 分区展示
```

你要记住这条链：

```text
Schema 是 AI 和工程系统之间的契约。
```

复习时重点看：

- JSON Schema 约束模型输出
- Pydantic 负责后端校验
- Codable 负责 iOS 解码
- 字段新增时客户端如何兼容

---

## 5. 财报 AI 应用：先拿事实，再让 AI 解读

第 6、7、8 课把项目从“笔记总结”升级为“财报分析”。

财报分析不能直接让模型猜：

```text
用户输入公司名
-> 市场识别
-> 找到对应数据源
-> 提取结构化财报事实
-> AI 根据事实做中文解读
```

这里形成了 Provider 架构：

```text
Global Router
-> SEC EDGAR Provider
-> A 股 Provider
-> 后续可扩展港股 / 同花顺 / 其他数据源
```

Provider 的职责：

- 识别公司、ticker、市场。
- 访问真实或可替代的数据源。
- 返回统一结构的财报事实。
- 数据源失败时给出可读原因。

AI 的职责：

- 不负责凭空编造指标。
- 只解释 Provider 已经拿到的事实。
- 在失败时允许后端返回规则降级结果。

复习时重点看：

- Provider 和 AI 分工
- 为什么 iOS 不关心 SEC / 东方财富 / 巨潮细节
- 为什么统一响应模型很重要

---

## 6. SSE 流式输出：把长等待变成渐进结果

第 9 课的核心是流式输出。

普通接口：

```text
请求 -> 等待 -> 一次性返回完整 JSON
```

SSE 流式接口：

```text
请求 -> status -> provider_result -> delta -> delta -> done
```

我们定义过的事件：

```text
status：告诉客户端当前阶段
provider_result：先把结构化财报事实发给客户端
delta：AI 生成的一段文本
done：明确告诉客户端流程完整结束
```

客户端处理流时，最重要的是不要把“已经收到的有效结果”丢掉：

```text
没有收到 provider_result 就失败 -> failure
收到 provider_result 后 AI 中断 -> partial
收到 done -> success
```

复习时重点看：

- SSE 和普通 HTTP 的区别
- 为什么需要 `done`
- 为什么需要 heartbeat
- 流式失败时怎么保留 partial state

---

## 7. 错误诊断和降级：AI 应用必须可观测

我们遇到过几类真实错误。

客户端网络超时：

```text
NSURLErrorDomain Code=-1001
The request timed out.
```

常见原因：

- 请求建立不了。
- 服务器没有持续返回字节。
- SSE 中间长时间静默。
- 手机和电脑不在可互通网段。

模型参数错误：

```text
Unsupported parameter: temperature
```

含义：

```text
某些模型不支持 temperature，需要后端按模型能力过滤参数。
```

API 额度错误：

```text
You have no credits remaining.
```

含义：

```text
ChatGPT Plus 订阅不等于 OpenAI API 额度。
```

后端降级原则：

```text
Provider 成功，AI 失败
-> 不应该整体失败
-> 返回 Provider 事实 + 规则摘要 / fallback summary
```

复习时重点看：

- iOS 打印 AFError / URLError / NSError 的完整原因
- 后端区分 Provider 错误和 AI 错误
- 降级结果如何通过 SSE 继续返回

---

## 8. RAG：下一阶段要掌握的核心概念

RAG 的全称是 Retrieval-Augmented Generation，检索增强生成。

它解决的问题：

```text
模型不知道你的私有知识、最新知识、业务规则，
直接回答容易幻觉。
```

不用 RAG：

```text
P(answer | question)
```

意思是：模型只根据问题和自身参数猜答案。

使用 RAG：

```text
P(answer | question + retrieved_context)
```

意思是：先检索相关资料，再让模型基于资料回答。

RAG 主干分成两条链路：

```text
资料准备：文档 -> chunk -> embedding -> vector store
用户提问：question -> embedding -> retrieve -> prompt -> answer
```

资料准备链路是离线过程：

```text
文档
-> 切成 chunk
-> 每个 chunk 生成 embedding
-> 把 chunk、embedding、来源元数据写入 vector store
```

它解决的是“知识怎么进入系统，并变成可检索资产”。

用户提问链路是在线过程：

```text
用户问题 question
-> 问题生成 embedding
-> 从 vector store 检索相关 chunk
-> 把 question + retrieved_context 拼进 prompt
-> 模型生成 answer
```

它解决的是“回答时怎么把相关知识拿出来，并放进模型上下文”。

在我们的财报应用里，要区分三件事：

```text
Provider 事实：这家公司发生了什么
RAG 知识：这些事实应该怎么理解
AI 生成：把事实和知识组织成中文分析
```

复习时重点看：

- RAG 为什么不是“让 AI 联网搜索”
- 资料准备链路和用户提问链路分别解决什么
- Chunk / Embedding / Vector Store / Retrieve / Prompt 分别做什么
- 为什么要把引用上下文展示给用户

---

## 9. 工程化规则：我们怎么推进这个项目

这个项目不是随手写 Demo，而是在训练真实 AI App 的工程节奏。

固定规则：

```text
OpenSpec First
Mock First
Small Step
Verify Always
Chinese Notes
Safe Keys
Python Backend
Moya Network
```

每做一个新流程，都要问：

- 规格写清楚了吗？
- 有 Mock 路径吗？
- 失败状态处理了吗？
- 后端和 iOS 的数据契约一致吗？
- 验证记录写了吗？
- 学习要点总结了吗？

---

## 10. 最适合你的复习方式

建议用“三轮复习法”。

第一轮：看知识图谱

```text
打开 ai_application_learning_knowledge_graph.html
只看节点和箭头
先能复述完整链路
```

目标是说出：

```text
业务问题为什么要经过后端、Provider、契约、AI 编排，再回到 iOS。
```

第二轮：按问题复习

每个主题只问自己一个问题：

- 为什么 iOS 不能直接调 OpenAI？
- 为什么 AI 输出要结构化？
- 为什么财报分析要先 Provider 后 AI？
- SSE 为什么需要 `done`？
- 流中断时为什么要保留 partial？
- RAG 和 Provider 有什么区别？

第三轮：回到代码

每个知识点都找一个对应代码位置：

```text
iOS ViewModel：状态如何变化
iOS Service：网络错误如何转换
FastAPI route：接口如何组织
Provider：数据如何进入统一模型
AI Service：Prompt 如何构建
Stream Service：SSE 如何发事件
RAG Service：上下文如何检索
```

能从代码里指出“这个知识点落在哪里”，才算真正掌握。

---

## 11. 当前学习状态自测

如果下面这些问题能讲清楚，说明当前阶段很扎实：

- AI App 为什么不是“前端 + ChatGPT API”？
- Backend Proxy 解决了哪些问题？
- Structured Outputs 为什么能降低前后端联调成本？
- Pydantic 和 Codable 在同一条链路里分别负责什么？
- Provider 架构为什么适合财报、多市场、多数据源？
- SSE 里 `status`、`provider_result`、`delta`、`done` 分别有什么意义？
- `-1001 timeout` 在流式应用里通常说明什么？
- OpenAI API 额度和 ChatGPT Plus 为什么是两套体系？
- RAG 的概率表达为什么是 `P(answer | question + context)`？
- Provider 事实、RAG 知识、AI 生成三者怎么分工？

---

## 12. 下一步学习建议

下一步不要急着上复杂向量库，先把 RAG 的概念讲透。

建议顺序：

1. 先学 RAG 是什么、为什么能减少幻觉。
2. 再学 RAG 的概率视角和标准流程。
3. 最后做一个 RAG v0：本地知识片段 + 关键词检索 + topK 上下文 + iOS 展示引用。

---

## 13. 每轮学习后的固定总结模板

以后每完成一个流程，都按这个格式复盘：

```text
本轮流程：

核心思想：

需要关注的技能知识点：
- iOS：
- 后端：
- AI 应用：
- 工程化：

下一步建议重点学习：
1.
2.
3.
```

这份模板的作用是：每次不仅知道“做了什么”，还知道“这个流程在 AI 应用能力树上对应哪块肌肉”。
