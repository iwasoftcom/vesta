Vesta

补齐短板的 S3 兼容对象存储 · v0.1.0

用 Rust 编写的分层、S3 兼容对象存储系统——单一二进制文件， 从笔记本电脑到 Raft 复制集群都无需更换软件即可扩展。

**在用 AI 辅助开发？**把这个链接交给你的编码智能体/LLM，而不是本页面—— 这是一份密集、面向机器优化的参考文档（安装方式、每个环境变量及其默认值、 精确的 API 调用），智能体可以直接使用，无需从营销文案中反推事实： [documentation.ai.md](https://iwasoft.com/products/vesta/0.1.0/docs/documentation.ai.md)

## Vesta 是什么

Vesta 针对当今对象存储方案（S3、GCS、Azure Blob、R2、B2、Wasabi、MinIO、 Ceph、SeaweedFS、Garage）中存在的功能缺口而设计。它讲的是真正的 S3 API—— SigV4 签名、分片上传、版本控制、条件请求、批量删除——并将**控制平面** （元数据：存储桶、对象索引、IAM）与**数据平面**（磁盘上按内容寻址的 数据块）完全分离，使二者可以独立替换或扩展。

## 设计原则

**控制平面与数据平面分离。**  
元数据与字节数据 分处于不同的 trait 边界之后。存储引擎、共识后端和加密层的替换无需触及 S3 API 层。

**不会有管理开关被遗忘在配置文件里。**  
速率限制、 GC 间隔、CORS、配额和策略都是运行时设置——通过管理控制台实时修改并复制， 而不是需要重启的环境变量。

**兼容性是一份契约，而非近似值。**  
SigV4（请求头、 预签名 URL、流式分块）、分片上传、版本控制和条件请求都针对真实的 AWS SDK 测试套件持续验证，而非精心挑选的示例。

## 与典型单二进制对象存储的区别

|  | 典型的 MinIO 式存储 | Vesta |
| --- | --- | --- |
| 共识机制 | 固定的纠删集 / 网关模型 | 具备动态成员关系的网络化 Raft——一个经过验证的引擎（[openraft](#architecture)）可作为可选后端接入同一写入路径 |
| 运行时配置 | 环境变量，修改需重启 | 管理控制台通过复制日志实时修改设置（速率限制、GC 间隔、CORS、配额）——无需重启 |
| 元数据持久性 | 因后端而异 | 带快照压缩的仅追加 WAL；每个节点独立持久化写入，并通过正常日志复制追赶进度 |
| 多租户 | 后期附加或缺失 | 租户为一等公民，支持按租户的存储桶配额和 SigV4 身份隔离 |
| AI 智能体访问 | 不适用 | 只读的 [MCP 服务器](#more)将原生搜索和 S3 Select 作为智能体工具暴露，并按密钥实现租户隔离 |

## 快速开始

运行服务器（容器镜像，或独立二进制文件）：

```
# Docker
docker run -p 9000:9000 iwasoftcom/vesta:0.1.0

# 或使用二进制文件
VESTA_DATA_DIR=/var/lib/vesta vesta
```

使用任意 S3 客户端与其通信：

```
aws --endpoint-url http://127.0.0.1:9000 s3 mb s3://photos
aws --endpoint-url http://127.0.0.1:9000 s3 cp ./x.jpg s3://photos/x.jpg
aws --endpoint-url http://127.0.0.1:9000 s3 ls s3://photos
```

## 功能一览

**速率限制**  
按租户的令牌桶，可从管理控制台实时 启用和调整；行为异常的调用方会收到规范的 `SlowDown` 响应， 而不是被直接断开连接。

**分布式共识**  
具备领导选举、动态成员关系和 持久日志复制的网络化 Raft——或者在同一写入路径之下选用经过验证的实现 `openraft`。

**纠删码与加密**  
Reed–Solomon 纠删码存储与 AES-256-GCM 静态加密，二者均支持去重安全（按内容寻址的数据块）。

**版本控制与对象锁定**  
完整的版本历史、删除标记， 以及带法律保留的 WORM 保留策略（GOVERNANCE/COMPLIANCE）。

**多租户**  
租户是一等公民：按租户的存储桶配额、 SigV4 身份隔离、存储桶策略和预定义 ACL。

**搜索、Select 与 Lambda**  
原生倒排索引元数据 搜索、S3 Select（CSV SQL），以及读时转换（类似 Object Lambda）。

**复制与事件**  
异步异地复制、变更流事件总线， 以及可插拔的 webhook 通知投递。

**生命周期与清单**  
过期与存储类别转换规则， 以及按需生成的 CSV 清单报告。

## 管理控制台

一个独立、无状态的管理应用（内嵌 React + MUI 界面）将写操作代理到某个 存储节点——它本身不保存任何数据；每一次更改都通过 S3 API 所使用的同一份 共识日志进行复制。

<table><tbody><tr><th>地址</th><td><code>http://localhost:9500</code>（环境变量 <code>VESTA_ADMIN_ADDR</code>，默认 <code>0.0.0.0:9500</code>）</td></tr><tr><th>连接目标</th><td>某个存储节点的管理 API，默认 <code>http://127.0.0.1:9000</code>（环境变量 <code>VESTA_ADMIN_NODES</code>）</td></tr><tr><th>默认用户</th><td>无——在创建<b>第一个</b>控制台用户（用户界面）之前，控制台处于开放状态并以管理员身份运作，创建后该窗口关闭。也可以在节点启动时预置：<code>VESTA_ADMIN_USER</code>/<code>VESTA_ADMIN_PASS</code></td></tr></tbody></table>

每个节点也以普通 JSON API 的形式暴露相同的操作： `http://<节点>:9000/_vesta/admin/*`（控制台自身调用的正是 这些接口）——便于脚本化。你最先要做的三件事：

```
# 1) 创建存储桶
curl -X POST http://127.0.0.1:9000/_vesta/admin/buckets \
  -H 'content-type: application/json' -d '{"name":"photos"}'

# 2) 创建租户（创建 API 密钥前必需）
curl -X POST http://127.0.0.1:9000/_vesta/admin/tenants \
  -H 'content-type: application/json' -d '{"name":"acme-corp"}'

# 3) 为该租户创建 API 密钥（SigV4 access/secret 对）
curl -X POST http://127.0.0.1:9000/_vesta/admin/keys \
  -H 'content-type: application/json' -d '{"tenant":"acme-corp"}'
# → {"access_key":"VESTA...","secret_key":"...","tenant":"acme-corp"}
```

一旦存在控制台用户或 API 密钥，这些调用就需要 `x-vesta-user`/`x-vesta-pass` 请求头（某个控制台用户 的凭据）——并且创建这第一个密钥会自动在整个集群范围内开启 S3 API 的 SigV4 强制校验，无需重启。

-   **用户、密钥与租户**——控制台账户、SigV4 API 密钥、按租户的配额。
-   **存储桶与策略**——创建/删除、存储桶策略 JSON、公开读取开关。
-   **集群**——节点健康状况、添加/移除成员、少数派写入与自动收缩开关。
-   **运行时设置**——速率限制、块 GC 间隔、CORS 来源：实时修改， 复制到每个节点，重启后依然保留。

## 架构

单一二进制文件、两个网络入口，以及严格的分层规则：S3 API 层从不直接 接触存储——一切都经由协调器；任何需要在整个集群生效的变更，都必须先经过 共识日志提交，才能被读取到。

S3 SDK · aws-cli SigV4 · 分片上传 · 版本控制 管理控制台 · AI 智能体（MCP） 无状态代理 · 按租户隔离的工具 S3 API · :9000 管理 API · :9500 协调器（Rust）：存储桶 · 对象 · 分片上传 · 策略 · 生命周期 · 搜索 共识日志（自研 Raft 或 openraft）——变更先提交，后可读 元数据（WAL） · 块存储（纠删码、加密、去重）

## 下载与源代码

-   **下载：**编译好的产物（Windows、Debian `.deb`、 RedHat `.rpm`）以及 Docker 镜像会按版本发布在 [iwasoft.com](https://iwasoft.com) → 产品 → Vesta。 源代码不包含在下载内容中。
-   **兼容性：**S3 API 接口（SigV4、分片上传、版本控制、条件请求） 持续针对真实的 AWS SDK 集成测试进行验证。
-   **如实说明现状：**项目仍处于早期开发阶段——尚未经过独立的安全 审计，也没有生产环境的运行经验。这是如实披露，而非对路线图的保留意见。

Vesta v0.1.0 · S3 兼容 · Rust、按内容寻址存储、网络化 Raft （自研或 openraft）。© iwasoft.
