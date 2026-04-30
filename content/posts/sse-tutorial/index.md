---
title: "SSE 教程"
date: 2026-04-30T20:08:20+08:00
draft: false
tags: ["SSE"]
comments: true
---
<center>
    <img src="./cover.png" alt="sse tutorial" title="sse tutorial" style="width: 100%;">
</center>

### 一、SSE 的本质
严格来说，HTTP 协议无法做到服务器主动推送信息（标准 HTTP 通信由客户端发起请求，服务器被动返回响应，服务器无法主动向客户端发送数据。）。 

为了让服务器“主动”发消息，HTTP 提供了流式传输，服务器通过设置响应头 `Content-Type: text/event-stream` 声明接下来是流信息，客户端接收到该头后，会保持连接并监听连接。

SSE（Server-Sent Events） 基于上述机制，使用流信息向客户端推送信息。

### 二、客户端 API
#### EventSource 对象
浏览器客户端通过内置的 `EventSource` 接口使用 SSE，浏览器首先生成一个 `EventSource` 对象，向服务器发起连接。
```js
// 同域或跨域均可
const source = new EventSource(url);

// 跨域时如需携带 Cookie
const source = new EventSource(url, { withCredentials: true });

// 连接状态检查
if (source.readyState === EventSource.OPEN) {
    console.log("连接已建立");
}
```
`url`可以是同域地址，也可以是跨域地址。跨域时通过第二个参数 `{ withCredentials: true }` 控制是否发送 Cookie 。

**重要属性**
- `EventSource`对象的`readyState`属性只读，反映当前连接状态。

| 值 | 常量 | 含义 |
|---|---|---|
| 0 | `EventSource.CONNECTING` | 正在连接，或断线后尝试重连 |
| 1 | `EventSource.OPEN` | 已连接，可以接收数据 |
| 2 | `EventSource.CLOSED` | 连接已关闭，且不会自动重连 |
- `EventSource`对象的`url`属性，可用于访问连接的URL。

#### 事件监听与连接管理
- `open`事件

连接一旦成功建立，就会触发`open`事件，可以在`onopen`属性定义回调函数。
```js
// 方式一：onopen 属性
source.onopen = function(event) {
    //...
}

// 方式二：addEventListener
source.addEventListener('open', function (event) {
    // ...
});
```
- `message`事件

客户端收到服务器发来的数据，就会触发`message`事件，数据存放在`event.data`中，可以在`onmessage`属性定义回调函数。
```js
// 方式一：onmessage 属性
source.onmessage = function (event) {
    const data = event.data; // 服务器返回的文本
    // ...
};

// 方式二
source.addEventListener('message', function(event){
    const data = event.data;
    // ...
})
```
注意：`event.data`始终是文本格式。如果过服务器发送的是JSON数据，需手动调用`JSON.parse()`解析。

- `error` 事件

连接中断或通信错误时触发`error`事件，可以在`onerror`属性定义回调函数。
```js
// 方式一：onerror 属性
source.onerror = function(event) {
    console.error("连接出错");
}

// 方式二：addEventListener
source.addEventListener('error', function (event) {
    console.error("连接出错");
});
```
- 自定义事件

默认情况下，服务器发来的数据，总是触发浏览器 `EventSource` 实例的 `message` 事件。但开发者也可以自定义 SSE 事件，这种情况下，发送回来的数据不会触发 `message` 事件。
```js
// 监听自定义事件（如服务器发来的 event: stock）
source.addEventListener('stock', (e) => {
    console.log('股票更新:', e.data);
})
```

- `close`方法

主动关闭连接。
```js
source.close(); // 关闭后不再自动重连
```

### 三、服务器实现
#### 数据格式
服务器向浏览器发送的 SSE 数据，必须是 UTF-8 编码的文本，具有如下的 HTTP 头信息。
```text
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
```
注意：`Content-Type` 必须指定 `event-stream` MIME类型。

每次发送的消息由一个或多个 `message` 组成，每个 message 以 `\n\n` 分隔。每个 message 内部由若干行构成，每行格式为：
```text
[field]: value\n
```
`field`可以为：`data`, `event`, `id`, `retry`。

此外，可包含冒号开头的注释行。通常，服务器每隔一段时间就会向浏览器发送一个注释，保持连接不中断。

示例：

```text
: This is a comment \n\n

data: 普通消息\n\n

data: 消息第一行\n
data: 消息第二行\n\n
```

#### `data` 字段
数据内容用 `data` 字段表示。
```text
data: message\n\n
```
如果数据很长，可以分成多行，前面行用 `\n` 结尾，最后一行用 `\n\n` 结尾。
```text
data: begin message\n
data: end message\n\n
```
一个发送 JSON 数据的示例：
```text
data: {\n
data: "foo": "bar",\n
data: "baz": 555\n
data: }\n\n
```

#### `id` 字段
数据标识符用 `id` 字段表示，相当于每一条数据的编号。
```text
id: msg001\n
data: message\n\n
```
浏览器会自动记录最后收到的 `id`，存储在 `lastEventId` 属性中；连接断开重连时，浏览器会自动添加请求头： `Last-Event-ID: msg001`，服务器可根据该头恢复发送后续消息。

#### `event` 字段
`event` 字段可用来表示自定义的事件类型，默认（不提供该字段时）是 `message` 事件。浏览器可以用 `addEventListener()` 监听该事件。

```text
event: foo\n
data: 这条消息会触发 foo 事件\n\n

data: 这条消息会触发默认的 message 事件\n\n

event: bar\n
data: 这条消息会触发 bar 事件\n\n
```
前端监听方式：
```js
source.addEventListener('foo', (e) => {
    console.log('自定义 foo 事件:', e.data);
});

source.addEventListener('bar', (e) => {
    console.log('自定义 bar 事件:', e.data);
})
```

#### `retry` 字段
服务器可以使用 `retry` 字段指定浏览器自动重连的时间间隔（单位：ms）
```text
retry: 10000\n\n    // 10s后尝试重连
```
两种情况会导致浏览器重新发起连接：
- 达到 `retry` 指定的时间间隔
- 网络错误导致连接意外中断
> 未指定 `retry` 时，浏览器使用默认重连间隔（通常为 3 s）。

### 四、SSE 的特点与注意事项
特点：
- 单向通信，服务器向客户端推送数据，客户端无法通过同一连接发送请求。如需发送数据需灵法HTTP请求。
- 基于 HTTP 协议，直接复用现有 HTTP/HTTPS，无需特殊协议（如 WebSocket 的 ws://），兼容现有网络基础设施。
- 自动重连，连接意外断开后，客户端内置机制会自动尝试重连，无需编写额外逻辑。
- 事件 ID 自动管理，支持为每条消息指定 `id`，浏览器自动记录并在 `Last-Event-ID` 头中发送。
- 支持断线续传，通过 `Last-Event-ID` 头记录最后收到的消息 ID，重连后可自动恢复丢失的消息。
- 文本与UTF-8，仅支持文本（UTF-8），无法原生发送二进制数据。二进制数据需 base64 编码或使用 WebSocket 。
- 支持自定义事件，通过 `event` 字段区分不同消息类型（如 `message`, `ping`, `update`），前端可为其绑定不同回调。

注意（常见问题）：
1. 连接数限制：同一域名下浏览器 SSE 连接数通常限制为 6 个（基于HTTP/1.1协议通信时），可通过升级到 HTTP/2 或采用多端口/域名分片解决。

在 HTTP/1.1 协议中，每个 SSE 连接都会占用一个独立的 TCP 连接。浏览器设定该限制，旨在防止单个网页或网站无限制地创建连接，消耗过多系统资源，从而拖慢整体网络性能。
HTTP/2 允许在同一个 TCP 连接上同时传输多个数据流，同一时间内服务器和客户端之间的最大连接数默认为 100 个（可根据需求调整）；此外，还可以采用多端口/域名分片方式，将连接分散到不同的端口
（`example.com:8001`, `:8002`）或子域名（`sse1.example.com`, `sse2.example.com`）。

2. 二进制数据：不支持二进制数据，需要 base64 编码或改用 WebSocket。
3. 适用场景：实时通知/股票行情/体育比分；服务器监控（日志实时输出），社交动态流等单向数据流，无需客户端频繁轮询。
4. 兼容性提醒：`EventSource` API 在现代浏览器中得到了广泛支持，包括 Chrome，Firefox，Safari，Edge 等，但 Internet Explorer 不支持。对于需要兼容 IE 的场景，可使用 `event-source-polyfill` 等库，
通过降级到长轮询来模拟 SSE 功能。
5. 安全性考量：SSE 受到浏览器同源策略限制，跨域场景需要正确配置 CORS。
必要响应头：
```text
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
```
安全建议：敏感数据使用 HTTPS 加密传输，限制允许的来源域，实现适当的身份验证
### 五、与其他技术对比
SSE 与 WebSocket 对比

| 特性 | SSE | WebSocket |
|---|---|---|
| 通信方向 | 单向（服务器->客户端） | 双向（全双工）|
| 协议基础 | HTTP | 独立 WebSocket 协议 |
| 数据传输格式 | 文本（UTF-8）| 文本或二进制数据 |
| 自动重连 | 内置支持 | 需手动实现 |
| 使用场景 | 实时通知、数据流、监控 | 聊天、协作编辑、游戏 |
| 复杂性 | 相对简单 | 相对复杂 |

SSE 与长轮询对比
- 连接机制差异
  - SSE持久连接：建立一次 HTTP 连接，服务器可随时推送多个事件。
  - 长轮询循环：一系列 HTTP 请求-响应，服务器挂起请求直到有数据或超时。
- 效率与资源消耗
  - 连接开销 SSE 更低
  - 实时性 SSE 更高
  - 实现复杂度 SSE 更简单

### 引用文章
- [阮一峰: Server-Sent Events 教程](https://www.ruanyifeng.com/blog/2017/05/server-sent_events.html)
- [Kimi 深度研究: Server-Sent Events 技术详解](https://www.kimi.com/preview/197bf3c5-a5c1-878a-8825-cfbcc8000519)