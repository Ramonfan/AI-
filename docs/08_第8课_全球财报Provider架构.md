# 第 8 课：全球财报 Provider 架构

## 为什么北京君正会报 SEC 错误

`北京君正` 是 A 股公司，股票代码 `300223`。上一版后端只有 SEC EDGAR 一个数据源，所以它会去 SEC 公司列表里找北京君正，当然找不到。

这不是用户输入错了，而是产品能力边界错了：我们把“SEC 财报分析”误当成了“全球财报分析”。

## 正确的后端结构

全球财报查询应该是 Provider 架构：

```text
iOS
-> FastAPI /api/financial-report/analyze
-> GlobalFinancialReportService
-> CompanyRoute 市场识别
-> SECProvider / CNINFOProvider / HKEXProvider / EDINETProvider / ...
-> 统一 FinancialReportAnalysisResponse
-> iOS 展示
```

这样 iOS 不需要知道公司来自哪个市场，它只关心统一的结构化结果。

## 本课先支持哪些市场

当前第一版：

- 美股 / SEC 披露主体：SEC EDGAR
- A 股第一版：东方财富 A 股列表自动映射公司名，东方财富财务摘要返回指标，巨潮资讯 CNINFO 公告兜底
- 常见中文别名：先映射成对应 ticker 再进入 Provider
- 港股 / 日股 / 英股：现在先识别出来并返回“暂未接入 Provider”的清晰说明

现在全局入口不再盲目“先查 SEC，失败再查 A 股”。新的路由流程是：

```text
输入公司名 / 股票代码
-> 内置别名映射
-> 识别 A 股代码、港股代码、市场后缀、中文公司名、英文 ticker
-> 得到 CompanyRoute
-> 只调用匹配市场的 Provider
-> 无法判断时才按已支持 Provider 顺序兜底尝试
```

现在 A 股不再依赖手动映射。查询流程是：

```text
输入公司名
-> 本地兜底映射
-> 本地 A 股列表缓存
-> 东方财富 A 股列表
-> 匹配公司名 / 股票代码
-> 得到证券代码
-> 东方财富财务摘要
-> 统一返回给 iOS
```

东方财富接口偶尔会直接断开连接。后端处理策略是：

```text
1. 先查本地兜底映射
2. 再查本地缓存的 A 股列表
3. 只有缓存未命中时才请求东方财富
4. 远端请求最多重试 2 次
5. 远端成功后刷新本地缓存
6. 远端失败时返回可读错误，不让 FastAPI 抛 ASGI 500
```

内置映射只保留为兜底，例如：

```text
公司名：北京君正
股票代码：300223
优先数据源：东方财富财务摘要
兜底数据源：巨潮资讯 CNINFO
```

`盛天网络` 已加入 A 股公司映射：

```text
公司名：盛天网络
股票代码：300494
优先数据源：东方财富财务摘要
兜底数据源：巨潮资讯 CNINFO
```

`英伟达` 会先映射为：

```text
中文输入：英伟达
标准查询：NVDA
数据源：SEC EDGAR
```

`腾讯` / `700` / `00700.HK` 现在会先识别为：

```text
市场：港股 / HKEX
状态：暂未接入 Provider
结果：返回清晰说明，不再误查 SEC
```

## A 股现在返回什么

SEC 的 companyfacts 已经提供结构化 XBRL JSON，所以可以直接提取 Revenue、Net Income 等指标。

A 股第一版现在先读取东方财富财务摘要，返回：

```text
营业总收入
归母净利润
扣非净利润
基本每股收益
加权净资产收益率
每股净资产
```

如果东方财富请求失败，再退回巨潮公告定位：

```text
公司名
-> 股票代码 / orgId
-> 巨潮公告列表
-> 最新年度/季度报告公告
-> 返回公告标题和 PDF 地址
```

下一步再做：

```text
下载 PDF
-> 提取文本 / 表格
-> 结构化利润表、资产负债表、现金流量表
-> 交给 AI 做中文分析
```

## 阶段成果怎么检验

### 1. 后端语法检查

```bash
cd backend
env PYTHONPYCACHEPREFIX=.python-cache .venv/bin/python -m py_compile app/config.py app/main.py app/models.py app/ai_service.py app/sec_edgar_service.py app/cninfo_service.py app/global_financial_report_service.py
```

### 2. 本地接口验证

启动后端：

```bash
cd backend
.venv/bin/python -m app.server
```

iOS 输入：

```text
北京君正
300223
盛天网络
300494
AAPL
Apple
英伟达
腾讯
700
00700.HK
```

预期：

- `AAPL` / `Apple` 走 SEC Provider
- `英伟达` / `NVDA` 走 SEC Provider
- `北京君正` / `300223` 走 CNINFO Provider
- `盛天网络` / `300494` 走 A 股 Provider
- `腾讯` / `700` / `00700.HK` 识别为港股 HKEX 暂未接入，不再误走 SEC

离线路由验证已经覆盖：

```text
英伟达 -> SEC / NVDA
AAPL -> SEC / AAPL
Apple -> SEC / Apple
北京君正 -> A 股 / 北京君正
300494 -> A 股 / 300494
腾讯 -> 港股 HKEX 暂未接入
700 -> 港股 HKEX 暂未接入
00700.HK -> 港股 HKEX 暂未接入
```

## 当前限制

这次网络审批没有允许我访问东方财富和巨潮接口，所以 A 股真实链路还没有在 Codex 环境里完成外网验证。代码已经按 Provider 架构落地，下一步需要在你本机网络下启动后端做真实联调。

另外，本次真实 SEC 网络联调也被执行环境审批拒绝；但已经用离线 SEC 替身验证：

```text
输入：英伟达
别名解析：NVDA
Provider：SEC EDGAR
接口结果：NVIDIA CORP / NVDA
```
