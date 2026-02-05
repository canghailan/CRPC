# CRPC (Concise RPC) 协议规范与技术白皮书：构建高性能、简明易用且全场景兼容的二进制通信框架



## 简介
CRPC (Concise RPC) 是一种遵循互联网标准设计的现代 RPC 协议。
它旨在消除开发者友好体验与高性能通信之间的鸿沟。
通过深度集成 CBOR (RFC 8949) 序列化协议、HTTP/2 传输层以及 Zstd/OpenZL 现代压缩算法，
构建了一个标准化、支持平滑演进的通信框架，实现“逻辑简明易用，传输极致性能”的设计愿景。



## 为什么不使用现有协议
- REST (JSON)：凭借资源导向架构和广泛支持成为 Web API 事实标准，但 JSON 序列化在 CPU 消耗和带宽占用上存在天然瓶颈
- JSON-RPC：作为轻量级的远程调用协议，易于实现和调试，但缺乏现代传输层支持和高效的二进制序列化，难以满足高性能场景需求
- gRPC (Protobuf)：通过 HTTP/2 和二进制格式极大提升了性能，但强制性的预编译 Schema（.proto）和复杂的工具链增加了开发心智负担，且难以在浏览器中直接使用

CRPC 通过内容协商机制（Content Negotiation）和透明压缩层，同时提供易用性和高性能。




## 传输层
### 默认传输：HTTP/2
CRPC 默认运行在 HTTP/2 之上，以利用其现代网络特性：
- 二进制分帧、多路复用：在单个 TCP 连接上并行处理多个 RPC 流，消除队头阻塞
- 流量控制：利用 HTTP/2 原生 WINDOW_UPDATE 机制实现反压，保护资源受限的节点


### 兼容传输：HTTP/1.1
为了确保兼容性，CRPC 支持在 HTTP/1.1 上运行：
- 仅支持特定通信方式：请求响应、服务端流
- 主要用于旧有系统集成、浏览器直接访问及开发调试


### 自定义传输： 基于 TCP、MQTT、WebSocket 等
参考 JSON-RPC。

#### Request
```json
{
    "crpc": 1,
    "method": "method_name",
    "params": {
        "param1": "value1",
        "param2": "value2"
    },
    "id": "1234567890"
}
```

#### Response
```json
{
    "crpc": 1,
    "result": {
        "key": "value"
    },
    "error": {
        "-1": "title",
        "-2": "detail",
        "-3": "instance",
        "-4": "type/status/ResponseCode",
        "...": "Custom"
    },
    "id": "1234567890"
}
```


### 安全传输：TLS 1.3
安全层作为可选项，但启用时建议以 TLS 1.3 为基准：
- 性能：优化首包时延
- 安全性：强制废弃不安全的加密套件，确保工业级通信安全



## 消息格式
### 内容协商 (Content Negotiation)
CRPC 运行在标准 HTTP 堆栈上，通过 ```Content-Type``` 和 ```Accept``` 头部自动切换编解码策略：
- CBOR 模式 (高性能)：```application/cbor```。CBOR 数据自带类型和长度信息（Major Type），结合 HTTP/2 DATA 帧的边界，实现了 0 字节的协议层开销
- JSON 模式 (兼容/调试)：```application/json```。提供纯文本的可读性，支持现有 Web 基础设施。支持浏览器直接调用、快速原型开发和 curl 命令行调试


### 序列化格式对比
| 特性             | JSON         | Protobuf       | **CBOR**             |
| ---------------- | ------------ | -------------- | ------------------- |
| **表示形式**    | 文本 (UTF-8) | 二进制         | **二进制**               |
| **自描述性**    | 是           | 否 (需 Schema) | **是 (无需 Schema)**    |
| **二进制支持**  | 需 Base64    | 原生           | **原生 (Major Type 2)** |
| **Schema 依赖** | 可选         | 强制           | **可选 (推荐 CDDL)**    |
| **解析效率**    | 低           | 高             | **高**                 |



## 通信⽅式
参考 gRPC 定义了四种标准的通信⽅式，并针对不同传输层进行了适配：
| 模式                         | HTTP/2 (CBOR/JSON)                                | HTTP/1.1 (JSON)                             |
| ---------------------------- | ------------------------------------------------- | ------------------------------------------ |
| **请求响应 (Unary)**         | 标准 Request-Response 帧序列                      | 标准 POST 请求-响应                            |
| **服务端流 (Server Stream)** | 响应 Body 为连续的 **CBOR Sequence** 或 ​**JSONL**​ | 利用 **Server-Sent Events (SSE)** 实现异步推流  |
| **客户端流 (Client Stream)** | 客户端在单个 Stream 中发送多个 DATA 帧              | 不支持                                        |
| **双向流 (Bidi Stream)**     | 全双工独立发送与接收 DATA 帧序列                    | 不支持                                        |



## 路由与方法
参考 gRPC 及 REST：
- HTTP Method: ```POST```
- HTTP Path: 指定 RPC 的 Method Name，例如 ```/{Service}/{Method}```

CRPC 可以被标准 Web 框架、中间件及基础设施轻松处理。



## 错误处理：Problem Details
CRPC 强制使用 Problem Details (RFC 9457) 及其二进制变体作为标准错误模型。
- JSON: 返回 ```application/problem+json```，字段包括 type, title, status, detail
- CBOR: 返回 ```application/concise-problem+cbor``` (RFC 9290)，使用负整数键映射标准字段，体积比 JSON 缩小约 60%



## 压缩
CRPC 将性能优化重心从序列化层移至透明的压缩层，通过 HTTP ```Content-Encoding``` 和 ```Accept-Encoding``` 头部协商：
| 算法        | 标识符    | 特点与应用场景                                                                                                    |
| ---------- | -------- | ---------------------------------------------------------------------------------------------------------------- |
| **Zstd**   | `zstd`   | ​**共享字典 (RFC 9842)**​。CRPC 默认推荐。通过预共享字典消除字段名冗余，使自描述格式（JSON/CBOR）达到接近 Protobuf 的压缩效率 |
| **OpenZL** | `openzl` | ​**格式感知**​。Meta 开源，通过 SDDL 描述数据结构，对 AI Tensor 或监控数据进行语义去重，效率远超通用算法                     |
| **LZ4**    | `lz4`    | ​**极致速度**​。CPU 损耗微乎其微，适用于超高频、低时延的指令交换                                                          |
| **Brotli** | `br`     | ​**静态高比率**​。适用于下发较大的服务配置或元数据包                                                                     |
| **Gzip**   | `gzip`   | ​**生态兼容**​。确保旧有系统和边缘节点的平滑接入                                                                         |



## 模式定义语言 (IDL)：CDDL 的灵活性
CRPC 推荐使用 CDDL (Concise Data Definition Language, RFC 8610) 定义服务契约：
- 表达力：比 Protobuf 更简洁，支持正则表达式、数值范围约束等高级校验
- 代码生成：支持多语言存根自动生成，确保端到端类型安全



## 安全架构：TLS 与 COSE 双重屏障
除了传输层的 TLS 1.3，CRPC 原生支持 COSE (CBOR Object Signing and Encryption, RFC 9052)：
- 签名：在经过非信任中间件（代理/网关）时保持消息的完整性和不可篡改性
- 端到端加密：即使 TLS 在网关层卸载，敏感业务数据依然能保持私密



## 协议对比
| 特性          | REST            | gRPC              | JSON-RPC     | **CRPC**                 |
| ------------ | --------------- | ----------------- | ------------ | ------------------------- |
| **序列化层** | JSON (文本)      | Protobuf (二进制)  | JSON (文本)   | **CBOR/JSON 内容协商**    |
| **传输层**   | HTTP 1.1/2       | HTTP/2 (Strict)   | 不限          | **HTTP/2 (Opt TLS 1.3)** |
| **压缩方案** | 多算法协商 (Gzip) | 消息级静态         | 无            | **OpenZL/Zstd 等算法协商** |
| **分帧开销** | 无               | 5-Byte 固定头     | 外层信封结构    | **无**                    |
| **错误模型** | 非标             | gRPC Status       | 专用对象       | **RFC 9457/9290 标准**    |
| **可读性**   | 优               | 差                | 优            | **优**                    |



## 核心优势
- 开发者友好：开发者在业务层处理熟悉的 JSON/CBOR 对象，无需强绑定复杂的 IDL 编译过程
- 标准化与生态融合：完全基于现有标准（HTTP/2, CBOR, Problem Details, Zstd-D），兼容现有的云原生基础设施
- 无缝平滑升级：现有项目可保留原有 JSON 逻辑，通过增加内容协商头即可原地升级为高性能二进制模式
- 极致性能：利用 HTTP/2 传输机制，OpenZL 的结构感知压缩、Zstd 字典压缩，可接近甚至超越 gRPC



## 示例
### 请求
```http
POST /Gretter/SayHello HTTP/2
Host: example.com
Content-Type: application/cbor
Accept-Encoding: zstd, openzl
Content-Encoding: zstd

{
  "name": "CRPC User"
}
```

### 响应
```http
HTTP/2 200 OK
Content-Type: application/cbor
Content-Encoding: zstd

{
  "message": "Hello, CRPC User!"
}
```
