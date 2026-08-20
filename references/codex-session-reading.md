# Codex 对话读取技术

> 当用户问"你能读取我跟codex之间的对话内容吗"或提到Codex话题名时使用。

## 存储结构

Codex会话文件存储在 `~/.codex/sessions/YYYY/MM/DD/` 目录下，文件名格式：
```
rollout-YYYY-MM-DDTHH-MM-SS-<uuid>.jsonl
```

话题索引在 `~/.codex/session_index.jsonl`（JSONL格式，每行一个对象）：
```json
{"id": "<uuid>", "thread_name": "话题名", "updated_at": "<ISO时间>"}
```

## 定位特定话题

1. 读取 `~/.codex/session_index.jsonl`
2. 按关键词搜索 `thread_name` 字段
3. 获取对应的 session `id`
4. 用 `id` 匹配 sessions 目录下的文件名

注意：`session_index.jsonl` 中同一个id可能出现多次（话题被重命名时），取最新 `updated_at` 的条目。

## JSONL文件解析

每个 rollout 文件是 JSONL 格式，每行一个JSON对象。关键结构：

```
type=session_meta   → 首行，含payload.base_instructions等元数据
type=response_item  → 实际消息，payload.type区分类型
type=event_msg      → 推理过程、token计数等，通常不需要
```

**提取对话消息的Python代码模板**：

```python
import json

filepath = "/Users/hui/.codex/sessions/YYYY/MM/DD/rollout-*.jsonl"
messages = []
with open(filepath, "r") as f:
    for line_num, line in enumerate(f, 1):
        line = line.strip()
        if not line:
            continue
        try:
            obj = json.loads(line)
            t = obj.get("type", "")
            payload = obj.get("payload", {})
            if t == "response_item" and isinstance(payload, dict):
                ptype = payload.get("type", "")
                if ptype == "message":
                    role = payload.get("role", "")
                    content = payload.get("content", [])
                    text = ""
                    if isinstance(content, list):
                        for c in content:
                            if isinstance(c, dict):
                                text += c.get("text", "")
                    elif isinstance(content, str):
                        text = content
                    # 过滤系统消息
                    if role in ("user", "assistant") and text and len(text) > 30:
                        if text.startswith("<permissions") or text.startswith("<recommended_plugins") or text.startswith("<environment"):
                            continue
                        messages.append({"role": role, "text": text, "line": line_num})
        except:
            pass
```

## 注意事项

- session_meta行（第1行）非常大（45KB+），包含base_instructions、dynamic_tools等系统消息，**不是用户输入**
- role="developer" 的消息是系统权限指令，**不是用户消息**
- role="user" 的第一条消息可能包含AGENTS.md内容+环境上下文，需用长度过滤（>30字符且不以`<`开头）
- 话题可能在多天有多个文件（同一session id跨日期续聊），需检查 `session_index.jsonl` 的 `updated_at` 判断是否有续聊
- 用Python `open()` 直接读取比 `terminal cat` 更可靠（terminal可能有输出截断）
- 不要用 `grep 'role.*user'` 搜索——JSONL每行是完整JSON对象，grep无法区分系统消息和真实用户输入
