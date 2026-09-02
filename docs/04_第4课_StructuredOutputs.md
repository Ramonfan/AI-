# 第 4 课：Structured Outputs

本课目标：把“让模型自己返回 JSON”升级成“后端用 JSON Schema 约束模型输出”。

---

## 1. 为什么需要 Structured Outputs

第 3 课里，我们是这样要求模型的：

```text
请返回严格 JSON，不要输出 Markdown，不要输出解释。
```

这属于 Prompt 约束。它能工作，但不是强保证。

模型仍然可能输出：

```text
下面是总结结果：
```json
{ ... }
```
```

或者：

```json
{
  "title": "...",
  "summary": "...",
  "keyword": []
}
```

注意这里 `keyword` 少了 `s`，iOS 就可能解析失败。

Structured Outputs 的作用是：把后端期望的结构用 JSON Schema 明确交给模型。

---

## 2. 当前目标结构

iOS 和后端约定的响应结构是：

```json
{
  "title": "标题",
  "summary": "摘要",
  "keywords": ["关键词"],
  "todos": ["行动项"]
}
```

后端 Pydantic 模型是：

```python
class SummarizeResponse(BaseModel):
    title: str
    summary: str
    keywords: list[str]
    todos: list[str]
```

第 4 课新增的 JSON Schema，就是把这个结构翻译成模型能理解的格式。

---

## 3. JSON Schema 是什么

JSON Schema 是一份“JSON 结构说明书”。

它告诉模型：

```text
结果必须是 object
必须有 title、summary、keywords、todos
title 和 summary 必须是 string
keywords 和 todos 必须是 string 数组
不能多返回其他字段
```

在代码里对应：

```python
{
    "type": "object",
    "additionalProperties": False,
    "required": ["title", "summary", "keywords", "todos"],
    "properties": {
        "title": {"type": "string"},
        "summary": {"type": "string"},
        "keywords": {
            "type": "array",
            "items": {"type": "string"},
        },
        "todos": {
            "type": "array",
            "items": {"type": "string"},
        },
    },
}
```

---

## 4. 在 Responses API 里怎么用

核心参数是 `text.format`：

```python
text={
    "format": {
        "type": "json_schema",
        "name": "note_summary",
        "strict": True,
        "schema": create_summary_schema(),
    }
}
```

学习理解：

```text
type=json_schema：启用 Structured Outputs
name：给这份输出格式起名
strict=True：要求模型严格遵守 schema
schema：真正的 JSON 结构说明
```

---

## 5. 为什么还要 Pydantic 校验

Structured Outputs 是模型输出层面的约束。

Pydantic 是后端运行时的校验。

两者关系：

```text
JSON Schema：告诉模型应该返回什么
Pydantic：后端收到结果后再检查一次
```

这叫双保险。

尤其是你现在使用的是 OpenAI 兼容网关：

```text
OPENAI_BASE_URL=http://ai-hub.nn.com/v1
```

兼容网关不一定完整支持所有 OpenAI 新参数，所以后端仍然要保留解析和校验。

---

## 6. 本课验收

后端语法检查：

```bash
env PYTHONPYCACHEPREFIX=.python-cache .venv/bin/python -m py_compile app/main.py app/models.py app/config.py app/ai_service.py
```

本地 Mock 分支检查：

```bash
.venv/bin/python - <<'PY'
import asyncio
from app.ai_service import summarize_note
from app.config import Settings

async def main():
    result = await summarize_note(
        "今天学习 Structured Outputs。",
        settings=Settings(openai_api_key="", openai_model="", openai_base_url=""),
    )
    print(result.model_dump_json())

asyncio.run(main())
PY
```

真实模型检查：

```bash
curl -X POST http://127.0.0.1:8000/api/summarize \
  -H "Content-Type: application/json" \
  -d '{"note":"今天学习了 Structured Outputs 和 JSON Schema。"}'
```

如果 AI Hub 返回“不支持 text.format / json_schema / strict”之类错误，说明兼容网关暂时不支持 Structured Outputs，需要降级到旧 JSON 模式或普通 prompt JSON。
