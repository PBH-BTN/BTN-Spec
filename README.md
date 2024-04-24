# BTN-Spec

BitTorrent Threat Network 标准规范。通过实现此规范，能够让让您的客户端接入任何其它符合此规范的 BTN 实例。


## 登录鉴权

所有发送到 BTN 实例的请求，都必须进行鉴权。尽管 BTN 实例可能对此不做要求，但你必须在所有发送到 BTN 实例的请求中，携带鉴权头。这样，BTN 实例将可以识别您的身份，并根据需要，从配置阶段开始就根据用户身份下发不同的配置文件。

BTN 使用 AppID + AppSecret 的组合来鉴权，以下是需要携带的 HTTP 请求头：

```
BTN-AppID: <AppID>
BTN-AppSecret: <AppSecret>
```

## 表明您的客户端实现

为了让 BTN 实例识别您的客户端的身份，您应该在 User-Agent 中携带一个 BTN 信息标记，其中包含：

* 您的实现名称和版本号（例如：PeerBanHelper/v3.2.0）
* 您的 BTN 实现版本号，此字段被固定为 `BTN-Protocol`（例如：BTN-Protocol/v3.0.0）

以下是一个标准 User-Agent 示例：

```
Java/17.0.1 PeerBanHelper/v3.2.0-dev BTN-Protocol/3.0.0
```

特别的，如果你在实验 BTN 协议，则应该将版本号置为 0，并附带 dev 标签：

```
Java/17.0.1 PeerBanHelper/v3.2.0-dev BTN-Protocol/0.0.0-dev
```

此 UA 允许 BTN 实例发送兼容响应，以适配多个不同 BTN 规范版本的客户端。

## 让服务器配置你

一旦准备就绪，您应该向 `CONFIG URL` 发送一个 GET 请求，要求 BTN 服务器为您下发配置文件。

以下是一个示例响应：

```json


```

