---
title: "JSON Lines"
date: 2026-05-30T10:35:23+08:00
draft: false
tags: []
comments: true
---
JSON Lines（JSONL/jsonl）是一种文件格式，用于存储结构化数据。常用于日志、流数据、会话数据集等。文件中每行包含一个有效的 JSON 值，使用换行符分隔，便于逐行处理。下面展示了以 JSONL 格式存储的 pi 会话示例。
```jsonl
{"type":"session","version":3,"id":"019e0265-4f6b-7086-b25f-161cbe6c7cb6","timestamp":"2026-05-07T12:24:14.444Z","cwd":"/home/yongy/Projects/roundshot"}
{"type":"model_change","id":"86c1c663","parentId":null,"timestamp":"2026-05-07T12:24:15.195Z","provider":"zai","modelId":"glm-5.1"}
{"type":"thinking_level_change","id":"4bddda8e","parentId":"86c1c663","timestamp":"2026-05-07T12:24:15.195Z","thinkingLevel":"medium"}
{"type":"message","id":"b1d0b47c","parentId":"4bddda8e","timestamp":"2026-05-07T12:24:20.043Z","message":{"role":"user","content":[{"type":"text","text":"探索项目"}],"timestamp":1778156660041}}
```
### 结构
JSONL 格式文件要求：
1. UTF-8 编码
2. 每行都是有效的 JSON 值（对象/数组/null/...，但空行无效）
3. 换行符（Linux/MacOS:`\n`，Windows:`\r\n`）

处理示例：
```python
import json

with open("data.jsonl", "r", encoding="utf-8") as f:
    for line in f:
        obj = json.loads(line)

        process(obj)    # 处理单行记录
```

### 优点
- 易于解析：可独立处理每一行。
- 内存高效：可单行/批量处理记录，无需加载整个文件内容。
- 支持流媒体传输：适合实时数据处理和日志分析。
- 人类友好：可读性比较好。

{{< tips info "使用提示" >}}
1. 文件的最后一行 JSON 值后也加上换行符，保持一致性，简化文件处理。
2. 可使用 `gzip` 或 `bzip2` 流压缩器压缩 JSONL 文件以节省空间。读取时可直接流式解压读取而无需解压至磁盘。
```python
import gzip
import orjson

with gzip.open("logs.jsonl.gz", "rb") as f:
    for line in f:
        obj = orjson.loads(line)
        process(obj)
```
{{< /tips >}}