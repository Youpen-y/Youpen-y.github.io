---
title: "Webhook"
date: 2026-05-27T22:03:24+08:00
draft: false
tags: ["programming"]
comments: true
---
最近一直对"钩子"比较感兴趣，因为这种技术就像是"传感器"，可以在环境状态变化时迅速通知"大脑"，引导"大脑"做出进一步的反应。随着 AI 智能化水平不断提升，钩子在 AI 领域的应用将会越来越广泛：一方面，它可以让人类在 AI 执行关键操作时及时收到通知；另一方面，在多智能体场景下，智能体之间可以通过钩子实现事件驱动的协同工作。

## 从本地到网络

Git Hooks 在命令执行的特定时刻触发特定的脚本，本质上是本地事件驱动；而今天要说的 Webhook 则把同样的思想搬到了网络上，通过 `HTTP POST` 在事件发生时通知远端服务，实现跨系统的事件驱动。从本地事件驱动的趋势来看，未来每个应用都可能会内置钩子机制，以便在关键事件发生时及时唤起 AI 介入。比如此前我在 AI 帮助下开发了 [ftrigger](https://github.com/Youpen-y/ftrigger)，一个用于监测文件系统变化的守护程序，在新增或修改文件时触发 AI 执行预定义的操作，和 Webhook 其实是一类思想。

## Webhook 起源

根据维基百科的说法，Webhook 名词由 Jeff Lindsay 创建，我找到了相应的文章如下（非常值得一读）。

{{< embed url="https://web.archive.org/web/20180630220036/http://progrium.com/blog/2007/05/03/web-hooks-to-revolutionize-the-web/" title="Web Hooks to Revolutionize the Web" >}}

从中可知：Webhook 的提出是为了解决轮询（Polling）问题的。传统模式下，客户端想知道服务器有没有新数据，只能不断去问"有了吗？"，这种方式既低效又浪费资源。Webhook 则是在服务器发生事件时主动推送给客户端，客户端只需要坐等通知。

### 工作原理

本质上，Webhook 就是用户自定义的 HTTP POST 回调。要让它工作起来，需要两个角色的配合：

{{< mermaid >}}
sequenceDiagram
    participant C as 客户端
    participant S as 服务器
    C->>S: 1. 注册回调 URL（Payload URL）
    Note over S: 事件等待中...
    S-->>S: 2. 事件触发（如：push 代码）
    S->>C: 3. HTTP POST 事件数据（JSON）
    C-->>C: 4. 处理数据，执行相应逻辑
    C->>S: 5. 返回 HTTP 200 确认
{{< /mermaid >}}

- **客户端**：告诉服务器"当某件事发生时，请往这个 URL 发一个 POST 请求"，这个 URL 通常称为 **Payload URL**。
- **服务器**：在对应事件发生时，将事件数据（通常是 JSON 格式）POST 到客户端指定的 URL。

### 示例

一个典型的 Webhook 流程如下：

1. 打开 [webhook.site](https://webhook.site/)，页面会自动分配一个唯一的 URL，如 `https://webhook.site/xxxxx`。
2. 用 `curl` 模拟一个 Webhook 请求：

```bash
curl -X POST https://webhook.site/xxxxx \
  -H "Content-Type: application/json" \
  -d '{"event": "push", "message": "Hello, Webhook!"}'
```

3. 回到 webhook.site 页面，就能实时看到收到的请求详情（Headers，Body 等）

上面只是一个简单的示例，实际上客户端在收到服务器数据后，可对数据做进一步操作。

```python
# 一个简单的 Webhook 接收服务
from http.server import HTTPServer, BaseHTTPRequestHandler

class WebhookHandler(BaseHTTPRequestHandler):
    def do_POST(self):
        content_length = int(self.headers['Content-Length'])
        body = self.rfile.read(content_length)
        print(f"Received webhook: {body.decode()}")
        self.send_response(200)
        self.end_headers()

HTTPServer(('0.0.0.0', 8080), WebhookHandler).serve_forever()
```

本地运行后，`http://localhost:8080` 就是一个 Webhook 端点，可以通过 `curl` 验证。
```bash
curl -X POST http://localhost:8080 \
  -H "Content-Type: application/json" \
  -d '{"event": "push", "message": "Hey, something happened"}'
```


也可以用 [ngrok](https://ngrok.com/) 将其暴露到公网：

```bash
ngrok http 8080
```

ngrok 会分配一个公网 URL（如 `https://xxxx.ngrok-free.app`），将它配置为 Payload URL，交给服务器，这样外部服务器的事件源就能把 Webhook 推送到你的本地服务了。以 GitHub Webhook 为例，你可以在仓库设置中配置一个 Payload URL（如 Discord 服务器的 `Webhook URL/github`），这样当你 push 代码、创建 Issue 或提交 Pull Request 时，Discord 频道就会自动收到一条包含事件详情的通知。当然，你也可以指向自己的服务，比如，在上述 `do_POST` 函数中实现自动触发 CI/CD 流水线、更新看板、甚至唤起 AI 处理等更复杂的逻辑。