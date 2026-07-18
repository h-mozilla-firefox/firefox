# MozNetLogger 配置说明

MozNetLogger 是植入 `nsHttpChannel` 的本地 HTTP 抓取器。它按 URL 正则规则，把命中的
请求/响应完整落盘为 JSON 文件，用于在**自己编译、自己运行**的 Firefox 上调试和排查网络流量。

> 抓取内容包含请求头、响应头和完整响应体，其中可能有 Cookie、Authorization、token 等
> 敏感信息。请仅对自己拥有或已获授权的服务使用，并妥善保管落盘文件。

## 配置文件位置

固定路径（硬编码）：`C:\firefox.json`

- 文件不存在、内容为空或 JSON 解析失败时，抓取功能静默关闭，不影响正常浏览。
- 配置在进程内只加载一次，**修改配置后需重启 Firefox** 才能生效。

## 配置结构

```json
{
  "httpLog": {
    "filters": [
      { "urlPattern": "<正则>", "saveDir": "<保存目录>" }
    ]
  }
}
```

字段说明：

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `httpLog.filters` | 数组 | 一组过滤规则，按顺序匹配，命中**第一条**即生效 |
| `urlPattern` | 字符串 | ECMAScript 正则，对完整 URL 做 `regex_search`（子串匹配），忽略大小写 |
| `saveDir` | 字符串 | 命中请求的 JSON 文件保存目录，不存在会自动创建 |

规则说明：

- `urlPattern` 或 `saveDir` 为空的规则会被跳过。
- 正则为**部分匹配**（`regex_search`），无需从头到尾匹配整条 URL。
- Windows 路径中的反斜杠在 JSON 里要转义为 `\\`。

## 输出格式

每个命中的请求在结束时写一个文件：

- 文件名：`<saveDir>\yyyyMMddHHmmssSSS.json`（本地时间，毫秒级时间戳）
- 内容：

```json
{
  "timestamp": "20260717141138123",
  "request": {
    "url": "https://api.example.com/v1/data",
    "method": "GET",
    "host": "api.example.com",
    "headers": "GET /v1/data HTTP/1.1\r\nHost: api.example.com\r\n..."
  },
  "response": {
    "status": 200,
    "headers": "HTTP/1.1 200 OK\r\nContent-Type: application/json\r\n...",
    "body": "<Base64 编码的响应体>"
  }
}
```

- `request.headers` / `response.headers` 是扁平化的原始报文头文本。
- `response.body` 是 **Base64 编码**，解码后即原始响应体（支持文本和二进制）。

## 配置 demo

见同目录下的 `MozNetLogger.demo.json`。

## 注意事项与已知限制

- **仅 Windows**：路径和分隔符按 Windows 硬编码。
- **正则必须合法**：Firefox 以 `-fno-exceptions` 编译，非法正则会导致进程终止（崩溃）。写入配置前请先验证正则语法。
- **大响应体**：响应体会整体缓存进内存再落盘，抓取大文件/流媒体会显著增加内存占用，并改变原有的流式传输行为，不建议匹配大文件或长连接（SSE/流式）接口。
- **无脱敏**：落盘内容为明文报文头 + Base64 响应体，不做任何敏感字段过滤。
- **匹配范围**：`urlPattern` 越宽，命中越多、落盘文件越多，建议尽量收窄到目标接口。
