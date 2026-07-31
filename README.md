此文档可能已过期。详细实现请参见：<https://github.com/PBH-BTN/Sparkle3> 源代码。

# BTN-Spec

BitTorrent Threat Network 标准规范。通过实现此规范，能够让让您的客户端接入任何其它符合此规范的 BTN 实例。

> [!TIP]
> 当前规范版本号：2.0.1 ，内部版本 `20`  
> 在实现非稳定版本 BTN-Spec 时，建议与 BTN 规范制定者联系。

## 设计理念

BTN-Spec 设计时考虑到中心化服务器容易遭到攻击或者数据泄露等情况，因此它被设计为可以在极低服务质量保证的情况下为大量用户提供服务。

* 不使用任何长连接
* 提交/获取数据时，不影响缓存数据
* 抛去需要动态环境的功能，其其余的规则分发等功能，应当可以在完全静态环境中部署（如：Github Pages），提高抗故障能力
* 最后，本规范设计时着重考虑了带宽消耗，避免服务器在任何非必要的情况下向客户端传输额外数据

## 客户端规范

* 客户端每次获取 BTN 响应后，都应该持久化缓存本地
* 如果初始化失败（且为网络错误或者服务器临时故障），则应当在一段时间后重试，这个时间不应该短于 600 秒。如果是服务器明确表明客户端版本/协议不兼容（403、400 状态码），则不应该继续重试（
* 如果提交数据失败，则客户端不应当进行重试：
  * 对于 Peers 快照数据：直接丢弃
  * 对于封禁列表数据：缓存直至下次提交
* 如果获取数据失败，最多进行 3 次重试，然后推迟到下次获取窗口
* 如果服务器要求客户端重定向，则应跟随重定向
* 对于同一个请求包含超过 1000 条数据的情况，则必须按每个请求最多 1000 条数据进行拆分。单个请求超过 1000 条数据将被服务端截断。超出部分会被丢弃
* 对于 GZIP 压缩传输的请求，如果解压后大小超过 33554432 字节，请求将在传输过程中被拒绝。IP 地址可能遭遇短暂速率限制或封禁
* 所有使用时间戳的位置，除非明确说明，否则均使用 UNIX 毫秒时间戳
* 所有设计数据长度、文件大小等的单位，除非明确说明，均使用字节（Byte）单位

## 登录鉴权

所有发送到 BTN 实例的请求，都必须进行鉴权。尽管 BTN 实例可能对此不做要求，但你必须在所有发送到 BTN 实例的请求中，携带鉴权头。这样，BTN 实例将可以识别您的身份，并根据需要，从配置阶段开始就根据用户身份下发不同的配置文件。

BTN 使用 AppID + AppSecret 的组合来鉴权，以下是需要携带的 HTTP 请求头：

```plain
X-BTN-AppID: <AppID>
X-BTN-AppSecret: <AppSecret>
```

如果用户未配置 BTN 鉴权，在支持的服务器实例上，使用下面的参数代替 BTN-AppID 和 BTN-AppSecret：

```plain
X-BTN-InstallationID: <InstallationID>
```

InstallationID 是唯一安装 ID，可以随机生成，但必须确保正确持久化。您也可以通过 MAC 地址计算，但通过 MAC 计算时必须与持久化的安装 ID 混合计算，这是因为同一台设备上可能由多个 BTN 客户端安装。  
对于支持的此方式连接 BTN 服务器实例，将自动为您创建一个匿名 BTN UserApp，并将操作绑定其上。但如果服务器不支持，也可能回退到鉴权失败。

## 表明您的客户端实现

为了让 BTN 实例识别您的客户端的身份，您应该在 User-Agent 中携带一个 BTN 信息标记，其中包含：

* 您的实现名称和版本号（例如：PeerBanHelper/v3.2.0）
* 您的 BTN 实现版本号，此字段被固定为 `BTN-Protocol`（例如：BTN-Protocol/v3.0.0）
* 系统信息（可选）

以下是一个标准 User-Agent 示例：

```plain
PeerBanHelper/9.3.14 (debian; Linux,7.0.0-28-generic,Resolute Raccoon7.0.0-28-generic) BTN-Protocol/2.0.1 BTN-Protocol-Version/20
```

特别的，如果你在实验 BTN 协议，则应该将版本号置为 0，并附带 dev 标签：

```plain
PeerBanHelper/9.3.14 (debian; Linux,7.0.0-28-generic,Resolute Raccoon7.0.0-28-generic) BTN-Protocol/0.0.0-dev BTN-Protocol-Version/0
```

## 工作量证明 (PoW)

为了避免对 API 请求的恶意滥用，提交端点可能执行 POW 验证。此类验证可以通过程序自动化完成而无需用户干预。可以缓解大规模请求 API 的恶意行为。  
我们尽可能保证各 API 免 POW 验证提交，但如果遭遇恶意攻击，则开关可能会被开启。

```json
"proof_of_work_captcha": {
        "endpoint": "https://sparkle.pbh-btn.com/ping/captcha/createSession"
}
```

endpoint 为 PoW 任务创建端点，则需要发送请求：

```url
GET <endpoint>?type=<模块configkey>
```

您将收到类似下面的响应：

```json
{
    "id": "c495327f-1909-4d2d-ba6b-91617f1e9110",
    "challengeBase64": "QCWfZutq+lC6auXXVXHzcQ==",
    "difficultyBits": 6,
    "algorithm": "SHA-256",
    "expireAt": 1785499028371
}
```

* `id`: 创建的会话 ID
* `challengeBase64`:要进行挑战的 byte[] 的 Base64 编码文本
* `difficultyBits`: 难度
* `algorithm`: 指定的哈希算法，目前仅使用 `SHA-256`
* `expireAt` 有效期截止时间 UNIX 毫秒时间戳

注意不要混用 configkey，后续可能会进行检查。

在完成计算后，重试需要提交的请求，并附带额外 HTTP 头：

```plain
X-BTN-PowID: <PoW ID>
X-BTN-PowSolution: <Base64 Encoded Pow Solution>
```

* `PoW ID`: 上面响应中给出的 ID
* `Base64 Encoded Pow Solution`: 计算结果 byte[] 的 Base64 编码文本

示例 Java 如下：

<details>
<summary>点击展开查看内容</summary>

```java
package com.ghostchu.peerbanhelper.util.pow;

import io.sentry.Sentry;

import java.nio.ByteBuffer;
import java.security.MessageDigest;
import java.security.SecureRandom;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.atomic.AtomicBoolean;

/**
 * PoW 无交互验证码（客户端）
 *
 */
public class PoWClient implements AutoCloseable {
    private final int threadCount = Runtime.getRuntime().availableProcessors();
    private final ExecutorService executor = Executors.newWorkStealingPool(threadCount);

    public byte[] solve(byte[] challenge, int difficultyBits, String algorithm) throws Exception {
        AtomicBoolean found = new AtomicBoolean(false);
        CompletableFuture<byte[]> resultFuture = new CompletableFuture<>();

        for (int t = 0; t < threadCount; t++) {
            int threadId = t;
            executor.submit(() -> {
                try {
                    MessageDigest digest = MessageDigest.getInstance(algorithm);
                    ByteBuffer buffer = ByteBuffer.allocate(8);
                    long nonce = new SecureRandom().nextLong() + threadId;
                    while (!found.get()) {
                        if (Thread.currentThread().isInterrupted()) {
                            return;
                        }
                        digest.reset();
                        digest.update(challenge);
                        buffer.clear();
                        buffer.putLong(nonce);
                        byte[] nonceBytes = buffer.array();
                        digest.update(nonceBytes);
                        byte[] hash = digest.digest();

                        if (hasLeadingZeroBits(hash, difficultyBits)) {
                            if (found.compareAndSet(false, true)) {
                                resultFuture.complete(nonceBytes.clone());
                            }
                            break;
                        }
                        nonce += threadCount;
                    }
                } catch (Exception e) {
                    Sentry.captureException(e);
                    resultFuture.completeExceptionally(e);
                }
            });
        }

        // wait for one thread to finish
        return resultFuture.get();
    }

    private boolean hasLeadingZeroBits(byte[] hash, int bits) {
        int fullBytes = bits / 8;
        int remainingBits = bits % 8;
        for (int i = 0; i < fullBytes; i++) {
            if (hash[i] != 0) return false;
        }
        if (remainingBits > 0) {
            int mask = 0xFF << (8 - remainingBits);
            return (hash[fullBytes] & mask) == 0;
        }
        return true;
    }

    @Override
    public void close() {
        this.executor.close();
    }
}
```

</details>


## 让服务器配置你

一旦准备就绪，您应该向 `CONFIG URL` 发送一个 GET 请求，要求 BTN 服务器为您下发配置文件。

以下是一个示例响应：

```json
{
    "min_protocol_version": 20,
    "max_protocol_version": 20,
    "ability": {
        "reconfigure": {
            "interval": 3600000,
            "random_initial_delay": 600000,
            "version": "9656e835-a39d-475f-83a0-232f439a0100",
            "config_key": "reconfigure"
        },
        "heartbeat": {
            "config_key": "heartbeat",
            "endpoint": "https://sparkle.ghostchu.com/ping/heartbeat",
            "interval": 1800000,
            "multi_if": true,
            "pow_captcha": false,
            "random_initial_delay": 3000
        },
        "ip_query": {
            "endpoint": "https://sparkle.ghostchu.com/ping/queryIp",
            "pow_captcha": false,
            "iframe_endpoint": "https://sparkle.pbh-btn.com/ping/queryIp/widget",
            "config_key": "ip_query"
        },
        "ip_allowlist": {
            "endpoint": "https://sparkle.ghostchu.com/ping/ruleIpAllowlist",
            "interval": 600000,
            "random_initial_delay": 15000,
            "pow_captcha": false,
            "config_key": "ip_allowlist"
        },
        "ip_denylist": {
            "endpoint": "https://sparkle.ghostchu.com/ping/ruleIpDenylist",
            "interval": 600000,
            "random_initial_delay": 15000,
            "pow_captcha": false,
            "config_key": "ip_denylist"
        },
        "rule_peer_identity": {
            "endpoint": "https://sparkle.ghostchu.com/ping/rulePeerIdentity",
            "interval": 2700000,
            "random_initial_delay": 15000,
            "pow_captcha": false,
            "config_key": "rule_peer_identity"
        },
        "submit_bans": {
            "config_key": "submit_bans",
            "endpoint": "https://sparkle.ghostchu.com/ping/syncBanHistory",
            "interval": 900000,
            "pow_captcha": false,
            "random_initial_delay": 15000
        },
        "submit_swarm": {
            "config_key": "submit_swarm",
            "endpoint": "https://sparkle.ghostchu.com/ping/syncSwarm",
            "interval": 900000,
            "pow_captcha": false,
            "random_initial_delay": 15000
        }
    },
    "proof_of_work_captcha": {
        "endpoint": "https://sparkle.pbh-btn.com/ping/captcha/createSession"
    }
}
```

在您的客户端实现收到此响应后，应首先检查自己的客户端是否满足 protocol_version 的需求，处于区间之内。如果不处于区间内，则应退出，并向用户报告错误。  
`ability` 对象内是服务器支持的能力及其配置列表，其具体内容由各个能力模块定义。  

`proof_of_work_captcha` 对象内是 Proof-of-Work (工作量证明) 验证码配置段，允许在无人值守的情况下计算数学难题以完成自主验证。主要用于缓解 BTN 服务器遭到恶意攻击的情况。

## 能力

服务器不必实现全部能力，同理客户端也是。但你总是应该尽最大可能实现本规范列出的所有能力，以提供最佳使用体验。  
特别的，在配置提交数据类的能力时，应首先征求用户同意以确保用户知道自己的部分数据将被提交到服务器。在未取得用户同意的情况下，不得执行这些能力。

### 提交封禁列表 `submit_bans`

此能力允许 BTN 兼容客户端向 BTN 实例提交当前活跃的封禁列表。

#### 配置

```json
{
    "config_key": "submit_bans",
    "endpoint": "https://sparkle.ghostchu.com/ping/syncBanHistory",
    "interval": 900000,
    "pow_captcha": false,
    "random_initial_delay": 15000
}
```

`interval`: 提交间隔（单位：毫秒）
`random_initial_delay`: 首次提交延迟随机偏移（单位：毫秒）。客户端首次提交应被计划在 `interval + random.nextLong(random_initial_delay)` 期间，以避免服务器出现请求处理尖峰，缓解服务器压力
`endpoint`: 指定此能力数据将被提交到哪个 API 端点。  
`pow_captcha`: 此端点是否收到 PoW 验证码保护

#### 请求

以下是请求示例：

```json
{
 "bans": [{
    "ban_at": 0,
    "peer_ip": "127.0.0.1",
    "peer_port": 0,
    "peer_id": "-qB0000-",
    "peer_client_name": "qBittorrent/0.0.0",
    "peer_progress": 0.0000000,
    "peer_flag": "d U",
    "torrent_identifier": "52fa13494a4571a951b46b1a04be19ab9d8089c3d3761956c99f5435e6f2c8ad",
    "torrent_is_private": false,
    "torrent_size": 0,
    "from_peer_traffic": 0,
    "to_peer_traffic": 0,
    "downloader_progress": 0.000000,
    "module": "模块名称",
    "rule": "规则名称",
    "description": "人类可读描述信息",
    "structured_data": "使用 JSON 格式表达的你的模块在封禁时用于决断的数据快照及决断阈值等技术性信息"
 }]
}
```

字段说明：

* `ban_at`: 封禁该 Peer 的 UNIX 毫秒时间戳
* `peer_ip`: Peer 的 IPv4 或 IPv6 地址
* `peer_port`: Peer 的端口号
* `peer_id`: Peer 的 Peer ID (请提供尽可能长的 PeerID，如遇 qB 此类会截断 PeerID 的下载器，提交阶段后的也可)
* `peer_client_name`: Peer 的客户端名称，如果下载器未提供，请提交空字符串
* `peer_progress`: 双精度浮点型的 Peer 任务进度 （0.0 = 0%, 1.0 = 100%）
* `peer_flag`: Peer Flag 信息，请参考 qBittorrent 的表示方法，如 d U O
* `torrent_identifier`: Torrent 识别码（参见后续文档介绍的计算方式）
* `torrent_is_private`: 是否为私有种子，如果未知请提交 false
* `torrent_size`: 种子大小（字节）
* `from_peer_traffic`: 由远程 Peer 传输给用户下载器的累计数据量，需要程序自己计算真实的值（计算值）
* `to_peer_traffic`: 由用户下载器传输到远程 Peer 的累计数据量，需要程序自己计算真实的值（计算值）
* `downloader_progress`: 记录创建时，用户下载器自己的任务进度快照
* `module`: 封禁操作的模块名称（使用自己的内部固定代号，尽量不要变动，且携带自己的命名空间/前缀，避免和它人的数据混在一起）
* `rule`: 产生封禁操作的规则名称
* `description`: 人类可读的封禁原因描述
* `structured_data`: JSON 格式的 JsonObject，包含了封禁时你的模块用于决策的元数据，尽可能提供多的信息

提交方式：

向此能力给定的 endpoint 发送 POST 请求。请求体必须且只能使用 GZIP 压缩，不支持未压缩的传输。  
附加请求头：

* Content-Encoding: gzip

#### 响应

期望响应：  

* 200 - 服务器成功处理此请求

错误、重定向响应：  
请参见：[通用响应处理](https://github.com/PBH-BTN/BTN-Spec/blob/main/README.md#%E9%80%9A%E7%94%A8%E5%93%8D%E5%BA%94%E5%A4%84%E7%90%86)

### 查询 IP 地址 `ip_query`

此能力允许 BTN 兼容客户端直接向 BTN 服务器请求数据查询，而无需用户打开网页手动操作。  
请注意不要频繁调用，此 IP 地址较为昂贵。若出现频繁调用，将启用 POW 验证码。

#### 配置

```json
{
    "endpoint": "https://sparkle.ghostchu.com/ping/queryIp",
    "pow_captcha": false,
    "iframe_endpoint": "https://sparkle.pbh-btn.com/ping/queryIp/widget",
    "config_key": "ip_query"
}
```

`interval`: 提交间隔（单位：毫秒）
`endpoint`: 指定此能力数据将被提交到哪个 API 端点。  
`pow_captcha`: 此端点是否收到 PoW 验证码保护  
`iframe_endpoint`: "嵌入式网页 iframe 地址"

#### iframe 嵌入

通过 URL 拼接，可以得到嵌入 URL，将提供一个 BTN 页面显示 IP 的数据：

```url
<iframe_endpoint>?ip=<IP>&appId=<BTN AppID>&appSecret=<BTN AppSecret>&installationId=<Installation ID>
```

#### 请求

以下是请求示例：

```json
GET <endpoint>?ip=<IP>
```

字段说明：

* `endpoint`: 此模块 JSON 配置中的 Endpoint 端点配置
* `IP`: 要查询的 IP 地址

提交方式：

向此能力给定的 endpoint 发送 GET 请求。或者使用 iframe 嵌入页面

#### 响应

期望响应：  

* 200 - 服务器成功处理此请求

错误、重定向响应：  
请参见：[通用响应处理](https://github.com/PBH-BTN/BTN-Spec/blob/main/README.md#%E9%80%9A%E7%94%A8%E5%93%8D%E5%BA%94%E5%A4%84%E7%90%86)

### 提交封禁列表 `submit_swarm`

此能力允许 BTN 兼容客户端向 BTN 实例提交种群信息。

Sparkle BTN 服务器上运行了一个类 Tracker 系统，通过此端点提交的数据将在上面形成一个跨用户、跨任务、跨网段的聚合列表并不停更新。Sparkle BTN 将通过此列表检查某个特定的 IP 地址在同一种子不同用户，或者不同种子不同用户上的任务元数据。

#### 配置

```json
{
    "config_key": "submit_swarm",
    "endpoint": "https://sparkle.ghostchu.com/ping/syncSwarm",
    "interval": 900000,
    "pow_captcha": false,
    "random_initial_delay": 15000
}
```

`interval`: 提交间隔（单位：毫秒）
`random_initial_delay`: 首次提交延迟随机偏移（单位：毫秒）。客户端首次提交应被计划在 `interval + random.nextLong(random_initial_delay)` 期间，以避免服务器出现请求处理尖峰，缓解服务器压力
`endpoint`: 指定此能力数据将被提交到哪个 API 端点。  
`pow_captcha`: 此端点是否收到 PoW 验证码保护

#### 请求

以下是请求示例：

```json
{
 "swarms": [{
    "torrent_identifier": "52fa13494a4571a951b46b1a04be19ab9d8089c3d3761956c99f5435e6f2c8ad",
    "torrent_is_private": false,
    "torrent_size": 0,
    "downloader": "用户下载器唯一识别 ID，如果你的程序仅通过 URL 识别不同的下载器，请通过加盐哈希等手段将其匿名化",
    "downloader_progress": 0.000000,
    "peer_ip": "127.0.0.1",
    "peer_port": 0,
    "peer_id": "-qB0000-",
    "peer_client_name": "qBittorrent/0.0.0",
    "peer_progress": 0.000000,
    "to_peer_traffic": 0,
    "to_peer_traffic_offset": 0,
    "from_peer_traffic": 0,
    "from_peer_traffic_offset": 0,
    "first_time_seen": 0,
    "last_time_seen": 0,
    "peer_last_flags": "d U O",
    "download_speed": 0,
    "download_speed_max": 0,
    "upload_speed": 0,
    "upload_speed_max": 0,
 }]
}
```

字段说明：

* `torrent_identifier`: Torrent 识别码（参见后续文档介绍的计算方式）
* `torrent_is_private`: 是否为私有种子，如果未知请提交 false
* `torrent_size`: 种子大小（字节）
* `downloader`: 用户下载器唯一识别 ID, 如果你的程序仅通过 URL 识别不同的下载器，请通过加盐哈希等手段将其匿名化
* `downloader_progress`: 用户下载器任务进度 （0.0 = 0%, 1.0 = 100%）
* `peer_ip`: Peer 的 IPv4 或 IPv6 地址
* `peer_port`: Peer 的端口号
* `peer_id`: Peer 的 Peer ID (请提供尽可能长的 PeerID，如遇 qB 此类会截断 PeerID 的下载器，提交阶段后的也可)
* `peer_client_name`: Peer 的客户端名称，如果下载器未提供，请提交空字符串
* `peer_progress`: 双精度浮点型的 Peer 任务进度 （0.0 = 0%, 1.0 = 100%）
* `from_peer_traffic`: 由远程 Peer 传输给用户下载器的数据量，需要程序自己计算真实的值（计算值）
* `to_peer_traffic`: 由用户下载器传输到远程 Peer 的数据量，需要程序自己计算真实的值（计算值）
* `from_peer_traffic_offset`: 本次会话由远程 Peer 传输给用户下载器的数据量，可以直接从下载器获取（原始值）
* `to_peer_traffic_offset`: 本次会话由用户下载器传输到远程 Peer 的数据量，可以直接从下载器获取（原始值）
* `first_time_seen`: 首次看到此 Peer 的 UNIX 毫秒时间戳
* `last_time_seen`: 上次看到此 Peer 的 UNIX 毫秒时间戳
* `peer_last_flags`: 该 Peer 最后一次获取到的 Flags 信息
* `download_speed`: （可选参数）用户下载器从远程 Peer 下载数据的速度
* `upload_speed`: （可选参数）远程 Peer 向用户下载器上传数据的速度
* `download_speed_max`: （可选参数）用户下载器从远程 Peer 下载数据的速度的历史最大值
* `upload_speed`: （可选参数）远程 Peer 向用户下载器上传数据的速度的历史最大值

提交方式：

向此能力给定的 endpoint 发送 POST 请求。请求体必须且只能使用 GZIP 压缩，不支持未压缩的传输。  
附加请求头：

* Content-Encoding: gzip

#### 响应

期望响应：  

* 200 - 服务器成功处理此请求

错误、重定向响应：  
请参见：[通用响应处理](https://github.com/PBH-BTN/BTN-Spec/blob/main/README.md#%E9%80%9A%E7%94%A8%E5%93%8D%E5%BA%94%E5%A4%84%E7%90%86)

### 心跳 `heartbeat`

该能力允许 BTN 网络请求 BTN 客户端 每隔一定时间向 BTN 服务器发送一个心跳请求，以更新在 BTN 网络上的状态。有时该功能还被用来检测 BTN 客户端的可用 IP 地址，以便更新在 BTN 网络上的记录。这取决于服务器的要求。

#### 配置

```json
{
    "config_key": "heartbeat",
    "endpoint": "https://sparkle.ghostchu.com/ping/heartbeat",
    "interval": 1800000,
    "multi_if": true,
    "pow_captcha": false,
    "random_initial_delay": 3000
}
```

* `interval`: 提交间隔（单位：毫秒）  
* `random_initial_delay`: 首次提交延迟随机偏移（单位：毫秒）。客户端首次提交应被计划在 `interval + random.nextLong(random_initial_delay)` 期间，以避免服务器出现请求处理尖峰，缓解服务器压力  
* `multi_if`: 是否使用系统上所有可用的网络接口都发送一次心跳请求

#### 请求

发送 POST 请求到端点，并附带下面的 body：

```json
{
    "ifaddr": "<系统网络接口名>"
}
```

* `系统网络接口名`: 发送此请求在系统上的标识符，如果使用默认接口发送(multi_if: false)，则此值固定为 `default`。

#### 响应

* 20x: 请求成功完成，可能为 200、204 等状态码

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

### 请求 IP 拒绝名单 `ip_denylist`

此端点提供正在被 BTN 封禁的 IP 地址和地址段。以一种更加内存高效的方式替代了原 rules (请求云端规则) 的 IP 黑名单集。请求时需要携带 `rev` 查询参数本地缓存的规则版本号。如果本地未缓存，此值固定为 `initial`；若没有规则更新，服务器将况返回 204 状态码，告知客户端规则未发生改变而节约传输带宽。

```json
{
    "endpoint": "https://sparkle.ghostchu.com/ping/ruleIpDenylist",
    "interval": 600000,
    "random_initial_delay": 15000,
    "pow_captcha": false,
    "config_key": "ip_denylist"
}
```

* `interval`: 提交间隔（单位：毫秒）  
* `random_initial_delay`: 首次提交延迟随机偏移（单位：毫秒）。客户端首次提交应被计划在 `interval + random.nextLong(random_initial_delay)` 期间，以避免服务器出现请求处理尖峰，缓解服务器压力  
* `endpoint`: 指定客户端应请求哪个端点获取规则信息

#### 响应

```plain
# IPV4
180.113.146.249
# IPV4 CIDR
127.0.0.1/24
# IPV6
2001:da8:1026:2f00::1
# IPV6 CIDR
2001:da8:1026:2f00::/56
# Multiple Comment
# Multiple Comment
# Multiple Comment
192.168.0.1
```

响应体为 IP 规则内容，注释以 `#` 或 `//` 开头。  
规则可能是 IPV4 或 IPV6 地址，以及它们的 CIDR 表达式。

响应的 HTTP 头会携带 `X-BTN-ContentVersion` 响应头，请保存此值，并在下一次请求中填入 `rev` 查询参数中。

### 请求 IP 允许名单 `ip_allowlist`

此端点提供正在被 BTN 允许的 IP 地址和地址段。以一种更加内存高效的方式替代了原 rules (请求云端规则) 的 IP 白名单集。请求时需要携带 `rev` 查询参数本地缓存的规则版本号。如果本地未缓存，此值固定为 `initial`；若没有规则更新，服务器将况返回 204 状态码，告知客户端规则未发生改变而节约传输带宽。



```json
{
    "endpoint": "https://sparkle.ghostchu.com/ping/ruleIpAllowlist",
    "interval": 600000,
    "random_initial_delay": 15000,
    "pow_captcha": false,
    "config_key": "ip_allowlist"
}
```

* `interval`: 提交间隔（单位：毫秒）  
* `random_initial_delay`: 首次提交延迟随机偏移（单位：毫秒）。客户端首次提交应被计划在 `interval + random.nextLong(random_initial_delay)` 期间，以避免服务器出现请求处理尖峰，缓解服务器压力  
* `endpoint`: 指定客户端应请求哪个端点获取规则信息

#### 响应

```plain
# IPV4
180.113.146.249
# IPV4 CIDR
127.0.0.1/24
# IPV6
2001:da8:1026:2f00::1
# IPV6 CIDR
2001:da8:1026:2f00::/56
# Multiple Comment
# Multiple Comment
# Multiple Comment
192.168.0.1
```

响应体为 IP 规则内容，注释以 `#` 或 `//` 开头。  
规则可能是 IPV4 或 IPV6 地址，以及它们的 CIDR 表达式。

响应的 HTTP 头会携带 `X-BTN-ContentVersion` 响应头，请保存此值，并在下一次请求中填入 `rev` 查询参数中。

### 请求云端规则 `rule_peer_identity`

此能力允许 BTN 兼容客户端从 BTN 实例上获取云端规则。

#### 配置

```json
{
    "endpoint": "https://sparkle.ghostchu.com/ping/rulePeerIdentity",
    "interval": 2700000,
    "random_initial_delay": 15000,
    "pow_captcha": false,
    "config_key": "rule_peer_identity"
}
```

* `interval`: 提交间隔（单位：毫秒）  
* `random_initial_delay`: 首次提交延迟随机偏移（单位：毫秒）。客户端首次提交应被计划在 `interval + random.nextLong(random_initial_delay)` 期间，以避免服务器出现请求处理尖峰，缓解服务器压力  
* `endpoint`: 指定客户端应请求哪个端点获取规则信息

#### 响应

期望响应：  

* 200 - 服务器成功处理此请求

错误、重定向响应：  
请参见：[通用响应处理](https://github.com/PBH-BTN/BTN-Spec/blob/main/README.md#%E9%80%9A%E7%94%A8%E5%93%8D%E5%BA%94%E5%A4%84%E7%90%86)

#### 请求

```
GET <endpoint>?rev=<rev>
```

* `rev`: 本地缓存的规则版本号。请求时需要携带 `rev` 查询参数本地缓存的规则版本号。如果本地未缓存，此值固定为 `initial`；若没有规则更新，服务器将况返回 204 状态码，告知客户端规则未发生改变而节约传输带宽。

#### 响应

```json
{
    "version": "1981c7af",
    "peer_id": {
        "hp/torrent 新变种": [
            "{\"method\":\"STARTS_WITH\",\"content\":\"-xm\"}"
        ]
    },
    "client_name": {
        "hp/torrent 新变种": [
            "{\"method\":\"STARTS_WITH\",\"content\":\"xm/torrent\"}"
        ],
        "BTN-全随机 PeerID (特征识别-粗略)": [
            "{\"method\":\"EQUALS\",\"content\":\"gopeed dev\"}"
        ]
    },
    "ip": {
        "BTN-多拨下载-2024-06-16": [
            "42.248.192.0/24",
            "..."
        ],
        "BTN-进度回退-2024-06-07": [
            "110.185.22.124/30",
            "..."
        ],
        "多拨黑名单": [
            "101.69.63.0/24",
            "..."
        ]
    },
    "port": {},
    "script": {}
}
```

* `rev`: 服务器返回的本次规则的版本号，在下次请求规则时应作为查询参数一同发送
* 其他内容可变，根据客户端和BTN服务端实现不同，此响应可能返回不同的响应结构，本示例中为 PBH （v1）结构。

期望响应：

* 200 - 服务器成功处理此请求
* 204 - 规则未发生更改

错误、重定向响应：  
请参见：[通用响应处理](https://github.com/PBH-BTN/BTN-Spec/blob/main/README.md#%E9%80%9A%E7%94%A8%E5%93%8D%E5%BA%94%E5%A4%84%E7%90%86)

此能力允许 BTN 兼容客户端从 BTN 实例上获取例外规则。命中例外规则的 Peer 将不会被封禁，已被封禁的 Peer 应解除封禁。它和请求云端规则非常相似，这并不是错误。  

#### 配置

```json
{
 "interval": 900000,
 "endpoint": "https://btn-dev-v2.ghostchu-services.top/ping/exception",
 "random_initial_delay": 5000
}
```

* `interval`: 提交间隔（单位：毫秒）  
* `random_initial_delay`: 首次提交延迟随机偏移（单位：毫秒）。客户端首次提交应被计划在 `interval + random.nextLong(random_initial_delay)` 期间，以避免服务器出现请求处理尖峰，缓解服务器压力  
* `endpoint`: 指定客户端应请求哪个端点获取规则信息

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
        String torrentInfoHandled = torrentInfoHash.toLowerCase(Locale.ROOT); // 转小写处理
        String salt = Hashing.crc32().hashString(torrentInfoHandled, StandardCharsets.UTF_8).toString(); // 使用 crc32 计算 info_hash 的哈希作为盐
        return Hashing.sha256().hashString(torrentInfoHandled + salt, StandardCharsets.UTF_8).toString(); // 在 info_hash 的明文后面追加盐后，计算 SHA256 的哈希值，结果应转全小写
    }
```

示例输入：`a5b24a285c3533d80ce62181813640ac4a0e6ed7`  
示例输出：`52fa13494a4571a951b46b1a04be19ab9d8089c3d3761956c99f5435e6f2c8ad`

中间过程生成的 salt 应为：`4063bf66` （大端序输出）  
如果您得到的结果是 `66bf6340`，则可能需要进行端序翻转：

```java
    public static byte[] flipBytes(byte[] a){
        byte[] b = new byte[a.length];
        for (int i = 0; i < b.length; i++) {
            b[i] = a[b.length - i - 1];
        }
        return b;
    }
```

## License

BTN-Spec 的文档和示例代码在 CC-0 (Public Domain) 协议下授权：

[![CC-0](https://mirrors.creativecommons.org/presskit/buttons/88x31/png/cc-zero.png)](https://creativecommons.org/publicdomain/zero/1.0/)

## 目前接入 BTN 协议的客户端列表

* [PeerBanHelper](https://github.com/PBH-BTN/PeerBanHelper)
