# BTN-Spec

BitTorrent Threat Network 标准规范。通过实现此规范，能够让让您的客户端接入任何其它符合此规范的 BTN 实例。

> [!TIP]
> 当前规范版本号：0.0.0-dev （草案、非稳定版本）  
> 在实现非稳定版本 BTN-Spec 时，建议与 BTN 规范制定者联系。


## 登录鉴权

所有发送到 BTN 实例的请求，都必须进行鉴权。尽管 BTN 实例可能对此不做要求，但你必须在所有发送到 BTN 实例的请求中，携带鉴权头。这样，BTN 实例将可以识别您的身份，并根据需要，从配置阶段开始就根据用户身份下发不同的配置文件。

BTN 使用 AppID + AppSecret 的组合来鉴权，以下是需要携带的 HTTP 请求头：

必须支持：

```
X-BTN-AppID: <AppID>
X-BTN-AppSecret: <AppSecret>
```

必须支持：

```
Authentication: Bearer <AppID>@<AppSecret>
```

向前兼容性：

```
BTN-AppID: <AppID>
BTN-AppSecret: <AppSecret>
```

我们规范推荐您支持第一和第二种，但由于仍有旧版本 PBH 运行，因此推荐您也支持第三种。  
三种有任意一种鉴权成功，则视为鉴权通过。  


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
{
	"min_protocol_version": 3,
	"max_protocol_version": 3,
	"ability": {
		"submit_peers": {
			"interval": 900000,
			"endpoint": "https://btn-dev-v2.ghostchu-services.top/ping/submitPeers",
			"random_initial_delay": 5000
		},
		"submit_bans": {
			"interval": 900000,
			"endpoint": "https://btn-dev-v2.ghostchu-services.top/ping/submitBans",
			"random_initial_delay": 5000
		},
		"rules": {
			"interval": 900000,
			"endpoint": "https://btn-dev-v2.ghostchu-services.top/ping/rules",
			"random_initial_delay": 5000
		},
		"reconfigure": {
			"interval": 900000,
			"random_initial_delay": 5000,
			"version": "3642e5eb-d435-4905-ad0e-e908470833c3"
		}
	}
}
```

在您的客户端实现收到此响应后，应首先检查自己的客户端是否满足 protocol_version 的需求，处于区间之内。如果不处于区间内，则应退出，并向用户报告错误。  
`ability` 对象内是服务器支持的能力及其配置列表，其具体内容由各个能力模块定义。  

## 能力

服务器不必实现全部能力，同理客户端也是。但你总是应该尽最大可能实现本规范列出的所有能力，以提供最佳使用体验。  
特别的，在配置提交数据类的能力时，应首先征求用户同意以确保用户知道自己的部分数据将被提交到服务器。在未取得用户同意的情况下，不得执行这些能力。

### 提交 Peers 列表 `submit_peers`

此能力允许 BTN 兼容客户端向 BTN 实例提交目前连接到下载器的 Peers 列表。

#### 配置

```json
{
	"interval": 900000,
	"endpoint": "https://btn-dev-v2.ghostchu-services.top/ping/submitPeers",
	"random_initial_delay": 5000
}
```

`interval`: 提交间隔（单位：毫秒）
`random_initial_delay`: 首次提交延迟随机偏移（单位：毫秒）。客户端首次提交应被计划在 `interval + random.nextLong(random_initial_delay)` 期间，以避免服务器出现请求处理尖峰，缓解服务器压力
`endpoint`: 指定此能力数据将被提交到哪个 API 端点。

#### 请求

以下是请求示例：

```json
{
	"populate_time": 1713971696000,
	"peers": [{
			"ip_address": "CE67:2B6F:646A:138B:9E4F:DD47:894E:608E",
			"peer_port": 12345,
			"peer_id": "-BC1234-",
			"client_name": "BitComet 1.2.3.4",
			"torrent_identifier": "<使用特定算法对 info_hash 进行加盐哈希>",
			"torrent_size": 12346789765,
			"downloaded": 3463465,
			"rt_download_speed": 133525,
			"uploaded": 2345754,
			"rt_upload_speed": 234456465,
			"peer_progress": 0.1245,
			"downloader_progress": 1,
			"peer_flag": "u I H X E P"
		},
		{
			"ip_address": "198.148.143.87",
			"peer_port": 23333,
			"peer_id": "-qB2312-",
			"client_name": "qBittorrent 2.3.1.2",
			"torrent_identifier": "<使用特定算法对 info_hash 进行加盐哈希>",
			"torrent_size": 12346789765,
			"downloaded": 3463465,
			"rt_download_speed": 133525,
			"uploaded": 2345754,
			"rt_upload_speed": 234456465,
			"peer_progress": 0.1245,
			"downloader_progress": 1,
			"peer_flag": "u I H X E P"
		}
	]
}
```

字段说明：
* populate_time - 数据打包时间
* ip_address - Peer 的 IPV4/IPV6 地址
* peer_port - Peer 连接的端口号
* peer_id - Peer ID，直接提交原始内容，无需过滤不可打印字符，如果不支持或未获取到，请使用空字符串填充
* client_ name - Peer ClientName，有时也被称为 User-Agent，如果不支持或未获取到，请使用空字符串填充
* torrent_identifier - 种子唯一 ID，基于 info_hash 使用[特定算法](https://github.com/PBH-BTN/BTN-Spec/blob/main/README.md#torrent-identifier-%E7%AE%97%E6%B3%95)加盐哈希，以匿名化处理
* torrent_size - 种子大小（单位：bytes）
* downloaded - 用户下载器从此 Peer 获取的数据总量 （单位：bytes），如果不支持，请使用 -1 值填充
* rt_download_speed - 用户下载器从此 Peer 获取数据的实时速度（单位：bytes），如果不支持，请使用 -1 值填充
* uploaded - 用户下载器向此 Peer 提供的数据总量 （单位：bytes），如果不支持，请使用 -1 值填充
* rt_upload_speed - 用户下载器向此 Peer 提供的数据的实时速度 （单位：bytes），如果不支持，请使用 -1 值填充
* peer_progress - Peer 在当前 torrent 的下载进度（浮点型，0 = 0%，1 = 100%）
* downloader_progress - 用户在当前 torrent 的下载进度（浮点型，0 = 0%，1 = 100%）
* peer_flag - BT 客户端显示的 “标志”，直接获取并填写在这里即可，如果不支持或未获取到，请使用空字符串填充

提交方式：  
向此能力给定的 endpoint 发送 POST 请求。请求体必须且只能使用 GZIP 压缩，不支持未压缩的传输。  
附加请求头：
  * Content-Encoding: gzip

#### 响应

期望响应：  
* 200 - 服务器成功处理此请求

错误、重定向响应：  
请参见：[通用响应处理](https://github.com/PBH-BTN/BTN-Spec/blob/main/README.md#%E9%80%9A%E7%94%A8%E5%93%8D%E5%BA%94%E5%A4%84%E7%90%86)

### 提交封禁列表 `submit_bans`

此能力允许 BTN 兼容客户端向 BTN 实例提交当前活跃的封禁列表。

#### 配置

```json
{
	"interval": 900000,
	"endpoint": "https://btn-dev-v2.ghostchu-services.top/ping/submitBans",
	"random_initial_delay": 5000
}
```

`interval`: 提交间隔（单位：毫秒）
`random_initial_delay`: 首次提交延迟随机偏移（单位：毫秒）。客户端首次提交应被计划在 `interval + random.nextLong(random_initial_delay)` 期间，以避免服务器出现请求处理尖峰，缓解服务器压力
`endpoint`: 指定此能力数据将被提交到哪个 API 端点。

#### 请求

以下是请求示例：

```json
{
	"populate_time": 1714280802047,
	"bans": [{
		"btn_ban": false,
		"module": "com.ghostchu.peerbanhelper.module.impl.rule.PeerIdBlacklist",
		"rule": "匹配 PeerId 规则: StringStartsWithMatcher{rule='-hp', hit=TRUE, miss=DEFAULT}",
		"peer": {
			"ip_address": "123.187.29.6",
			"peer_port": 60874,
			"peer_id": "-HP0001-",
			"client_name": "HP 0.0.0.1",
			"torrent_identifier": "<使用特定算法对 info_hash 进行加盐哈希>",
			"torrent_size": 5044211712,
			"downloaded": 0,
			"rt_download_speed": 0,
			"uploaded": 0,
			"rt_upload_speed": 0,
			"peer_progress": 0.0,
			"downloader_progress": 0.3929437099724969,
			"peer_flag": "I E",
                        "ban_unique_id": "123456789"
		}
	}]
}
```

字段说明：
* populate_time - 数据打包时间
* bans - 封禁列表
* btn_ban - 是否是被 BTN 的规则封禁的 Peer
* module - 执行封禁规则模块名称
* ip_address - Peer 的 IPV4/IPV6 地址
* peer_port - Peer 连接的端口号
* peer_id - Peer ID，直接提交原始内容，无需过滤不可打印字符，如果不支持或未获取到，请使用空字符串填充
* client_ name - Peer ClientName，有时也被称为 User-Agent，如果不支持或未获取到，请使用空字符串填充
* torrent_identifier - 种子唯一 ID，基于 info_hash 使用[特定算法](https://github.com/PBH-BTN/BTN-Spec/blob/main/README.md#torrent-identifier-%E7%AE%97%E6%B3%95)加盐哈希，以匿名化处理
* torrent_size - 种子大小（单位：bytes）
* downloaded - 用户下载器从此 Peer 获取的数据总量 （单位：bytes），如果不支持，请使用 -1 值填充
* rt_download_speed - 用户下载器从此 Peer 获取数据的实时速度（单位：bytes），如果不支持，请使用 -1 值填充
* uploaded - 用户下载器向此 Peer 提供的数据总量 （单位：bytes），如果不支持，请使用 -1 值填充
* rt_upload_speed - 用户下载器向此 Peer 提供的数据的实时速度 （单位：bytes），如果不支持，请使用 -1 值填充
* peer_progress - Peer 在当前 torrent 的下载进度（浮点型，0 = 0%，1 = 100%）
* downloader_progress - 用户在当前 torrent 的下载进度（浮点型，0 = 0%，1 = 100%）
* peer_flag - BT 客户端显示的 “标志”，直接获取并填写在这里即可，如果不支持或未获取到，请使用空字符串填充
* ban_unique_id - 唯一封禁ID，在解封之前此值应保持不变

提交方式：  
向此能力给定的 endpoint 发送 POST 请求。请求体必须且只能使用 GZIP 压缩，不支持未压缩的传输。  
附加请求头：
  * Content-Encoding: gzip

#### 响应

期望响应：  
* 200 - 服务器成功处理此请求

错误、重定向响应：  
请参见：[通用响应处理](https://github.com/PBH-BTN/BTN-Spec/blob/main/README.md#%E9%80%9A%E7%94%A8%E5%93%8D%E5%BA%94%E5%A4%84%E7%90%86)

### 允许重新配置 `reconfigure`

此能力允许 BTN 兼容客户端检测 BTN 实例上的配置更改，并在需要时重新配置 BTN 兼容客户端。

#### 配置

```json
{
	"interval": 10800000,
	"random_initial_delay": 5000,
	"version": "0d1a867c-c665-460a-992a-94e983b40ec1"
}
```

* `interval`: 提交间隔（单位：毫秒）  
* `random_initial_delay`: 首次提交延迟随机偏移（单位：毫秒）。客户端首次提交应被计划在 `interval + random.nextLong(random_initial_delay)` 期间，以避免服务器出现请求处理尖峰，缓解服务器压力  
* `version`: 指定当前配置版本号，当两次检查版本号不一致时，将会触发重新配置

#### 响应

期望响应：  
* 200 - 服务器成功处理此请求

错误、重定向响应：  
请参见：[通用响应处理](https://github.com/PBH-BTN/BTN-Spec/blob/main/README.md#%E9%80%9A%E7%94%A8%E5%93%8D%E5%BA%94%E5%A4%84%E7%90%86)

#### 请求

和初次配置一样，会请求 config-url 配置端点。

#### 响应

和初次配置一样。

### 提交规则命中统计数据

此能力允许 BTN 兼容客户端向 BTN 实例提交当前客户端配置的规则列表（包括用户配置的），以及每个规则的命中统计。

TODO: 等待施工！

#### 响应

期望响应：  
* 200 - 服务器成功处理此请求

错误、重定向响应：  
请参见：[通用响应处理](https://github.com/PBH-BTN/BTN-Spec/blob/main/README.md#%E9%80%9A%E7%94%A8%E5%93%8D%E5%BA%94%E5%A4%84%E7%90%86)

## 通用响应处理

BTN 实现客户端应该合理的处理服务器的响应。对于重定向响应（301/302），则自动跟随。

对于错误响应，服务器应该按照 [HTTP 响应状态码规范](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Status) 返回合理的响应码；客户端则仅需要在收到这些状态码后，将状态码和响应体告知用户（如：在控制台打印），然后进行错误处理即可。  

## Torrent Identifier 算法

为了匿名用户下载内容（我们不关心你在下载什么，也不想承担存储它带来的隐私风险！），**所有 BTN 实现客户端必须严格执行此算法，匿名处理用户的种子信息**。

以下是 Java 实现的代码（使用 Google Guava 库）：

```java
    /**
     * 获取种子不可逆匿名识别符
     *
     * @return 不可逆匿名识别符
     */
    public String getHashedIdentifier(String torrentInfoHash) {
        String salt = Hashing.crc32().hashString(torrentInfoHash, StandardCharsets.UTF_8).toString(); // 使用 crc32 计算 info_hash 的哈希作为盐
        return Hashing.sha256().hashString(torrentInfoHash + salt, StandardCharsets.UTF_8).toString(); // 在 info_hash 的明文后面追加盐后，计算 SHA256 的哈希值，结果应转全小写
    }
```


## License

BTN-Spec 的文档和示例代码在 CC-0 (Public Domain) 协议下授权：

[![CC-0](https://mirrors.creativecommons.org/presskit/buttons/88x31/png/cc-zero.png)](https://creativecommons.org/publicdomain/zero/1.0/)


## 目前接入 BTN 协议的客户端列表

* [PeerBanHelper](https://github.com/PBH-BTN/PeerBanHelper) （最新版支持：0.0.0-dev）
