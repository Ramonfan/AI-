# 第 3 课：后端 Proxy 与 AI 调用

本课目标：搭建一个最小后端，让 iOS 通过后端调用模型。

---

## 为什么需要后端

iOS 客户端不应该直接保存模型 API Key。

原因：

- App 包可以被逆向
- 用户可以绕过客户端限制
- 成本会失控
- 无法做统一日志和限流

正确链路：

```text
iOS App
→ 你的后端 /api/summarize
→ OpenAI Responses API
→ 后端解析结果
→ 返回结构化 JSON 给 iOS
```

---

## 第一版接口

```http
POST /api/summarize
```

请求：

```json
{
  "note": "今天学习了 Prompt、结构化输出和 SwiftUI 状态管理..."
}
```

响应：

```json
{
  "title": "AI 学习笔记",
  "summary": "这段笔记总结了 AI 应用开发的基础概念。",
  "keywords": ["Prompt", "SwiftUI", "结构化输出"],
  "todos": ["复习 Prompt", "完成 SwiftUI 页面"]
}
```

---

## 本课任务

1. 创建后端项目
2. 实现 `/api/health`
3. 实现 `/api/summarize`
4. 先返回 Mock JSON
5. 再接 OpenAI Responses API
6. iOS 把 `MockSummaryService` 替换成 `RemoteSummaryService`

---

## 本课实现分层

```text
app/main.py
```

负责 FastAPI 路由。它只处理 HTTP 请求、输入校验和错误转换。

```text
app/config.py
```

负责读取 `.env` 中的后端配置。

```text
app/ai_service.py
```

负责 AI 业务逻辑：没有 Key 时返回 Mock，有 Key 时调用 OpenAI Responses API。

这一层分开后，后续升级 Structured Outputs、RAG、流式输出时，不需要把所有逻辑塞进路由函数里。

---

## 验收清单

- [ ] 后端能启动
- [ ] `/api/health` 正常返回
- [ ] `/api/summarize` 能返回 JSON
- [ ] iOS 不保存模型 API Key
- [ ] iOS 能调用后端并展示结果
- [ ] 后端请求失败时 iOS 能展示错误
- [ ] 填入真实 `OPENAI_API_KEY` 后能返回模型生成结果
