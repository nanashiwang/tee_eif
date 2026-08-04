# AWS Nitro Enclave PoO Parent Gateway 第三方中转站全链路部署文档

本文面向第三方中转站研发和 SRE，说明如何在 AWS Nitro Enclave 平台部署
Proof-of-Observation 链路。本文假设 EIF 文件已经构建完成，不包含重新构建 EIF
或复现 PCR 的步骤。

## 交付物清单

开始部署前，第三方应先拿到以下材料：

| 材料 | 用途 |
|---|---|
| 已构建 EIF 文件，例如 `proof.eif` | 部署到 Nitro Enclave |
| EIF SHA256 | 安装后校验文件是否被替换或损坏 |
| 预期 PCR0 / trust metadata | 给 verifier 或信任元数据发布使用 |
| EIF 对应源码 commit / 构建版本 | 审计、回滚和后续复现构建使用 |
| Gateway 源码分支、tag、commit 或发布 binary | 部署 PoO Parent Gateway |
| 父 VM 私网 IP / DNS | 远程中转站访问 Gateway 时使用 |
| 上游 allowlist，例如 `dashscope.aliyuncs.com:443` | 配置 Gateway 出网上游准入 |
| mTLS CA / client cert / client key | 远程中转站访问 Gateway 时使用；同机 loopback 模式不需要 |

推荐实际执行顺序：

1. 准备 EC2 父 VM 和 Nitro Enclave 运行环境。
2. 部署已构建好的 EIF，并启动 Enclave。
3. 部署 PoO Parent Gateway。
4. 改造并启动中转站。
5. 发起端到端验收请求，确认响应包含 `proof`。

本文按交付对象分为四部分：

1. 整体链路
2. 中转站改造部署
3. Gateway 部署
4. EIF 部署

注意：正文为了方便不同团队分工，把中转站、Gateway、EIF 分成独立章节；它不是严格执行顺序。
首次部署时请先完成第 4 节 EIF 部署，并用固定 `EnclaveCID=16` 启动 Enclave，
再执行第 3 节 Gateway 部署。

## 1. 整体链路

### 1.1 整体架构图

```mermaid
flowchart LR
  client["Client / SDK<br/>发起模型请求"]
  relay["第三方中转站<br/>鉴权、模型/账号/代理选择<br/>注入 JSON proof / SSE proof"]
  gateway["PoO Parent Gateway<br/>接收 PoO 请求<br/>连接 Enclave 并管理出网"]
  enclave["AWS Nitro Enclave<br/>观察上游请求/响应<br/>生成 proof"]
  upstream["上游模型服务<br/>OpenAI / Claude / DashScope 等"]
  verifier["Verifier / trust metadata<br/>旁路校验 proof"]

  client -->|"业务请求"| relay
  relay -->|"PoO 请求<br/>含可选代理配置"| gateway
  gateway -->|"送入 TEE 观察"| enclave
  enclave -->|"请求经 Gateway egress 出网"| upstream
  upstream -->|"模型响应"| enclave
  enclave -->|"响应 + proof"| gateway
  gateway -->|"返回给中转站"| relay
  relay -->|"模型响应 + proof"| client
  client -.->|"proof + trust metadata"| verifier
```

第三方中转站可以和 Gateway 部署在同一台父 VM，也可以远程部署；同机模式使用
loopback，远程模式使用 HTTPS + mTLS。Gateway 和 Enclave 之间、Enclave 出网 lane、
固定 Enclave CID、端口和 readiness 等细节，在第 3、4 节展开。

图中主要构成部分和后续部署章节的对应关系：

| 构成部分 | 后续章节 | 说明 |
|---|---|---|
| 第三方中转站 | 第 2 节 | 接入 Gateway、传递代理配置、处理非流式和流式 proof 输出 |
| PoO Parent Gateway | 第 3 节 | 部署 Gateway、配置 mTLS、allowlist、出网链路和 readiness |
| AWS Nitro Enclave / `proof.eif` | 第 4 节 | 安装已构建 EIF，固定 CID 启动并管理 Enclave |
| 上游模型服务和可选代理 | 第 2.7、3.5、3.6 节 | 明确实际上游 `host:443`、代理传递方式和 Gateway 侧准入 |
| Client / Verifier / trust metadata | 第 1.6、2.11、4.3、5 节 | Client 接收 proof；Verifier 使用 proof、trust metadata 和 EIF 发布记录做旁路校验 |

### 1.2 组件职责

| 组件 | 部署位置 | 职责 |
|---|---|---|
| 第三方中转站 | 父 VM 本机或独立机器 | 接收用户 OpenAI-compatible / Claude / Responses 请求，选择账号和代理，把上游 HTTP 请求封装成 PoO frame，调用 Parent Gateway |
| PoO Parent Gateway | AWS Nitro 父 VM | 暴露 `POST /v1/proof/relay`，连接 Enclave，分配动态 egress lane，按请求处理 direct / HTTP CONNECT / SOCKS5 出网 |
| Nitro Enclave / EIF | AWS Nitro Enclave | 在 TEE 内观察上游请求和响应，生成字段级 proof，通过 `RESP_TRAILER` 返回 |
| 上游模型服务 | 公网或代理后方 | 例如 `api.openai.com:443`、`api.anthropic.com:443`、`dashscope.aliyuncs.com:443` |

### 1.3 无代理链路

```text
Client
  -> 第三方中转站
  -> POST /v1/proof/relay
  -> PoO Parent Gateway
  -> AF_VSOCK <EnclaveCID>:5005
  -> Nitro Enclave PoO runtime
  -> AF_VSOCK CID=3, dynamic lane port 18000-18999
  -> PoO Parent Gateway egress
  -> upstream:443
```

### 1.4 有代理链路

```text
Client
  -> 第三方中转站
  -> POST /v1/proof/relay
     Header: X-PoO-Proxy-URL: socks5h://user:pass@proxy.example.com:1080
  -> PoO Parent Gateway
  -> AF_VSOCK <EnclaveCID>:5005
  -> Nitro Enclave PoO runtime
  -> AF_VSOCK CID=3, dynamic lane port 18000-18999
  -> PoO Parent Gateway egress
  -> proxy
  -> upstream:443
```

`X-PoO-Proxy-URL` 是敏感信息，可能包含代理用户名和密码。禁止把该 header
原值写入 access log、业务日志、APM、错误响应或审计事件。

### 1.5 端口规划

| 端口 / 地址 | 默认值 | 方向 | 用途 |
|---|---:|---|---|
| Proof Relay API | `15005` | 中转站 -> Gateway | `POST /v1/proof/relay` |
| Admin API | `127.0.0.1:15007` | 运维 -> Gateway | `/healthz`、`/readyz`、`/metrics` |
| Enclave relay vsock | `5005` | Gateway -> Enclave | PoO frame control stream |
| Enclave metrics vsock | `5006` | Gateway -> Enclave | readiness / metrics probe |
| Egress lane ports | `18000-18999` | Enclave -> Gateway | 单请求动态 egress lane |

生产环境建议固定 Enclave CID，例如：

```bash
POO_ENCLAVE_CID=16
```

Nitro CLI 的 `run-enclave --enclave-cid` 只允许指定 `4` 或更大的 CID。本文后续命令统一使用
`16`；如果现场已经占用该 CID，可以选择另一个 `>=4` 的值，但 EIF 启动参数和 Gateway
`POO_PARENT_VSOCK_CID` 必须始终一致。

生产环境使用：

```bash
POO_EGRESS_LANE_LISTEN_MODE=direct-vsock
```

不要再启动旧静态 `vsock-proxy 8444 dashscope.aliyuncs.com 443` 这类服务。
旧静态 proxy 会和动态 lane 架构混淆，且不能支持每次请求携带不同代理。

### 1.6 响应 proof 形态

非流式响应建议在最终 JSON 顶层注入：

```json
{
  "id": "resp_xxx",
  "object": "response",
  "output": [],
  "proof": {
    "v": 2,
    "alg": "ed25519"
  }
}
```

上面的 `proof` 只用于展示注入位置，不代表实际 proof schema。实际字段、版本和
校验规则以随 `proof.eif` 一起交付的 trust metadata、PoO 版本和 verifier 规范为准。

流式响应建议在上游 SSE 内容和 `data: [DONE]` 后追加一个事件：

```text
event: tee.proof
data: {"v":2,"alg":"ed25519", "...":"..."}
```

如果 PoO 是生产强制能力，Gateway 不可用、proof 缺失或 proof schema 不合法时，
中转站应 fail closed，不应静默回退直连。

## 2. 中转站改造部署

### 2.1 改造目标

第三方中转站原链路通常是：

```text
中转站 -> upstream
中转站 -> proxy -> upstream
```

改造后：

```text
中转站 -> PoO Parent Gateway -> Enclave -> Gateway egress -> upstream
中转站 -> PoO Parent Gateway -> Enclave -> Gateway egress -> proxy -> upstream
```

中转站不需要知道 Enclave CID、`5005`、`5006` 或 `18000-18999` lane 端口。
这些都是 Gateway 内部实现细节。

### 2.2 推荐接入边界

建议把 PoO 接在中转站“统一上游 HTTP 执行层”，而不是散落到每个模型接口或
provider 里：

```text
handler / router
  -> 鉴权、模型映射、账号选择、代理选择
  -> 为本次请求创建 request-scoped proof recorder
  -> provider executor 按原方式构造普通上游 HTTP request
  -> 统一 HTTP client / transport wrapper 调用 PoO Parent Gateway
  -> handler 注入 JSON proof 或追加 SSE proof
```

这样改造后，原有 provider executor 不需要理解 PoO frame，也不需要知道 Gateway
协议细节。Go 中转站可以用 `RoundTripper + recorder` 模式；Node.js 中转站可以在统一
HTTP client、fetch/axios adapter 或 gateway client 层实现等价 wrapper。

需要逐项确认：

- 所有需要 PoO 的 provider、插件 executor、自定义 HTTP client、uTLS client 都经过同一个 wrapper。
- PoO 启用时，本地 transport 不再直接访问上游；最终代理 URL 只传给 Gateway。
- 未命中 PoO 策略或 `enabled=false` 的请求，仍保持原 direct / local-proxy 链路。
- proof 输出只在 handler/response 层做，避免 provider 层直接拼接下游协议。

### 2.3 错误、回退和防重复提交语义

生产推荐 `required=true`，即 Gateway 不可用、proof 缺失、proof schema 不合法或
PoO frame 错误时 fail closed。

如果灰度阶段使用 `required=false`，只能在“确认请求尚未提交给 Gateway / Enclave /
上游”时回退原链路。实现上建议让 PoO SDK 或 wrapper 返回类似 `Submitted` 的状态：

| 场景 | Submitted | 建议行为 |
|---|---:|---|
| Gateway 配置错误、URL 不合法、请求体超过本地限制 | `false` | `required=false` 可回退原链路；`required=true` 返回错误 |
| Gateway 请求已经写出后网络断开 | `true` | 返回本次请求错误，不换账号、不换模型、不 fallback |
| Gateway HTTP relay endpoint 返回非 2xx，或 Gateway frame 协议错误 | `true` | 返回本次请求错误，不触发账号 cooldown |
| Gateway 返回 frame response，且 `RESP_HEAD` 中的上游 status 非 2xx | `true` | 按上游错误语义处理；如果有完整 proof，最终错误 JSON 顶层也注入 proof |
| 非流式缺 proof | `true` | 生产返回错误；不要把无 proof 响应伪装成已验证 |
| 流式模型内容已写出后 proof 出错 | `true` | 追加 `event: tee.error` 后关闭连接 |

一旦请求可能已经提交到 Gateway、Enclave 或上游模型，就不能再做 credential fallback、
model pool fallback 或换代理重试，否则同一个用户请求可能被重复提交给多个上游账号。
如果响应头或模型内容已经写给客户端，也不能再改 HTTP status，只能按当前流式协议结束
或追加 `tee.error`。

### 2.4 接口范围

可以按业务选择接入范围。常见建议：

- `/openai/v1/chat/completions`
- `/openai/v1/responses`
- `/openai/responses`
- Claude Messages / Anthropic-compatible 接口

如果第三方中转站做了模型映射，例如用户请求 `qwen3.7-plus`，最终上游目标是
`dashscope.aliyuncs.com:443`，则 PoO frame 中的 upstream host 必须是实际访问的
上游 host。

### 2.5 Gateway 客户端配置

在中转站配置中增加：

```js
pooParentGateway: {
  enabled: process.env.POO_PARENT_GATEWAY_ENABLED === 'true',
  required: process.env.POO_PARENT_GATEWAY_REQUIRED !== 'false',
  url: process.env.POO_PARENT_GATEWAY_URL || 'http://127.0.0.1:15005/v1/proof/relay',
  authMode: process.env.POO_PARENT_GATEWAY_AUTH_MODE || 'none',
  timeoutMs: parseInt(process.env.POO_PARENT_GATEWAY_TIMEOUT_MS) || 600000,
  maxBodyBytes: parseInt(process.env.POO_PARENT_GATEWAY_MAX_BODY_BYTES) || 64 * 1024 * 1024,
  mtls: {
    caFile: process.env.POO_PARENT_GATEWAY_CA_FILE || '',
    certFile: process.env.POO_PARENT_GATEWAY_CERT_FILE || '',
    keyFile: process.env.POO_PARENT_GATEWAY_KEY_FILE || '',
    servername: process.env.POO_PARENT_GATEWAY_SERVER_NAME || ''
  }
}
```

父 VM 本机中转站 `.env`：

```bash
POO_PARENT_GATEWAY_ENABLED=true
POO_PARENT_GATEWAY_REQUIRED=true
POO_PARENT_GATEWAY_URL=http://127.0.0.1:15005/v1/proof/relay
POO_PARENT_GATEWAY_AUTH_MODE=none
POO_PARENT_GATEWAY_TIMEOUT_MS=600000
POO_PARENT_GATEWAY_MAX_BODY_BYTES=67108864
```

远程中转站 `.env`：

```bash
POO_PARENT_GATEWAY_ENABLED=true
POO_PARENT_GATEWAY_REQUIRED=true
POO_PARENT_GATEWAY_URL=https://<parent_vm_private_dns_or_ip>:15005/v1/proof/relay
POO_PARENT_GATEWAY_AUTH_MODE=mtls
POO_PARENT_GATEWAY_CA_FILE=/etc/poo-parent-gateway/client-ca.pem
POO_PARENT_GATEWAY_CERT_FILE=/etc/poo-parent-gateway/relay-client.pem
POO_PARENT_GATEWAY_KEY_FILE=/etc/poo-parent-gateway/relay-client-key.pem
POO_PARENT_GATEWAY_SERVER_NAME=poo-parent-gateway.internal
POO_PARENT_GATEWAY_TIMEOUT_MS=600000
POO_PARENT_GATEWAY_MAX_BODY_BYTES=67108864
```

约束：

- `authMode=none` 只能用于 `http://127.0.0.1`、`http://localhost`、`http://[::1]`。
- 远程访问必须使用 HTTPS + mTLS，并用安全组 / 防火墙限制来源。
- `required=true` 是生产推荐值。

如果中转站采用 YAML 配置，也可以表达为：

```yaml
poo-parent-gateway:
  enabled: true
  required: true
  url: "http://127.0.0.1:15005/v1/proof/relay"
  auth-mode: "none"
  timeout: 10m
  max-body-bytes: 67108864
```

### 2.6 请求封装协议

中转站向 Gateway 发起：

```text
POST /v1/proof/relay
Content-Type: application/vnd.poo.frames
X-PoO-Proxy-URL: optional proxy URL
X-PoO-Tenant-ID: optional tenant id
X-PoO-Account-ID: optional account id
X-PoO-Request-ID: optional request id

<REQ_HEAD frame><REQ_BODY frame>
```

Frame 格式：

```text
1 byte frame_type + 4 bytes big-endian payload_length + payload
```

Frame type：

| 名称 | 值 | 方向 | 说明 |
|---|---:|---|---|
| `REQ_HEAD` | `0x01` | 中转站 -> Gateway / Enclave | JSON metadata，包含 upstream、headers、token 等 |
| `REQ_BODY` | `0x02` | 中转站 -> Gateway / Enclave | 最终上游请求体 bytes；空 body 使用 zero-length frame |
| `RESP_HEAD` | `0x10` | Enclave / Gateway -> 中转站 | JSON metadata，包含上游 status 和 headers |
| `RESP_CHUNK` | `0x11` | Enclave / Gateway -> 中转站 | 上游响应 body bytes，可出现多次 |
| `RESP_TRAILER` | `0x12` | Enclave / Gateway -> 中转站 | JSON trailer，携带 proof |
| `ERR` | `0x1f` | Enclave / Gateway -> 中转站 | Enclave 协议错误或 TEE 内错误 |

合法请求序列必须是：

```text
REQ_HEAD + REQ_BODY + EOF
```

v1 请求侧只允许一个 `REQ_HEAD` 和一个 `REQ_BODY`。不能追加第二个 `REQ_BODY`、
`RESP_*` frame、未知 frame type 或尾随字节。Gateway 只会改写首个
`REQ_HEAD.egress_port`，不会改写 upstream、headers、nonce 或 body bytes。

`REQ_HEAD` 的核心字段：

```json
{
  "nonce": "<random request nonce>",
  "egress_port": 0,
  "upstream": {
    "host": "dashscope.aliyuncs.com",
    "method": "POST",
    "path": "/compatible-mode/v1/chat/completions",
    "headers": {
      "content-type": "application/json"
    },
    "headersOrdered": [
      ["Host", "dashscope.aliyuncs.com"],
      ["Content-Type", "application/json"],
      ["Authorization", ""],
      ["Content-Length", ""]
    ]
  },
  "token": "<upstream-api-key>"
}
```

说明：

- `egress_port` 由 Gateway 改写，中转站传 `0` 或占位值即可。
- `upstream.host` 只能是上游 `host` 或 `host:443`，v1 只支持 HTTPS 443；不要把 scheme、path 或非 443 端口放进 host。
- `headers` 是普通 JSON object，用于兼容基础解析。
- 如需保留出站 header 顺序和大小写，使用 `headersOrdered`；Enclave 会优先使用它。
- `headersOrdered` 中 `Authorization` 和 `Content-Length` 可以传空字符串哨兵值，Enclave 会分别用 `token` 和实际 body length 填充。
- 不应转发 hop-by-hop headers，例如 `connection`、`transfer-encoding`、
  `keep-alive`、`proxy-connection`、`upgrade`。
- 请求 body 放入 `REQ_BODY`。

### 2.7 代理配置传递

中转站账号已有代理配置时，转换为 URL 后放入 `X-PoO-Proxy-URL`：

```text
http://host:port
https://host:port
socks5://host:port
socks5h://host:port
http://user:pass@host:port
socks5h://user:pass@host:port
```

没有代理时不发送该 header。

Gateway 不会证明“使用了哪个代理”。proxy URL 只决定本次网络出口，不进入 proof
statement。

### 2.8 非流式处理

中转站收到 Gateway frame response 后：

1. 读取 `RESP_HEAD`，得到上游 HTTP status 和 headers。
2. 拼接所有 `RESP_CHUNK` 得到上游 body。
3. 读取 `RESP_TRAILER` 得到 proof。
4. 按现有业务逻辑做协议转换。
5. 在最终 JSON 顶层增加 `proof` 字段。

如果 `RESP_HEAD` 中的上游 status 是非 2xx，但 Gateway / Enclave 已经返回完整
`RESP_TRAILER`，仍应把 proof 注入最终错误 JSON envelope 顶层。不要只因为上游
status 非 2xx 就丢弃 proof；账号保护、限流、cooldown 等业务逻辑继续按原有上游
错误语义执行。

无法把 proof 注入 JSON 时，建议返回中转站自己的 `502 upstream invalid response`，
不要返回 `multipart/mixed`，很多 OpenAI-compatible 客户端不兼容。

如果在 `RESP_TRAILER` 前收到 `ERR` frame、连接关闭或 frame 解析失败，按 PoO
错误处理；生产 `required=true` 时不能把无 proof 的模型响应返回为已验证响应。

### 2.9 流式处理

流式请求建议：

1. 收到 `RESP_HEAD` 后设置客户端响应头。
2. 原样转发 `RESP_CHUNK` 中的 SSE 字节。
3. 等 `RESP_TRAILER` 到达。
4. 若上游已发送 `data: [DONE]`，在之后追加 `event: tee.proof`。
5. 如果 PoO 出错且已经开始返回内容，追加 `event: tee.error` 后关闭连接。

不要在收到上游 `[DONE]` 后立刻 `res.end()`，否则会丢失最后的 proof trailer。
如果收到 `ERR` frame 或连接在 `RESP_TRAILER` 前断开，且已经向客户端写出模型内容，
只能追加 `tee.error` 或按原流式错误协议结束，不能再 fallback 到其他账号或模型。

### 2.10 中转站部署和重启示例

以下以 Node.js 中转站为例：

```bash
cd /opt/relay-service
git fetch origin
git checkout <poo-enabled-branch-or-tag>
git pull --ff-only origin <poo-enabled-branch-or-tag>

npm ci --omit=dev
npm test
```

写入环境变量：

```bash
sudo install -d -m 0755 /etc/relay-service
sudo tee /etc/relay-service/poo.env >/dev/null <<'EOF'
POO_PARENT_GATEWAY_ENABLED=true
POO_PARENT_GATEWAY_REQUIRED=true
POO_PARENT_GATEWAY_URL=http://127.0.0.1:15005/v1/proof/relay
POO_PARENT_GATEWAY_AUTH_MODE=none
POO_PARENT_GATEWAY_TIMEOUT_MS=600000
POO_PARENT_GATEWAY_MAX_BODY_BYTES=67108864
EOF
```

如果使用 systemd，确保 service 引入该环境文件：

```ini
EnvironmentFile=-/etc/relay-service/poo.env
```

重启：

```bash
sudo systemctl daemon-reload
sudo systemctl restart relay-service
sudo systemctl status relay-service --no-pager
```

如果是当前 `claude-relay-service` 的 npm daemon 方式：

```bash
cd /opt/relay-service
npm run service:stop || true
npm run service:start:daemon
```

### 2.11 中转站验收请求

非流式 OpenAI Responses 示例：

```bash
curl -sS -X POST "http://<relay_host>:<relay_port>/openai/v1/responses" \
  -H "Authorization: Bearer <relay_api_key>" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen3.7-plus",
    "input": "用一句话回答：PoO 链路是否正常？",
    "stream": false
  }' | jq .
```

期望最终 JSON 顶层包含：

```text
.proof
```

流式示例：

```bash
curl -N -X POST "http://<relay_host>:<relay_port>/openai/v1/responses" \
  -H "Authorization: Bearer <relay_api_key>" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen3.7-plus",
    "input": "用一句话回答：PoO 流式链路是否正常？",
    "stream": true
  }'
```

期望末尾出现：

```text
event: tee.proof
data: {...}
```

### 2.12 中转站改造验收清单

第三方中转站完成代码接入后，至少验证：

- `enabled=false` 时保持原 direct / local-proxy 链路。
- `enabled=true` 且请求命中业务策略时，上游请求会打到 Parent Gateway。
- 普通 HTTP client、uTLS client、插件 provider 和自定义 client 都不会绕过 PoO wrapper。
- 账号代理、请求级代理和全局代理优先级正确；`direct` / `none` 不发送 `X-PoO-Proxy-URL`。
- 非流式 JSON object 顶层能注入 `proof`，已有顶层 `proof` 时会被本次 PoO proof 覆盖。
- 非流式响应不是 JSON object 时，返回明确错误，不返回客户端不兼容的 `multipart/mixed`。
- 流式 SSE 在原协议终止帧之后追加 `event: tee.proof`。
- 流式模型内容已写出后 proof 出错，追加 `event: tee.error`，不切换账号或模型。
- 收到 `ERR` frame 或 `RESP_TRAILER` 前断连时，按 PoO 错误处理，不返回伪验证响应。
- PoO 请求提交前失败且 `required=false` 时，可以回退原链路。
- PoO 请求提交后失败时，不进行 credential fallback、model pool fallback 或账号 cooldown。
- 日志中不会输出完整 `proxyURL`、代理 userinfo、`Authorization` 或上游 API key。

## 3. Gateway 部署

以下命令均在 AWS Nitro 父 VM 上执行。

Gateway GitHub 仓库：

```text
https://github.com/Demo-9876/tcp-to-vsock-gateway
```

### 3.1 前置条件

Gateway 依赖正在运行的 Enclave CID。首次部署时，如果还没有启动 Enclave，请先执行
第 4 节 EIF 部署，再回到本节。

确认 Enclave 已启动并记录 CID：

```bash
sudo nitro-cli describe-enclaves
```

示例输出字段：

```json
{
  "EnclaveName": "proof",
  "EnclaveCID": 16,
  "State": "RUNNING"
}
```

后续命令设置：

```bash
export POO_ENCLAVE_NAME=proof
export ENCLAVE_CID=16
echo "ENCLAVE_CID=${ENCLAVE_CID}"

sudo nitro-cli describe-enclaves | jq -e \
  --arg name "$POO_ENCLAVE_NAME" \
  --argjson cid "$ENCLAVE_CID" \
  '.[] | select(.EnclaveName == $name and .State == "RUNNING" and .EnclaveCID == $cid)'
```

如果该命令没有匹配结果，说明 Enclave 未启动、名字不对，或没有用固定 CID `16`
启动。先回到第 4 节按固定 CID 启动 EIF，再继续部署 Gateway。

确认父 VM 能访问上游：

```bash
curl -I --connect-timeout 10 https://dashscope.aliyuncs.com/ || true
curl -I --connect-timeout 10 https://api.openai.com/ || true
curl -I --connect-timeout 10 https://api.anthropic.com/ || true
```

确认没有旧静态 `vsock-proxy`：

```bash
ps -ef | egrep 'vsock-proxy|8443|8444|8445|egress'
systemctl list-units --type=service --all | egrep 'egress|vsock|dashscope' || true
```

如果看到旧服务，例如 `tee-egress-dashscope.service`：

```bash
sudo systemctl disable --now tee-egress-dashscope.service
```

### 3.2 安装 Go 和工具

Amazon Linux：

```bash
sudo dnf update -y
sudo dnf install -y git curl ca-certificates gcc make jq
```

如果系统没有 `dnf`：

```bash
sudo yum install -y git curl ca-certificates gcc make jq
```

Gateway `go.mod` 要求 Go `1.24` 或更高版本：

```bash
go version
```

如需手动安装 Go，请从内部制品库或 Go 官方下载对应架构 tarball，并先校验
SHA256。示例：

```bash
set -euo pipefail

cd /tmp
GO_VERSION=1.24.3
GO_TARBALL_SHA256="<replace-with-expected-sha256>"
ARCH=$(uname -m)

case "$ARCH" in
  x86_64) GO_ARCH=amd64 ;;
  aarch64) GO_ARCH=arm64 ;;
  *) echo "unsupported arch: $ARCH" >&2; exit 1 ;;
esac

GO_TARBALL="go${GO_VERSION}.linux-${GO_ARCH}.tar.gz"

if [ "$GO_TARBALL_SHA256" = "<replace-with-expected-sha256>" ]; then
  echo "set GO_TARBALL_SHA256 before installing Go" >&2
  exit 1
fi

curl -fLO "https://go.dev/dl/${GO_TARBALL}"
printf '%s  %s\n' "$GO_TARBALL_SHA256" "$GO_TARBALL" | sha256sum -c -

sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "$GO_TARBALL"
echo 'export PATH=/usr/local/go/bin:$PATH' | sudo tee /etc/profile.d/go.sh >/dev/null
export PATH=/usr/local/go/bin:$PATH

go version
```

### 3.3 拉取和构建 Gateway

必须部署包含 direct-vsock accept 修复的版本，即 commit `5ac85a1` 或之后的版本。

```bash
sudo mkdir -p /opt/poo
sudo chown "$USER":"$USER" /opt/poo

cd /opt/poo
git clone https://github.com/Demo-9876/tcp-to-vsock-gateway.git
cd tcp-to-vsock-gateway

GIT_REF=docs/poo-parent-gateway-request-proxy-design
git fetch origin "$GIT_REF"
git checkout "$GIT_REF"
git pull --ff-only origin "$GIT_REF"

git rev-parse --short=12 HEAD
git merge-base --is-ancestor 5ac85a1 HEAD
```

最后一行必须返回成功。如果失败，说明当前 checkout 不包含 direct-vsock accept 修复，
不要继续构建部署。

如果第三方具备 GitHub SSH 权限，也可以把 clone 地址替换为：

```bash
git clone git@github.com:Demo-9876/tcp-to-vsock-gateway.git
```

构建前测试：

```bash
go test ./...
```

构建：

```bash
VERSION=poo-parent-gateway-$(date -u +%Y%m%d%H%M%S)
COMMIT=$(git rev-parse --short=12 HEAD)
BUILD_TIME=$(date -u +%Y-%m-%dT%H:%M:%SZ)

mkdir -p dist
go build \
  -trimpath \
  -ldflags="-s -w -X main.version=${VERSION} -X main.commit=${COMMIT} -X main.buildTime=${BUILD_TIME}" \
  -o dist/tcp-to-vsock-gateway \
  ./cmd/tcp-to-vsock-gateway

./dist/tcp-to-vsock-gateway --version
sha256sum ./dist/tcp-to-vsock-gateway
```

### 3.4 安装目录和运行用户

```bash
sudo useradd --system --no-create-home --shell /sbin/nologin poo-gateway 2>/dev/null || true

sudo mkdir -p /etc/poo-parent-gateway
sudo mkdir -p /var/lib/poo-parent-gateway/spool
sudo mkdir -p /var/log/poo-parent-gateway

sudo chown -R poo-gateway:poo-gateway /var/lib/poo-parent-gateway
sudo chown -R poo-gateway:poo-gateway /var/log/poo-parent-gateway
sudo chmod 0700 /var/lib/poo-parent-gateway/spool

sudo install -o root -g root -m 0755 \
  /opt/poo/tcp-to-vsock-gateway/dist/tcp-to-vsock-gateway \
  /usr/local/bin/tcp-to-vsock-gateway

/usr/local/bin/tcp-to-vsock-gateway --version
```

### 3.5 父 VM 本机中转站配置

如果中转站与 Gateway 同机，使用 loopback + `authMode=none`：

```bash
export ENCLAVE_CID=16
echo "ENCLAVE_CID=${ENCLAVE_CID}"

sudo tee /etc/poo-parent-gateway/env >/dev/null <<EOF
POO_PARENT_PROOF_RELAY_LISTEN=127.0.0.1:15005
POO_PARENT_LEGACY_INGRESS_LISTEN=
POO_PARENT_ADMIN_LISTEN=127.0.0.1:15007
POO_PARENT_AUTH_MODE=none
POO_PARENT_CLIENT_POLICY_FILE=

POO_PARENT_VSOCK_CID=${ENCLAVE_CID}
POO_PARENT_VSOCK_PORT=5005
POO_PARENT_VSOCK_METRICS_PORT=5006
POO_PARENT_SHUTDOWN_TIMEOUT_MS=300000
POO_PARENT_LOG_LEVEL=info

POO_EGRESS_LANE_LISTEN_MODE=direct-vsock
POO_EGRESS_PORT_RANGE=18000-18999
POO_EGRESS_PORT_COOLDOWN_MS=5000
POO_EGRESS_ROUTE_IDLE_TTL_MS=300000
POO_EGRESS_LEASE_TTL_MS=30000
POO_EGRESS_DEFAULT_CONNECT_TIMEOUT_MS=10000
POO_EGRESS_MAX_ACTIVE_ROUTES=1000
POO_EGRESS_MAX_ACTIVE_LEASES=4096
POO_EGRESS_DEFAULT_ROUTE_CONCURRENCY=1
POO_EGRESS_MAX_ROUTE_CONCURRENCY=16
POO_EGRESS_ALLOWED_TARGETS=dashscope.aliyuncs.com:443,api.openai.com:443,api.anthropic.com:443

POO_RELAY_MAX_METADATA_BYTES=16384
POO_RELAY_MAX_REQ_HEAD_BYTES=1048576
POO_RELAY_MAX_FRAME_BYTES=67108864
POO_RELAY_MAX_REQUEST_BYTES=268435456
POO_RELAY_MAX_BUFFERED_BYTES=4194304
POO_RELAY_SPOOL_DIR=/var/lib/poo-parent-gateway/spool
POO_RELAY_MAX_SPOOL_BYTES=1073741824
POO_RELAY_IO_TIMEOUT_MS=300000

TTVG_CONNECT_TIMEOUT=5s
TTVG_IDLE_TIMEOUT=300s
TTVG_READY_CACHE_TTL=1s
TTVG_MAX_CONNS=1024
TTVG_TCP_KEEPALIVE=30s
EOF
```

如需允许更多上游，把 `POO_EGRESS_ALLOWED_TARGETS` 增加为逗号分隔的
`host:443` 列表。

`POO_EGRESS_ALLOWED_TARGETS` 是 Gateway 全局强制 allowlist。它不配置，或显式配置为空：

```bash
POO_EGRESS_ALLOWED_TARGETS=
```

时，Gateway 会使用内置默认值：

```text
api.openai.com:443
api.anthropic.com:443
generativelanguage.googleapis.com:443
aiplatform.googleapis.com:443
api.mistral.ai:443
api.cohere.ai:443
api.together.xyz:443
openrouter.ai:443
api.deepseek.com:443
api.x.ai:443
api.groq.com:443
api.perplexity.ai:443
api.fireworks.ai:443
api.replicate.com:443
api.novita.ai:443
api.sambanova.ai:443
api.ai21.com:443
api.cerebras.ai:443
api-inference.huggingface.co:443
router.huggingface.co:443
dashscope.aliyuncs.com:443
ark.cn-beijing.volces.com:443
ark.cn-shanghai.volces.com:443
open.bigmodel.cn:443
api.moonshot.cn:443
api.minimax.chat:443
api.stepfun.com:443
api.baichuan-ai.com:443
api.lingyiwanwu.com:443
api.siliconflow.cn:443
hunyuan.tencentcloudapi.com:443
qianfan.baidubce.com:443
```

线上可以用当前 binary 打印实际内置列表，避免文档和程序版本漂移：

```bash
/usr/local/bin/tcp-to-vsock-gateway --print-default-targets
```

### 3.6 远程中转站 mTLS 配置

如果中转站不在父 VM 本机，Proof Relay API 需要绑定父 VM 私网地址并启用 mTLS。

生成示例证书：

```bash
mkdir -p /tmp/poo-mtls
cd /tmp/poo-mtls

openssl genrsa -out client-ca.key 4096
openssl req -x509 -new -nodes \
  -key client-ca.key \
  -sha256 \
  -days 3650 \
  -subj "/CN=poo-parent-gateway-client-ca" \
  -out client-ca.pem

openssl genrsa -out server-key.pem 2048
openssl req -new \
  -key server-key.pem \
  -subj "/CN=poo-parent-gateway" \
  -out server.csr

cat > server-ext.cnf <<EOF
subjectAltName = IP:<parent_vm_private_ip>,DNS:poo-parent-gateway.internal
extendedKeyUsage = serverAuth
EOF

openssl x509 -req \
  -in server.csr \
  -CA client-ca.pem \
  -CAkey client-ca.key \
  -CAcreateserial \
  -out server.pem \
  -days 825 \
  -sha256 \
  -extfile server-ext.cnf

openssl genrsa -out relay-client-key.pem 2048
openssl req -new \
  -key relay-client-key.pem \
  -subj "/CN=relay-prod" \
  -out relay-client.csr

cat > relay-client-ext.cnf <<EOF
subjectAltName = URI:spiffe://poo/relay-prod
extendedKeyUsage = clientAuth
EOF

openssl x509 -req \
  -in relay-client.csr \
  -CA client-ca.pem \
  -CAkey client-ca.key \
  -CAcreateserial \
  -out relay-client.pem \
  -days 825 \
  -sha256 \
  -extfile relay-client-ext.cnf
```

上面是最小可运行示例：同一个 CA 同时签发 Gateway server cert 和 relay client cert。
生产环境可以拆分 server CA 和 client CA；无论采用哪种方式，CA 私钥都不应长期留在父 VM
或中转站机器上，证书签发完成后应转移到受控密钥管理位置。

安装 Gateway 侧证书：

```bash
sudo install -o root -g root -m 0644 client-ca.pem /etc/poo-parent-gateway/client-ca.pem
sudo install -o root -g root -m 0644 server.pem /etc/poo-parent-gateway/server.pem
sudo install -o root -g poo-gateway -m 0640 server-key.pem /etc/poo-parent-gateway/server-key.pem
```

把以下文件安全分发到中转站机器：

```text
client-ca.pem
relay-client.pem
relay-client-key.pem
```

在中转站机器安装证书文件：

```bash
sudo install -d -m 0750 /etc/poo-parent-gateway
sudo install -o root -g root -m 0644 client-ca.pem /etc/poo-parent-gateway/client-ca.pem
sudo install -o root -g root -m 0644 relay-client.pem /etc/poo-parent-gateway/relay-client.pem
sudo install -o root -g root -m 0640 relay-client-key.pem /etc/poo-parent-gateway/relay-client-key.pem
```

如果中转站服务不是 root 运行，请把 `relay-client-key.pem` 的 group 改为中转站运行
用户所在组，并保持权限不大于 `0640`。

写 Gateway client policy：

```bash
sudo tee /etc/poo-parent-gateway/client-policy.yaml >/dev/null <<'EOF'
clients:
  - san_uri: spiffe://poo/relay-prod
    allowed_targets:
      - dashscope.aliyuncs.com:443
      - api.openai.com:443
      - api.anthropic.com:443
    max_concurrency: 4
EOF

sudo chmod 0644 /etc/poo-parent-gateway/client-policy.yaml
```

`client-policy.yaml` 里的 `allowed_targets` 是 client 级收窄规则：

- 不配置 `allowed_targets` 或配置为空列表时，该 client 不额外收窄目标，只受全局 `POO_EGRESS_ALLOWED_TARGETS` 限制。
- 配置了 `allowed_targets` 时，请求目标必须同时命中全局 `POO_EGRESS_ALLOWED_TARGETS` 和该 client 的 `allowed_targets`。
- client 级 `allowed_targets` 不能放大全局 allowlist。如果两者没有交集，`--check-config` 会失败。

远程模式 env：

```bash
export ENCLAVE_CID=16
echo "ENCLAVE_CID=${ENCLAVE_CID}"
export PARENT_VM_PRIVATE_IP=<parent_vm_private_ip>

sudo tee /etc/poo-parent-gateway/env >/dev/null <<EOF
POO_PARENT_PROOF_RELAY_LISTEN=${PARENT_VM_PRIVATE_IP}:15005
POO_PARENT_LEGACY_INGRESS_LISTEN=
POO_PARENT_ADMIN_LISTEN=127.0.0.1:15007
POO_PARENT_AUTH_MODE=mtls
POO_PARENT_MTLS_CA_FILE=/etc/poo-parent-gateway/client-ca.pem
POO_PARENT_MTLS_CERT_FILE=/etc/poo-parent-gateway/server.pem
POO_PARENT_MTLS_KEY_FILE=/etc/poo-parent-gateway/server-key.pem
POO_PARENT_CLIENT_POLICY_FILE=/etc/poo-parent-gateway/client-policy.yaml

POO_PARENT_VSOCK_CID=${ENCLAVE_CID}
POO_PARENT_VSOCK_PORT=5005
POO_PARENT_VSOCK_METRICS_PORT=5006
POO_PARENT_SHUTDOWN_TIMEOUT_MS=300000
POO_PARENT_LOG_LEVEL=info

POO_EGRESS_LANE_LISTEN_MODE=direct-vsock
POO_EGRESS_PORT_RANGE=18000-18999
POO_EGRESS_PORT_COOLDOWN_MS=5000
POO_EGRESS_ROUTE_IDLE_TTL_MS=300000
POO_EGRESS_LEASE_TTL_MS=30000
POO_EGRESS_DEFAULT_CONNECT_TIMEOUT_MS=10000
POO_EGRESS_MAX_ACTIVE_ROUTES=1000
POO_EGRESS_MAX_ACTIVE_LEASES=4096
POO_EGRESS_DEFAULT_ROUTE_CONCURRENCY=1
POO_EGRESS_MAX_ROUTE_CONCURRENCY=16
POO_EGRESS_ALLOWED_TARGETS=dashscope.aliyuncs.com:443,api.openai.com:443,api.anthropic.com:443

POO_RELAY_MAX_METADATA_BYTES=16384
POO_RELAY_MAX_REQ_HEAD_BYTES=1048576
POO_RELAY_MAX_FRAME_BYTES=67108864
POO_RELAY_MAX_REQUEST_BYTES=268435456
POO_RELAY_MAX_BUFFERED_BYTES=4194304
POO_RELAY_SPOOL_DIR=/var/lib/poo-parent-gateway/spool
POO_RELAY_MAX_SPOOL_BYTES=1073741824
POO_RELAY_IO_TIMEOUT_MS=300000

TTVG_CONNECT_TIMEOUT=5s
TTVG_IDLE_TIMEOUT=300s
TTVG_READY_CACHE_TTL=1s
TTVG_MAX_CONNS=1024
TTVG_TCP_KEEPALIVE=30s
EOF
```

安全组 / 防火墙要求：

- `15005/tcp` 只允许中转站机器或私网 LB 访问。
- `15007/tcp` 默认只在父 VM 本机访问，不对公网开放。
- 不开放 `18000-18999/tcp`，这些是 vsock lane，不是公网 TCP 端口。

### 3.7 配置校验

```bash
sudo bash -lc 'set -a; . /etc/poo-parent-gateway/env; set +a; /usr/local/bin/tcp-to-vsock-gateway --check-config'
```

期望输出：

```text
configuration ok
```

### 3.8 systemd service

下面的 Gateway unit 默认采用第 4.5 节的 `proof-enclave.service` 管理 Enclave，
因此会在启动 Gateway 时请求 systemd 先拉起 Enclave。生产环境推荐使用这种方式。
注意：这只保证 `proof-enclave.service` 的启动命令已执行完成，不代表 Enclave 内部
`5005/5006` 服务已经 ready。后续必须等待 Gateway `/readyz` 返回 `200` 后，再继续
中转站验收或接入流量。

如果现场只采用第 4.4 节的手动 `nitro-cli run-enclave` 方式，不安装
`proof-enclave.service`，请删除 `[Unit]` 中的 `proof-enclave.service` 依赖，只保留：

```ini
After=network-online.target
Wants=network-online.target
```

```bash
sudo tee /etc/systemd/system/poo-parent-gateway.service >/dev/null <<'EOF'
[Unit]
Description=PoO Parent Gateway
After=network-online.target proof-enclave.service
Wants=network-online.target proof-enclave.service

[Service]
Type=simple
EnvironmentFile=/etc/poo-parent-gateway/env
ExecStart=/usr/local/bin/tcp-to-vsock-gateway
User=poo-gateway
Group=poo-gateway
Restart=always
RestartSec=2s
LimitNOFILE=65536
TimeoutStopSec=330s

NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/lib/poo-parent-gateway /var/log/poo-parent-gateway

[Install]
WantedBy=multi-user.target
EOF
```

启动：

```bash
sudo systemctl daemon-reload
sudo systemctl enable poo-parent-gateway
sudo systemctl restart poo-parent-gateway
sudo systemctl status poo-parent-gateway --no-pager
```

等待 Gateway ready：

```bash
READY=0
for i in $(seq 1 60); do
  if curl -fsS http://127.0.0.1:15007/readyz >/dev/null; then
    echo "poo-parent-gateway is ready"
    READY=1
    break
  fi
  sleep 2
done

if [ "$READY" != "1" ]; then
  echo "poo-parent-gateway did not become ready within 120 seconds" >&2
  curl -i http://127.0.0.1:15007/readyz || true
  exit 1
fi

curl -fsS http://127.0.0.1:15007/readyz
```

最后一次 `curl` 应返回 `200`。如果 120 秒后仍不是 `200`，脚本会以非 0 状态退出；
不要继续启动中转站 PoO 流量，先按第 3.9 节检查 Enclave CID、Gateway env 和日志。

查看日志：

```bash
sudo journalctl -u poo-parent-gateway -n 100 --no-pager
sudo journalctl -u poo-parent-gateway -f
```

### 3.9 Gateway 验证

健康检查：

```bash
curl -i http://127.0.0.1:15007/healthz
curl -i http://127.0.0.1:15007/readyz
```

期望都是 `200`。如果 `/readyz` 是 `503`，先确认：

```bash
sudo nitro-cli describe-enclaves
grep -E 'POO_PARENT_VSOCK_CID|POO_PARENT_VSOCK_PORT|POO_PARENT_VSOCK_METRICS_PORT' /etc/poo-parent-gateway/env
sudo journalctl -u poo-parent-gateway -n 100 --no-pager
```

远程 mTLS 模式还需要从中转站机器验证 TLS 握手和网络路径。下面命令使用
`--resolve` 把证书中的 DNS SAN 指向父 VM 私网 IP；如果直接使用私网 IP URL，则
server cert 的 SAN 必须包含该 IP。

```bash
curl -sS \
  --resolve poo-parent-gateway.internal:15005:<parent_vm_private_ip> \
  --cacert /etc/poo-parent-gateway/client-ca.pem \
  --cert /etc/poo-parent-gateway/relay-client.pem \
  --key /etc/poo-parent-gateway/relay-client-key.pem \
  -o /tmp/poo-mtls-smoke.body \
  -w '%{http_code}\n' \
  https://poo-parent-gateway.internal:15005/v1/proof/relay
```

期望返回 `405`。这只表示请求已经通过 TLS、client certificate CA 校验和安全组，
到达 Proof Relay 应用层，但方法不是 `POST`。该检查不会执行
`client-policy.yaml` 的 identity lookup；policy identity 仍必须通过第 2.11 节的
端到端 PoO 请求验证。若端到端请求返回 `403 forbidden`，重点检查 client certificate
identity 是否匹配 `client-policy.yaml` 中的 `san_uri` / `san_dns` / `subject`。
如果这里出现 TLS/SAN/certificate 错误或连接失败，先修证书、DNS/SNI 或安全组。

关键 metrics：

```bash
curl -s http://127.0.0.1:15007/metrics | grep -E \
  'ttvg_proof_relay_requests_total|ttvg_proof_relay_preflight_failures_total|ttvg_proof_relay_aborts_total|ttvg_egress_routes_cached_active|ttvg_egress_routes_cached_idle|ttvg_egress_leases_active|ttvg_egress_lane_ports_active|ttvg_egress_route_failures_total|ttvg_egress_proxy_failures_total|ttvg_egress_bytes_enclave_to_upstream_total|ttvg_egress_bytes_upstream_to_enclave_total'
```

发起中转站请求后，正常现象：

- `ttvg_proof_relay_requests_total` 增长。
- 请求期间 `ttvg_egress_leases_active` / `ttvg_egress_lane_ports_active` 短暂增长。
- `ttvg_egress_bytes_enclave_to_upstream_total` 和
  `ttvg_egress_bytes_upstream_to_enclave_total` 增长。
- 不再出现 `lane accept failed ... address family not supported by protocol`。

### 3.10 常见 Gateway 故障

`lane accept failed ... address family not supported by protocol`

- Gateway 版本过旧，没有包含 direct-vsock accept 修复。
- 升级到 commit `5ac85a1` 或之后版本，重新构建部署。

`连 vsock-proxy 失败: tried parent CID(s) [3] on port 18000; Connection reset by peer`

- 常见原因是 Gateway 配成 `tcp-loopback`，但 Enclave 仍是 `direct-vsock`。
- 生产应保持 `POO_EGRESS_LANE_LISTEN_MODE=direct-vsock`。

`TLS 写请求头失败: unexpected end of file`

- 如果 Gateway 日志同时有 `lane accept failed`，先修 Gateway 版本。
- 如果 Gateway 日志有 `egress connect failed`，检查父 VM 到上游或代理的连通性。

`target_not_allowed`

- `POO_EGRESS_ALLOWED_TARGETS` 或 client policy 未包含实际上游 `host:443`。

## 4. EIF 部署

以下命令均在 AWS Nitro 父 VM 上执行。本文使用已经构建好的 EIF，不重新构建。

### 4.1 EC2 前置条件

实例必须支持并启用 Nitro Enclaves。

建议实例族：

```text
m7g.xlarge / m7g.2xlarge
c7g.xlarge / c7g.2xlarge
```

至少预留：

- 2 vCPU 给 Enclave。
- 2048 MiB 内存给 Enclave。
- 父 VM 额外保留 Gateway 和中转站资源。

如果实例尚未启用 Enclave，需要停机后修改属性：

```bash
aws ec2 modify-instance-attribute \
  --instance-id i-xxxxxxxxxxxxxxxxx \
  --enclave-options 'Enabled=true'
```

### 4.2 安装 Nitro CLI 和 allocator

Amazon Linux：

```bash
sudo dnf update -y
sudo dnf install -y aws-nitro-enclaves-cli aws-nitro-enclaves-cli-devel jq
```

如果系统使用 `yum`：

```bash
sudo yum install -y aws-nitro-enclaves-cli aws-nitro-enclaves-cli-devel jq
```

验证：

```bash
nitro-cli --version
```

配置 allocator：

```bash
sudo tee /etc/nitro_enclaves/allocator.yaml >/dev/null <<'EOF'
---
memory_mib: 2048
cpu_count: 2
EOF
```

启动 allocator：

```bash
sudo systemctl enable --now nitro-enclaves-allocator.service
sudo systemctl status nitro-enclaves-allocator.service --no-pager
```

如果启动失败，检查：

- EC2 是否启用 Enclave。
- `cpu_count` 是否超过实例可用 CPU。
- `memory_mib` 是否超过可预留内存。

### 4.3 安装已构建 EIF

假设第三方收到的 EIF 文件已经上传到父 VM：

```text
/tmp/proof.eif
```

安装到标准路径：

```bash
sudo mkdir -p /opt/proof-of-observation/eif
sudo install -o root -g root -m 0644 \
  /tmp/proof.eif \
  /opt/proof-of-observation/eif/proof.eif

sha384sum /opt/proof-of-observation/eif/proof.eif || true
sha256sum /opt/proof-of-observation/eif/proof.eif
ls -lh /opt/proof-of-observation/eif/proof.eif
```

生产发布记录中至少保存：

- EIF 文件 SHA256。
- EIF 对应源码 commit / 构建版本。
- 预期 PCR0 / trust metadata。
- 发布人和发布时间。

### 4.4 手动启动 Enclave

生产不要使用 `--debug-mode`。

```bash
sudo nitro-cli run-enclave \
  --enclave-name proof \
  --eif-path /opt/proof-of-observation/eif/proof.eif \
  --enclave-cid 16 \
  --cpu-count 2 \
  --memory 2048
```

查看运行状态：

```bash
sudo nitro-cli describe-enclaves
```

记录：

```text
ENCLAVE_ID=<EnclaveID>
ENCLAVE_CID=16
```

Gateway 配置中的 `POO_PARENT_VSOCK_CID` 必须等于固定 `EnclaveCID=16`。
如果现场选择了其他固定 CID，所有 `--enclave-cid`、`POO_ENCLAVE_CID` 和
`POO_PARENT_VSOCK_CID` 都必须同步替换为同一个值。

Debug 环境可以临时用：

```bash
sudo nitro-cli run-enclave \
  --enclave-name proof \
  --eif-path /opt/proof-of-observation/eif/proof.eif \
  --enclave-cid 16 \
  --cpu-count 2 \
  --memory 2048 \
  --debug-mode
```

查看 console：

```bash
sudo nitro-cli console --enclave-id <ENCLAVE_ID>
```

正常应看到类似：

```text
listening on vsock :5005
metrics :5006
```

### 4.5 systemd 管理 Enclave

写启动脚本：

```bash
sudo tee /usr/local/bin/run-proof-enclave.sh >/dev/null <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

EIF=/opt/proof-of-observation/eif/proof.eif
NITRO_CLI="${NITRO_CLI:-/usr/bin/nitro-cli}"
ENCLAVE_NAME="${POO_ENCLAVE_NAME:-proof}"
ENCLAVE_CID="${POO_ENCLAVE_CID:-16}"
CPU_COUNT="${POO_ENCLAVE_CPU_COUNT:-2}"
MEMORY_MIB="${POO_ENCLAVE_MEMORY_MIB:-2048}"

RUNNING_CID="$(
  "$NITRO_CLI" describe-enclaves |
    jq -r --arg name "$ENCLAVE_NAME" '.[] | select(.EnclaveName == $name and .State == "RUNNING") | .EnclaveCID' |
    head -n1
)"

if [ -n "$RUNNING_CID" ] && [ "$RUNNING_CID" != "null" ]; then
  if [ "$RUNNING_CID" = "$ENCLAVE_CID" ]; then
    echo "enclave ${ENCLAVE_NAME} is already running with CID ${ENCLAVE_CID}; skip"
    exit 0
  fi
  echo "enclave ${ENCLAVE_NAME} is running with CID ${RUNNING_CID}, expected ${ENCLAVE_CID}" >&2
  exit 1
fi

"$NITRO_CLI" run-enclave \
  --enclave-name "$ENCLAVE_NAME" \
  --eif-path "$EIF" \
  --enclave-cid "$ENCLAVE_CID" \
  --cpu-count "$CPU_COUNT" \
  --memory "$MEMORY_MIB"
EOF

sudo chmod 0755 /usr/local/bin/run-proof-enclave.sh
```

写停止脚本：

```bash
sudo tee /usr/local/bin/stop-proof-enclave.sh >/dev/null <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

ENCLAVE_NAME="${POO_ENCLAVE_NAME:-proof}"
NITRO_CLI="${NITRO_CLI:-/usr/bin/nitro-cli}"

mapfile -t ENCLAVE_IDS < <(
  "$NITRO_CLI" describe-enclaves |
    jq -r --arg name "$ENCLAVE_NAME" '.[] | select(.EnclaveName == $name) | .EnclaveID'
)

if [ "${#ENCLAVE_IDS[@]}" -eq 0 ]; then
  echo "no enclave named ${ENCLAVE_NAME}; skip"
  exit 0
fi

for enclave_id in "${ENCLAVE_IDS[@]}"; do
  "$NITRO_CLI" terminate-enclave --enclave-id "$enclave_id"
done
EOF

sudo chmod 0755 /usr/local/bin/stop-proof-enclave.sh
```

写 service：

```bash
sudo tee /etc/systemd/system/proof-enclave.service >/dev/null <<'EOF'
[Unit]
Description=proof-of-observation Nitro Enclave
After=nitro-enclaves-allocator.service
Requires=nitro-enclaves-allocator.service

[Service]
Type=oneshot
Environment=POO_ENCLAVE_NAME=proof
Environment=POO_ENCLAVE_CID=16
Environment=POO_ENCLAVE_CPU_COUNT=2
Environment=POO_ENCLAVE_MEMORY_MIB=2048
ExecStart=/usr/local/bin/run-proof-enclave.sh
RemainAfterExit=yes
ExecStop=/usr/local/bin/stop-proof-enclave.sh

[Install]
WantedBy=multi-user.target
EOF
```

启动：

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now proof-enclave.service
sudo systemctl status proof-enclave.service --no-pager
sudo nitro-cli describe-enclaves
```

注意：该 unit 是 `Type=oneshot + RemainAfterExit=yes`，只负责执行启动和停止命令，
不会持续监督 Enclave 进程。如果 Enclave 后续异常退出，systemd 可能仍显示该 unit
为 active；生产需要用 Gateway `/readyz`、metrics 或外部监控发现异常。

`proof-enclave.service` 默认以 root 运行，因此脚本内部直接调用 `nitro-cli`，
不再依赖 `sudo`。如果现场 `nitro-cli` 不在 `/usr/bin/nitro-cli`，可以在 unit 中增加
`Environment=NITRO_CLI=<actual-path>`。如果需要手动执行脚本，请用
`sudo /usr/local/bin/run-proof-enclave.sh` 或
`sudo /usr/local/bin/stop-proof-enclave.sh`。

Gateway 已部署后的恢复示例：

```bash
sudo systemctl restart proof-enclave.service
sudo systemctl restart poo-parent-gateway.service
curl -fsS http://127.0.0.1:15007/readyz
```

最后一行必须返回成功后，才恢复中转站 PoO 流量。

该 unit 默认只管理 `EnclaveName=proof` 的 Enclave。如果一个父 VM 上部署多个
Enclave，请给每个 Enclave 使用不同 `POO_ENCLAVE_NAME` 和 `POO_ENCLAVE_CID`，
并为它们拆分独立 service。

### 4.6 Gateway 部署后的固定 CID 校验

本小节依赖第 3 节 Gateway 已经部署完成。首次部署时，如果当前只是在执行第 4 节
启动 EIF，可以先跳过本小节，完成第 3 节后再回来执行。

每次 Enclave 重启后确认仍是固定 CID：

```bash
sudo nitro-cli describe-enclaves | jq -e \
  '.[] | select(.EnclaveName == "proof" and .State == "RUNNING" and .EnclaveCID == 16)'
```

确认 Gateway 配置仍指向同一个 CID：

```bash
grep '^POO_PARENT_VSOCK_CID=16$' /etc/poo-parent-gateway/env

sudo systemctl restart poo-parent-gateway
curl -fsS http://127.0.0.1:15007/readyz
```

最后一行必须返回成功；如果失败，先按第 3.9 节检查 Gateway readiness。

如果 `describe-enclaves` 显示 `proof` 不是 CID `16`，不要动态修改 Gateway
去追随漂移 CID；应先停止错误 CID 的 Enclave，再用 `--enclave-cid 16` 重新启动。

### 4.7 停止 Enclave

```bash
sudo nitro-cli describe-enclaves
sudo nitro-cli terminate-enclave --enclave-id <ENCLAVE_ID>
```

或在确定同机只有该 Enclave 时：

```bash
sudo nitro-cli terminate-enclave --all
```

### 4.8 EIF 验收

EIF-only 验收不依赖 Gateway，只确认 Enclave 已经以固定 CID 运行：

```bash
sudo nitro-cli describe-enclaves | jq .
```

确认输出中包含：

```json
{
  "EnclaveName": "proof",
  "EnclaveCID": 16,
  "State": "RUNNING"
}
```

Gateway 部署后再执行 Gateway readiness 和端到端验收：

```bash
curl -fsS http://127.0.0.1:15007/readyz
```

端到端请求后，确认：

```bash
curl -s http://127.0.0.1:15007/metrics | grep -E \
  'ttvg_proof_relay_requests_total|ttvg_egress_bytes_enclave_to_upstream_total|ttvg_egress_bytes_upstream_to_enclave_total'
```

业务响应应包含 `proof`，流式响应应包含 `event: tee.proof`。

## 5. 全链路上线检查表

- [ ] EC2 实例已启用 Nitro Enclaves。
- [ ] `nitro-cli --version` 可用。
- [ ] `nitro-enclaves-allocator.service` 正常。
- [ ] 已安装已构建 EIF 到 `/opt/proof-of-observation/eif/proof.eif`。
- [ ] 生产 Enclave 未使用 `--debug-mode`。
- [ ] `sudo nitro-cli describe-enclaves` 显示 `RUNNING`。
- [ ] Enclave 使用固定 `EnclaveCID=16` 启动。
- [ ] Gateway `POO_PARENT_VSOCK_CID` 等于固定 `EnclaveCID=16`。
- [ ] Gateway 版本包含 commit `5ac85a1` 或之后的 direct-vsock accept 修复。
- [ ] 未运行旧静态 `vsock-proxy` egress service。
- [ ] `POO_EGRESS_LANE_LISTEN_MODE=direct-vsock`。
- [ ] `POO_EGRESS_ALLOWED_TARGETS` 包含实际上游 `host:443`。
- [ ] Gateway `/readyz` 返回 `200`。
- [ ] 中转站启用 `POO_PARENT_GATEWAY_ENABLED=true`。
- [ ] 生产中转站启用 `POO_PARENT_GATEWAY_REQUIRED=true`。
- [ ] 同机模式只用 loopback `authMode=none`。
- [ ] 远程模式使用 HTTPS + mTLS + 安全组来源限制。
- [ ] 远程模式已从中转站机器完成 TLS/network smoke test，`GET /v1/proof/relay` 返回 `405`。
- [ ] 远程模式端到端 PoO 请求没有返回 `403 forbidden`，client certificate identity 已命中 client policy。
- [ ] 非流式响应顶层有 `proof`。
- [ ] 非流式上游错误 JSON 在具备完整 `RESP_TRAILER` 时，最终错误 envelope 顶层也有 `proof`。
- [ ] 流式响应末尾有 `event: tee.proof`。
- [ ] Gateway metrics 中 proof relay 和 egress bytes 增长。
- [ ] 日志没有泄露 `Authorization`、上游 API key、完整 proxy URL、proxy 用户名或密码。

## 6. 参考资料

本文整合并更新了以下内部文档中的部署信息：

- `claude-relay-service/docs/poo-parent-gateway-integration-plan.md`
- `claude-relay-service/docs/poo-parent-gateway-third-party-integration.md`
- `tcp-to-vsock-gateway/docs/poo-parent-gateway-deployment.md`
- `tcp-to-vsock-gateway/docs/poo-parent-gateway-request-proxy-design.md`
- `/Users/admin/Documents/Codex/2026-07-08/ni/outputs/aws-nitro-enclave-proof-of-observation-deployment-guide.md`

其中早期 Nitro guide 中的固定 `vsock-proxy` egress 方案，已被当前
PoO Parent Gateway 的动态 direct-vsock lane 替代。新部署不需要为每个上游模型
启动固定 `8443/8444/8445` egress proxy。
