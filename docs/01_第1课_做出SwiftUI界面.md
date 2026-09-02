# 第 1 课：做出 SwiftUI 界面

本课目标：先不接 AI，不写后端，只做出 Mini AI Notes 的静态界面和基本交互。

---

## 你要做出来的界面

第一版页面包含：

```text
标题：Mini AI Notes
输入区：用户输入一段笔记
按钮：总结
结果区：标题、摘要、关键词、待办事项
状态：空状态、loading、错误状态
```

---

## 为什么先做 UI

AI 应用也首先是一个 App。

如果 UI 状态不清楚，后面接入模型后会更乱：

- 请求中怎么显示？
- 请求失败怎么显示？
- 没输入内容能不能点按钮？
- 返回结构化结果后怎么展示？
- 长文本会不会撑爆页面？

所以第 1 课先把产品形态固定下来。

---

## 建议 SwiftUI 模型

```swift
struct NoteSummary {
    var title: String
    var summary: String
    var keywords: [String]
    var todos: [String]
}
```

页面状态：

```swift
enum SummaryState {
    case idle
    case loading
    case success(NoteSummary)
    case failure(String)
}
```

---

## 建议页面拆分

```text
MiniAINotesApp
ContentView
NoteInputView
SummaryResultView
KeywordChipsView
TodoListView
```

第 1 课可以先全部写在 `ContentView.swift`，等功能跑通后再拆组件。

---

## 第 1 课任务

### 任务 1：创建 iOS 项目

用 Xcode 创建一个 SwiftUI App：

```text
Product Name: MiniAINotes
Interface: SwiftUI
Language: Swift
```

建议放在：

```text
AI-Learning-Labs/MiniAINotes/ios/MiniAINotes
```

### 任务 2：实现基础页面

页面需要有：

- 标题
- 多行输入框
- 总结按钮
- 结果区域

### 任务 3：加入 Mock 数据

点击“总结”按钮后，先不要请求网络，直接展示固定结果：

```text
标题：AI 应用学习计划
摘要：这段笔记主要讨论了 iOS 工程师如何通过小应用学习 AI 开发。
关键词：iOS、AI、SwiftUI、Prompt
待办：
1. 创建 SwiftUI 项目
2. 完成输入页面
3. 接入 Mock 总结结果
```

### 任务 4：加入基础状态

至少支持：

- 输入为空时按钮不可点击
- 点击后短暂 loading
- loading 完成后展示结果

---

## 验收清单

- [ ] App 能运行
- [ ] 可以输入一段笔记
- [ ] 输入为空时不能点击总结
- [ ] 点击总结后出现 loading
- [ ] loading 后展示 Mock 总结结果
- [ ] 结果包含标题、摘要、关键词、待办
- [ ] 页面在小屏幕上不明显拥挤

---

## 本课复盘问题

用中文回答：

1. 为什么 AI App 要先设计状态，而不是先接 API？
2. `NoteSummary` 为什么要用结构化模型？
3. `SummaryState` 比多个 `Bool` 状态更好吗？为什么？
4. 后面接入真实 AI 时，第 1 课的哪些代码可以保留？

