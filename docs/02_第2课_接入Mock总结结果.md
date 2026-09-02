# 第 2 课：接入 Mock 总结结果

本课目标：把“点击按钮后展示固定结果”升级成一个更接近真实 AI 调用的本地流程。

---

## 本课重点

你要先学会设计 iOS 侧的 AI 调用边界。

虽然第 2 课仍然不请求网络，但代码结构要模拟真实调用：

```text
View
→ ViewModel
→ SummaryService
→ MockSummaryService
→ NoteSummary
```

这样第 3 课接后端时，只需要把 Mock Service 换成真实网络 Service。

---

## 建议协议

```swift
protocol SummaryService {
    func summarize(note: String) async throws -> NoteSummary
}
```

Mock 实现：

```swift
struct MockSummaryService: SummaryService {
    func summarize(note: String) async throws -> NoteSummary {
        try await Task.sleep(nanoseconds: 800_000_000)
        return NoteSummary(
            title: "AI 应用学习计划",
            summary: "这段笔记主要讨论了 iOS 工程师如何通过小应用学习 AI 开发。",
            keywords: ["iOS", "AI", "SwiftUI", "Prompt"],
            todos: ["创建 SwiftUI 项目", "完成输入页面", "接入 Mock 总结结果"]
        )
    }
}
```

---

## 本课任务

1. 创建 `SummaryService` 协议
2. 创建 `MockSummaryService`
3. 创建 `SummaryViewModel`
4. 把 loading / success / failure 状态放到 ViewModel
5. 让 View 只负责展示和触发 action

---

## 验收清单

- [ ] View 不直接生成 Mock 数据
- [ ] Mock 数据来自 `MockSummaryService`
- [ ] ViewModel 管理状态
- [ ] 可以模拟 loading
- [ ] 可以模拟失败状态
- [ ] 代码结构方便替换为真实网络服务

