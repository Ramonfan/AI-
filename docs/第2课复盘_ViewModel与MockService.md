# 第 2 课复盘：ViewModel 与 Mock Service

## 本课完成了什么

把第 1 课集中在 `ContentView.swift` 里的模型、状态和 Mock 总结逻辑拆成了更清楚的结构：

```text
ContentView
→ SummaryViewModel
→ SummaryService
→ MockSummaryService
→ NoteSummary
```

## 关键文件

- `Models/NoteSummary.swift`：总结结果的数据模型。
- `Models/SummaryState.swift`：页面状态，包括 idle、loading、success、failure。
- `Services/SummaryService.swift`：总结服务协议。
- `Services/MockSummaryService.swift`：本地 Mock 实现。
- `ViewModels/SummaryViewModel.swift`：管理输入、状态和服务调用。
- `ContentView.swift`：只负责展示和触发用户动作。

## 学到的核心点

1. View 不应该直接生成 Mock 数据。
2. ViewModel 适合管理 loading、success、failure 这类页面状态。
3. Service 协议能隔离 Mock 实现和后续真实后端实现。
4. 先 Mock 再接真实 API，可以减少网络、密钥、模型成本带来的干扰。

## 正常路径

```text
用户输入笔记
→ 点击总结
→ ContentView 调用 viewModel.summarize()
→ SummaryViewModel 设置 loading
→ MockSummaryService 返回 NoteSummary
→ SummaryViewModel 设置 success
→ ContentView 展示结构化结果
```

## 失败路径

Mock 服务支持模拟失败：

```text
输入内容包含“失败”
→ MockSummaryService 抛出 simulatedFailure
→ SummaryViewModel 设置 failure
→ ContentView 展示错误状态
```

## Context7 记录

本课涉及 SwiftUI、ObservableObject、@Published、async/await、Task.sleep 等 API。当前会话中 Context7 的 `resolve-library-id/query-docs` 工具没有暴露出来，所以按照项目规则记录原因，并使用 Apple 官方稳定 API 完成实现。

## 验证结果

已执行 generic iOS build：

```text
xcodebuild -project MiniAINotes.xcodeproj -scheme MiniAINotes -destination generic/platform=iOS -derivedDataPath /tmp/MiniAINotesDerivedData CODE_SIGNING_ALLOWED=NO build
```

结果：

```text
BUILD SUCCEEDED
```

