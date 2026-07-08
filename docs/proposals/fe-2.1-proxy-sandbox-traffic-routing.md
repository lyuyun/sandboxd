# 特性设计文档


**标题**：node-ctl proxy 支持 Sandbox 流量路由  
**版本**：v1.0  
**日期**：2026-06-29  
**状态**：草稿

---

## 1. 需求概述

Kuasar Sandbox 平台的沙箱（microVM）需要通过 HTTPS 向外暴露 envd Connect RPC、code-interpreter 等服务端口，供 SDK 和用户应用访问。当前平台已具备沙箱生命周期管理能力，但**数据面路由转发**是沙箱可用性的关键前提：没有正确的路由装配和流量转发，沙箱即使启动成功也无法对外提供服务。

本特性聚焦于 node-ctl proxy 子命令及其承载的数据面能力：在节点上以独立进程运行一组 proxy worker（由 proxy master 管理），接收来自 cluster-router 的 HTTPS 入站流量，按 `<port>-<sid>.<domain>` Host 解析目标沙箱，将流量精确路由到对应 microVM 的 floatingip 或 UDS 端点；同时实现 paused 沙箱的透明 park/wake 唤醒机制，以及与 conductor 进程解耦的高可用保障。

**当前版本策略**：覆盖 proxy.mode=external 的完整设计（生产首选），兼顾 internal/off 两种降级模式；MMDS v2 sidecar 在 proxy 进程内内嵌，与路由表共享生命周期。

---

## 2. 需求分析

### 2.1 Goals & Non-Goals

#### Goals

1. **数据面路由正确性**：按 `<port>-<sid>.<domain>` 精确路由，端口 49983 走 envd UDS，端口 49999 走 code-interpreter UDS，其他端口走 floatingip:port；验收：SDK 发起的 envd RPC / 用户端口请求均能正常响应。
2. **控制面故障零数据面中断**：conductor 进程崩溃或重启期间，已建立连接和 running 沙箱的新连接不中断；验收：conductor kill 后 running 沙箱访问无感知。
3. **Paused 沙箱透明唤醒（park/wake）**：流量到达 paused 沙箱时自动触发 resume，并在 park_timeout 内完成透传；验收：连接在 wake 完成前挂起，完成后正常建立，客户端无需重试。
4. **Token 鉴权**：enforce 模式下 `X-Access-Token` 不匹配时返回 401；每次 connect/create 重新 mint token，旧 token 即时失效；验收：伪造 token 请求返回 401。
5. **MMDS v2 集成**：proxy 内嵌 MMDS HTTP 服务，按 floatingip 查路由表返回 `accessTokenHash`；验收：guest envd 可成功完成 MMDS 握手完成 re-key。
6. **多 worker 水平扩展**：SO_REUSEPORT 下多个 worker 共享端口，单 worker 崩溃不影响整体；验收：kill 一个 worker 后，存量连接由 OS 层 RST 通知客户端重连，新连接由其余 worker 处理。

#### Non-Goals

- 出站流量策略（CIDR allow/deny）
- 跨节点路由调度（cluster-router 职责）
- 沙箱生命周期管理
- L2 ARP 代答、GENEVE 跨宿主隧道（sandbox-vswitch 职责）
- DNS 出站访问控制

### 2.2 交付范围

| 编号 | 交付项 | 说明 |
|------|--------|------|
| D-1 | proxy worker 进程框架 | `node-ctl proxy` 子命令，SO_REUSEPORT TLS 监听，多实例 |
| D-2 | routesync 客户端 | config-socket plugin 平面订阅者，全量 + 增量路由同步 |
| D-3 | 路由表 + 流量分发 | proxyshm seqlock hash O(1) 查表，按端口分发至 UDS/floatingip |
| D-4 | eBPF flowtable 集成 | 新建连接注册 flowtable，已建连接内核 TC hook 直通 |
| D-5 | park/wake 机制 | paused 沙箱 park 队列 + wake 上行帧 + 超时 404 |
| D-6 | MMDS v2 sidecar | 内嵌 MMDS HTTP，deterministic mmds_secret 验证 |
| D-7 | Token 鉴权 | off/log/enforce 三档，envdsign.CheckDataPlaneAuth |
| D-8 | proxy.mode 三档 | external / internal / off，conductor 侧按 mode 启动内嵌或外置 proxy |
| D-9 | 可观测性 | Prometheus metrics + 结构化日志 |

### 2.3 友商分析

| 维度 | E2B（参照系）| agent-substrate（Google）| kuasar-sandbox（本设计）|
|------|------------|------------------------|----------------------|
| 数据面实现 | Linux bridge + iptables L4 | Envoy sidecar L7 用户态代理 | eBPF/TC 内核态转发 + proxy worker L7 终止 |
| 控制面故障影响数据面 | 有（iptables 规则随进程维护）| 有（Envoy 依赖控制面下发配置）| **无**（bpffs pin，flowtable 独立于控制面）|
| Paused 沙箱透明恢复 | ❌ 客户端需感知并重试 | ❌ 无 pause 概念 | ✅ server-side park/wake |
| 路由查表复杂度 | O(n) iptables 规则匹配 | DNS pull + Envoy xDS | **O(1)** proxyshm seqlock hash（外置 mmap）|
| 多进程水平扩展 | 单进程 | 单 sidecar per pod | **SO_REUSEPORT 多 worker，独立故障域** |
| MMDS/元数据安全下发 | ❌ 无 | △ 依赖 K8s ConfigMap | ✅ HMAC 确定性密钥，不落明文 |

**核心竞争优势**：eBPF flowtable 接管已建连接后，宿主机上的转发路径完全绕过用户态 proxy worker，吞吐量接近线速；同时 conductor 崩溃不中断数据面，这是 E2B 和 agent-substrate 均未解决的问题。

### 2.4 需求约束

1. **依赖 sandbox-vswitch 网络槽**：proxy 的 floatingip 路由依赖 vswitch assign 的 floatingip 地址，conductor 在沙箱创建时调 `connector-ctl vswitch attach` 完成分配后，floatingip 才写入 routesync RouteEntry；proxy 无法独立运行于 vswitch 之前。
2. **config-socket 必须先于 proxy master 就绪**：proxy master 启动时连接 config-socket plugin 平面，若 conductor 未就绪则退避重连，数据面处于降级（无路由）状态，直至同步完成。
3. **TLS 证书共享**：proxy `--data-listen` 与 conductor `api.listen` 使用同一套通配证书（`*.<domain>` + `api.<domain>`），运维需确保两个进程挂载相同证书路径。
4. **SO_REUSEPORT 限制**：同一节点上所有 proxy worker 必须 bind 相同 `--data-listen` 地址；混合节点场景下需配置为 agent-vip 而非 0.0.0.0（防与 k8s NodePort 端口空间冲突）。
5. **park_timeout 对用户可见**：park 期间 SDK 连接处于等待状态，park_timeout（默认 30 s）超时后返回 404，用户需处理此错误；`park_timeout` 策略值由 conductor 在 routesync hello 帧下推，proxy 配置文件仅作初始值。

---

## 3. 解决方案

### 3.1 User Stories

#### 3.1.1 Story A：SDK 访问 Running 沙箱的 envd 端口

作为一个 **SDK 用户**，我想要通过 `<49983-sid.domain>` 发起 Connect RPC 调用，以便于在运行中的沙箱内执行代码和文件操作。

**场景**：沙箱创建成功（state=running），SDK 持有 `EnvdAccessToken`，发起 HTTPS 请求至 cluster-router；cluster-router 按 Host 将请求转发至 proxy worker :8443；proxy worker 解析 Host 得 `(sid, port=49983)`，查 proxyshm 命中，校验 `X-Access-Token`，将连接路由至 envd UDS，后续流量经 eBPF flowtable 内核直通。用户感知到 100 ms 内的连接建立延迟。

#### 3.1.2 Story B：流量触发 Paused 沙箱自动恢复

作为一个 **SDK 用户**，我想要向已暂停的沙箱发起请求后自动等待其恢复，以便于无需在客户端手动调用 connect API 实现透明的 cold-start 体验。

**场景**：沙箱处于 paused 状态（已拍快照），SDK 发起请求；proxy worker 查 proxyshm state=paused，将连接入 ParkQueue，同时写 wake pipe → proxy master → conductor；conductor 执行 resume（恢复快照）；恢复完成后 routesync 广播 `upsert(state=running)` + notify pipe 唤醒 workers；ParkQueue 解除，连接建立；park_timeout 内整个过程对 SDK 透明。

#### 3.1.3 Story C：Serve 进程重启期间已建连接不中断

作为一个 **平台运维人员**，我想要重启 node-ctl conductor 进程（升级/配置变更），以便于不影响用户已建立的沙箱连接。

**场景**：conductor 重启；proxy worker 维持已建连接（eBPF flowtable 已接管内核转发）；running 沙箱的新连接依赖 proxyshm 路由缓存继续服务；paused 沙箱的 Wake 请求暂时无人应答，等 conductor 重启完成后 proxy master 自动重连并重新同步路由表，Wake 随之恢复。运维人员可无感升级 conductor。

#### 3.1.4 Story D：单 Worker 崩溃后快速恢复

作为一个 **平台运维人员**，我想要 proxy master 在 worker 崩溃时由其余 worker 继续承接新连接，以便于节点数据面具备高可用能力。

**场景**：proxy worker 1 崩溃；OS 内核 SO_REUSEPORT 将新连接分发至其余 worker；master `superviseProxyWorker` goroutine 重启该 worker；重启后 worker 直接读 proxyshm，无需全量 routesync 重同步；宕机期间该 worker 上的存量 TCP 连接断开，新连接已由其余 worker 处理。

### 3.2 架构影响分析

本特性在现有 node-ctl 二进制中新增 `proxy` 子命令，不引入新的外部依赖组件，架构元素影响如下：

- **新增进程**：`proxy master + worker × K`（external 模式）；internal 模式由 conductor 进程内嵌，off 模式不启动。
- **新增接口**：config-socket plugin 平面的 routesync 帧协议（h2c + 4B LE + JSON），conductor 侧同步新增实现。
- **技术选型**：Go 标准库 `net/http` + `crypto/tls`；eBPF flowtable 操作复用 sandbox-vswitch 已 pin 的 BPF map（`/sys/fs/bpf/vswitch/map_flowtable`），不新增 BPF 程序。
- **特性树影响**：本特性是沙箱生命周期的必要前提——无 proxy 则沙箱创建成功但不可访问；两者并行开发，proxy 可以 `proxy.mode=off` 模式先行合入。

### 3.3 功能规格

#### 静态规格

| 规格项 | 值 |
|--------|-----|
| 节点 worker 数量（生产建议）| 2（最大 4） |
| 数据面监听端口 | :8443（SO_REUSEPORT） |
| 路由查表复杂度 | O(1)（proxyshm seqlock hash）|
| park 队列上限（per-sid）| 无硬上限，受 park_timeout 时间窗约束 |
| 路由表事件通道缓冲 | 1024 帧/subscriber |
| eBPF flowtable 条目上限 | 与 sandbox-vswitch 共享，参见 vswitch 设计 |
| OS 版本要求 | Linux kernel ≥ 5.15（BPF_MAP_TYPE_LRU_HASH） |
| TLS 最低版本 | TLS 1.2（建议 1.3） |

#### 与存量特性的配套关系

| 存量特性 | 关系 | 约束 |
|---------|------|------|
| 沙箱 CRUD | 强依赖：create 写路由、delete 广播删除 | proxy master/worker 必须与 conductor 在同一节点 |
| Auto-suspend | 协作：suspend 触发 routesync upsert(paused)；流量触发 wake | park_timeout 须 > resume P99 |
| Secure 模式 | 协作：mmds.enabled=true 时 proxy 内嵌 MMDS；conductor 生成 mmds_secret 经 routesync 下发 | MMDS sidecar 须与 proxy worker 同进程 |
| 网络基础设施（vswitch）| 依赖：floatingip 由 connector-ctl vswitch attach 分配后才写入 RouteEntry | vswitch 须先于 proxy 就绪 |

| 规划特性（未实现）| 关系 | 约束 |
|----------------|------|------|
| 高性能数据面（eBPF flowtable）| 协作：proxy 新建连接后注册 flowtable，后续由 TC hook 内核转发（当前全程用户态 splice）| bpffs 须挂载于 /sys/fs/bpf |

### 3.4 风险及设计约束

| 风险 | 影响 | 缓解措施 |
|------|------|---------|
| conductor 重启期间 paused 沙箱 wake 无人应答 | park 超时后用户请求 404 | park_timeout 默认 30 s，conductor 重启 < 5 s；告警 conductor 重启时间异常 |
| proxy worker 订阅通道积压（> 1024 帧）| subscriber 被丢弃，需全量重同步 | 指数退避重连；监控 `proxy_routesync_reconnect_total` |
| bpffs 未挂载或 flowtable map 不存在 **[flowtable 实现后生效]** | proxy 降级为纯用户态转发（仍可用，但内核旁路不生效）| 启动时检查 `/sys/fs/bpf/vswitch/map_flowtable`，缺失则 warn 并降级 |
| TLS 证书更新期间短暂服务中断 | 新建 TLS 握手失败 | 双 worker 滚动重启：先重启 worker-1，再重启 worker-2 |
| park 队列 goroutine 泄漏（sid 永不 resume）| 内存缓慢增长 | park_timeout 超时后强制释放所有 goroutine；Dead 路由删除时同步清队列 |

**升级兼容性**：routesync 协议以 `{"type":"hello","hello":{"version":1,...}}` 携带版本号；proxy master 拒绝 version > 自身能力的 hello 帧并回退重连，保证滚动升级期间 conductor/proxy master 版本不同时不崩溃。

### 3.5 可选替代方案

| 方案 | 描述 | 放弃原因 |
|------|------|---------|
| 纯 iptables/nftables DNAT | 不需要 proxy 进程，直接在宿主机规则链转发 | 规则随沙箱密度线性膨胀；控制面重启时规则管理复杂；无 park/wake 能力 |
| Envoy xDS proxy | 复用成熟代理组件 | 引入 C++ 依赖；xDS 协议替换 routesync 增加集成成本；MMDS 无法内嵌 |
| 用户态纯 splice（不集成 eBPF）| 实现简单 | 当前即为此方案；高并发下每个报文均需用户态往返，吞吐受限；叠加 eBPF flowtable 后此方案退化为首包处理 + 回退模式 |

---

## 4. 详细设计

### 4.0 完整进程拓扑

#### 4.0.1 节点进程全景图

下图展示 proxy master/worker 与节点上所有上下文进程、内核组件、集群层之间的完整拓扑关系（`proxy.mode=external`）。

```
┌─ Cluster 层 ───────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                                                                                                │
│  ┌───────────────────────────────────────────┐   ┌────────────────────────────────────────────────────────┐   │
│  │  cluster-ctl registry / scaler            │   │  cluster-ctl router                                    │   │
│  │  ▲ heartbeat / route_upsert / build_event │   │  Host: api.<domain>        → node-ctl conductor  :443  │   │
│  │  ▼ command(create/connect/delete/key_put) │   │  Host: <port>-<sid>.<dom>  → proxy worker         :8443│   │
│  └───────────────────────┬───────────────────┘   └──────────────────────────────────────┬─────────────────┘   │
└──────────────────────────┼──────────────────────────────────────────────────────────────┼─────────────────────┘
                           │ node-link (mTLS / h2c)                                        │ HTTPS
                           │ 上行: register · heartbeat · route_upsert · build_event        │ :443 控制面
                           │ 下行: command(create / connect / delete / key_put / build)     │ :8443 数据面
                           ▼                                                               ▼
┌─ Node 宿主机 ──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                                                                                                                                    │
│  ┌─ node-ctl conductor serve ────────────────────────────────────────┐  ┌─ node-ctl proxy master (root netns) ─────────────────────────────────┐  │
│  │ [node-ctl.service  Restart=on-failure]                            │  │ [node-ctl-proxy.service]  proxy_socket /run/sandbox/proxy.sock       │  │
│  │ api.listen :443 (TLS / h2c)                                      │  │                                                                      │  │
│  │                                                                  │  │  ┌─ RouteSyncClient (plugin 平面) ─────────────────────────────────┐  │  │
│  │  ┌─ HTTPServer ──────────────────────────────────────────┐       │  │  │ 上行: register 帧（首帧）/ wake 帧（按需）                    │  │  │
│  │  │ /sandboxes  /templates  /health                       │       │  │  │ 下行: hello/upsert/delete/bookmark                            │  │  │
│  │  │ X-API-KEY / Bearer 鉴权                               │       │  │  │ → 写 proxyshm（seqlock CAS）+ 广播 notify pipe                │  │  │
│  │  └────────────────────────────────────────────────────────┘       │  │  │ 断连: 指数退避 100ms→30s                                      │  │  │
│  │                                                                  │  │  └──────────────────────────────────────────────────────────────────┘  │  │
│  │  ┌─ orchestrator core ──────────────────────────────────┐         │  │  收 wake pipe（workers）→ 去重 → 上报 Wake 帧                      │  │
│  │  │  沙箱状态机 running / paused                          │         │  │  广播 notify pipe → workers 检测 GlobalRev 变化                   │  │
│  │  │  Reconcile(对账收养) / Reaper(5s)                     │         │  │  startCommandInNetNS(sw0_mgmt) + fd 继承: data/forward/mmds        │  │
│  │  │  BuildPool(2s) / StreamAuthority                      │         │  │                                                                      │  │
│  │  └────────────────────────────────────────────────────────┘         │  │  ┌─ proxy worker × K（mgmt netns sw0_mgmt）──────────────────┐    │  │
│  │                                                                  │  │  │ SO_REUSEPORT :8443（继承 data listener fd）                │    │  │
│  │  ┌─ resource controller ────────────────────────────────┐         │  │  │ TLSListener · HostRouter（<port>-<sid> / E2b-* header）   │    │  │
│  │  │ [resource_listen.socket UDS]                        │         │  │  │ auth: envdsign.CheckDataPlaneAuth                         │    │  │
│  │  │  AdmissionCtrl / 令牌桶 / 四水位                    │         │  │  │ proxyshm WorkerView（mmap 只读, seqlock O(1)）            │    │  │
│  │  │  Active Reclaimer ◄── sandbox-ctl                   │         │  │  │   {Profile, State, FloatingIP, EnvdUDS, CiUDS,            │    │  │
│  │  └────────────────────────────────────────────────────────┘         │  │  │    AccessToken, TrafficAccessToken, MmdsSecret}           │    │  │
│  │                                                                  │  │  │ Dispatcher（profile × port）                             │    │  │
│  │  ┌─ config-socket (UDS, h2c, 4 planes) ──────────────────┐       │  │  │   e2b + 49983/49999 → KindUDS splice                     │    │  │
│  │  │ /run/sandbox/node-ctl.socket                          │       │  │  │   bare + 49983/49999 → KindDeny → 501                    │    │  │
│  │  │ task  plane ◄── sandbox-ctl (SO_PEERCRED)            │       │  │  │   any  + other       → KindTCP via mg0→sw-mX            │    │  │
│  │  │  POST /internal/task/launchspec                       │       │  │  │                         [FlowTableWriter 计划中]         │    │  │
│  │  │ admin plane ◄── node-ctl CLI                         │       │  │  │   paused → park + wake pipe ──► master                  │    │  │
│  │  │  /internal/admin/*  (manifest-key 等)                │       │  │  │   KindNotFound → 404                                     │    │  │
│  │  │                                                       │       │  │  │ MMDS v2（mmds fd 继承, 127.0.0.1:19254）                 │    │  │
│  │  │ plugin plane ◄──────────────────────────────────────── │───── ┤  │  │   PUT /latest/api/token → {sid}.{hex-HMAC-SHA256}       │    │  │
│  │  │  routesync 下行帧:                                    │◄──── ┤  │  │   GET /（X-metadata-token）→ {instanceID,envID,hash}    │    │  │
│  │  │  hello / upsert / delete / bookmark                   │───── ►  │  │ MetricsServer: GET /metrics → :9090                     │    │  │
│  │  │  SO_PEERCRED pid in plugin_pidfile                    │       │  │  └──────────────────────────────────────────────────────────┘    │  │
│  │  │ api   plane ◄── external REST clients                │       │  └──────────────────────────────────────────────────────────────────┘  │
│  │  └────────────────────────────────────────────────────────┘       │                                                                        │
│  │                                                                  │  proxyshm: /run/sandbox/proxy-routes.shm（65536 slots）               │
│  │  ┌─ node-link ─────────────────────────────────────────────┐      │  master 写（seqlock CAS）/ workers mmap 只读                          │
│  │  │  → cluster-ctl registry (mTLS h2c)                     │      │                                                                        │
│  │  │  上行: register/heartbeat/route_upsert                  │      │                                                                        │
│  │  │  下行: handleCommand                                    │      │                                                                        │
│  │  └─────────────────────────────────────────────────────────┘      │                                                                        │
│  │                                                                  │                                                                        │
│  │  ┌─ SQLite (WAL) ────────────────────────────────────────┐        │                                                                        │
│  │  │ /var/lib/sandbox/node-ctl.db                          │        │                                                                        │
│  │  │ sandboxes / builds / manifest_keys                    │        │                                                                        │
│  │  └────────────────────────────────────────────────────────┘        │                                                                        │
│  └─────────────────────────┬──────────────────────────────────────────┘                                                                        │
│                            │ D-Bus / sdbus                                                                                                     │
│                            │ StartUnit / StopUnit / ListUnitsByPatterns                                                                        │
│                            │ + connector-ctl vswitch attach/detach  (sandbox-ctl 子进程调用)                                                   │
│                            ▼                                                                                                                   │
│  ┌─ systemd [sandbox 单元，由 conductor 通过 D-Bus 动态生成与管理] ──────────────────────────────────────────────────────────────────────┐   │
│  │                                                                                                         │   │
│  │  sandbox-runner@<sid>.service × N   Type=oneshot  Restart=no  KillMode=control-group  ← node-ctl 生成  │   │
│  │    ExecStart: node-ctl run-sandbox --sandbox-id=<sid> --config-socket=...                               │   │
│  │                                                                                                         │   │
│  │  sandbox-builder@<bid>.service × M  Type=oneshot  Restart=no  KillMode=control-group  ← node-ctl 生成  │   │
│  │    ExecStart: node-ctl run-builder --build-id=<bid> --config-socket=...                                 │   │
│  │                                                                                                         │   │
│  └────────────────────────────────────────────┬────────────────────────────────────────────────────────────┘   │
│                                               │ execve  (node-ctl run-sandbox / run-builder)                   │
│                                               ▼                                                                 │
│  ┌─ sandbox-ctl × (N+M) [host netns，不做命名空间隔离] ───────────────────────────────────────────────────  ┐   │
│  │                                                                                                          │   │
│  │  ── config-socket task plane → node-ctl conductor                                                       │   │
│  │     POST /internal/task/launchspec   SO_PEERCRED 鉴权   FetchLaunchSpec / FetchBuildSpec                 │   │
│  │                                                                                                          │   │
│  │  ── resource_listen.socket → resource controller (node-ctl conductor 内嵌，可选)                         │   │
│  │     Admit / Settled / Heartbeat(~30s) / OOMReport / Release / Reattach                                   │   │
│  │     帧格式: [4B LE 长度][JSON]                                                                           │   │
│  │                                                                                                          │   │
│  │  ┌─ Forwarder  (e2b profile 专有，pkg/sandbox/forward.go) ──────────────────────────────────────────┐   │   │
│  │  │  host 侧创建 UDS 监听器；每条新连接经两层协议向 guest 建立 vsock 反向通道:                       │   │   │
│  │  │  层1: dial vsock.sock → CONNECT <port>\n → CH virtio-vsock → guest virtio_vsock → sandbox-init   │   │   │
│  │  │  层2: TypeConnect{Network,Address} → sandbox-init dial 目标本地端口                               │   │   │
│  │  │  /run/sandbox/<sid>/envd.sock ←── proxy dial(49983) → TypeConnect{tcp,127.0.0.1:49983} → envd    │   │   │
│  │  │  /run/sandbox/<sid>/ci.sock   ←── proxy dial(49999) → TypeConnect{tcp,127.0.0.1:49999} → ci     │   │   │
│  │  │  Pause()/CloseActive()/Resume(): 快照静止期冻结新连接，折叠活跃 relay                              │   │   │
│  │  └───────────────────────────────────────────────────────────────────────────────────────────────── ┘   │   │
│  │                                                                                                          │   │
│  │  ctl.sock  /run/sandbox/<sid>/ctl.sock  ← snapshot/exec CLI 子进程通过 --run-root 定位                  │   │
│  │    snapshot_request: 暂停 VM → CH API → 创建快照 → 返回路径                                             │   │
│  │    exec_request: 握手 → stdio MUX relay                                                                 │   │
│  │                                                                                                          │   │
│  │  注(cgroup): --cgroup-adopt：sandbox-ctl 与 CH 同处 unit cgroup；--cgroup-isolated（计划）：CH 移入子 cgroup│  │
│  │                                                                                                          │   │
│  │  ┌─ cloud-hypervisor ──────────────────────────────────────────────────────────────────────────────  ┐  │   │
│  │  │  tap fd ←── connector-ctl vswitch open-port (SCM_RIGHTS; tapfd 模式: setns 进入 sw0_vswitch netns) │  │   │
│  │  │  vsock.sock  /run/sandbox/<sid>/vsock.sock  ← 层1入口  CONNECT <port>\n → CH virtio-vsock → guest  │  │   │
│  │  │                                                                                                    │  │   │
│  │  │  ┌─ Guest VM [KVM 硬件隔离, guest netns] ──────────────────────────────────────────────────  ┐    │  │   │
│  │  │  │  sandbox-init (PID 1) → POST /init  (envd re-key, Secure 模式)                            │    │  │   │
│  │  │  │  sandbox-init ← serveReverseChannel: 接受 virtio-vsock → 读层2 TypeConnect → dial → relay  │    │  │   │
│  │  │  │  envd :49983  code-interp :49999  (经 vsock 通道)                                          │    │  │   │
│  │  │  │  user app :PORT  (via virtio-net, 不经 vsock)                                              │    │  │   │
│  │  │  │    入向: proxy worker dial(floatingip:PORT) → mg0(mgmt netns) → sw-mX(switch netns)         │    │  │   │
│  │  │  │           → TC DNAT → bpf_redirect → sw0-tN tap → CH → guest                               │    │  │   │
│  │  │  │    回程: guest → CH → tap → sw0-tN TC SNAT → sw-mX → mg0 → proxy worker TCP socket         │    │  │   │
│  │  │  │    规划: transit NIC TC flowtable 命中 → 绕过 proxy worker 直通 → guest                    │    │  │   │
│  │  │  │  MMDS: 169.254.169.254:80 ──(vswitch DNAT)──► 127.0.0.1:19254 (proxy MMDS, mgmt netns)    │    │  │   │
│  │  │  │  DNS:  169.254.169.253:53 ──(vswitch DNAT)──► 127.0.0.1:19253 (CoreDNS)                   │    │  │   │
│  │  │  │  出站: guest NIC → CH tap → vswitch TC eBPF(GENEVE) → transit NIC → 外部网关               │    │  │   │
│  │  │  └──────────────────────────────────────────────────────────────────────────────────────────── ┘    │  │   │
│  │  └────────────────────────────────────────────────────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                                                 │
│  ┌─ sandbox-vswitch [connector-ctl vswitch serve, sw0_vswitch netns] ──────────────────────────────────────┐   │
│  │  ← connector-ctl vswitch attach --inner-ip=... --port=N  (sandbox 启动时，sandbox-ctl 调)              │   │
│  │  ← connector-ctl vswitch detach --port=N                 (sandbox 销毁时)                              │   │
│  │  ← connector-ctl vswitch open-port --port=N → tap fd (SCM_RIGHTS) → CH  (tapfd 模式)                  │   │
│  │                                                                                                          │   │
│  │  mgmt-service DNAT (mgmt-extract netns sw0_mgmt 内):                                                    │   │
│  │    169.254.169.254:80  (UDP+TCP) → 127.0.0.1:19254   proxy worker MMDS v2                              │   │
│  │    169.254.169.253:53  (UDP+TCP) → 127.0.0.1:19253   CoreDNS                                          │   │
│  │                                                                                                          │   │
│  │  GENEVE 跨宿主: sandbox floatingip ←── transit NIC (eth1, 已移入此 netns) ──► gateway / 对端节点        │   │
│  └──────────────────────────────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                                                 │
│  ┌─ cache-ctl / store-ctl [sandbox-accelerator] ──────────┐                                                     │
│  │  node-ctl.service 强依赖 After=cache-ctl store-ctl      │                                                     │
│  │  snapshot ingest / chunk 去重 / L1-L3 分层缓存          │                                                     │
│  └─────────────────────────────────────────────────────────┘                                                     │
│                                                                                                                 │
│  ┌─ Linux 内核 ────────────────────────────────────────────────────────────────────────────────────────────  ┐   │
│  │  bpffs  /sys/fs/bpf/vswitch/                                                                             │   │
│  │  ├─ prog_tc_ingress / prog_tc_egress   TC hook, attach on transit NIC (eth1)                            │   │
│  │  │                                                                                                       │   │
│  │  ├─ map_flowtable   BPF_MAP_TYPE_LRU_HASH  [计划中，未实现]                                             │   │
│  │  │    key:  {src_ip, src_port, dst_ip, dst_port} (网络字节序)                                            │   │
│  │  │    val:  {floatingip, port}                                                                           │   │
│  │  │    ↑ proxy worker FlowTableWriter  bpf_map_update_elem  (新建 TCP 连接后写入) [未实现]               │   │
│  │  │    ↑ proxy worker              bpf_map_delete_elem  (连接关闭后清理)      [未实现]                   │   │
│  │  │    ← TC hook 读取: 已建连接在内核路径直接转发, 报文不再经过 proxy worker 用户态 [未实现]               │   │
│  │  │                                                                                                       │   │
│  │  ├─ map_conntrack   SYNACK / ESTABLISHED / CLOSING  (连接状态追踪)                                       │   │
│  │  └─ map_egress_policy / map_egress_allow_lpm / map_egress_deny_lpm  (出站策略)              │   │
│  │                                                                                                          │   │
│  │  cgroup v2  当前: unit cgroup（--cgroup-adopt，sandbox-ctl + CH 同处）                                  │   │
│  │             计划: <unit-cgroup>/sandbox-<sid>/（--cgroup-isolated，仅 CH）                              │   │
│  │  ├─ memory.max / memory.current / memory.stat     ← resource controller 读取 (Active Reclaimer)        │   │
│  │  └─ cpu.stat / cpuset.cpus.effective              ← metrics API 读取                          │   │
│  └──────────────────────────────────────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

#### 4.0.2 proxy master / worker 内部模块与 IPC 汇总

```
外部入站 (cluster-router → :8443 HTTPS)
         │
         ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────────────┐
│  node-ctl proxy master 进程（root netns）  [node-ctl-proxy.service]                                 │
│                                                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────────────────────────┐  │
│  │  goroutine: RouteSyncClient  (常驻, config-socket plugin plane)                               │  │
│  │  连接: PUT /internal/plugin/{id}/register  [/run/sandbox/node-ctl.socket  UDS h2c]           │  │
│  │  SO_PEERCRED pid 鉴权  (conductor 侧验 plugin_pidfile)                                        │  │
│  │  上行:  register 帧（首帧）/ wake 帧（按需, per-sid singleflight）                             │  │
│  │  下行:  hello    → 更新 auth/park_timeout 策略                                               │  │
│  │         upsert   → 写 proxyshm slot（seqlock CAS）+ 广播 notify pipe                         │  │
│  │         delete   → 清 proxyshm slot + 广播 notify pipe                                       │  │
│  │         bookmark → 清孤儿 slot（首次全量结束）                                                │  │
│  │  断连重试: 指数退避 100ms → 200ms → 400ms … max 30s                                           │  │
│  └──────────────────────────────────────────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────────────────────────────────────────┐  │
│  │  goroutine: wake pipe reader                                                                  │  │
│  │  从所有 workers 读 wake pipe → 去重（per-sid singleflight）→ 上报 routesync Wake 帧           │  │
│  └──────────────────────────────────────────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────────────────────────────────────────┐  │
│  │  goroutine: notify pipe writer                                                                │  │
│  │  proxyshm GlobalRev 增加 → 向所有 workers 广播 notify pipe（1字节写入，触发 re-lookup）        │  │
│  └──────────────────────────────────────────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────────────────────────────────────────┐  │
│  │  proxyshm writer  (seqlock CAS, /run/sandbox/proxy-routes.shm, 65536 slots)                  │  │
│  │  SandboxSlot: {SandboxID, Profile, State, FloatingIP, EnvdUDS, CiUDS,                        │  │
│  │               AccessToken, TrafficAccessToken, MmdsSecret, SnapshotLocation}                 │  │
│  └──────────────────────────────────────────────────────────────────────────────────────────────┘  │
│  startCommandInNetNS(sw0_mgmt, "node-ctl proxy worker") × K                                         │
│  fd 继承: dataLn(:8443 SO_REUSEPORT) · forwardLn(UDS) · mmdsLn(127.0.0.1:19254)                    │
└──────────────────────────────────────────────────────┬──────────────────────────────────────────────┘
                                                       │ exec 进入 mgmt netns (sw0_mgmt)
                                                       ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────────────┐
│  node-ctl proxy worker × K（mgmt netns sw0_mgmt）                                                   │
│                                                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────────────────────────┐  │
│  │  goroutine: Accept 循环  (dataLn: TLSListener :8443 SO_REUSEPORT, 继承 fd)                   │  │
│  └──────────────────────────────┬───────────────────────────────────────────────────────────────┘  │
│                                 │ per-conn goroutine                                               │
│                                 ▼                                                                   │
│  ┌──────────────────────────────────────────────────────────────────────────────────────────────┐  │
│  │  goroutine: 请求处理                                                                          │  │
│  │  1. HostRouter.Parse(req.Host)       → (sid, port)     失败 → 400                            │  │
│  │  2. proxyshm.WorkerView.Lookup(sid)  → RouteEntry      未命中 → 404                          │  │
│  │  3. envdsign.CheckDataPlaneAuth(req, entry.AccessToken) 不符 → 401 (enforce 模式)            │  │
│  │  4. RouteForTarget(profile, port):                                                            │  │
│  │     ┌─────────────────────────────────────────────────────────────────────────────────────┐  │  │
│  │     │  KindUDS  (e2b + 49983/49999)                                                       │  │  │
│  │     │    dial forwardLn UDS → io.Copy 双向 splice                                         │  │  │
│  │     │    (envd.sock / ci.sock，Forwarder 持有；内部经 vsock 穿透到 guest envd/ci)          │  │  │
│  │     │                                                                                    │  │  │
│  │     │  KindDeny (bare + 49983/49999) → 501 Not Implemented                               │  │  │
│  │     │                                                                                    │  │  │
│  │     │  KindTCP  (any + other port)                                                        │  │  │
│  │     │    dial TCP(entry.FloatingIP:port) via mg0 → sw-mX → guest                         │  │  │
│  │     │    FlowTableWriter.Register / .Remove [bpffs，计划中]                               │  │  │
│  │     │    io.Copy 双向 splice                                                               │  │  │
│  │     │                                                                                    │  │  │
│  │     │  state=paused                                                                      │  │  │
│  │     │    ParkQueue.Park(sid, conn, deadline=now+park_timeout)                            │  │  │
│  │     │    write wake pipe ──► master → routesync Wake 帧 → conductor                      │  │  │
│  │     │    等待 notify pipe → GlobalRev 变更 → re-lookup                                    │  │  │
│  │     │    UnparkAll(sid) [wake 成功] → 重走 KindUDS/KindTCP 分支                           │  │  │
│  │     │    Cancel(sid)    [超时/delete] → conn.Close() → 503                               │  │  │
│  │     │                                                                                    │  │  │
│  │     │  KindNotFound (无路由 / 超时未 resume) → 404                                        │  │  │
│  │     └─────────────────────────────────────────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────────────────────────────────────────┐  │
│  │  goroutine: MMDS v2 server  (mmdsLn: HTTP/1.1  127.0.0.1:19254, mgmt netns, 继承 fd)        │  │
│  │  PUT /latest/api/token  ByFloatingIP(src_ip) → sid → mmds_secret                            │  │
│  │    token = sid + "." + hex(HMAC-SHA256(mmds_secret, sid))  (确定性，无存储，无 TTL)          │  │
│  │  GET / (X-metadata-token)  验 token → {instanceID, envID, accessTokenHash}                  │  │
│  │    accessTokenHash = hex(sha512(accessToken))                                               │  │
│  └──────────────────────────────────────────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────────────────────────────────────────┐  │
│  │  goroutine: MetricsServer  (HTTP  127.0.0.1:9090)  GET /metrics → Prometheus text format     │  │
│  └──────────────────────────────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────────────────────────────┘
        │ UDS splice (forwardLn)   │ TCP splice via mg0   │ bpffs               │ wake pipe → master
        ▼                         ▼                      ▼                    ▼        → routesync
  envd.sock / ci.sock       floatingip:PORT        /sys/fs/bpf/vswitch/  /run/sandbox/node-ctl.socket
  (host UDS, sandbox-ctl    (mg0→sw-mX→tap→CH)    map_flowtable          (conductor plugin plane)
   Forwarder 持有)
```

#### 4.0.3 proxy master/worker 启动与路由同步序列

```
conductor(node-ctl.service)   proxy master              proxy worker × K        sandbox-ctl
         │                         │                          │                      │
         │   [master 启动]          │                          │                      │
         │   startCommandInNetNS   │  exec + fd 继承           │                      │
         │                         ├─────────────────────────►│                      │
         │                         │  data/forward/mmds ln    │ Accept 循环就绪        │
         │                         │                          │ MMDS server 就绪      │
         │   [master 注册握手]      │                          │                      │
         │   PUT /plugin/.../register                          │                      │
         │◄────────────────────────┤                          │                      │
         │   SO_PEERCRED 鉴权      │                          │                      │
         │   → hello 帧 (policy)   │                          │                      │
         ├────────────────────────►│  写 proxyshm + notify    │                      │
         │                         ├─────────────────────────►│                      │
         │                         │                          │                      │
         │   [全量同步]             │                          │                      │
         │   upsert(sid-1, running)│                          │                      │
         ├────────────────────────►│ seqlock CAS proxyshm     │                      │
         │   upsert(sid-2, paused) │ + 广播 notify pipe       │                      │
         ├────────────────────────►├─────────────────────────►│ WorkerView.Lookup    │
         │   ...                   │                          │   可见最新路由         │
         │   bookmark              │                          │                      │
         ├────────────────────────►│ 清孤儿 slot               │                      │
         │                         │                          │                      │
         │   [增量: sandbox 创建]   │                          │                      │
         │◄── Admit ────────────────────────────────────────────────────────────────┤
         │   upsert(new-sid, running)                          │                      │
         ├────────────────────────►│ 写 proxyshm + notify     │                      │
         │                         ├─────────────────────────►│                      │
         │                         │                          │                      │
         │   [增量: sandbox 暂停]   │                          │                      │
         │   upsert(sid-X, paused) │                          │                      │
         ├────────────────────────►│ 写 proxyshm + notify     │                      │
         │                         ├─────────────────────────►│                      │
         │                         │                          │                      │
         │   [worker 收到针对 sid-X 的请求]                    │                      │
         │                         │  state=paused            │                      │
         │                         │  ParkQueue.Park(sid-X)   │                      │
         │                         │◄─── wake pipe ───────────┤                      │
         │                         │ 去重 singleflight        │                      │
         │   wake(sid-X) ──────────┤                          │                      │
         │◄────────────────────────┤                          │                      │
         │   resumeIfPaused(sid-X) │                          │                      │
         │   upsert(sid-X, running)│                          │                      │
         ├────────────────────────►│ 写 proxyshm + notify     │                      │
         │                         ├─────────────────────────►│ notify pipe 触发      │
         │                         │                          │ re-lookup → running  │
         │                         │                          │ UnparkAll(sid-X)     │
         │                         │                          │ 重走 KindUDS/KindTCP │
         │                         │                          │                      │
         │   [增量: sandbox 销毁]   │                          │                      │
         │   delete(sid-Y)         │                          │                      │
         ├────────────────────────►│ 清 proxyshm slot         │                      │
         │                         │ + 广播 notify pipe        │                      │
         │                         ├─────────────────────────►│ Cancel(sid-Y)        │
         │                         │                          │ → conn.Close() → 404 │
         │                         │                          │                      │
         │   [断连重连]             │ (conductor 重启/通道中断) │                      │
         │                         │ 指数退避 100ms → 30s      │                      │
         │   PUT /plugin/.../register                          │                      │
         │◄────────────────────────┤                          │                      │
         │   重新注册 → hello + 全量 upsert + bookmark          │                      │
         ├────────────────────────►│ 写 proxyshm + notify     │                      │
         │                         ├─────────────────────────►│                      │

   [数据面: envd 端口连接建立（port=49983 为例）]

   proxy worker       Forwarder(sandbox-ctl)    cloud-hypervisor   sandbox-init  envd:49983
        │                    │                         │                │               │
        │ KindUDS:            │                         │                │               │
        │ dial(forwardLn UDS) │                         │                │               │
        ├───────────────────►│                         │                │               │
        │                    │ [层1] dial vsock.sock   │                │               │
        │                    │       CONNECT <port>\n  │                │               │
        │                    ├────────────────────────►│                │               │
        │                    │                         │ virtqueue → guest               │
        │                    │                         ├───────────────►│               │
        │                    │ [层2] TypeConnect{tcp, 127.0.0.1:49983} │               │
        │                    ├─────────────────────────────────────────►│               │
        │                    │                         │                │ dial :49983   │
        │                    │                         │                ├──────────────►│
        │◄───────────────────────────────────────────────────────────────────────────── │
        │    io.Copy relay（proxy worker ↔ forwardLn ↔ vsock ↔ sandbox-init ↔ envd）

   [数据面: 用户端口连接建立（port=PORT，sandbox-ctl 不参与）]

   proxy worker    mg0(mgmt netns)→sw-mX(sw netns)    cloud-hypervisor   guest user:PORT
        │                   │                                 │                  │
        │ KindTCP:           │                                 │                  │
        │ dial(floatingip:PORT)                               │                  │
        ├──────────────────►│                                 │                  │
        │                   │ TC DNAT(floatingIP→inner_ip)    │                  │
        │                   │ bpf_redirect → sw0-tN tap fd    │                  │
        │                   ├────────────────────────────────►│                  │
        │                   │                                 │ virtio-net → guest│
        │                   │                                 ├─────────────────►│
        │                   │                                 │                  │ TCP 连接建立
        │◄─────────────────────────────────────────────────────────────────────── │
        │    TCP splice（proxy worker ↔ mg0 ↔ sw-mX ↔ tap ↔ CH ↔ guest）
        │    规划: transit NIC TC hook flowtable 命中 → 内核直通，绕过 proxy worker
```

#### 4.0.4 关键 IPC 通道一览

**控制面**

| 通道 | 类型 | 端点 | 通信方向 | 鉴权方式 |
|------|------|------|---------|---------|
| config-socket plugin 平面 | UDS h2c 双向流 | `/run/sandbox/node-ctl.socket` | proxy master ↔ conductor (routesync) | SO_PEERCRED + plugin_pidfile |
| config-socket task 平面 | UDS h2c | `/run/sandbox/node-ctl.socket` | sandbox-ctl → conductor | SO_PEERCRED (sandbox-ctl pid) |
| config-socket admin 平面 | UDS h2c | `/run/sandbox/node-ctl.socket` | node-ctl CLI → conductor | SO_PEERCRED + socket 0600 |
| resource_listen.socket | UDS 帧化 JSON | `/run/sandbox/resource.sock` | sandbox-ctl → resource controller | UDS socket 权限 |
| node-link | mTLS h2c / plain h2c | cluster-ctl registry host:port | conductor ↔ cluster-ctl | mTLS 证书 |
| proxyshm | mmap 共享内存 | `/run/sandbox/proxy-routes.shm` | master 写（seqlock CAS）/ workers 只读 mmap | 文件权限 |
| wake/notify pipe | pipe fd（继承） | in-process | workers → master（wake）/ master → workers（notify）| fd 继承（进程内） |
| connector-ctl vswitch | subprocess SCM_RIGHTS | (tapfd 模式) | sandbox-ctl → cloud-hypervisor（TAP fd 一次性交接）| — |
| D-Bus / sdbus | D-Bus | systemd socket | conductor → systemd (StartUnit/StopUnit) | D-Bus policy |
| Prometheus metrics | HTTP loopback | `127.0.0.1:9090` | 监控采集 → proxy worker | — |

**数据面**

| 通道 | 类型 | 端点 | 通信方向 | 鉴权方式 |
|------|------|------|---------|---------|
| envd.sock / ci.sock | UDS (AF_UNIX stream) | `/run/sandbox/<sid>/envd.sock` | proxy worker → Forwarder（vsock 反向通道入口）| — (上层 token 鉴权) |
| vsock.sock | UDS (AF_UNIX stream) | `/run/sandbox/<sid>/vsock.sock` | Forwarder → CH virtio-vsock（层1 CONNECT + 层2 TypeConnect）| — (host-only socket) |
| floatingip:PORT | TCP | sandbox floatingip:PORT | **入向**（当前）proxy worker → mg0(mgmt netns) → sw-mX(switch netns) TC DNAT → bpf_redirect → sw0-tN tap → CH → guest；**回程**（当前）guest → tap → sw0-tN TC SNAT → sw-mX → mg0 → proxy worker TCP socket；**规划**：transit NIC TC hook flowtable 命中 → 绕过 proxy worker 内核直通 | envdsign.CheckDataPlaneAuth (proxy 层) |
| bpffs map_flowtable **[计划中，未实现]** | 内核 BPF map | `/sys/fs/bpf/vswitch/map_flowtable` | proxy worker 写 / TC hook 读（transit NIC 双向命中） | CAP_SYS_ADMIN (root only) |
| MMDS v2 HTTP | HTTP/1.1 loopback | `127.0.0.1:19254` (mgmt netns) | guest envd → proxy worker（经 vswitch DNAT, mgmt netns）| MMDS session token |

#### 4.0.5 proxy.mode=internal 进程拓扑差异

`proxy.mode=internal` 将 serve 与 proxy 合并为同一个 Go 进程运行，主要用于开发/单机调试场景。与 `external` 模式的差异如下：

**进程拓扑对比**

```
proxy.mode=external（生产）                       proxy.mode=internal（开发/调试）
──────────────────────────────────────────────────────────────────────────────────
node-ctl.service                                  node-ctl.service
  └── node-ctl conductor serve (root netns)          └── node-ctl conductor serve
        │ config-socket plugin plane                        ┌── [内嵌] proxy 逻辑
        │ (routesync IPC → master)                          │     TLSListener :8443
        │                                                   │     HostRouter
node-ctl-proxy.service                                      │     Dispatcher / ParkQueue
  └── node-ctl proxy master (root netns)                    │     MMDS v2 server
        RouteSyncClient → config-socket                     │     MetricsServer
        proxyshm writer (seqlock)                      无 routesync IPC
        startCommandInNetNS(sw0_mgmt, worker)          无 proxy master/worker 分进程
     └── node-ctl proxy worker × K (mgmt netns)        RouteTable 为 conductor 内部对象
           proxyshm WorkerView (mmap 只读)             无 proxyshm
           TLSListener :8443 (SO_REUSEPORT, fd 继承)
           MMDS v2 (fd 继承, mgmt netns)
```

**关键差异汇总**

| 维度 | external | internal |
|------|----------|----------|
| 进程数 | 1 conductor + 1 master + K workers | 1 进程（conductor + proxy 合一） |
| proxy master 进程 | 有（root netns，RouteSyncClient + proxyshm 写）| 无 |
| proxy worker 进程 | 有（mgmt netns，K 个，继承 fd）| 无 |
| routesync IPC | config-socket plugin plane (UDS h2c) master↔conductor | 无（进程内直接共享） |
| 路由表存储 | proxyshm mmap（master 写，workers 只读，seqlock）| 进程内 RouteTable（无序列化） |
| wake 机制 | worker 写 wake pipe → master 去重 → routesync Wake 帧 → conductor | Dispatcher 直接调用 orchestrator.Wake(sid) |
| MMDS v2 server | proxy worker 进程内（mgmt netns，继承 fd）| conductor 进程内 |
| data_listen netns | proxy worker 在 mgmt netns 中 Accept | conductor 进程，host netns |
| SO_REUSEPORT | workers 共享 :8443（fd 继承）| 单监听器 |
| 故障隔离 | conductor 崩溃不影响 workers（数据面继续服务）| conductor 崩溃 = 控制面 + 数据面同时不可用 |
| 适用场景 | 生产环境（高可用、多 worker、mgmt netns 隔离）| 本地开发、单机测试（配置简单） |

**park/wake 差异（internal 模式下无 wake pipe / wake 帧）**

```
external 模式:
  worker Dispatcher: state=paused
    → ParkQueue.Park(sid, conn)
    → write wake pipe ──► master（去重 singleflight）
    → master: routesync Wake 帧 → conductor
    → conductor: resumeIfPaused(sid) → upsert(sid, running) → proxyshm + notify pipe
    → worker: notify pipe 触发 → re-lookup → running → UnparkAll(sid)

internal 模式:
  Dispatcher: state=paused
    → ParkQueue.Park(sid, conn)
    → orchestrator.Wake(sid)         ← 进程内直接调用（无 IPC）
    → sandbox 恢复后更新 RouteTable
    → UnparkAll(sid)
```

**注意事项**

- `proxy.mode=internal` 不支持多 worker 水平扩展，:8443 仅有单监听器。
- 生产环境必须使用 `proxy.mode=external`，以实现 serve/proxy 故障隔离与独立重启。
- `proxy.mode=off` 时不启动任何数据面监听，详见 §4.0.6。

#### 4.0.6 proxy.mode=off 进程拓扑说明

`proxy.mode=off` 是纯控制面模式，node-ctl conductor 启动后**不构建任何数据面组件**，适用于离线管控、CI 环境预热、proxy 功能先行合入等场景。

**进程拓扑**

```
node-ctl.service
  └── node-ctl conductor serve
        │
        ├── HTTPServer          :443  (TLS / h2c dev)   ← 控制面 API，正常提供
        ├── orchestrator core                            ← 沙箱状态机，正常运行
        ├── resource controller  resource.sock           ← 资源准入，正常运行
        ├── config-socket        node-ctl.socket         ← task/admin/api 平面，正常运行
        │     plugin 平面 ← 无 proxy master 连接，但平面本身保持监听
        ├── node-link            → cluster-ctl registry  ← 正常上报
        └── SQLite (WAL)         node-ctl.db             ← 正常读写

无 node-ctl-proxy.service（proxy master 不启动）
无 proxy worker（mgmt netns 无进程）
        ✗  TLSListener :8443            ← 不启动
        ✗  HostRouter / proxyshm        ← 不构建
        ✗  Dispatcher / ParkQueue       ← 不构建
        ✗  MMDS v2 server               ← 不启动
        ✗  RouteSyncClient / routesync  ← 不构建（plugin 平面只监听，不推）
        ✗  FlowTableWriter / bpffs 写入 ← 不执行

无外部入站流量路径
```

**与其他模式的关键差异**

| 维度 | external | internal | off |
|------|----------|----------|-----|
| 数据面监听 :8443 | proxy worker × K（mgmt netns）| conductor 内嵌 | **无** |
| routesync 推送 | 有（conductor → proxy master → proxyshm）| 有（in-process）| **无**（plugin 平面保持监听但不推）|
| MMDS v2 server | proxy worker 进程内（mgmt netns）| conductor 进程内 | **不启动** |
| eBPF flowtable 写入 | 有 | 有 | **无** |
| 沙箱生命周期管理 | 正常 | 正常 | **正常**（控制面完整） |
| 外部流量可达性 | 可达 | 可达 | **不可达**（无数据面） |
| `data_requests_total` | 计数 | 计数 | `result="off"` 计数（如有请求打到控制面 :443） |

**启动校验约束**

| 条件 | 结果 |
|------|------|
| `proxy.mode=off` | 正常启动，数据面组件全部跳过 |
| `proxy.mode=off` + `mmds.enabled=true` | **启动失败**（MMDS 依赖数据面路由表，off 模式无路由表） |
| `proxy.mode=off` + `proxy.auth=enforce` | 配置项被忽略（无数据面，auth 配置无意义） |

**典型使用场景**

1. **并行开发**：proxy 功能尚未合入时，`proxy.mode=off` 可先行合入控制面代码，沙箱创建/销毁/暂停等生命周期 API 完整可用，仅缺数据面访问能力。
2. **离线管控节点**：节点仅作资源池成员注册与沙箱调度（cluster-ctl 分配），数据面由其他节点承载（跨节点 migration 场景）。
3. **CI/预热环境**：自动化测试只需验证控制面 API，不需要真实流量，避免 TLS 证书与端口占用。
4. **运维排障**：临时禁用数据面入站（如端口冲突、证书失效），保留控制面以便执行 `node-ctl sandbox list/stop` 等运维操作。

### 4.1 整体拓扑与数据流

下图为 `proxy.mode=external`（生产部署）的节点拓扑；internal 模式下 conductor 进程内嵌 proxy，无 master/worker 拆分。

```
外部 SDK / 用户浏览器
         │ HTTPS  Host: <port>-<sid>.<domain>
         ▼
  cluster-ctl router（集群层，按 Host 分流）
         │ HTTPS :8443（数据面） / :443（控制面 → conductor）
         ▼
┌─── host root netns ────────────────────────────────────────────────────────┐
│  node-ctl conductor  ·  node-ctl proxy master                              │
│  :8443 listener socket（master 在 root netns 创建，exec worker 时 fd 继承）│
│                                                                            │
│  ┌─ node-ctl proxy master ────────────────────────────────────────────┐   │
│  │  ├─ exec 并监管 K 个 worker（startCommandInNetNS→proxy_netns）    │   │
│  │  │    fd 继承：data/forward/mmds listener                         │   │
│  │  ├─ 收 wake pipe → 去重入队 → 上报 routesync Wake 帧             │   │
│  │  ├─ 广播 notify pipe → 通知 worker 路由版本变化                  │   │
│  │  └─ routesync subscriber（config-socket plugin 平面）             │   │
│  │       ← conductor 下行：hello / upsert / delete / bookmark        │   │
│  │       → conductor 上行：register / wake                           │   │
│  │       写入 ──► mmap 共享路由表（proxyshm）                        │   │
│  └───────────────────────────────────────────────────────────────────┘   │
│                                                                            │
│  ┌─ mmap 共享路由表（proxyshm）────────────────────────────────────────┐  │
│  │  文件：/run/sandbox/proxy-routes.shm  容量：65536 槽              │  │
│  │  master 写（seqlock CAS） / worker 只读 mmap                      │  │
│  │  每记录：sid · profile · state · floatingip · envd_uds            │  │
│  │          ci_uds · access_token · traffic_access_token             │  │
│  │          mmds_secret · snap_loc                                   │  │
│  └───────────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────────────┘
         │ fork into proxy_netns（startCommandInNetNS）
         ▼ 继承 :8443 fd（SO_REUSEPORT accept 无需同 netns）
┌─── mgmt netns (sw0_mgmt, proxy_netns) ─────────────────────────────────────┐
│  mg0（mgmt veth peer，--mgmt-extract=sw0_mgmt:mg0:floatingIP_cidr）        │
│                                                                             │
│  ┌─ node-ctl proxy worker × K ──────────────────────────────────────────┐  │
│  │  SO_REUSEPORT :8443（继承自 master 的 fd）                           │  │
│  │                                                                      │  │
│  │  TLS 终止 → Host / E2b-Sandbox-* 解析 → (sid, port)               │  │
│  │  auth: envdsign.CheckDataPlaneAuth                                   │  │
│  │         X-Access-Token  或  envd 文件签名 URL（代理验签）            │  │
│  │                                                                      │  │
│  │  state=running，e2b profile:                                         │  │
│  │    port 49983 → envd.sock（KindUDS splice）                         │  │
│  │    port 49999 → ci.sock（KindUDS splice）                           │  │
│  │    其他端口   → floatingip:port（KindTCP splice via mg0→sw-mX）     │  │
│  │  state=running，bare profile:                                        │  │
│  │    port 49983/49999 → KindDeny → 501 Not Implemented               │  │
│  │    其他端口   → floatingip:port（KindTCP splice via mg0→sw-mX）     │  │
│  │                                                                      │  │
│  │  state=paused:                                                       │  │
│  │    → per-sid park 队列（挂起，最长 park_timeout）                   │  │
│  │    → wake pipe ──► master ──► routesync Wake 帧 ──► conductor       │  │
│  │                                                                      │  │
│  │  KindNotFound（无路由 / 超时未 resume）→ 404                         │  │
│  │                                                                      │  │
│  │  路由读取: mmap 共享路由表（只读，seqlock O(1) hash 查询）           │  │
│  │  路由感知: master notify pipe → worker 检测 GlobalRev 变化          │  │
│  │                                                                      │  │
│  │  MMDS v2（mmds.enabled=true）:                                       │  │
│  │    PUT /latest/api/token → {sid}.{hex-HMAC-SHA256(mmds_secret,sid)} │  │
│  │    GET /（envd 传 X-metadata-token）→ {instanceID,envID,accessTokenHash} │
│  └───────────────────────────────────────────────────────────────────── ┘  │
└─────────────────────────────────────────────────────────────────────────────┘

  eBPF flowtable（/sys/fs/bpf/vswitch/）[计划中，未实现]
  {src_ip,src_port,dst_ip,dst_port} → {floatingip,port}
  TC hook 内核直通（已建连接绕过 proxy 用户态）
```

**internal 模式对比**（`proxy.mode=internal`）：conductor（`node-ctl conductor serve`）进程内嵌 `proxy.NewWithDialer`，以 `Orchestrator` 路由视图直接解析路由，无 master/worker 进程、无 proxyshm；TCP 拨号可通过 `proxy_netns` 在指定 netns 内发起。external 模式下 conductor 的数据面入口退化为 `proxyForwarder`，将请求经 UDS（`proxy_socket`）转给已注册的 proxy master。

### 4.2 proxy 进程模块划分

#### 4.2.1 核心模块

**Proxy master（root netns，`node-ctl-proxy.service`）**

| 模块 | 职责 |
|------|------|
| `RouteSyncClient` | plugin 平面 h2c 客户端（连接 conductor config-socket）；首帧 register；收 hello → 全量 upsert + bookmark + 增量更新；收到 upsert/delete → 写 proxyshm + 广播 notify pipe；指数退避重连 |
| `proxyshm.Table`（写端）| mmap 写入 `/run/sandbox/proxy-routes.shm`（65536 固定槽，seqlock CAS）；master 是唯一写入方 |
| wake pipe reader | 读 workers 写入的 wake pipe（1 字节/sid）；per-sid singleflight 去重 → 向 conductor 发 routesync Wake 帧 |
| notify pipe writer | proxyshm GlobalRev 更新后向所有 worker 写 notify pipe（1 字节广播），触发 worker re-lookup |
| `startCommandInNetNS` | exec worker 进程进入 mgmt netns（`sw0_mgmt`），fd 继承 dataLn / forwardLn / mmdsLn |

**Proxy worker × K（mgmt netns，继承 fd，`startCommandInNetNS` 启动）**

| 模块 | 职责 |
|------|------|
| `TLSListener` | 继承 dataLn fd（`:8443` SO_REUSEPORT），TLS 握手，Accept 循环 |
| `HostRouter` | 解析 HTTP Host header，提取 `(sid, port)`；两种寻址：① `<port>-<sid>.<domain>`；② `E2b-Sandbox-Id` + `E2b-Sandbox-Port` header |
| `proxyshm.WorkerView`（只读）| 只读 mmap 同一 shm 文件；seqlock O(1) Lookup；notify pipe 触发 re-check |
| `envdsign.CheckDataPlaneAuth` | `off/log/enforce` 三档；校验 `X-Access-Token` header 或 HMAC 签名 URL |
| `RouteForTarget` / `Dispatcher` | 按 profile × port 决策：e2b 49983/49999 → KindUDS（dial forwardLn EnvdUDS/CiUDS）；bare 49983/49999 → KindDeny → 501；其他端口 → KindTCP（dial FloatingIP:port via mg0）|
| `ParkQueue` | paused 状态下 park 当前 conn；同时写 wake pipe → master；notify pipe 触发 UnparkAll；park_timeout → Cancel → 503 |
| `MMDSServer` | 继承 mmdsLn fd（`127.0.0.1:19254`，mgmt netns）；`PUT /latest/api/token` → `sid.hex(HMAC-SHA256(mmds_secret, sid))`；`GET /` → `{instanceID, envID, accessTokenHash: hex(sha512(token))}`；token 确定性，每个 worker 独立可验 |
| `FlowTableWriter` | **[计划中]** KindTCP 建连后写 eBPF flowtable（`bpf_map_update_elem`）；连接关闭后清理 |
| `MetricsServer` | Prometheus `/metrics` |

**envd.sock / ci.sock 的本质：vsock 反向通道转发器**

`RouteEntry.EnvdUDS`（`/run/sandbox/<sid>/envd.sock`）和 `CiUDS`（`ci.sock`）并非 envd/code-interpreter 进程本身的 socket，而是 `sandbox-ctl run` 启动时通过 `--connect` 参数在 host 侧创建的 **UDS 监听器**，由 `Forwarder` goroutine 持有。

**端到端数据面完整路径（以 port 49983 envd 为例）：**

```
客户端 SDK  (HTTPS / gRPC-web)
  │  目标: <49983-sid>.domain:443
  ▼
cluster-router  [集群入口，L7 TLS 终止]
  │  转发到节点 proxy worker :8443（HTTP/2 或 HTTP/1.1）
  ▼
proxy worker  [mgmt netns，startCommandInNetNS 启动]
  │  TLSListener.Accept
  │  HostRouter.Parse(Host) → (sid, port=49983)
  │  proxyshm.WorkerView.Lookup(sid) → Route{KindUDS,
  │                                     UDS:"/run/sandbox/<sid>/envd.sock"}
  │  envdsign.CheckDataPlaneAuth(X-Access-Token)   [enforce 模式]
  │  Dispatcher: KindUDS → dial forwardLn(EnvdUDS)
  ▼
sandbox-ctl Forwarder  [host netns，goroutine]
  │  acceptLoop 在 /run/sandbox/<sid>/envd.sock（UDS）上接受连接
  │
  │  ── 层1: 建立 host↔guest vsock 通道 ──
  │  OpenForward(): dial /run/sandbox/<sid>/vsock.sock（UDS，CH 暴露的 vsock 设备入口）
  │  发送 CONNECT <vsock-port>\n
  │  （等待 sandbox-init 完成 AF_VSOCK accept，层1 通道就绪）
  │
  │  ── 层2: 协商 guest 侧目标 ──
  │  发送 TypeConnect{Network:"tcp", Address:"127.0.0.1:49983"}
  │  等待 TypeConnectAck
  │
  │  ── 层3: 数据搬运（fwd frame protocol）──
  │  启动 fwd.Relay.Run()
  │    plain→framed: 用户字节 → FrameData；TCP EOF → FrameEOF；读错误 → FrameRST
  │    framed→plain: FrameData → write；FrameEOF → CloseWrite()；FrameRST → abort
  ▼
cloud-hypervisor  [virtio-vsock 设备后端，host 用户态]
  │  收到 CONNECT 请求 → 写入 virtio-vsock virtqueue（共享内存）
  │  通知 guest kernel（virtio kick）
  ▼
guest kernel virtio_vsock 驱动
  │  从 virtqueue 读取连接请求 → 递交给 AF_VSOCK 监听方
  ▼
sandbox-init  [guest PID 1，serveReverseChannel goroutine]
  │  AF_VSOCK listener 接受连接 → 层1 通道就绪（Forwarder ↔ sandbox-init 双向字节流）
  │
  │  ── 层2: 在已建立的 vsock 通道内协商 guest 侧目标 ──
  │  Forwarder 发 TypeConnect{Network:"tcp", Address:"127.0.0.1:49983"}
  │  sandbox-init 读 TypeConnect → net.Dial("tcp", "127.0.0.1:49983") → 发 TypeConnectAck
  │  ── 层3: 数据搬运（fwd frame protocol）──
  │  双方启动 fwd.Relay.Run():
  │    Forwarder:    plain=UDS conn，framed=vsock conn
  │    sandbox-init: plain=TCP conn，framed=vsock conn
  │    帧格式: [type u8][len u16BE][payload]；FrameData/FrameEOF/FrameRST
  │    FrameEOF 在带内传递 TCP half-close（CH vsock proxy 不透传 shutdown(SHUT_WR)）
  ▼
envd 进程  [guest 内，监听 127.0.0.1:49983]

← 字节流完整路径:
   客户端 ↔ proxy worker
           ↕ forwardLn UDS (EnvdUDS)
         Forwarder [层1 vsock 通道 + 层2 TypeConnect 协商 + 层3 fwd frame relay]
           ↕ vsock.sock → CH virtio-vsock virtqueue（层3 fwd frames）
         sandbox-init [层2 TypeConnect 处理 + 层3 fwd frame relay]
           ↕ TCP loopback (127.0.0.1:49983)（层3 plain 侧裸字节）
         envd
```

**vsock 通道的三层协议**

Forwarder 从 vsock.sock 到 guest envd 的过程涉及三个独立的协议层：

*第一层：vsock 连接建立（Forwarder ↔ CH，通过 virtio-vsock virtqueue）*

vsock.sock 是 CH 实现的 vsock 设备 host 侧 Unix socket 入口。Forwarder dial vsock.sock 后，先发送连接请求（`CONNECT <port>\n`），指定 guest 侧 vsock 监听端口。CH 收到后通过 **virtio-vsock virtqueue**（共享内存，类似 virtio-net 但完全绕过网络栈）将连接请求推给 guest 内核的 `virtio_vsock` 驱动，驱动交给 guest 内监听该端口的 sandbox-init（serveReverseChannel goroutine）。至此 Forwarder 与 sandbox-init 之间建立了一条双向字节流通道。

*第二层：TypeConnect 帧（Forwarder → sandbox-init，运行在已建立的 vsock channel 上）*

vsock 通道建立后，Forwarder 发送应用层帧 `TypeConnect{Network:"tcp", Address:"127.0.0.1:49983"}`，这是 sandbox-ctl 自定义的 IPC 协议，告诉 sandbox-init 在 guest 内部 dial 哪个本地服务。sandbox-init 收到后 dial `127.0.0.1:49983`（envd），回 `TypeConnectAck`，双方进入第三层。

*第三层：fwd frame protocol（双方对称，跑在已建立的 vsock channel 上）*

`TypeConnectAck` 之后，双方均启动 `fwd.Relay.Run()`，对 vsock channel 进行带帧的双向搬运。帧格式为 `[type u8][len u16BE][payload]`，三种帧类型：`FrameData` 携带数据字节；`FrameEOF` 带内传递 TCP half-close（触发对端 `CloseWrite()`）；`FrameRST` 中止双向连接。引入帧的原因是 CH 的 vsock proxy 不会把 `shutdown(SHUT_WR)` 跨 UDS↔vsock 边界透传，裸字节 relay 无法保留 TCP half-close 语义。

| 层次 | 参与方 | 协议 | 作用 |
|------|--------|------|------|
| 第一层 | Forwarder ↔ CH vsock.sock ↔ guest kernel virtio_vsock | `CONNECT <port>\n` + virtio-vsock virtqueue | 建立 host→guest 虚拟字节流通道 |
| 第二层 | Forwarder → sandbox-init（跑在第一层通道内）| `TypeConnect{Network, Address}` / `TypeConnectAck` | 告知 sandbox-init 在 guest 内 dial 哪个本地服务 |
| 第三层 | Forwarder ↔ sandbox-init（跑在第一层通道内，第二层握手完成后）| fwd frame（FrameData / FrameEOF / FrameRST）| 数据搬运 + TCP half-close 保留 |

> virtio-vsock 与 virtio-net 的区别：virtio-net 传输 Ethernet 帧，经 tap → TC eBPF 进入网络栈；virtio-vsock 传输纯字节流，彻底绕过 guest 和 host 的网络栈，是 sandbox-ctl ↔ guest 之间的专用 IPC 通道。

关键含义：
- **sandbox-ctl 是数据面上的字节 relay**，不只是控制面组件；每个数据字节都经过 sandbox-ctl 进程的 `io.Copy` goroutine 搬运
- `envd.sock` 是 sandbox-ctl 在 host 侧创建的 UDS 监听器，并非 envd 自身的 socket
- `vsock.sock` 是 cloud-hypervisor 暴露的 vsock 设备入口，sandbox-ctl Forwarder 通过它向 guest 内发起反向连接
- **sandbox-ctl cgroup 隔离**：当前（`--cgroup-adopt`）sandbox-ctl 与 CH 同处 unit cgroup，`memory.high` 节流时存在 relay 链路死锁风险，当前接受该代价；计划（`--cgroup-isolated`，未实现）sandbox-ctl 留在 unit cgroup，CH 移入子 cgroup，彻底消除节流路径

`--connect` 指令由 orchestrator 在 `LaunchSpec` 中注入（`sandboxcfg.ConnectSpecs()`），仅对 e2b profile 生效；bare profile 不暴露控制端口。快照（pause）期间 Forwarder 会先 `Pause()` 阻止新连接，再 `CloseActive()` 折叠活跃 relay，保证快照窗口内无半开连接。

**用户端口（port ≠ 49983/49999）端到端数据面完整路径：**

用户端口走 TCP splice。当前实现中 proxy worker 全程承运（KindTCP → `io.Copy` 双向 splice）。eBPF flowtable 加速尚未实现，以下阶段二为设计规格。

阶段一：经过 proxy worker 用户态（当前实现）

```
客户端 SDK  (HTTPS / TCP)
  │  目标: <PORT>-<sid>.domain:443
  ▼
cluster-router  [集群入口，L7 TLS 终止]
  │  转发到节点 proxy worker :8443（SO_REUSEPORT）
  ▼
proxy worker  [mgmt netns，startCommandInNetNS 启动]
  │  TLSListener.Accept
  │  HostRouter.Parse(Host) → (sid, port=PORT)
  │  proxyshm.WorkerView.Lookup(sid) → Route{KindTCP,
  │                                     Addr:"x.x.x.x:PORT"}
  │  envdsign.CheckDataPlaneAuth(X-Access-Token)   [enforce 模式]
  │  Dispatcher: KindTCP → dial TCP(FloatingIP:PORT) via mg0
  │  io.Copy 双向 splice 启动（用户态字节搬运，全程经过 proxy worker）
  ▼
mg0  [mgmt veth peer，mgmt netns (sw0_mgmt)]
  │  内核路由：floatingIP/20 → mg0（vswitch 启动时安装）
  │  veth pair
  ▼
sw-mX  [mgmt veth switch 侧，switch netns（sw0_vswitch）]
  │  TC hook: tc/ingress_mx
  │  DNAT: dst floatingIP → dst inner_ip
  │  bpf_redirect(slot->ifindex)  → sw0-tN
  ▼
sw0-tN  [tap device，switch netns]
  │  CH 通过 SCM_RIGHTS 持有此 tap fd（connector-ctl vswitch open-port 交接）
  ▼
cloud-hypervisor  [virtio-net backend，host 用户态]
  │  从 tap fd 读取以太帧 → 经 virtio-net virtqueue 送入 guest
  ▼
guest 网卡 (virtio-net)  [guest netns]
  ▼
用户 app  [guest 内，监听 :PORT]

← 当前入向：客户端 → proxy worker（io.Copy splice）→ mg0 → veth → sw-mX DNAT → tap → CH virtio-net → guest
← 当前回程：guest → CH virtio-net → tap → sw0-tN TC SNAT(inner_ip→floatingIP) → sw-mX → mg0 → proxy worker TCP socket
← 规划入向：transit NIC TC hook flowtable 命中，改写 dst→floatingIP → mg0 → sw-mX DNAT → tap → guest（绕过 proxy worker）
← 规划回程：guest → tap → sw0-tN SNAT → mg0 → transit NIC TC hook flowtable 反向命中，改写 src→nodeIP → client
```

阶段二：flowtable 写入后，TC hook 内核直通（绕过 proxy worker）【计划中】

```
客户端后续数据包  (已建连接)
  │
  ▼
transit NIC (eth1)  [TC hook: prog_tc_ingress]
  │  bpf_map_lookup_elem(map_flowtable, {src,sport,dst,dport}) → 命中
  │  内核改写 dst → FloatingIP:PORT；proxy worker 用户态 splice 循环退出
  ▼
mg0  [mgmt veth peer，mgmt netns (sw0_mgmt)]
  │  内核路由：floatingIP/20 → mg0（flowtable 改写后命中此路由）
  │  veth pair
  ▼
sw-mX  [switch netns]
  │  TC DNAT(floatingIP→inner_ip) → bpf_redirect → sw0-tN tap
  ▼
CH virtio-net backend → virtio-net virtqueue → guest 用户 app

回程（guest → client，内核直通）:
guest 用户 app → CH virtio-net → tap → sw0-tN
  → TC SNAT(inner_ip→floatingIP) → sw-mX → mg0 → host kernel
  ▼
transit NIC (eth1)  [TC hook: prog_tc_ingress，反向]
  │  bpf_map_lookup_elem(map_flowtable, {src=floatingIP,sport,...}) → 命中
  │  改写 src FloatingIP → nodeIP（还原为客户端所见的节点地址）
  ▼
client

← 规划入向：transit NIC TC hook flowtable 命中，改写 dst→floatingIP → mg0 → sw-mX DNAT → tap → guest（绕过 proxy worker）
← 规划回程：guest → tap → sw0-tN SNAT → mg0 → transit NIC TC hook flowtable 反向命中，改写 src→nodeIP → client
← 双向均绕过 proxy worker 用户态，吞吐量接近线速
```

连接关闭时，proxy worker 保留一个轻量 goroutine 监听 FIN/RST，收到后执行 `bpf_map_delete_elem` 清理 flowtable 条目，防止 4 元组复用时命中过期条目。**[flowtable 实现后生效；当前无 flowtable，连接关闭仅回收 io.Copy goroutine]**

与 envd/ci 路径的关键差异：

| 维度 | envd/ci（port 49983/49999） | 用户端口（其他 port） |
|------|----------------------------|----------------------|
| 转发层 | UDS → vsock（始终经过 sandbox-ctl） | TCP → mg0 → sw-mX TC DNAT → tap（当前全程经过 proxy worker；flowtable 实现后首包经 proxy worker，后续 transit NIC flowtable 内核直通）|
| 数据路径中的进程 | proxy worker + sandbox-ctl（全程字节 relay） | 当前：proxy worker 全程 io.Copy；flowtable 实现后：首包经 proxy worker，后续绕过所有用户态 |
| 网络命名空间穿越 | mgmt netns UDS → vsock 虚拟设备 | mgmt netns → vswitch netns → guest netns |
| 快照/暂停影响 | Forwarder.Pause() 阻断新连接 | paused 状态下 ParkQueue.Park()，写 wake pipe → master → conductor |
| 吞吐量上限 | sandbox-ctl io.Copy goroutine 瓶颈 | 当前：proxy worker io.Copy；flowtable 实现后：TC hook 线速转发 |

#### 4.2.2 proxy.mode 三档行为对比

| 模式 | proxy 进程 | MMDS 位置 | 生产适用性 |
|------|-----------|-----------|-----------|
| `external` | `node-ctl-proxy.service` → proxy master（root netns）+ proxy worker × K（mgmt netns）| 每个 worker 均可响应（继承 mmdsLn fd）| ✅ 推荐生产 |
| `internal` | conductor 进程内嵌单实例（无 master/worker 拆分）| conductor 进程内 | ⚠️ 开发/低流量场景；conductor 崩溃连数据面一并挂 |
| `off` | 不启动 | 无 | 🔧 调试 / 纯控制面验证 |

### 4.3 routesync 协议详细设计

#### 4.3.1 帧格式

```
[4B LE uint32 帧长度][JSON 正文（最大 1 MiB）]
```

#### 4.3.2 握手流程

```
proxy master 连接 config-socket
    │
    ├─→ PUT /internal/plugin/{id}/register（SO_PEERCRED 鉴权）
    │     register 帧作为请求 body 随 PUT 立即发出（proxy master 上行，首帧）
    │   {"type":"register","register":{
    │     "subscribe":{"kind":"route_wake"},
    │     "proxy":{"socket":{"path":"/run/sandbox/proxy-1.sock"}},
    │     "mmds":true}}
    │
    │← hello 帧（conductor 读到 register 后下行，含策略）
    │   {"type":"hello","hello":{"version":1,
    │     "policy":{"domain":"...","auth_mode":"off|log|enforce",
    │               "park_timeout_ms":90000}}}
    │
    │← upsert × N（全量 running + paused 路由快照）
    │← bookmark（全量同步完成标记）
    │
    │  [增量阶段]
    │← upsert（单条路由变更）
    │← delete（沙箱销毁）
    │
    ├─→ wake（proxy master 上行，paused 沙箱触发唤醒）
    │   {"type":"wake","sid":"abc123"}
```

proxy master 在收到 `bookmark` 后，清理 proxyshm 中**未在本次全量同步中出现**的旧槽（孤儿清理），并广播 notify pipe，确保 workers 的路由视图与 conductor 严格一致。

#### 4.3.3 RouteEntry 数据结构

`routesync.RouteEntry` 是 conductor 向 proxy master 推送的单条沙箱路由信息，master 将其写入 proxyshm；workers 通过 `proxyshm.mmapRecord`（与 `RouteEntry` 字段一一对应）读取路由。

```go
// orchestrator/internal/routesync/proto.go
type RouteEntry struct {
    SandboxID          string `json:"sid"`
    Profile            string `json:"profile"`                   // "e2b" | "bare"
    TemplateID         string `json:"template_id,omitempty"`     // MMDS envID 字段来源
    State              string `json:"state"`                     // "running" | "paused"
    // dead 沙箱通过 delete 帧推送，proxy 侧不会收到 state=dead 的 upsert
    EnvdUDS            string `json:"envd_uds,omitempty"`        // e2b 49983 UDS 路径
    CiUDS              string `json:"ci_uds,omitempty"`          // e2b 49999 UDS 路径
    FloatingIP         string `json:"floatingip,omitempty"`      // mg0 dial 目标 IP
    AccessToken        string `json:"access_token,omitempty"`    // sb.EnvdAccessToken
    TrafficAccessToken string `json:"traffic_access_token,omitempty"` // SDK create 返回的兼容 token；不用于数据面鉴权
    SnapshotLocation   string `json:"snap_loc,omitempty"`        // "" | "local" | "remote"
    MmdsSecret         string `json:"mmds_secret,omitempty"`     // hex(keys.MmdsSecret(manifestKey, id))；确定性，每个 worker 可独立验 MMDS token
    Group              string `json:"group,omitempty"`           // cluster 路由分片键
    RouteKey           string `json:"route_key,omitempty"`       // cluster session affinity 键
}
```

**与旧架构的关键差异**

| 字段 | 旧（sandbox-orchestrator）| 新（orchestrator）|
|------|--------------------------|------------------|
| `State` 值域 | `running \| paused \| saved` | `running \| paused`（dead → delete 帧）|
| `MigrationToken` | 有（saved 状态跨机恢复凭证）| **不存在** |
| `TrafficAccessToken` | 无 | 有（SDK 兼容，不作数据面鉴权）|
| 存储方式 | 进程内 `sync.Map[sid → RouteEntry]`（每个 worker 独立副本）| proxyshm mmap（master 写，65536 固定槽，seqlock；workers 只读 mmap）|
| 鉴权字段 | `AccessToken` → `ConstantTimeCompare` | `AccessToken` → `envdsign.CheckDataPlaneAuth` |

**proxyshm 物理结构（`proxyshm.mmapRecord`，定长字段，无 JSON）**

```
mmapRecord:
  Seq / Hash / Status / SyncGen / Rev      ← seqlock + 槽状态控制字段
  SandboxID          [128]byte
  Profile            [16]byte
  TemplateID         [128]byte
  State              [16]byte
  EnvdUDS            [256]byte
  CiUDS              [256]byte
  FloatingIP         [64]byte
  AccessToken        [256]byte
  TrafficAccessToken [256]byte
  SnapshotLocation   [32]byte
  MmdsSecret         [128]byte

文件路径: /run/sandbox/proxy-routes.shm（defaultCapacity = 65536 槽）
```

### 4.4 流量路由决策树

```
收到入站连接
    │
    ├─ 解析 Host → (sid, port)
    │   失败 → 400 Bad Request
    │
    ├─ proxyshm.WorkerView.Lookup(sid)
    │   未命中 → 404 Not Found（KindNotFound）
    │
    ├─ envdsign.CheckDataPlaneAuth（auth_mode=enforce）
    │   bare profile + 49983/49999 → KindDeny → 501 Not Implemented
    │   X-Access-Token / 签名 URL 验证失败 → 401 Unauthorized
    │
    ├─ route.State == "running"  → RouteForTarget(profile, port)
    │   ├─ e2b + 49983 → KindUDS: dial forwardLn(EnvdUDS)
    │   │                  └─ sandbox-ctl Forwarder acceptLoop
    │   │                       → vsock 层1 CONNECT + 层2 TypeConnect{127.0.0.1:49983}
    │   │                       → guest envd:49983 → io.Copy 双向 splice
    │   ├─ e2b + 49999 → KindUDS: dial forwardLn(CiUDS)  （同上，目标 127.0.0.1:49999）
    │   └─ any + 其他端口 → KindTCP: dial TCP FloatingIP:port via mg0
    │                        → io.Copy 双向 splice
    │                        → [计划中] 写 eBPF flowtable + TC hook 接管
    │
    ├─ route.State == "paused"
    │   → ParkQueue.Park(sid, conn, deadline=now+park_timeout)
    │   → write wake pipe ──► master 去重 singleflight → routesync Wake 帧 → conductor
    │     conductor 触发 resume → upsert(running) → proxyshm + notify pipe
    │   → notify pipe 触发 re-lookup → running → UnparkAll(sid) → 重走上方
    │   → park_timeout 到期：Cancel(sid) → conn.Close() → 503
    │
    └─ KindNotFound（超时/删除后无路由）
        → 404 Not Found
```

### 4.4.1 透明转发的 envd 接口清单

> proxy 按 `(sid, port)` 路由，**不解析 HTTP 路径**，不区分 gRPC 和 REST。
> 以下接口均由 guest 内 envd 实现，proxy 层只做认证（X-Access-Token）+ 路由 + 透传。
>
> **实际能力由 guest 内 envd daemon 的版本决定**：下表列出的是当前版本 envd 实现的接口集合。node-ctl proxy 对路径无感知，接口的增删、行为变更、版本迭代均由 envd 二进制决定，proxy 无需跟随修改。不同模板镜像内置的 envd 版本不同，同一 node-ctl 节点上运行的不同沙箱可能暴露不同的接口子集，调用方需根据实际 envd 版本判断能力边界。

#### port 49983 — envd（仅 e2b profile）

envd 以 chi.Mux 为底层路由器，同时挂载 Connect-RPC handler 和 oapi-codegen 生成的 REST handler，监听 guest 内 `127.0.0.1:49983`；通过 sandbox-ctl vsock 反向通道穿透到 host 侧 `envd.sock`。

**协议兼容性**

proxy HTTP 链路两端非对称：

| 链路段 | 协议 | 实现依据 |
|--------|------|---------|
| 客户端 → proxy | HTTP/2（无 TLS 时 h2c；有 TLS 时 ALPN 协商）| `netserve.go:36` `h2c.NewHandler` / `TLSConfig.NextProtos: ["h2", "http/1.1"]` |
| proxy → envd | HTTP/1.1 | `proxy.go:109` `ForceAttemptHTTP2: false` |

| 客户端协议 | 可用性 | 原因 |
|-----------|--------|------|
| **Connect 协议**（`application/connect+proto`，e2b SDK 默认）| ✅ 完整可用 | Connect 协议设计即兼容 HTTP/1.1 后端；unary / server-stream / client-stream 均正常 |
| **gRPC-Web 协议**（`application/grpc-web`）| ✅ 可用 | trailer 编码在 response body 内，不依赖 HTTP/2 trailer 帧 |
| **gRPC 协议**（`application/grpc`）| ❌ 不可用 | gRPC 要求端到端 HTTP/2；proxy→envd 降为 HTTP/1.1 后 `grpc-status` trailer 丢失，envd 亦拒绝 HTTP/1.1 上的 `application/grpc` 请求 |

**Connect-RPC 接口**（Connect 协议 / gRPC-Web 协议可用；gRPC 协议不可用）

| 路径 | 流式方向 | 说明 |
|------|---------|------|
| `/process.Process/List` | Unary | 列举 guest 内所有受管进程 |
| `/process.Process/Connect` | Server-stream | attach 到已有进程，服务端推送 stdout/stderr/exit 事件；stdin 输入须通过 `SendInput`/`StreamInput` 独立发送 |
| `/process.Process/Start` | Server-stream | 启动新进程，stdout/stderr/exit 事件以 stream 返回 |
| `/process.Process/SendInput` | Unary | 向进程 stdin 写入数据 |
| `/process.Process/StreamInput` | Client-stream | 客户端流式写入 stdin；经 HTTP/1.1 chunked request body 透传，语义正确 |
| `/process.Process/SendSignal` | Unary | 向进程发送 Unix 信号 |
| `/process.Process/CloseStdin` | Unary | 关闭进程 stdin（发送 EOF）|
| `/process.Process/Update` | Unary | 更新进程元数据（超时、标签等）|
| `/filesystem.Filesystem/Stat` | Unary | 查询路径属性（类型、大小、mtime）|
| `/filesystem.Filesystem/MakeDir` | Unary | 创建目录（含 mkdir -p）|
| `/filesystem.Filesystem/Move` | Unary | 移动/重命名文件或目录 |
| `/filesystem.Filesystem/Remove` | Unary | 删除文件或目录 |
| `/filesystem.Filesystem/ListDir` | Unary | 列举目录内容 |
| `/filesystem.Filesystem/WatchDir` | Server-stream | 监听目录变更事件（inotify 封装），长连接推送 |
| `/filesystem.Filesystem/CreateWatcher` | Unary | 创建目录监听器，返回 watcher handle |
| `/filesystem.Filesystem/GetWatcherEvents` | Unary | 轮询获取指定 watcher 的事件列表 |
| `/filesystem.Filesystem/RemoveWatcher` | Unary | 销毁 watcher，释放 inotify 资源 |

> `Start`、`Connect`、`WatchDir` 等 server-stream RPC：proxy 设置 `FlushInterval: -1`（`proxy.go:128`），每个响应帧立即下发，不在 proxy 层缓冲。

**REST 接口（HTTP/1.1 或 HTTP/2，oapi-codegen 生成）**

| 方法 + 路径 | 说明 |
|------------|------|
| `GET  /files` | 文件下载；`path` query param 指定 guest 内绝对路径 |
| `POST /files` | 文件上传；支持 `multipart/form-data` 多文件或 `application/octet-stream` 单文件（xattr `user.e2b.*` 存储元数据）|
| `POST /files/compose` | 多文件合并写入目标路径，底层使用 `copy_file_range` 内核零拷贝；写临时文件后原子 rename，避免中途失败破坏目标文件 |
| `GET  /envs` | 返回 guest 内当前生效的环境变量键值表 |
| `GET  /health` | 健康探针（`{"status":"ok"}`）；sandbox-ctl 在 VM 启动后直接调此接口确认 envd 就绪，亦可通过 proxy 透传 |
| `POST /init` | envd re-key：接受 MMDS 颁发的 `accessToken`，仅在 `mmds.enabled=true` 时生效 |
| `POST /freeze` | 冻结 guest 内所有受管进程组（pause 前调用）|
| `POST /unfreeze` | 解冻（resume 后调用）|
| `POST /fsfreeze` | 冻结 guest 内文件系统写入（暂停 I/O）|
| `POST /fsthaw` | 解冻文件系统 |
| `POST /collapse` | 压缩 envd 自身堆内存（将匿名页合并为 2 MiB 透明大页），pause 快照前调用以减少脏页数量 |
| `GET  /metrics` | 返回 guest 内主机资源快照（JSON）：CPU 核数/使用率、内存总量/已用/缓存、磁盘总量/已用；数据来自 `/proc/stat` 和 `/proc/meminfo`，是 guest 整机视图，不区分进程（envd 自身占用包含在内，无法单独分辨）|

**认证拆分**：proxy 在 `AuthMiddleware` 校验 `X-Access-Token`，通过后透传；envd `auth.go` 的 bypass 列表仅包含 `GET /files` 和 `POST /files`，其余路径（含 `POST /files/compose`）envd 侧仍会做自身校验。

#### port 49999 — code-interpreter（仅 e2b profile）

code-interpreter 进程监听 `127.0.0.1:49999`，通过 `ci.sock` 穿透到 host。proxy 按 `port == 49999 → KindUDS(ci.sock)` 路由，接口内容由 code-interpreter 自身定义，proxy 不感知。

#### 其他端口 — 用户自定义服务（TCP，floatingIP 路由）

`port ∉ {49983, 49999}` → `KindTCP → floatingIP:port`，由 vswitch DNAT 到 guest 内对应服务，proxy 不做协议解析。bare profile 访问 49983/49999 → `KindDeny(501 Not Implemented)`。

### 4.5 eBPF flowtable 集成（计划中，proxy 当前代码尚未实现）

> **注**：`internal/proxy/` 当前版本中不存在 `FlowTableWriter`、`bpf_map_update_elem` 等实现。以下为设计规格，待实现后生效。现阶段用户端口全程由 proxy worker io.Copy 双向 splice 承运。

已建 TCP 连接在 proxy worker 完成第一个 splice 循环后，写入 vswitch 共享 flowtable（`/sys/fs/bpf/vswitch/map_flowtable`）：

```go
// key: 4 元组（网络字节序）
type FlowKey struct {
    SrcIP   [4]byte
    SrcPort uint16
    DstIP   [4]byte
    DstPort uint16
}

// value: 目标 floatingip + port
type FlowValue struct {
    FloatingIP [4]byte
    Port       uint16
    _          [2]byte // padding
}
```

写入后，vswitch TC hook 在内核直接按 flowtable 转发，proxy worker 退出该连接的 splice 循环（保留 goroutine 监听 FIN/RST 以清理 flowtable 条目）。

**连接关闭时**：proxy worker 监听到 EOF/RST 后，执行 `bpf_map_delete_elem` 清理对应 flowtable 条目，避免旧条目干扰后续重用相同 4 元组的新连接。

### 4.6 park/wake 详细流程

```
proxy worker（state=paused）       proxy master              conductor              sandbox
      │                                 │                        │                    │
      │ ParkQueue.Park(sid, conn)       │                        │                    │
      │ write wake pipe ───────────────►│                        │                    │
      │                                 │ singleflight(sid)      │                    │
      │                                 ├──── Wake 帧 ───────────►│                    │
      │                                 │                        │ OnWake(sid)        │
      │                                 │                        │ resumeIfPaused()   │
      │                                 │                        ├── lc.Start(runner@sid)──►
      │                                 │                        │   （D-Bus StartUnit）│ restore snapshot
      │                                 │                        │                    │ envd ready
      │                                 │                        │◄───────────────────┤
      │                                 │                        │ publishUpsert      │
      │                                 │◄─ upsert(running) ─────┤                    │
      │                                 │ proxyshm write         │                    │
      │                                 │ notify pipe broadcast  │                    │
      │◄─ notify pipe 触发 re-lookup ───┤                        │                    │
      │ ParkQueue.UnparkAll(sid)        │                        │                    │
      ├── splice ────────────────────────────────────────────────────────────────────►│

异常路径（sid unknown/dead）：
      │                                 │                        │
      │                                 │◄─ delete 帧 ───────────┤
      │                                 │ proxyshm clear         │
      │                                 │ notify pipe broadcast  │
      │◄─ notify pipe 触发 re-lookup ───┤                        │
      │ ParkQueue.Cancel(sid) → 404    │                        │
```

**两级去重保证**：多个 proxy worker 同时收到同一 paused 沙箱的请求时，各自写 wake pipe；proxy master 通过 singleflight 合并同 sid 的 wake，只向 conductor 发一次 Wake 帧。conductor 侧 `OnWake` 内部同样有 `o.sf.Do(sid, ...)` 兜底，防止极端竞态下的重复 resume，确保每次 restore 只触发一次 `lc.Start`，避免 cloud-hypervisor 状态混乱。

### 4.7 MMDS v2 设计

proxy worker 在继承自 master 的 mmdsLn fd（`127.0.0.1:19254`，mgmt netns）上运行 HTTP/1.1 服务。vswitch 将 guest 内 `169.254.169.254:80` 的流量 DNAT 至此地址。

**mmds_secret 确定性推导**（conductor 侧计算，经 routesync → proxyshm 下发）：

```
mmds_secret = HMAC-SHA256(manifest_key_bytes, "kuasar-mmds-v1:" + sid)
```

manifest_key 明文不离开 conductor 内存；N 个 proxy worker 通过 proxyshm 读取相同 mmds_secret，无需共享状态即可对等响应。

**两段式 MMDS 握手流程**：

```
envd（guest 内）                         proxy MMDS server
     │ PUT /latest/api/token              │
     ├───────────────────────────────────►│
     │                                    │ ByFloatingIP(src_ip) → sid
     │                                    │ token = sid + "." + hex(HMAC-SHA256(mmds_secret, sid))
     │◄─── token ─────────────────────────┤  (确定性，无存储，任意 worker 均可计算)
     │ GET /latest/meta-data/             │
     │   X-metadata-token: <token>        │
     ├───────────────────────────────────►│
     │                                    │ 验 token → 返回 metadata
     │◄─── {accessTokenHash: hex(SHA512(EnvdAccessToken))} ──┤
     │ 验 SHA512 hash == conductor 经 proxyshm 下发的 AccessToken hash
     │ envd re-key 完成，后续 RPC 验 token
```

### 4.8 API 设计

#### 4.8.1 对外 API 变更（proxy 数据面入口）

proxy（master/worker）本身不暴露控制面 REST API；其对外接口是 **HTTPS 数据面**，行为变更如下：

| 场景 | 请求格式 | 响应变化 |
|------|---------|---------|
| 正常转发 | `Host: <port>-<sid>.<domain>`，`X-Access-Token: <token>` | 透传后端响应，无附加 header |
| 鉴权失败 | 同上，token 不匹配 | **新增** 401 `{"message":"unauthorized"}` |
| Paused 沙箱唤醒中 | 同上，state=paused | **新增** 连接挂起最长 park_timeout，恢复后透传 |
| 沙箱不存在 | 同上 | 404（现有行为，已有） |

**数据面认证**：proxy 对 `X-Access-Token` 支持 `off / log / enforce` 三档（默认 enforce）。bare profile 无 envd，proxyshm 中 `AccessToken == ""`，proxy 无论哪档均跳过 token 校验，数据面请求**无认证保护**；e2b profile 有 access token，enforce 模式下 token 不匹配返回 401。

服务级别：P99 连接建立延迟（running 沙箱）< 50 ms；吞吐量峰值受限于宿主机 NIC 带宽（不受 proxy worker 用户态限制）。

#### 4.8.2 周边 API 调用变更

| 调用方向 | 接口 | 变化 |
|---------|------|------|
| proxy master → conductor | config-socket plugin 平面 `PUT /internal/plugin/{id}/register` | **新增** |
| proxy master → conductor | routesync wake 上行帧 | **新增** |
| conductor → proxy master | routesync upsert/delete/hello/bookmark 下行帧 | **新增** |
| proxy → vswitch bpffs | `bpf_map_update_elem` / `bpf_map_delete_elem` on `map_flowtable` | **新增** |
| guest envd → proxy worker | MMDS v2 HTTP `169.254.169.254:80` → `127.0.0.1:19254`（vswitch DNAT）| 新增 MMDS sidecar，接口协议已有定义 |

**过载风险评估**：routesync 为 push 模型，conductor 主动推送增量变更，proxy worker 本地查 proxyshm O(1)，无额外 RPC 调用在热路径上；flowtable 写入为内核调用，延迟 < 1 µs；整体热路径不引入新的外部依赖过载风险。

#### 4.8.3 上游调用方接入约定

node-ctl proxy 上游通过 `--data-listen`（TCP）接入，比如：集群入口 `cluster-ctl route` 和集群级网关 `agent-gateway`。他们的连接方式相同，区别在于沙箱寻址方式不同。

**上游入口示意**

```
client
  │
  ├─── cluster-ctl route ──────────────────────────→ --data-listen (TCP)  ─┐
  │     集群级入口；解析 sid → DataEndpoint；注入 X-Access-Token             │
  │     Host / port 由 client 指定，route 原样透传                           ├─→ node-ctl proxy → guest
  │                                                                          │
  └─── agent-gateway ──────────────────────────────→ --data-listen (TCP) ─┘
        集群级网关；本地 sid→node map 查表定位目标节点；
        透传 E2b-Sandbox-Id / E2b-Sandbox-Port / X-Access-Token
        port 由调用方通过 E2b-Sandbox-Port 指定（缺省 49983）
```

**设计约定：port 由 client 指定，cluster-ctl route / agent-gateway 负责透传**

`Host: <port>-<sid>.<domain>`、`E2b-Sandbox-Id`、`E2b-Sandbox-Port` 均由**客户端用户**在请求中指定：

- **cluster-ctl route**：Host label 路径直接透传客户端 Host；by-(group, route-key) 路径读取客户端的 `E2b-Sandbox-Port`（缺省 49983）拼入 Host label 后转发
- **agent-gateway**：原样透传 `E2b-Sandbox-Id` / `E2b-Sandbox-Port`
- 若客户端未指定 port，两者均以 **49983**（envd 入口）作为缺省值
- port 合法性由 node-ctl proxy 在路由决策时保证（KindDeny / KindTCP / KindUDS 分类）

**cluster-ctl route 对 proxy 的要求**

| 项 | 说明 |
|----|------|
| 连接端点 | proxy 的 `--data-listen`（TCP，SO_REUSEPORT） |
| 沙箱寻址 | `Host: <port>-<sid>.<domain>`（router 解析 sid 后保持 Host 原样） |
| 认证注入 | `X-Access-Token: <token>`（router 从 registry 取 access_token 后注入） |
| Transport | HTTP/1.1 keepalive（`MaxIdleConns:512, MaxIdleConnsPerHost:64, IdleConnTimeout:90s`）；proxy 须接受复用连接 |
| CONNECT 隧道 | router 发送 `CONNECT <port>-<sid>.<domain> HTTP/1.1` + `X-Access-Token`；proxy 须处理 CONNECT 并双向 splice |

**agent-gateway 对 proxy 的要求**

agent-gateway 持有 `sid → node` 映射，收到请求后直接查表定位目标节点，无需回查 registry，转发到该节点的 `--data-listen`。

| 项 | 说明 |
|----|------|
| 连接端点 | 目标节点 proxy 的 `--data-listen`（TCP，SO_REUSEPORT） |
| 路由依据 | 本地 `sid → node` map 查表，直接命中目标节点 |
| 沙箱寻址 | header 对 `E2b-Sandbox-Id`（sid）+ `E2b-Sandbox-Port`（port，缺省 49983） |
| 认证透传 | `X-Access-Token` 原样透传 |
| CONNECT 隧道 | 发送 `CONNECT <target> HTTP/1.1` + `E2b-Sandbox-Id` + `E2b-Sandbox-Port` + `X-Access-Token`；proxy 须处理链式 CONNECT |

**proxy 须同时满足的能力点**

1. `ParseSandbox()` 优先读 `E2b-Sandbox-Id`，fallback 到 Host label，两类上游均可正确解析 `(sid, port)`。
2. `X-Access-Token` 鉴权对两类上游统一执行。
3. CONNECT 隧道（`serveConnect`）在 HTTP/1.1 和 HTTP/2 两种协议下均能正常 splice。

### 4.9 数据库设计

proxy 进程**无独立数据库**，路由状态完全来自 routesync 内存同步。conductor 侧 SQLite `sandboxes` 表中以下字段为 proxy 路由装配的数据源：

| 字段 | 用途 |
|------|------|
| `floatingip` | proxy 转发目标 IP |
| `envd_uds` | port 49983 UDS 路径 |
| `ci_uds` | port 49999 UDS 路径 |
| `envd_access_token` | proxy 数据面鉴权 token（`envdsign.CheckDataPlaneAuth` 比对值，经 routesync → proxyshm `AccessToken` 字段下发）|
| `traffic_access_token` | SDK create API 返回的兼容 token；经 proxyshm `TrafficAccessToken` 字段下发；**不用于**数据面鉴权 |
| `state` | running/paused 决定 proxy 路由行为 |
| `snapshot_ref` | paused 快照位置（local/remote 决定 RouteEntry.SnapshotLocation）|

**无新增 DB 字段**（本特性不扩展 sandboxes schema）。

### 4.10 可观测性设计

#### 4.10.1 指标设计（Prometheus）

```
# 数据面请求计数（proxy.Counter.Inc，当前已实现）
data_requests_total{result}
# result: ok | badrequest | notfound | denied | unauthorized | route_error | upstream_error

# 数据面连接计数（per-port-type，计划中）
proxy_connections_total{state, port_type}
# state: routed | parked | rejected_auth | rejected_notfound
# port_type: envd | ci | user

# 在途连接数（gauge，计划中）
proxy_connections_active{port_type}

# park/wake 延迟直方图（计划中）
proxy_park_duration_seconds{result}
# result: woken | timeout | cancelled

# routesync 重连次数（计划中）
proxy_routesync_reconnect_total

# proxyshm 已占用槽数（gauge，计划中）
proxy_proxyshm_slots{state}
# state: running | paused

# flowtable 写入延迟（计划中）
proxy_flowtable_write_duration_seconds

# MMDS 请求计数（计划中）
proxy_mmds_requests_total{method, result}
```

#### 4.10.2 日志与事件

**关键日志（结构化 JSON）**：

| 事件 | level | 关键字段 |
|------|-------|---------|
| routesync 连接建立 | info | master_id, conductor_addr |
| routesync 重连 | warn | master_id, attempt, backoff_ms |
| 路由鉴权失败 | warn | sid, remote_addr（不记录 token） |
| park 开始 | debug | sid, park_timeout_ms |
| wake 发送 | info | sid |
| wake 成功（park 解除）| info | sid, park_duration_ms |
| park 超时 | warn | sid, park_timeout_ms |
| flowtable 写入失败 | error | sid, errno（降级为纯用户态转发） |
| MMDS session token 颁发 | debug | floating_ip, sid |

### 4.11 配置项与特性开关

#### node-ctl.yaml proxy 相关配置

```yaml
proxy:
  mode: "external"          # external | internal | off
  auth: "enforce"           # off | log | enforce（off 仅测试用）
  park_timeout: "90s"       # paused 沙箱唤醒等待超时
  mmds:
    enabled: false          # true = 启动 MMDS v2 sidecar
```

#### node-ctl-proxy.service 配置项（proxy.yaml）

proxy master 通过 `node-ctl proxy --config /etc/node-ctl/proxy.yaml` 启动，所有参数来自 YAML 配置文件（`ProxyFileConfig`）；`--worker` flag 为内部标志，由 master 的 `startCommandInNetNS` 传入 worker 进程，不由外部配置。

| 配置键 | 默认值 | 说明 |
|--------|--------|------|
| `config_socket` | `/run/sandbox/node-ctl.socket` | conductor config-socket 路径 |
| `proxy_socket` | `<dir(config_socket)>/proxy.sock` | conductor proxyForwarder UDS |
| `shm_path` | `<dir(config_socket)>/proxy-routes.shm` | proxyshm 文件路径 |
| `data_listen` | `:8443` | SO_REUSEPORT 数据面监听（master 创建，worker 继承）|
| `workers` | `1` | worker 进程数 |
| `auth` | `enforce` | off / log / enforce |
| `park_timeout` | `30s` | bootstrap 初始值；conductor hello 帧下推后覆盖 |
| `mmds_listen` | `""` | MMDS HTTP 监听地址（mgmt netns，空=禁用）|
| `metrics_listen` | `""` | Prometheus metrics 端点（空=禁用）|

### 4.12 sandbox-vswitch 网络层

sandbox-vswitch 是 kuasar-sandbox 的 L2/L3 数据面底座，proxy 的用户端口路由（floatingip:port）和 MMDS 管理流量均建立在它之上。

#### 4.12.1 总体架构

**图一：vswitch 内部设备拓扑**

```
┌──────────────── host root netns ─────────────────────────────────────┐
│  conductor · proxy master · :8443 listener socket（worker 继承 fd）  │
└──────────────────────────────────────────────────────────────────────┘

┌──────────────── mgmt netns (sw0_mgmt) ───────────────────────────────┐
│  mg0  ← --mgmt-extract=sw0_mgmt:mg0:floatingIP_cidr                 │
│  proxy worker（proxy_netns=sw0_mgmt，继承 root netns :8443 fd）      │
│  MMDS v2 listener 127.0.0.1:19254（route_localnet=1）               │
│         │                                                            │
└─────────┼────────────────────────────────────────────────────────────┘
          │ veth pair（mg0 ↔ sw-mX）
┌─────────┼────────────────────────────────── sw0_vswitch netns ───────────────────────────────┐
│         │                    transit NIC (eth1)                                               │
│       sw-mX                  ← vswitch start 从 caller netns 移入此 netns                    │
│    TC ingress_mx              TC ingress_transit                                              │
│         │                         │                                                           │
│         └─────────────────────────┘                                                           │
│                  bpf_redirect(slot->ifindex)                                                  │
│                          │                                                                    │
│                          ▼                                                                    │
│                  sw0-tN  (tap device，TUNSETPERSIST 持久化)                                  │
│                  TC ingress_nx                                                                │
│                          │                                                                    │
│           tap fd（open-port 在此 netns 内打开，SCM_RIGHTS 交接）                             │
│                          │                                                                    │
│                 cloud-hypervisor（virtio-net backend）                                        │
│                 ← tapfd 模式：sandbox-ctl startCH() setns 进入此 netns fork CH               │
│                 ← tap-name 模式：CH 在 host root netns，tap fd 跨 netns 可用                 │
│                          │ virtio-net virtqueue                                               │
│                    ┌─────┴──────┐                                                            │
│                    │ guest netns │  （KVM 硬件隔离，与宿主机任何 netns 均独立）              │
│                    └────────────┘                                                            │
│                                                                                              │
│  每个设备挂载 TC ingress eBPF 程序（shared block）                                            │
│  BPF map pin 在 bpffs → 控制面进程退出后数据面不中断                                         │
└──────────────────────────────────────────────────────────────────────────────────────────────┘
```

**图二：用户端口数据面路径（tapfd 模式，CH 在 switch netns）**

```
                       客户端 SDK
                           │ HTTPS/TCP
                           ▼
                     cluster-router
                           │ :8443
                           ▼
┌──── host root netns ──────────────────────────────────────────────────────┐
│  :8443 listener socket（proxy master 创建；worker fork 后继承 fd accept） │
└───────────────────────────────────────────────────────────────────────────┘
                           │ 连接送入继承 fd
                           ▼
┌──── mgmt netns (sw0_mgmt) ─────────────────────────────────────────────────────────────────┐
│                                                                                            │
│  proxy worker ◄── 本节点回程（sw-mX → mg0，SNAT src inner_ip→floatingIP）                │
│       │                                                                                    │
│       │ dial(floatingIP:PORT)   内核路由: floatingIP/20 → mg0                             │
│       ▼                                                                                    │
│      mg0（mgmt veth peer）                                                                 │
│       │ veth pair                                                                          │
└───────┼────────────────────────────────────────────────────────────────────────────────────┘
        │              sw0_vswitch netns
┌───────┼────────────────────────────────────────────────────────────────────────────────────┐
│       ▼                      bpf_redirect(slot->ifindex)    transit NIC (eth1)              │
│     sw-mX ──────────────────────────────────────────► sw0-tN ◄──── GENEVE ──► 跨节点       │
│  TC ingress_mx                                        TC ingress_nx  TC ingress_transit     │
│  DNAT: floatingIP→inner_ip                            SNAT: inner_ip→floatingIP            │
│                                                            │                                │
│                                               tap fd（在此 netns 内持有）                   │
│                                          cloud-hypervisor ◄┘                                │
│                                          （setns 进入此 netns，tapfd 模式）                 │
│                                                            │ virtio-net virtqueue            │
│                                         ┌──────────────────┴────────────────┐               │
│                                         │          guest netns               │               │
│                                         │      virtio-net NIC  user app:PORT │               │
│                                         └────────────────────────────────────┘               │
└────────────────────────────────────────────────────────────────────────────────────────────┘
```

控制面（`connector-ctl vswitch serve/start`）只负责配置 BPF map、管理 netns、移动设备；一旦 `start` 完成，转发逻辑完全在内核 TC eBPF 程序中执行，控制面进程退出不影响已建立连接。transit NIC 在 `start` 时由 caller netns 移入 switch netns，mgmt veth peer（mg0）直接创建在 `--mgmt-extract` 指定的 mgmt netns。

**CH 所在 netns 取决于网络获取模式**

| 模式 | 配置 | CH netns | 说明 |
|------|------|---------|------|
| tapfd（生产）| `network.tapfd.exec: [connector-ctl, vswitch, open-port, sw0, --port=N]` | **sw0_vswitch netns** | `open-port` 发送 switch netns fd；`startCH()` `setns(CLONE_NEWNET)` 后 fork |
| tap-name（开发/测试）| `network.tap: <tap-name>` | **host root netns** | 无 tapfd 交接，`netnsFile == nil`，CH 直接 `cmd.Start()` |

tap fd 本身跨 netns 可用（内核对象引用），tap-name 模式下 CH 虽在 root netns 仍可驱动 switch netns 内的 tap 设备。

**用户端口完整数据面路径**

| 子路径 | 方向 | 路径 |
|--------|------|------|
| 本节点 proxy | 入向 | proxy worker(mgmt netns) → dial(floatingIP) → mg0 → veth → sw-mX TC DNAT(floatingIP→inner_ip) → bpf_redirect → sw0-tN tap fd → CH(switch netns) virtio-net → guest |
| 本节点 proxy | 回程 | guest → CH virtio-net → tap → sw0-tN TC SNAT(inner_ip→floatingIP) → sw-mX → mg0(mgmt netns) → proxy worker TCP socket |
| 跨节点 GENEVE | 入向 | transit NIC → TC eBPF(GENEVE 解封装) → bpf_redirect → sw0-tN → tap fd → CH virtio-net backend → guest 网卡 → user app |
| 跨节点 GENEVE | 出向 | guest → CH virtio-net backend → tap fd → sw0-tN TC SNAT + GENEVE 封装 → transit NIC → 外部网关 |

#### 4.12.2 端口模式

##### tap 模式（microVM，kuasar 默认）

```
connector-ctl vswitch open-port sw0 --port=N
  → 进入 switch netns
  → open(/dev/net/tun) + TUNSETIFF(IFF_TAP|IFF_NO_PI|IFF_VNET_HDR)
  → TUNSETPERSIST(1)          ← tap 设备持久化，进程退出不消失
  → SCM_RIGHTS 把 fd 发到 TAPFD_SOCKET
  → cloud-hypervisor 接收 fd，作为 virtio-net backend
```

**为什么不需要 veth pair**：TAP 设备天生有两端——网络栈端（sw0-tN，位于 switch netns，TC eBPF 挂载于此）和字符设备端（fd，不绑定 netns）。cloud-hypervisor 持有 fd，直接以用户态读写原始 Ethernet 帧，充当 guest virtio-net ↔ host tap 的桥接器，无需任何 netns 间通道。

**tap 模式下用户端口数据面路径**

```
本节点 proxy worker 入向:
  proxy worker dial(floatingIP:PORT)  [mgmt netns，经 mg0 出]
    → mg0(mgmt netns) → veth → sw-mX(switch netns)
    → TC DNAT(floatingIP→inner_ip) → bpf_redirect → sw0-tN
    → tap fd（CH 持有）→ CH virtio-net backend → guest 网卡 → user app:PORT

本节点 proxy worker 回程:
  user app:PORT → guest 网卡 → CH virtio-net backend → tap fd → sw0-tN
    → TC SNAT(inner_ip→floatingIP) → bpf_redirect → sw-mX → mg0
    → mgmt netns (sw0_mgmt) → proxy worker TCP socket

跨节点 GENEVE 入向:
  transit NIC → TC eBPF(GENEVE 解封装) → sw0-tN（switch netns）
    → tap fd（CH 通过 SCM_RIGHTS 持有）
    → CH virtio-net backend（用户态，读 Ethernet 帧写 virtqueue）
    → guest virtio-net 驱动 → guest 网卡 → user app:PORT

跨节点 GENEVE 出向:
  user app:PORT → guest virtio-net 驱动（写 virtqueue）
    → CH virtio-net backend（读 virtqueue，写 tap fd）
    → sw0-tN（switch netns）→ TC eBPF(GENEVE 封装)
    → transit NIC → 外部网关
```

##### veth 模式（容器类沙箱）

```
switch netns                   sandbox netns
  sw0-pN ───── veth pair ─────── sw0-nN
  (switch side，TC eBPF)          (attach 时移入)
```

#### 4.12.3 slot_id 转发模型

所有转发决策以 **slot_id** 为唯一键，由整数算术推导，不依赖报文中的 src IP/MAC：

| 入向端 | slot_id 推导方式 |
|--------|----------------|
| 沙箱设备（sw0-tN/pN）| `ifindex_to_slot[ifindex]` BPF map 查表 |
| mgmt 回程（sw0-mX）| `floating_ip - base_ip`（浮动 IP 段内偏移）|
| 外部入向（transit NIC）| `UDP_dst - geneve_port_base` |

slot 分配通过 mmap'd BPF array + 原子 CAS 实现（`inner_ip` 字段：`0=Free`, `0xFFFFFFFF=Reserved`, 真实 IP=`Allocated`），无全局锁。

#### 4.12.4 三条数据面通路

**① sandbox → 管理服务（MMDS 等，169.254.169.254）**

```
guest 发出 dst=169.254.169.254:80
  → virtio-net → tap fd → sw0-tN
  → TC ingress eBPF：命中 mgmt_cidrs[]
      SNAT src(inner_ip) → floating_ip
      DNAT dst:80 → 127.0.0.1:19254  （mgmt_svc_fwd BPF map）
  → sw0-mX → mgmt netns
  → proxy worker MMDS server（看到 src = floating_ip）
回程：反向 DNAT，guest 始终看到 src=169.254.169.254
```

**② sandbox ↔ 用户端口（两条子路径：本节点 proxy worker / 跨节点 GENEVE）**

**② a. 本节点 proxy worker → sandbox（mgmt veth 路径）**

入向（proxy worker 发起）：
```
proxy worker  [mgmt netns (sw0_mgmt)]
  │  dial TCP(floatingIP:PORT)
  │  内核路由：floatingIP/20 → mg0（vswitch 启动时安装于 mgmt netns）
  ▼
mg0  [mgmt veth peer，mgmt netns (sw0_mgmt)]
  │  veth pair
  ▼
sw-mX  [mgmt veth switch 侧，switch netns]
  │  TC hook: tc/ingress_mx
  │  DNAT: dst floatingIP → dst inner_ip
  │  bpf_redirect(slot->ifindex) → sw0-tN
  ▼
sw0-tN  [tap device，switch netns]
  → tap fd（CH 持有）→ CH virtio-net backend → virtio-net virtqueue
  → guest 网卡 → user app:PORT
```

回程（sandbox 回复 proxy worker）：
```
guest user app → guest 网卡 (virtio-net)
  → CH virtio-net backend → tap fd → sw0-tN
  │  TC hook: tc/ingress_nx
  │  识别 dst 在 host 范围（非 mgmt_cidrs，非 GENEVE 外部）
  │  SNAT: src inner_ip → src floatingIP
  │  bpf_redirect → sw-mX
  ▼
sw-mX → mg0 → mgmt netns (sw0_mgmt) → proxy worker TCP socket
```

规划入向（transit NIC 内核直通，绕过 proxy worker）：
```
transit NIC (eth1)  [TC hook: prog_tc_ingress]
  │  bpf_map_lookup_elem(map_flowtable, {src,sport,dst,dport}) → 命中
  │  改写 dst → floatingIP:PORT；proxy worker 用户态 splice 循环退出
  ▼
mg0  [mgmt veth peer，mgmt netns (sw0_mgmt)]
  │  内核路由：floatingIP/20 → mg0
  │  veth pair
  ▼
sw-mX  [switch netns]
  │  TC DNAT: dst floatingIP → dst inner_ip
  │  bpf_redirect → sw0-tN
  ▼
sw0-tN → tap fd → CH virtio-net backend → guest user app:PORT
```

规划回程（双向均内核直通）：
```
guest user app → CH virtio-net → tap → sw0-tN
  │  TC SNAT: src inner_ip → src floatingIP
  │  bpf_redirect → sw-mX → mg0 → host kernel
  ▼
transit NIC (eth1)  [TC hook: prog_tc_ingress，反向]
  │  bpf_map_lookup_elem(map_flowtable, {src=floatingIP,sport,...}) → 命中
  │  改写 src floatingIP → nodeIP（还原为节点地址）
  ▼
client
```

**② b. 跨节点外部流量（GENEVE 封装）**

出向：
```
guest user app → guest 网卡 (virtio-net)
  → CH virtio-net backend → tap fd → sw0-tN（tap 设备，switch netns）
  → TC ingress eBPF（sw0-tN）：
      GENEVE 封装：outer src=transit_ip, outer dst=gateway_ip
      UDP dst = geneve_port_base + slot_id   ← 每沙箱独享一个 UDP 端口
  → transit NIC → 外部网关
```

入向：
```
gateway → transit NIC → TC ingress eBPF：
      slot_id = UDP_dst - geneve_port_base    ← O(1) 算术，无 hash
      验证 outer src == transit_gateway_ip + VNI
      GENEVE 解封装；强制改写 h_dest = 派生 port MAC
  → sw0-tN（tap 设备，switch netns）
  → tap fd（cloud-hypervisor 通过 SCM_RIGHTS 持有）
  → CH virtio-net backend → virtio-net virtqueue
  → guest 网卡 → user app:PORT
```

> **与 proxy 的关系**：用户端口流量（非 49983/49999）由 proxy worker（mgmt netns）通过 mgmt veth 路径（本节点）将 TCP 连接送达 sandbox；sandbox-ctl 不参与此路径；proxy worker 完成 L7 鉴权后 io.Copy 全程 splice（flowtable 接管后完全绕过 proxy worker 用户态）。

**③ tap fd 交接（cloud-hypervisor 拿到虚拟网卡）**

`connector-ctl vswitch open-port` 是一次性操作：持久 tap 常驻 switch netns；cloud-hypervisor fd 关闭时 tap 的网络栈端仍存在（TUNSETPERSIST），可供下次快照恢复时重新交接。

#### 4.12.5 隔离机制

| 机制 | 实现方式 |
|------|---------|
| 沙箱间二层隔离 | 无 port→port 转发路径；不同 slot 的帧不可互达 |
| ARP 全代答 | switch netns 内 eBPF 代答所有 ARP，防止沙箱探测邻居 |
| 源 MAC 强制改写 | 出口强制将 h_source 改写为派生 port MAC，防止 MAC 伪造 |
| GENEVE 入向验证 | 检查 outer src IP + VNI，丢弃非法封装报文 |
| slot 分配不信任 sandbox | slot_id 由宿主机算术推导，sandbox 声称的 src IP/MAC 不参与转发决策 |

#### 4.12.6 控制面与数据面解耦

BPF map 以 `BPF_OBJ_PIN` 钉在 bpffs（`/sys/fs/bpf/vswitch/`），TC filter 附着在网络设备上，均与 `vswitch serve` 进程生命周期解耦：

- `vswitch serve` 崩溃 → 已建 TCP 连接继续转发
- `vswitch serve` 重启 → 重新 `open()` 已 pin 的 map，重挂 TC filter（若已存在则幂等）
- `cloud-hypervisor` 崩溃 → tap fd 引用计数归零，tap 设备的字符设备端关闭，但网络栈端（sw0-tN）因 TUNSETPERSIST 仍留在 switch netns，下次重启时重新 open-port 交接新 fd

#### 4.12.7 与 proxy 的依赖边界

| proxy 所需 | vswitch 提供 |
|-----------|-------------|
| `floating_ip`（用户端口路由目标）| `vswitch-ctl attach` 返回 slot_id → node-ctl 计算 floating_ip 后写入 RouteEntry |
| MMDS 请求中的 src IP 识别 | SNAT 将 inner_ip → floating_ip，MMDS server 凭 floating_ip 查路由表得 sid |
| 用户端口数据面通路 | mgmt veth（mg0→sw-mX TC DNAT→sw0-tN tap）将 floating_ip:port 流量送达 guest 内部服务 |

proxy 不直接操作 vswitch；依赖链为：`node-ctl serve (vswitch-ctl attach)` → `floatingip 写入 RouteEntry` → `routesync 下推 proxy`。

**vswitch 与 vsock 的边界**

vswitch 只承载网络数据包（IP 层），vsock 是完全独立的 host↔guest 通信机制。两条路径互不相交：

| 端口类型 | 数据面路径 | sandbox-ctl 参与 | vswitch 参与 | 底层传输 |
|---------|-----------|----------------|------------|---------|
| 用户端口（非 49983/49999）本节点 | node-proxy → mg0 → sw-mX TC DNAT → sw0-tN tap → CH virtio-net → guest；回程 sw0-tN SNAT → sw-mX → mg0 → node-proxy | ❌ 不经手 | ✅ mgmt veth + tap | 网络包，走 TC eBPF（DNAT/SNAT）|
| 用户端口（非 49983/49999）跨节点 | transit NIC GENEVE 解封装 → sw0-tN tap → CH virtio-net → guest；出向 guest → sw0-tN GENEVE 封装 → transit NIC → 外部网关 | ❌ 不经手 | ✅ GENEVE + tap | 网络包，走 TC eBPF（GENEVE 封/解装）|
| envd/ci 端口（49983/49999）| node-proxy → envd.sock → Forwarder → vsock.sock → CH virtio-vsock → sandbox-init → envd | ✅ Forwarder relay | ❌ 不经过 vswitch | virtio-vsock virtqueue（共享内存）；三层协议：层1 `CONNECT` 建立 vsock 通道，层2 `TypeConnect` 协商目标端口，层3 fwd frame（FrameData/FrameEOF/FrameRST）数据搬运 + TCP half-close 保留 |

vsock 由 cloud-hypervisor 实现，走 virtio-vsock virtqueue（共享内存），与 vswitch 的 tap 设备、TC eBPF 程序、GENEVE 隧道完全无关。两者是并行的两套 host↔guest 通道，面向不同用途：vswitch tap 承载对外可路由的网络流量（用户端口，sandbox-ctl/Forwarder 全程不参与），vsock 承载 sandbox-ctl 与 guest 之间的内部 IPC（含 envd/ci 数据面 relay，Forwarder 做三层协议 relay：层1 vsock 通道建立、层2 TypeConnect 服务协商、层3 fwd frame 数据搬运）。

---

## 5. 性能

### 关键路径延迟目标

| 路径 | P50 | P99 | 说明 |
|------|-----|-----|------|
| running 沙箱新连接建立（TLS + 路由查表 + splice 建立）| < 10 ms | < 50 ms | **部分实测**：proxy worker 自身处理（loopback keep-alive）268 µs；loopback 新建 TLS 连接（RSA 2048 自签名）4.5 ms；真实部署另加网络 RTT（数据中心内 ~0.5–2 ms）及 TLS session ticket 复用影响；端到端 P50/P99 需在实际部署中标定 |
| paused 沙箱 wake 延迟（park 开始到 splice 建立）| < 2 s | < 30 s | **估算**：受 resume（快照恢复）P99 主导，实际值取决于快照大小与存储速度 |
| proxyshm.WorkerView.Lookup（seqlock O(1) hash）| ~51 ns | < 500 ns | **参考值**（原 sync.Map benchmark，Xeon E5-2680 v4）：命中 51 ns/miss 19 ns；proxyshm 定长 hash 无 GC 压力，预期相近；待 benchmark 验证 |
| flowtable 写入（bpf_map_update_elem）| < 1 µs | < 5 µs | **估算**：syscall 开销约 100–300 ns + BPF LRU hash 更新，待 benchmark |
| 已建连接 TC hook 转发（字节级吞吐）| 接近 NIC 线速 | — | eBPF 内核旁路，绕过 proxy worker 用户态 |
| routesync 增量 upsert 应用延迟（纯 CPU）| ~12 µs | — | **实测参考**：JSON decode 11.5 µs；master 写 proxyshm（seqlock CAS）+ notify pipe 广播，替代原 sync.Map Store 51 ns；端到端含 UDS 传输，待 benchmark |
| MMDS session token 颁发（HMAC 计算）| ~2.5 µs | — | **实测**：HMAC-SHA256 benchmark；含 HTTP handler 开销端到端待实测 |

### 容量分析

| 维度 | 单节点目标 | 说明 |
|------|-----------|------|
| 并发沙箱数 | 4096 | vswitch BPF 编译常量 `MAX_PORTS=4096`（`bpf/types.go`）硬上限；proxyshm 默认 65536 槽（`defaultCapacity`），内存约 `65536 × sizeof(mmapRecord)` 固定预分配 |
| 并发 TCP 连接数 | ~1 万 | per-conn 用户态：2 goroutine × 初始栈 2 KiB（`runtime/stack.go stackMin=2048`，Linux）+ 2 × io.Copy 堆缓冲 32 KiB（`io/io.go:418`）= 68 KiB；内核态：用户端口（TCP→TCP）两侧各 tcp_rmem default 128 KiB + tcp_wmem default 16 KiB = 288 KiB，envd/ci（TCP→UDS）TCP 侧 144 KiB + UDS 侧 core/[rw]mem_default 208 KiB = 352 KiB；用户端口合计约 356 KiB/conn，envd/ci 约 420 KiB/conn；按最贵路径 420 KiB × 1万 ≈ 4 GiB proxy 内存预算（节点 16 GiB）推算 |
| park 队列（全部 paused 同时唤醒）| per-sid 无硬上限 | `ParkQueue` 每个 parked conn：1 goroutine（初始栈 2 KiB）+ entry header ≈ 2 KiB；waiter 数 = 流量速率 × park_timeout，park_timeout 到期后自动回收；最多 4096 个 sid 同时唤醒 |
| routesync 全量同步时间（4096 条）| < 500 ms（估算，待 benchmark 验证）| 单条 upsert Msg JSON 实测 461 B，4096 条合计 ~1.80 MiB；Range 遍历期间逐条 WriteMsg 不 flush，Bookmark 后一次性 flush；传输层为 UDS h2c（非 loopback TCP），内核 copy 无网络延迟 |

---

## 6. 可靠性

### 6.1 故障管理

| 失效场景 | 数据面影响 | 自愈路径 |
|---------|-----------|---------|
| `node-ctl conductor` 崩溃 | 控制面中断；running 沙箱 proxy worker 凭 proxyshm 缓存继续转发；paused 沙箱 wake 挂起至 park_timeout | systemd Restart=on-failure → Reconcile 收养 → proxy master 自动重连重同步 |
| proxy worker × 1 崩溃 | 该 worker 上连接 RST；新连接由其余 K-1 worker 承接 | master `superviseProxyWorker` goroutine 自动重启 worker（无需全量 routesync 重同步，proxyshm 已有最新路由）|
| proxy worker × K（全部）崩溃 | 数据面中断 | master 依次重启所有 worker；running 沙箱的 microVM 不受影响 |
| bpffs `map_flowtable` 不可访问 | flowtable 集成降级；proxy 继续纯用户态 splice 转发（功能不损失，吞吐下降）| 日志告警 `FlowtableUnavailable`；运维检查 vswitch 状态 |
| routesync 通道积压（> 1024 帧）| proxy subscriber 被丢弃；指数退避重连后全量重同步 | 重连间隔：100ms → 200ms → 400ms … 最大 30s |
| conductor 与 proxy 版本不匹配 | hello 帧 version 字段检测到不兼容时 proxy master 主动关闭连接并告警 | 滚动升级时先升 conductor，再重启 proxy master（master 重启时 worker 随之重启）|

### 6.2 系统防呆

- **park 超时强制释放**：`ParkQueue` 在 `park_timeout` 到期后无论 conductor 是否响应都执行 Cancel，返回 404；防止请求永久挂起。
- **delete 帧解除挂起**：conductor 发 `delete` 帧 → master 清除 proxyshm 槽 + notify pipe 广播 → worker re-lookup KindNotFound → `ParkQueue.Cancel(sid)` 立即解除该 sid 所有 parked 连接，防止孤儿 goroutine。
- **全量同步孤儿清理**：`bookmark` 收到后，master 清除本次同步未出现的 proxyshm 槽并广播 notify pipe，防止路由视图与 conductor 状态永久分叉。
- **MMDS token 无状态**：token = `sid + "." + hex(HMAC-SHA256(mmds_secret, sid))`，确定性计算，无存储、无 TTL；token 泄露风险由 mmds_secret 轮换（sandbox 销毁）覆盖。
- **Forwarder 快照静默期保护**：快照（pause）开始前 `Forwarder.Pause()` 阻止新连接进入 envd.sock/ci.sock，`CloseActive()` 折叠所有活跃 sandbox-init relay 连接（层1通道 + 层2字节流一并关闭），保证快照窗口内不存在半开的 vsock 连接；快照完成后 `Resume()` 重新开放监听。防止快照期间 relay goroutine 持有 vsock channel 导致 VM 状态不一致。
- **sandbox-ctl cgroup 隔离防死锁**：当前（`--cgroup-adopt`）sandbox-ctl 与 CH 同处 systemd unit cgroup；`memory.high` 节流时 Forwarder relay goroutine（层1 virtqueue 写入 + 层2 io.Copy）可能被一并卡住，造成整条 `proxy worker ↔ Forwarder ↔ CH virtio-vsock ↔ sandbox-init ↔ envd` 链路死锁；该风险当前已知并接受。计划（`--cgroup-isolated`，未实现）：sandbox-ctl 留在 unit cgroup，CH 移入子 cgroup `<unit-cgroup>/sandbox-<sid>/`；`memory.high` 仅节流 CH，sandbox-ctl 可随时发出 balloon inflate 命令解压；`KillMode=control-group` 覆盖整个子树，CH 不会因 sandbox-ctl 退出而成为孤儿。
- **tap fd TUNSETPERSIST 防用户端口中断**：用户端口数据面路径（`vswitch tap → CH virtio-net → guest`）与 sandbox-ctl 完全解耦。cloud-hypervisor 崩溃时 tap fd 引用计数归零，字符设备端关闭；但 sw0-tN 因 `TUNSETPERSIST(1)` 仍留在 switch netns，下次 CH 重启时 `connector-ctl vswitch open-port` 重新交接新 fd，用户端口数据面可自愈，无需重建 vswitch 端口或重写 eBPF flowtable 条目。

### 6.3 过载控制

- **routesync 背压**：conductor 侧订阅通道缓冲 1024 帧，积压即丢弃 subscriber（非 block），防止 slow proxy master 拖慢 conductor 的状态广播。
- **park 队列无硬上限**：per-sid 的 park 连接数由 park_timeout 自然淘汰，不设硬上限（防止误杀合法慢 resume）；极端场景（大量同时唤醒）下 goroutine 数量受 OS 内存限制，建议配合 node-level max_connections 使用。
- **连接速率限制**：未内置（依赖上层 cluster-router 或负载均衡器做连接速率限制）。

### 6.4 冗余设计

- **多 worker 冗余（SO_REUSEPORT）**：生产建议 2 个 worker；任一 worker 崩溃时 OS 将新连接分发至存活 worker，存量连接 RST 后 SDK 重连至存活 worker。
- **eBPF flowtable bpffs pin**：flowtable BPF map pin 至 `/sys/fs/bpf/vswitch/`，proxy 进程崩溃后 map 持久存在，TC hook 继续按已记录条目转发已建 TCP 连接。
- **conductor 与 proxy 进程解耦**：运维可独立重启 conductor 或 proxy master（master 重启时 worker 随之退出重启），互不依赖进程生命周期。

### 6.5 资源残留

- **TCP 连接**：proxy 进程退出时 OS 自动关闭所有 socket，客户端收到 RST；无残留。
- **eBPF flowtable 条目**：proxy 监听连接 EOF/RST 后清理对应条目；proxy 异常崩溃时，对应连接的 flowtable 条目将残留至 LRU 淘汰（`BPF_MAP_TYPE_LRU_HASH`，由 vswitch 控制 max_entries，不影响新连接）。
- **park goroutine**：park_timeout 超时后自动回收；正常关闭时 `ParkQueue.Close()` 取消所有 context。

### 6.6 健康检查

proxy worker 不暴露独立健康探针端点；`node-ctl-proxy.service`（proxy master）的存活状态由 systemd 监控（`Type=exec`，进程退出即为不健康，自动重启）；worker 由 master `superviseProxyWorker` goroutine 监管，master 崩溃时 worker 随之退出。

`/metrics` 端点上的 `proxy_routesync_reconnect_total` 连续增长可作为 proxy master 健康告警依据。

### 6.7 SLI/SLO 治理

**CUJ（Critical User Journey）**：用户通过 SDK 访问 running 沙箱的 envd 端口

| SLI | 目标 SLO | 告警阈值 |
|-----|---------|---------|
| 连接建立成功率（非 4xx/5xx）| ≥ 99.9%（30 天滚动）| < 99.5% 触发告警 |
| 连接建立 P99 延迟（running 沙箱）| < 50 ms | > 200 ms 触发告警 |
| park/wake 成功率（paused 沙箱）| ≥ 99%（park_timeout 内成功）| < 95% 触发告警 |

**告警预埋**：

| 告警名称 | 触发条件 | 级别 |
|---------|---------|------|
| `ProxyAuthRejectionSpike` | `rate(proxy_connections_total{state="rejected_auth"}[5m]) > 10` | warning |
| `ProxyRoutesynReconnecting` | `rate(proxy_routesync_reconnect_total[5m]) > 0` | warning |
| `ProxyParkTimeoutHigh` | `rate(proxy_park_duration_seconds_count{result="timeout"}[5m]) > 1` | critical |
| `ProxyMasterDown` | `up{job="node-ctl-proxy"} == 0` | critical |
| `ProxyFlowtableWriteError` | `rate(proxy_flowtable_write_errors_total[5m]) > 0` | warning |

### 6.8 数据可靠性设计

routesync 路由表为**纯内存状态**（master 维护 proxyshm，workers 通过 mmap 读取），proxy master 重启后通过全量重同步从 conductor 恢复，proxyshm 随之重建；路由真相源为 conductor 的 SQLite `sandboxes` 表，proxy 不维护独立状态，无数据一致性风险。

---

## 7. 安全设计

### 7.1 数据面认证模型

proxy 对每条入站连接执行 `authorized()`（`auth.go`），调用 `envdsign.CheckDataPlaneAuth`，令牌来源于 `Route.AccessToken`（由 proxyshm 读取，conductor 经 routesync 推送；sandbox 创建时生成，`connect/create` 响应体携带）。

| 模式 | 行为 |
|------|------|
| `AuthOff` | 跳过所有 token 校验（开发/测试用） |
| `AuthLog` | 校验失败记录日志，不拒绝请求 |
| `AuthEnforce` | `envdsign.CheckDataPlaneAuth`（X-Access-Token header 或 envd 签名文件 URL）；不匹配返回 401（**生产必须使用**）|

`authorized()` 有两个短路豁免（`auth.go:22`），任一成立直接放行（不调用 `CheckDataPlaneAuth`）：

| 豁免条件 | 代码位置 | 说明 |
|---------|---------|------|
| `authMode == AuthOff` | `auth.go:22` | 全局关闭校验 |
| `route.AccessToken == ""` | `auth.go:22` | 路由无 token（当前 bare profile 实际不走此路径，见风险 2）|

其余情况均进入 `envdsign.CheckDataPlaneAuth`，该函数统一处理 `X-Access-Token` header 和 envd pre-signed 文件 URL（`/files?signature=...`）两种鉴权方式；proxy 验签后 envd 在 guest 内再次校验（纵深防御）。

### 7.2 e2b profile 与 bare profile 安全层次对比

`RouteForTarget`（`proxy.go`）对两种 profile 生成不同的路由类型，导致安全纵深不同：

| 维度 | e2b profile | bare profile |
|------|-------------|--------------|
| 控制端口（49983/49999）| KindUDS，proxy 校验 `X-Access-Token` | KindDeny，直接拒绝（501）|
| 用户端口（其他）| KindTCP，proxy 校验 `X-Access-Token` | KindTCP，proxy 校验 `X-Access-Token` |
| Guest 内鉴权 | envd 额外校验 `X-Access-Token`（纵深防御）| 无（无 envd，in-guest 无认证层）|
| 浮动 IP 直连暴露面 | envd 仍会拒绝未授权请求 | VM 监听端口直接裸露，无 fallback |

**vswitch 已提供的结构性隔离**：vswitch eBPF 程序不存在 port→port 转发路径，ARP 全代答，sandbox 间通信在数据面上被结构性阻断（`vswitch.md §1.2`）。因此攻击面不在 sandbox 互访，而在**宿主机进程绕过 proxy 直连浮动 IP**。

### 7.3 已识别风险

#### 风险 1：宿主机进程直连浮动 IP（bare profile 单点防御）

**描述**：proxy 的 `X-Access-Token` 校验是 bare profile 的唯一认证层。宿主机上能访问浮动 IP 段的任意进程（含受攻击的运维工具、异常 sandbox-ctl 等）可绕过 proxy 直接建立 TCP 连接到 bare sandbox 的监听端口，此时无任何认证保障。

**代码依据**：`proxy.go RouteForTarget`（bare 非控制端口返回 `KindTCP`，无 in-guest 兜底）；`auth.go:20` 豁免条件（`route.AccessToken == ""` 直接放行）。

**影响**：若浮动 IP 网络隔离失效，攻击者可无鉴权访问 bare sandbox 内任意监听端口。e2b profile 不受此影响（envd 仍校验 token）。

#### 风险 2：`EnvdAccessToken` 语义歧义引发的潜在认证降级

**描述**：`Route.AccessToken` 的 struct 注释为 `"" = no data-plane auth, e.g. bare"`，配套测试也传入空 token 给 bare profile（`proxy_test.go`），与当前实现（`orch.go:452` 传入 `sb.EnvdAccessToken`）存在分歧。若未来开发者依注释"修正"实现，会使 bare profile TCP 路由的 `AccessToken` 变为空，触发 `auth.go:20` 豁免，关闭 proxy 层认证。

**代码依据**：`RouteForTarget` 注释 `"" = no data-plane auth, e.g. bare"`；`auth.go:20` 当 `route.AccessToken == ""` 时直接返回 `true`。

**影响**：一次"修复注释不一致"的 PR 即可误关闭 bare profile 的唯一认证层。

#### 风险 3：`AuthOff` 全局豁免与 bare profile 共存

**描述**：`auth.mode=off` 关闭所有 token 校验，无论 profile 类型。当节点同时承载 bare profile sandbox 时，`AuthOff` 会使 bare sandbox 的所有 TCP 端口完全无保护。当前无配置校验阻止两者共存。

**代码依据**：`auth.go:20` `mode == config.AuthOff → return true`；§4.0.6 启动校验注释仅覆盖 MMDS + off 场景，未覆盖 bare + AuthOff 场景。

**影响**：运维失误（忘记切换回 AuthEnforce）导致生产 bare sandbox 完全无认证暴露。

### 7.4 消减方案

#### P0：消除代码层歧义（极低成本，无行为变化）

**措施 A：token 语义分离**

将 bare profile 的代理认证 token 从 `EnvdAccessToken` 切换为 `TrafficAccessToken`，两者均在 sandbox 创建时生成、经 routesync 下推，当前仅 `EnvdAccessToken` 被路由使用，`TrafficAccessToken` 语义上即为数据面流量 token。

此修改涉及两条代码路径，均须同步修改：

**路径 1（internal mode）：`orch.go:452`**

```go
// orch.go — RouteForTarget 调用处（当前 ~line 452）
token := sb.EnvdAccessToken
if sb.Profile() == types.ProfileBare {
    token = sb.TrafficAccessToken  // bare: 数据面 token，与 envd token 解耦
}
return proxy.RouteForTarget(string(sb.Profile()), sb.EnvdUDS, sb.CiUDS, sb.FloatingIP, token, port)
```

**路径 2（external mode）：conductor routesync 推送**

external mode 的 proxy master 通过 routesync 从 conductor 拉取 `RouteEntry.AccessToken`，该值由 conductor 在 `routeEntry()` 中填充（`orch/routes.go`）。当前 conductor 推送的是 `sb.EnvdAccessToken`；需同步修改推送侧，对 bare profile 改填 `sb.TrafficAccessToken`：

```go
// conductor — routesync RouteEntry 构造处（bare profile 分支）
entry.AccessToken = sb.EnvdAccessToken  // 当前
// 改为：
if sb.Profile() == types.ProfileBare {
    entry.AccessToken = sb.TrafficAccessToken
} else {
    entry.AccessToken = sb.EnvdAccessToken
}
```

`RouteEntry.AccessToken` 字段无需改名（已是通用字段名），只有填值逻辑需要按 profile 区分。

同步修正 `proxy.go Route.AccessToken` 注释，删除误导性的 `e.g. bare` 说明：

```go
// AccessToken is the expected data-plane bearer token checked by AuthMiddleware.
// Empty string disables per-route auth (only acceptable when auth mode is off or log).
AccessToken string
```

**措施 B：禁止 bare profile + AuthOff 共存**

在 conductor 配置加载或启动校验阶段增加约束：

```go
if cfg.HasBareProfile() && cfg.Proxy.Auth == config.AuthOff {
    return errors.New("proxy.auth=off is not permitted when bare-profile sandboxes are enabled; use log or enforce")
}
```

或等效地：routesync hello 帧下推 auth 模式后，proxy master 侧检查；若 `hello.policy.auth == off` 且存在 bare profile 路由则记 ERROR 告警并强制使用 log 模式，拒绝静默降级。

#### P1：网络层缩小浮动 IP 可达面（中等成本，运维变更）

在宿主机 default netns 增加 iptables `owner` 规则，将浮动 IP 段的出向连接权限锁定到 proxy master/worker 和 conductor 进程运行用户（如 `uid=node-ctl`）：

```bash
# 只允许 node-ctl 进程用户（uid=1001）建立到浮动 IP 段的连接
iptables -I OUTPUT -d <floating-ip-cidr> \
  -m owner ! --uid-owner node-ctl \
  -j REJECT --reject-with icmp-admin-prohibited
```

此规则与 vswitch eBPF 的 sandbox 间隔离形成双重网络层防护：sandbox 间流量被 eBPF 结构性阻断，宿主机其他进程的直连流量被 iptables 阻断，proxy 成为唯一合法的浮动 IP 访问路径。

实施注意：需在 vswitch attach/detach 流程中同步维护浮动 IP 集合（建议使用 ipset 动态管理）。

#### P2：bare profile 端口 allowlist（中等成本，代码变更）

在 sandbox 配置（`X-Kuasar-Sandbox-Launch`）中增加 `bare_allowed_ports` 字段，`Dispatcher` 对 bare profile 增加端口白名单过滤：

```go
if entry.Profile == types.ProfileBare && !entry.AllowedPorts.Contains(port) {
    return ErrPortDenied  // 返回 403
}
```

未在白名单声明的端口即使持有合法 token 也无法访问，token 泄露后的爆炸半径收缩到显式声明的端口集合。

#### P3：轻量 auth sidecar（较高成本，新增组件）

bare profile 缺少 in-guest 认证层是与 e2b profile 的结构性差距。可为 bare profile 提供可选的极简 init 进程（非完整 envd），职责单一：

- 从 MMDS 或启动参数获取 `TrafficAccessToken`
- 监听代理端口，校验 `X-Access-Token` header
- 校验通过后 TCP 转发到 guest 内应用实际端口

实现后，bare profile 获得与 e2b 对等的纵深防御：proxy 层 token 校验 + in-guest sidecar 校验，浮动 IP 直连不再裸露应用端口。

### 7.5 风险与消减措施汇总

| 风险 | 严重度 | 成立条件 | 消减措施 | 优先级 |
|------|--------|---------|---------|--------|
| 注释/实现分歧导致 bare token 被误清空 | 高（一次 PR 即可触发）| 开发者依注释"修复"代码 | P0-A：token 语义分离 + 注释修正 | **P0** |
| `AuthOff` + bare profile 运维失误共存 | 高（认证完全关闭）| 配置错误 | P0-B：启动校验拒绝共存 | **P0** |
| 宿主机进程绕过 proxy 直连浮动 IP | 高（bare 无 in-guest 兜底）| 宿主机网络未隔离 | P1 iptables owner 规则 | P1 |
| token 泄露后多端口暴露 | 中（需先获取 token）| token 从 API 响应泄露 | P2 端口 allowlist | P2 |
| bare profile 无纵深防御（结构性差距）| 中（需突破 proxy + 网络隔离）| 攻击者能直连浮动 IP | P3 轻量 auth sidecar | P3 |

**vswitch 已覆盖（无需额外消减）**：sandbox 间 eBPF 结构性隔离（无 port→port 转发路径），跨 sandbox 直接攻击在数据面被静态阻断。

---

## 4. User Story 分工

| US 编号 | 描述 | 负责团队 | 预估工时 |
|--------|------|---------|---------|
| US-2.1-1 | `node-ctl proxy` 子命令框架：TLSListener、HostRouter、SO_REUSEPORT 绑定 | orchestrator | M（1w）|
| US-2.1-2 | RouteSyncClient：config-socket plugin 平面 h2c 客户端、全量 + 增量同步、指数退避重连 | orchestrator | M（1w）|
| US-2.1-3 | RouteTable + Dispatcher：运行态路由查表、三档端口分发（UDS/floatingip/park）| orchestrator | S（3d）|
| US-2.1-4 | ParkQueue + WakeSender：park/wake 机制、per-sid singleflight、超时清理 | orchestrator | M（1w）|
| US-2.1-5 | FlowTableWriter：新建连接写 eBPF flowtable，关闭时清理；bpffs 不可达时降级 | orchestrator | S（3d）|
| US-2.1-6 | AuthMiddleware：off/log/enforce 三档，ConstantTimeCompare | orchestrator | S（2d）|
| US-2.1-7 | MMDSServer：内嵌 MMDS v2 HTTP，ByFloatingIP 查路由表，session token TTL | orchestrator | M（1w）|
| US-2.1-8 | serve 侧 routesync 广播：StreamAuthority、upsert/delete/bookmark/hello 下行帧实现 | orchestrator | M（1w）|
| US-2.1-9 | proxy.mode=internal：serve 内嵌单实例 proxy（复用同模块代码）| orchestrator | S（3d）|
| US-2.1-10 | node-proxy@.service systemd 单元模板 + 运维文档（启动标志、TLS 配置、worker 数建议）| infra | S（2d）|
| US-2.1-11 | 可观测性：Prometheus metrics、结构化日志、告警规则配置 | orchestrator | S（3d）|
| US-2.1-12 | 集成测试：running/paused/saved 场景 e2e 验证，auth 失败验证，worker 故障恢复验证 | QA | M（1w）|

**工时说明**：S=3d，M=1w；US-2.1-1 ～ 2.1-8 为 P0 核心路径，预计 6 周交付；US-2.1-9（internal 模式）和 US-2.1-10/11/12 可并行或紧随推进。
