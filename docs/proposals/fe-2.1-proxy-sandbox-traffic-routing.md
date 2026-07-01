# 特性设计文档

**US/FE 编号**：FE-2.1  
**标题**：node-ctl proxy 支持 Sandbox 流量路由  
**版本**：v1.0  
**日期**：2026-06-29  
**状态**：草稿

---

## 1. 需求概述

Kuasar Sandbox 平台的沙箱（microVM）需要通过 HTTPS 向外暴露 envd Connect RPC、code-interpreter 等服务端口，供 SDK 和用户应用访问。当前平台已具备沙箱生命周期管理能力，但**数据面路由转发**是沙箱可用性的关键前提：没有正确的路由装配和流量转发，沙箱即使启动成功也无法对外提供服务。

本特性（FE-2.1）聚焦于 node-ctl proxy 子命令及其承载的数据面能力：在节点上以独立进程运行一组 node-proxy worker，接收来自 cluster-router 的 HTTPS 入站流量，按 `<port>-<sid>.<domain>` Host 解析目标沙箱，将流量精确路由到对应 microVM 的 floatingip 或 UDS 端点；同时实现 paused 沙箱的透明 park/wake 唤醒机制，以及与 serve 进程解耦的高可用保障。

**当前版本策略**：覆盖 proxy.mode=external 的完整设计（生产首选），兼顾 internal/off 两种降级模式；MMDS v2 sidecar 在 proxy 进程内内嵌，与路由表共享生命周期。

---

## 2. 需求分析

### 2.1 Goals & Non-Goals

#### Goals

1. **数据面路由正确性**：按 `<port>-<sid>.<domain>` 精确路由，端口 49983 走 envd UDS，端口 49999 走 code-interpreter UDS，其他端口走 floatingip:port；验收：SDK 发起的 envd RPC / 用户端口请求均能正常响应。
2. **控制面故障零数据面中断**：serve 进程崩溃或重启期间，已建立连接和 running 沙箱的新连接不中断；验收：serve kill 后 running 沙箱访问无感知。
3. **Paused 沙箱透明唤醒（park/wake）**：流量到达 paused 沙箱时自动触发 resume，并在 park_timeout 内完成透传；验收：连接在 wake 完成前挂起，完成后正常建立，客户端无需重试。
4. **Token 鉴权**：enforce 模式下 `X-Access-Token` 不匹配时返回 401；每次 connect/create 重新 mint token，旧 token 即时失效；验收：伪造 token 请求返回 401。
5. **MMDS v2 集成**：proxy 内嵌 MMDS HTTP 服务，按 floatingip 查路由表返回 `accessTokenHash`；验收：guest envd 可成功完成 MMDS 握手完成 re-key。
6. **多 worker 水平扩展**：SO_REUSEPORT 下多个 worker 共享端口，单 worker 崩溃不影响整体；验收：kill 一个 worker 后，存量连接由 OS 层 RST 通知客户端重连，新连接由其余 worker 处理。

#### Non-Goals

- 出站流量策略（CIDR allow/deny，属于 FE-NET-1.1）
- 跨节点路由调度（cluster-router 职责）
- 沙箱生命周期管理（FE-1 范畴）
- L2 ARP 代答、GENEVE 跨宿主隧道（sandbox-vswitch 职责）
- DNS 出站访问控制（FE-NET-1.2）

### 2.2 交付范围

| 编号 | 交付项 | 说明 |
|------|--------|------|
| D-1 | proxy worker 进程框架 | `node-ctl proxy` 子命令，SO_REUSEPORT TLS 监听，多实例 |
| D-2 | routesync 客户端 | config-socket plugin 平面订阅者，全量 + 增量路由同步 |
| D-3 | 路由表 + 流量分发 | 本地 sync.Map O(1) 查表，按端口分发至 UDS/floatingip |
| D-4 | eBPF flowtable 集成 | 新建连接注册 flowtable，已建连接内核 TC hook 直通 |
| D-5 | park/wake 机制 | paused 沙箱 park 队列 + wake 上行帧 + 超时 404 |
| D-6 | MMDS v2 sidecar | 内嵌 MMDS HTTP，deterministic mmds_secret 验证 |
| D-7 | Token 鉴权 | off/log/enforce 三档，ConstantTimeCompare |
| D-8 | proxy.mode 三档 | external / internal / off，serve 侧按 mode 启动内嵌或外置 proxy |
| D-9 | 可观测性 | Prometheus metrics + 结构化日志 |

### 2.3 友商分析

| 维度 | E2B（参照系）| agent-substrate（Google）| kuasar-sandbox（本设计）|
|------|------------|------------------------|----------------------|
| 数据面实现 | Linux bridge + iptables L4 | Envoy sidecar L7 用户态代理 | eBPF/TC 内核态转发 + node-proxy L7 终止 |
| 控制面故障影响数据面 | 有（iptables 规则随进程维护）| 有（Envoy 依赖控制面下发配置）| **无**（bpffs pin，flowtable 独立于控制面）|
| Paused 沙箱透明恢复 | ❌ 客户端需感知并重试 | ❌ 无 pause 概念 | ✅ server-side park/wake |
| 路由查表复杂度 | O(n) iptables 规则匹配 | DNS pull + Envoy xDS | **O(1)** sync.Map in-process |
| 多进程水平扩展 | 单进程 | 单 sidecar per pod | **SO_REUSEPORT 多 worker，独立故障域** |
| MMDS/元数据安全下发 | ❌ 无 | △ 依赖 K8s ConfigMap | ✅ HMAC 确定性密钥，不落明文 |

**核心竞争优势**：eBPF flowtable 接管已建连接后，宿主机上的转发路径完全绕过用户态 node-proxy，吞吐量接近线速；同时 serve 崩溃不中断数据面，这是 E2B 和 agent-substrate 均未解决的问题。

### 2.4 需求约束

1. **依赖 sandbox-vswitch 网络槽**：proxy 的 floatingip 路由依赖 vswitch assign 的 floatingip 地址，node-ctl serve 在沙箱创建时调 `vswitch-ctl attach` 完成分配后，floatingip 才写入 routesync RouteEntry；proxy 无法独立运行于 vswitch 之前。
2. **config-socket 必须先于 proxy worker 就绪**：proxy worker 启动时连接 config-socket plugin 平面，若 serve 未就绪则退避重连，数据面处于降级（无路由）状态，直至同步完成。
3. **TLS 证书共享**：proxy `--data-listen` 与 serve `api.listen` 使用同一套通配证书（`*.<domain>` + `api.<domain>`），运维需确保两个进程挂载相同证书路径。
4. **SO_REUSEPORT 限制**：同一节点上所有 proxy worker 必须 bind 相同 `--data-listen` 地址；混合节点场景下需配置为 agent-vip 而非 0.0.0.0（防与 k8s NodePort 端口空间冲突）。
5. **park_timeout 对用户可见**：park 期间 SDK 连接处于等待状态，park_timeout（默认 90 s）超时后返回 404，用户需处理此错误；`park_timeout` 策略值由 serve 在 routesync hello 帧下推，proxy 启动标志仅作初始值。

---

## 3. 解决方案

### 3.1 User Stories

#### 3.1.1 Story A：SDK 访问 Running 沙箱的 envd 端口

作为一个 **SDK 用户**，我想要通过 `<49983-sid.domain>` 发起 Connect RPC 调用，以便于在运行中的沙箱内执行代码和文件操作。

**场景**：沙箱创建成功（state=running），SDK 持有 `EnvdAccessToken`，发起 HTTPS 请求至 cluster-router；cluster-router 按 Host 将请求转发至 node-proxy :8443；node-proxy 解析 Host 得 `(sid, port=49983)`，查路由表命中，校验 `X-Access-Token`，将连接路由至 envd UDS，后续流量经 eBPF flowtable 内核直通。用户感知到 100 ms 内的连接建立延迟。

#### 3.1.2 Story B：流量触发 Paused 沙箱自动恢复

作为一个 **SDK 用户**，我想要向已暂停的沙箱发起请求后自动等待其恢复，以便于无需在客户端手动调用 connect API 实现透明的 cold-start 体验。

**场景**：沙箱处于 paused 状态（已拍快照），SDK 发起请求；node-proxy 查路由表 state=paused，将连接入 park 队列，同时向 serve 发 `wake` 上行帧；serve 执行 resume（恢复快照）；恢复完成后 routesync 广播 `upsert(state=running)`；node-proxy park 队列解除，连接建立；park_timeout 内整个过程对 SDK 透明。

#### 3.1.3 Story C：Serve 进程重启期间已建连接不中断

作为一个 **平台运维人员**，我想要重启 node-ctl serve 进程（升级/配置变更），以便于不影响用户已建立的沙箱连接。

**场景**：serve 重启；node-proxy worker 维持已建连接（eBPF flowtable 已接管内核转发）；running 沙箱的新连接依赖 proxy 本地路由缓存继续服务；paused 沙箱的 Wake 请求暂时无人应答，等 serve 重启完成后 proxy 自动重连并重新同步路由表，Wake 随之恢复。运维人员可无感升级 serve。

#### 3.1.4 Story D：单 Worker 崩溃后快速恢复

作为一个 **平台运维人员**，我想要 node-proxy 在单个 worker 崩溃时由其余 worker 继续承接新连接，以便于节点数据面具备高可用能力。

**场景**：node-proxy@1.service 崩溃；OS 内核 SO_REUSEPORT 将新连接分发至其余 worker；systemd 在 RestartSec=1 内重启该 worker；重启后 worker 走全量路由同步，恢复完整路由表；宕机期间该 worker 上的存量 TCP 连接断开，新连接已由其余 worker 处理。

### 3.2 架构影响分析

本特性在现有 node-ctl 二进制中新增 `proxy` 子命令，不引入新的外部依赖组件，架构元素影响如下：

- **新增进程**：`node-proxy worker`（external 模式）；internal 模式由 serve 进程内嵌，off 模式不启动。
- **新增接口**：config-socket plugin 平面的 routesync 帧协议（h2c + 4B LE + JSON），serve 侧同步新增实现。
- **技术选型**：Go 标准库 `net/http` + `crypto/tls`；eBPF flowtable 操作复用 sandbox-vswitch 已 pin 的 BPF map（`/sys/fs/bpf/vswitch/map_flowtable`），不新增 BPF 程序。
- **特性树影响**：FE-2.1 是 FE-1（沙箱生命周期）的必要前提——无 proxy 则沙箱创建成功但不可访问；两者并行开发，FE-2.1 可以 `proxy.mode=off` 模式先行合入。

### 3.3 功能规格

#### 静态规格

| 规格项 | 值 |
|--------|-----|
| 节点 worker 数量（生产建议）| 2（最大 4） |
| 数据面监听端口 | :8443（SO_REUSEPORT） |
| 路由查表复杂度 | O(1)（sync.Map） |
| park 队列上限（per-sid）| 无硬上限，受 park_timeout 时间窗约束 |
| 路由表事件通道缓冲 | 1024 帧/subscriber |
| eBPF flowtable 条目上限 | 与 sandbox-vswitch 共享，参见 vswitch 设计 |
| OS 版本要求 | Linux kernel ≥ 5.15（BPF_MAP_TYPE_LRU_HASH） |
| TLS 最低版本 | TLS 1.2（建议 1.3） |

#### 与存量特性的配套关系

| 存量特性 | 关系 | 约束 |
|---------|------|------|
| FE-1.1 沙箱 CRUD | 强依赖：create 写路由、delete 广播删除 | proxy 必须与 serve 在同一节点 |
| FE-1.2 Auto-suspend | 协作：suspend 触发 routesync upsert(paused)；流量触发 wake | park_timeout 须 > resume P99 |
| FE-1.3 Secure 模式 | 协作：mmds.enabled=true 时 proxy 内嵌 MMDS；serve 生成 mmds_secret 经 routesync 下发 | MMDS sidecar 须与 proxy worker 同进程 |
| FE-7.1 网络基础设施 | 依赖：floatingip 由 vswitch-ctl attach 分配后才写入 RouteEntry | vswitch 须先于 proxy 就绪 |

| 规划特性（未实现）| 关系 | 约束 |
|----------------|------|------|
| FE-7.2 高性能数据面 | 协作：proxy 新建连接后注册 flowtable，后续由 TC hook 内核转发（当前 FE-2.1 阶段全程用户态 splice）| bpffs 须挂载于 /sys/fs/bpf |

### 3.4 风险及设计约束

| 风险 | 影响 | 缓解措施 |
|------|------|---------|
| serve 重启期间 paused 沙箱 wake 无人应答 | park 超时后用户请求 404 | park_timeout 默认 90 s，serve 重启 < 5 s；告警 serve 重启时间异常 |
| proxy worker 订阅通道积压（> 1024 帧）| subscriber 被丢弃，需全量重同步 | 指数退避重连；监控 `proxy_routesync_reconnect_total` |
| bpffs 未挂载或 flowtable map 不存在 **[FE-7.2 实现后生效]** | proxy 降级为纯用户态转发（仍可用，但内核旁路不生效）| 启动时检查 `/sys/fs/bpf/vswitch/map_flowtable`，缺失则 warn 并降级 |
| TLS 证书更新期间短暂服务中断 | 新建 TLS 握手失败 | 双 worker 滚动重启：先重启 worker-1，再重启 worker-2 |
| park 队列 goroutine 泄漏（sid 永不 resume）| 内存缓慢增长 | park_timeout 超时后强制释放所有 goroutine；Dead 路由删除时同步清队列 |

**升级兼容性**：routesync 协议以 `{"type":"hello","hello":{"version":1,...}}` 携带版本号；proxy 拒绝 version > 自身能力的 hello 帧并回退重连，保证滚动升级期间 serve/proxy 版本不同时不崩溃。

### 3.5 可选替代方案

| 方案 | 描述 | 放弃原因 |
|------|------|---------|
| 纯 iptables/nftables DNAT | 不需要 proxy 进程，直接在宿主机规则链转发 | 规则随沙箱密度线性膨胀；控制面重启时规则管理复杂；无 park/wake 能力 |
| Envoy xDS proxy | 复用成熟代理组件 | 引入 C++ 依赖；xDS 协议替换 routesync 增加集成成本；MMDS 无法内嵌 |
| 用户态纯 splice（不集成 eBPF）| 实现简单 | FE-2.1 当前即为此方案；高并发下每个报文均需用户态往返，吞吐受限；FE-7.2 叠加 eBPF flowtable 后此方案退化为首包处理 + 回退模式 |

---

## 4. 详细设计

### 4.0 完整进程拓扑

#### 4.0.1 节点进程全景图

下图展示 node-proxy worker 与节点上所有上下文进程、内核组件、集群层之间的完整拓扑关系。

```
┌─ Cluster 层 ───────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                                                                                                │
│  ┌───────────────────────────────────────────┐   ┌────────────────────────────────────────────────────────┐   │
│  │  cluster-ctl registry / scaler            │   │  cluster-ctl router                                    │   │
│  │  ▲ heartbeat / route_upsert / build_event │   │  Host: api.<domain>        → node-ctl serve  :443      │   │
│  │  ▼ command(create/connect/delete/key_put) │   │  Host: <port>-<sid>.<dom>  → node-proxy wkr  :8443     │   │
│  └───────────────────────┬───────────────────┘   └──────────────────────────────────────┬─────────────────┘   │
└──────────────────────────┼──────────────────────────────────────────────────────────────┼─────────────────────┘
                           │ node-link (mTLS / h2c)                                        │ HTTPS
                           │ 上行: register · heartbeat · route_upsert · build_event        │ :443 控制面
                           │ 下行: command(create / connect / delete / key_put / build)     │ :8443 数据面
                           ▼                                                               ▼
┌─ Node 宿主机 (default netns) ───────────────────────────────────────────────────────────────────────────────────┐
│                                                                                                                 │
│  ┌─ node-ctl serve ──────────────────────────────┐      ┌─ node-proxy worker × K ────────────────────────────┐ │
│  │ [node-ctl.service  Restart=on-failure]         │      │ [node-proxy@{1..K}.service  Restart=always  1s]    │ │
│  │ api.listen :443  (TLS / h2c dev)              │      │ --data-listen :8443 (SO_REUSEPORT)                 │ │
│  │                                               │      │ --tls-cert / --tls-key  (与 serve 同一证书)         │ │
│  │  ┌─ HTTPServer ──────────────────────────┐    │      │ --auth enforce                                      │ │
│  │  │ /sandboxes  /templates  /health       │    │      │ --park-timeout 90s  (hello 帧下推后覆盖)            │ │
│  │  │ X-API-KEY / Bearer 鉴权               │    │      │                                                     │ │
│  │  └───────────────────────────────────────┘    │      │  ┌─ TLSListener :8443 ──────────────────────────┐  │ │
│  │                                               │      │  │  SO_REUSEPORT; Accept 循环; TLS 握手          │  │ │
│  │  ┌─ orchestrator core ───────────────────┐    │      │  └──────────────────────────────────────────────┘  │ │
│  │  │  沙箱状态机 running / paused / dead    │    │      │  ┌─ HostRouter ──────────────────────────────────┐  │ │
│  │  │  Reconcile(对账收养) / Reaper(5s)     │    │      │  │  Host → (sid, port)                            │  │ │
│  │  │  BuildPool(2s) / StreamAuthority      │    │      │  │  支持 <port>-<sid>.<domain>                    │  │ │
│  │  └───────────────────────────────────────┘    │      │  │  支持 E2b-Sandbox-Id header                    │  │ │
│  │                                               │      │  └──────────────────────────────────────────────┘  │ │
│  │  ┌─ resource controller ─────────────────┐    │      │  ┌─ RouteTable (sync.Map, O(1)) ─────────────────┐  │ │
│  │  │ [resource_listen.socket UDS]          │    │      │  │ sid → RouteEntry {                             │  │ │
│  │  │  AdmissionCtrl / 令牌桶 / 四水位      │    │      │  │   state: running|paused|saved                  │  │ │
│  │  │  Active Reclaimer                     │    │      │  │   floatingip  envd_uds  ci_uds                 │  │ │
│  │  │  ◄── sandbox-ctl Admit/Settled/       │    │      │  │   access_token  mmds_secret                    │  │ │
│  │  │       Heartbeat/OOMReport/Release      │    │      │  │   snap_loc  migration_token }                  │  │ │
│  │  └───────────────────────────────────────┘    │      │  └──────────────────────────────────────────────┘  │ │
│  │                                               │      │  ┌─ Dispatcher ──────────────────────────────────┐  │ │
│  │  ┌─ config-socket (UDS, h2c, 4 planes) ──┐   │      │  │ running + 49983 → envd.sock  (UDS splice)      │  │ │
│  │  │ /run/sandbox/node-ctl.socket           │   │      │  │ running + 49999 → ci.sock    (UDS splice)      │  │ │
│  │  │                                        │   │      │  │ running + other → floatingip:port (TCP splice) │  │ │
│  │  │ task  plane ◄── sandbox-ctl           │   │      │  │           + write bpffs map_flowtable [FE-7.2]  │  │ │
│  │  │  /internal/task/launchspec            │   │      │  │ paused → ParkQueue (挂起) + WakeSender          │  │ │
│  │  │  SO_PEERCRED pid == sandbox-ctl pid   │   │      │  │ saved  → 409 Conflict + migration_token         │  │ │
│  │  │                                        │   │      │  │ 无路由 → 404 / 410                              │  │ │
│  │  │ admin plane ◄── node-ctl CLI          │   │      │  └──────────────────────────────────────────────┘  │ │
│  │  │  /internal/admin/*  (manifest-key等)  │   │      │  ┌─ ParkQueue + WakeSender ──────────────────────┐  │ │
│  │  │                                        │   │      │  │ Park(sid, conn, deadline=now+park_timeout)     │  │ │
│  │  │ plugin plane ◄─────────────────────── │───┼──────┤  │ WakeSender.Wake(sid)  per-sid singleflight     │  │ │
│  │  │  routesync 下行帧:                    │◄──┼──────┤  │ → wake 上行帧 → serve OnWake()                 │  │ │
│  │  │  hello / upsert / delete / bookmark   │───┼──────►  │ UnparkAll(sid) 解除 / Cancel(sid) 超时 → 404   │  │ │
│  │  │  SO_PEERCRED pid in plugin_pidfile    │   │      │  └──────────────────────────────────────────────┘  │ │
│  │  │                                        │   │      │  ┌─ RouteSyncClient ──────────────────────────────┐  │ │
│  │  │ api   plane ◄── external REST clients │   │      │  │ PUT /internal/plugin/{id}/register              │  │ │
│  │  │  X-API-KEY / Bearer token              │   │      │  │ SO_PEERCRED 鉴权 (plugin_pidfile)               │  │ │
│  │  └────────────────────────────────────────┘   │      │  │ 上行: register 帧 (首帧) / wake 帧 (按需)       │  │ │
│  │                                               │      │  │ 下行: hello→upsert×N→bookmark→增量 upsert/del  │  │ │
│  │  ┌─ node-link ───────────────────────────┐    │      │  │ bookmark 后清孤儿路由; 指数退避重连 100ms→30s   │  │ │
│  │  │  → cluster-ctl registry (mTLS h2c)    │    │      │  └──────────────────────────────────────────────┘  │ │
│  │  │  上行: register/heartbeat/route_upsert │    │      │  ┌─ AuthMiddleware ───────────────────────────────┐  │ │
│  │  │  下行: handleCommand                  │    │      │  │ off | log | enforce (--auth 标志 / hello 下推)  │  │ │
│  │  └───────────────────────────────────────┘    │      │  │ enforce: ConstantTimeCompare(X-Access-Token,   │  │ │
│  │                                               │      │  │          route.AccessToken)  → 401 on mismatch │  │ │
│  │  ┌─ SQLite (WAL) ────────────────────────┐    │      │  └──────────────────────────────────────────────┘  │ │
│  │  │ /var/lib/sandbox/node-ctl.db          │    │      │  ┌─ MMDS v2 server ───────────────────────────────┐  │ │
│  │  │ sandboxes / builds / manifest_keys    │    │      │  │ --mmds-listen 127.0.0.1:19254 (HTTP/1.1)        │  │ │
│  │  └───────────────────────────────────────┘    │      │  │ ByFloatingIP(src_ip) → sid → mmds_secret        │  │ │
│  └────────────────────────┬──────────────────────┘      │  │ PUT /latest/api/token → HMAC token (确定性)     │  │ │
│                           │ D-Bus / sdbus                │  │ GET /latest/meta-data → accessTokenHash         │  │ │
│                           │ StartUnit / StopUnit /       │  └──────────────────────────────────────────────┘  │ │
│                           │ ListUnitsByPatterns           │  ┌─ FlowTableWriter [计划中，FE-7.2，未实现] ───────┐  │ │
│                           │                              │  │ 新建 TCP 连接后:                                 │  │ │
│                           │ + vswitch-ctl attach/detach  │  │ bpf_map_update_elem(map_flowtable,              │  │ │
│                           │   (subprocess, node-ctl 调)  │  │   {src4,sport,dst4,dport}, {floatingip,port})   │  │ │
│                           │                              │  │ 连接关闭后: bpf_map_delete_elem 清条目           │  │ │
│                           │                              │  └──────────────────────────────────────────────┘  │ │
│                           │                              │  ┌─ MetricsServer ─────────────────────────────────┐  │ │
│                           │                              │  │ --metrics-listen 127.0.0.1:9090                  │  │ │
│                           │                              │  │ Prometheus /metrics                              │  │ │
│                           │                              │  └──────────────────────────────────────────────┘  │ │
│                           │                              └────────────────────────────────────────────────────┘ │
│                           ▼                                                                                     │
│  ┌─ systemd [sandbox 单元，由 serve 通过 D-Bus 动态生成与管理] ───────────────────────────────────────────┐   │
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
│  │  ── config-socket task plane →  node-ctl serve                                                          │   │
│  │     /internal/task/launchspec    SO_PEERCRED 鉴权    FetchLaunchSpec / FetchBuildSpec                    │   │
│  │                                                                                                          │   │
│  │  ── resource_listen.socket    → resource controller (node-ctl serve 内嵌)                                │   │
│  │     Admit / Settled / Heartbeat(~30s) / OOMReport / Release / Reattach                                   │   │
│  │     帧格式: [4B LE 长度][JSON]   (同 routesync 帧协议)                                                   │   │
│  │                                                                                                          │   │
│  │  ┌─ Forwarder  (e2b profile 专有，pkg/sandbox/forward.go) ──────────────────────────────────────────┐   │   │
│  │  │  host 侧创建 UDS 监听器；每条新连接经两层协议向 guest 建立 vsock 反向通道:                       │   │   │
│  │  │  层1(连接建立): dial vsock.sock → CONNECT <port>\n                                               │   │   │
│  │  │                → CH virtio-vsock virtqueue → guest kernel virtio_vsock 驱动 → relay 进程         │   │   │
│  │  │  层2(应用协议): TypeConnect{Network,Address} 帧 → guest relay dial 目标本地端口                  │   │   │
│  │  │                                                                                                   │   │   │
│  │  │  /run/sandbox/<sid>/envd.sock  ←── node-proxy dial (port 49983)                                  │   │   │
│  │  │    层2 帧: TypeConnect{tcp, 127.0.0.1:49983} → guest relay → envd:49983                         │   │   │
│  │  │                                                                                                   │   │   │
│  │  │  /run/sandbox/<sid>/ci.sock    ←── node-proxy dial (port 49999)                                  │   │   │
│  │  │    层2 帧: TypeConnect{tcp, 127.0.0.1:49999} → guest relay → code-interp:49999                  │   │   │
│  │  │                                                                                                   │   │   │
│  │  │  Pause()/CloseActive()/Resume(): 快照静止期冻结新连接，折叠活跃 relay                              │   │   │
│  │  └───────────────────────────────────────────────────────────────────────────────────────────────── ┘   │   │
│  │                                                                                                          │   │
│  │  ctl.sock  /run/sandbox/<sid>/ctl.sock              ← snapshot/exec CLI 子进程通过 --run-root 定位      │   │
│  │    snapshot_request: 暂停 VM → CH API → 创建快照 → 返回路径 (一次性, 关闭连接)                          │   │
│  │    exec_request:     握手 → stdio MUX relay (ExecHandler 接管连接生命周期)                              │   │
│  │                                                                                                          │   │
│  │  注(cgroup): 当前 --cgroup-adopt：sandbox-ctl 与 CH 同处 unit cgroup；memory.high 节流存在死锁风险      │   │
│  │              计划 --cgroup-isolated：sandbox-ctl 留 unit cgroup，CH 移入子 cgroup <unit>/<sid>/          │   │
│  │                                                                                                          │   │
│  │  ┌─ cloud-hypervisor [child process，可选进入 tap netns] ────────────────────────────────────────────┐  │   │
│  │  │  TAP fd ←── vswitch-ctl open-port (SCM_RIGHTS via TAPFD_SOCKET UDS)                               │  │   │
│  │  │    virtio-net backend: 用户端口数据包经 tap ↔ virtio-net ↔ guest（sandbox-ctl 不经手）             │  │   │
│  │  │  vsock.sock  /run/sandbox/<sid>/vsock.sock  ← 层1入口: Forwarder/exec_relay dial 此 socket          │  │   │
│  │  │              CONNECT <port>\n → CH virtio-vsock virtqueue → guest kernel virtio_vsock 驱动        │  │   │
│  │  │                                                                                                    │  │   │
│  │  │  ┌─ Guest VM [KVM 硬件隔离, guest netns] ────────────────────────────────────────────────────  ┐  │  │   │
│  │  │  │                                                                                              │  │  │   │
│  │  │  │  sandbox-init (PID 1)   → POST /init  (envd re-key, Secure 模式)                            │  │  │   │
│  │  │  │                                                                                              │  │  │   │
│  │  │  │  vsock relay           ← 层1 终点: 接受 virtio-vsock 连接 (host Forwarder CONNECT <port>)    │  │  │   │
│  │  │  │                           读层2 TypeConnect{Network,Address} → dial 目标本地端口             │  │  │   │
│  │  │  │                           在 vsock channel 与本地 TCP 连接之间做字节 relay                   │  │  │   │
│  │  │  │                                                                                              │  │  │   │
│  │  │  │  envd          :49983  ← vsock relay dial (TypeConnect{tcp,127.0.0.1:49983})                │  │  │   │
│  │  │  │  code-interp   :49999  ← vsock relay dial (TypeConnect{tcp,127.0.0.1:49999})                │  │  │   │
│  │  │  │                                                                                              │  │  │   │
│  │  │  │  user app      :PORT  (sandbox-ctl 不在此路径；经 virtio-net 而非 vsock relay)              │  │  │   │
│  │  │  │    入向: node-proxy dial(floatingip:PORT) → mg0(root netns) → veth → sw-mX(switch netns)    │  │  │   │
│  │  │  │           → TC DNAT(floatingip→inner_ip) → bpf_redirect → sw0-tN tap → CH virtio-net → guest │  │  │   │
│  │  │  │    回程: guest → CH virtio-net → tap → sw0-tN TC SNAT(inner_ip→floatingip) → sw-mX → mg0   │  │  │   │
│  │  │  │           → host root netns → node-proxy TCP socket                                          │  │  │   │
│  │  │  │    当前: node-proxy 全程 splice（write flowtable 待 FE-7.2 实现）                             │  │  │   │
│  │  │  │    FE-7.2 入向: transit NIC TC hook flowtable 命中 → 改写 dst→floatingIP → mg0 → sw-mX →   │  │  │   │
│  │  │  │           DNAT → tap → CH → guest（绕过 node-proxy）                                        │  │  │   │
│  │  │  │    FE-7.2 回程: guest → tap → sw0-tN SNAT → mg0 → transit NIC TC hook flowtable 反向命中   │  │  │   │
│  │  │  │           → 改写 src→nodeIP → client                                                        │  │  │   │
│  │  │  │                                                                                              │  │  │   │
│  │  │  │  MMDS 握手     169.254.169.254:80 ──(vswitch DNAT)──► 127.0.0.1:19254 (node-proxy MMDS)     │  │  │   │
│  │  │  │                                                                                              │  │  │   │
│  │  │  │  DNS 查询      169.254.169.253:53  ──(vswitch DNAT)──► 127.0.0.1:19253 (CoreDNS)           │  │  │   │
│  │  │  │                                                                                              │  │  │   │
│  │  │  │  出站流量      guest 网卡(virtio-net) → CH virtio-net backend → tap fd                       │  │  │   │
│  │  │  │    → vswitch TC eBPF(GENEVE 封装) → transit NIC (eth1) → 外部网关 / 跨宿主 overlay          │  │  │   │
│  │  │  └──────────────────────────────────────────────────────────────────────────────────────────── ─┘  │  │   │
│  │  └────────────────────────────────────────────────────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                                                 │
│  ┌─ sandbox-vswitch [vswitch-ctl serve, sw0_vswitch netns] ──────────────────────────────────────────────  ┐   │
│  │  ← vswitch-ctl attach --port=N --mac=... --floatingip=... --egress-policy=...  (node-ctl 子进程调用)    │   │
│  │  ← vswitch-ctl detach --port=N                                                  (沙箱销毁时)            │   │
│  │  ← vswitch-ctl open-port --port=N  → TAP fd (SCM_RIGHTS) → cloud-hypervisor   (沙箱启动时)            │   │
│  │                                                                                                          │   │
│  │  mgmt-service DNAT (mgmt-extract netns 内):                                                              │   │
│  │    169.254.169.254:80  (UDP+TCP) → 127.0.0.1:19254   node-proxy MMDS v2                               │   │
│  │    169.254.169.253:53  (UDP+TCP) → 127.0.0.1:19253   CoreDNS                                          │   │
│  │                                                                                                          │   │
│  │  GENEVE 跨宿主: sandbox floatingip ←──── transit NIC (eth1) ────► gateway / 对端节点                    │   │
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
│  │  ├─ map_flowtable   BPF_MAP_TYPE_LRU_HASH  [FE-7.2，未实现]                                             │   │
│  │  │    key:  {src_ip, src_port, dst_ip, dst_port} (网络字节序)                                            │   │
│  │  │    val:  {floatingip, port}                                                                           │   │
│  │  │    ↑ node-proxy FlowTableWriter  bpf_map_update_elem  (新建 TCP 连接后写入) [未实现]                 │   │
│  │  │    ↑ node-proxy                  bpf_map_delete_elem  (连接关闭后清理)      [未实现]                 │   │
│  │  │    ← TC hook 读取: 已建连接在内核路径直接转发, 报文不再经过 node-proxy 用户态 [未实现]                 │   │
│  │  │                                                                                                       │   │
│  │  ├─ map_conntrack   SYNACK / ESTABLISHED / CLOSING  (连接状态追踪)                                       │   │
│  │  └─ map_egress_policy / map_egress_allow_lpm / map_egress_deny_lpm  (出站策略, FE-NET-1.1)              │   │
│  │                                                                                                          │   │
│  │  cgroup v2  当前: unit cgroup（--cgroup-adopt，sandbox-ctl + CH 同处）                                  │   │
│  │             计划: <unit-cgroup>/sandbox-<sid>/（--cgroup-isolated，仅 CH）                              │   │
│  │  ├─ memory.max / memory.current / memory.stat     ← resource controller 读取 (Active Reclaimer)        │   │
│  │  └─ cpu.stat / cpuset.cpus.effective              ← metrics API 读取 (FE-9.1)                          │   │
│  └──────────────────────────────────────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

#### 4.0.2 node-proxy worker 内部模块与 IPC 汇总

```
外部入站 (cluster-router → :8443 HTTPS)
         │
         ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────────────┐
│  node-proxy worker 进程  [node-proxy@N.service]                                                      │
│                                                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────────────────────────┐  │
│  │  goroutine: Accept 循环  (TLSListener :8443 SO_REUSEPORT)                                    │  │
│  └──────────────────────────────┬───────────────────────────────────────────────────────────────┘  │
│                                 │ per-conn goroutine                                               │
│                                 ▼                                                                   │
│  ┌──────────────────────────────────────────────────────────────────────────────────────────────┐  │
│  │  goroutine: 请求处理                                                                          │  │
│  │  1. HostRouter.Parse(req.Host) → (sid, port)               失败 → 400                         │  │
│  │  2. RouteTable.Lookup(sid)     → RouteEntry                未命中 → 404                       │  │
│  │  3. AuthMiddleware.Check(X-Access-Token, entry.AccessToken) 不符 → 401 (enforce 模式)         │  │
│  │  4. Dispatcher.Dispatch(entry.State, port, conn)                                              │  │
│  │     ┌─────────────────────────────────────────────────────────────────────────────────────┐  │  │
│  │     │  state=running                                                                      │  │  │
│  │     │  port 49983 → dial UDS(entry.EnvdUDS)  →  io.Copy 双向 splice                      │  │  │
│  │     │               (envd.sock = host UDS，Forwarder 持有；内部经 vsock 穿透到 guest envd:49983)    │  │  │
│  │     │  port 49999 → dial UDS(entry.CiUDS)   →  io.Copy 双向 splice                      │  │  │
│  │     │               (ci.sock   = host UDS，Forwarder 持有；内部经 vsock 穿透到 guest code-interp:49999)  │  │  │
│  │     │  other port → dial TCP(entry.FloatingIP:port)                                       │  │  │
│  │     │               → FlowTableWriter.Register(4-tuple, floatingip, port)  [bpffs 写]    │  │  │
│  │     │               → io.Copy 双向 splice (首包及后续直到 flowtable 接管)                  │  │  │
│  │     │               → 连接关闭: FlowTableWriter.Remove(4-tuple)           [bpffs 清]     │  │  │
│  │     │                                                                                    │  │  │
│  │     │  state=paused                                                                      │  │  │
│  │     │  ParkQueue.Park(sid, conn, deadline)                                               │  │  │
│  │     │  WakeSender.Wake(sid)  →→→ wake 帧 →→→ RouteSyncClient → serve                   │  │  │
│  │     │  等待: UnparkAll(sid) [wake 成功] → 重走 running 分支                               │  │  │
│  │     │        Cancel(sid)    [超时/delete] → conn.Close() → 503 sandbox_wake_timeout      │  │  │
│  │     │                                                                                    │  │  │
│  │     │  state=saved   → 409 Conflict, X-Migration-Token header                            │  │  │
│  │     │  不存在/dead   → 404/410                                                           │  │  │
│  │     └─────────────────────────────────────────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────────────────────────┐  │
│  │  goroutine: RouteSyncClient  (常驻, config-socket plugin plane)                               │  │
│  │                                                                                              │  │
│  │  连接: PUT /internal/plugin/{id}/register  [config-socket UDS h2c]                          │  │
│  │        SO_PEERCRED pid 鉴权  (serve 侧验 plugin_pidfile)                                    │  │
│  │                                                                                              │  │
│  │  上行写:  {"type":"register", "subscribe":{"kind":"route_wake"}, "mmds":true, ...}           │  │
│  │           {"type":"wake",     "sid":"abc123"}   (Dispatcher 触发, singleflight/sid)          │  │
│  │                                                                                              │  │
│  │  下行读:  {"type":"hello",    "hello":{"version":1, "policy":{...}}}   → 更新 auth/timeout  │  │
│  │           {"type":"upsert",   "route":{...}}   → RouteTable.ApplyUpsert(entry)              │  │
│  │           {"type":"delete",   "sid":"..."}     → RouteTable.ApplyDelete(sid)                │  │
│  │                                                 + ParkQueue.Cancel(sid)                     │  │
│  │           {"type":"bookmark"}                  → 清理本次未出现的孤儿路由条目               │  │
│  │                                                                                              │  │
│  │  断连重试: 指数退避 100ms → 200ms → 400ms … max 30s                                         │  │
│  └──────────────────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────────────────────────┐  │
│  │  goroutine: MMDS v2 server  (HTTP/1.1  127.0.0.1:19254)                                     │  │
│  │                                                                                              │  │
│  │  请求来源:  guest envd  →  169.254.169.254:80  →(vswitch DNAT)→  127.0.0.1:19254            │  │
│  │            src_ip = sandbox floatingip  (DNAT 保留原始 src)                                 │  │
│  │                                                                                              │  │
│  │  PUT /latest/api/token                                                                       │  │
│  │    ByFloatingIP(src_ip) → sid → route.MmdsSecret                                           │  │
│  │    token = sid + "." + hex(HMAC-SHA256(mmds_secret, sid))  (确定性，无存储，无 TTL)          │  │
│  │                                                                                              │  │
│  │  GET /latest/meta-data/  (X-metadata-token: <token>)                                        │  │
│  │    验 token → 返回 {accessTokenHash: hex(SHA512(EnvdAccessToken))}                          │  │
│  │    envd 比对 hash, re-key 完成                                                              │  │
│  └──────────────────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────────────────────────┐  │
│  │  goroutine: MetricsServer  (HTTP  127.0.0.1:9090)                                            │  │
│  │  GET /metrics → Prometheus text format                                                       │  │
│  └──────────────────────────────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────────────────────────────┘
        │ UDS splice                    │ TCP splice          │ bpffs              │ routesync h2c
        ▼                              ▼                    ▼                   ▼
  envd.sock / ci.sock           floatingip:PORT    /sys/fs/bpf/vswitch/   /run/sandbox/
  (host UDS, sandbox-ctl        (vswitch 转发)     map_flowtable          node-ctl.socket
   Forwarder 持有；每条连接
   经 vsock 反向通道穿透到
   guest 内 127.0.0.1:49983/9)
```

#### 4.0.3 proxy worker 启动与路由同步序列

```
systemd                node-proxy worker               node-ctl serve              sandbox-ctl
   │                          │                               │                         │
   │  StartUnit               │                               │                         │
   ├─────────────────────────►│                               │                         │
   │                          │                               │                         │
   │           [注册握手]      │ PUT /internal/plugin/proxy-N/register                   │
   │                          │   + register 帧（随请求 body 即时写入，不等 200 OK）      │
   │                          ├──────────────────────────────►│                         │
   │                          │                               │ SO_PEERCRED 鉴权         │
   │                          │                               │ ReadMessage → register  │
   │                          │◄── 200 OK + hello 帧 (policy) ┤ 推送策略                │
   │                          │                               │                         │
   │           [全量同步阶段]  │◄── upsert(sid-1, running) ───┤ BeginSync               │
   │                          │◄── upsert(sid-2, paused) ────┤                         │
   │                          │◄── upsert(sid-3, saved)  ────┤                         │
   │                          │◄── ...                        │                         │
   │                          │◄── bookmark ──────────────────┤ 全量结束                │
   │                          │  清理孤儿条目                  │                         │
   │                          │                               │                         │
   │           [增量阶段]      │                               │                         │
   │                          │              sandbox 创建时    │◄── Admit ───────────────┤
   │                          │◄── upsert(new-sid, running) ─┤                         │
   │                          │                               │                         │
   │                          │              sandbox 暂停时    │                         │
   │                          │◄── upsert(sid-X, paused) ────┤                         │
   │                          │                               │                         │
   │       [proxy 收到针对 sid-X 的请求]                       │                         │
   │                          │ Resolve(sid-X)                │                         │
   │                          │   内部: waiters[sid-X] 挂起   │                         │
   │                          │         pending[sid-X]=true  │                         │
   │                          │ 上行: wake(sid-X) ────────────►│                         │
   │                          │                               │ resumeIfPaused(sid-X)   │
   │                          │◄── upsert(sid-X, running) ────┤ resume 完成             │
   │                          │  notifyLocked(sid-X)          │                         │
   │                          │  Resolve 返回 → 重走 running 分│                         │
   │                          │                               │                         │
   │                          │              sandbox 销毁时    │                         │
   │                          │◄── delete(sid-Y) ─────────────┤                         │
   │                          │  ApplyDelete →                │                         │
   │                          │    notifyLocked(sid-Y)        │                         │
   │                          │    （解除 waiters，Resolve     │                         │
   │                          │      返回 err → 503/404）      │                         │
   │                          │                               │                         │
   │           [断连重连]      │  (serve 重启 / 通道中断)       │                         │
   │                          │  指数退避 100ms → 30s          │                         │
   │                          │ PUT /internal/plugin/proxy-N/register                   │
   │                          │   + register 帧（重新注册）    │                         │
   │                          ├──────────────────────────────►│ 重新注册                 │
   │                          │◄── hello + 全量 upsert + bm ──┤ 重走全量同步             │
   │                          │                               │                         │
   │           [数据面: envd 端口连接建立（port=49983 为例）]  │                         │
   │                          │                               │                         │
   │  node-proxy worker    Forwarder(sandbox-ctl)    cloud-hypervisor   vsock relay   envd:49983
   │        │                    │                         │                │               │
   │        │ Dispatcher:        │                         │                │               │
   │        │ dial(envd.sock)    │                         │                │               │
   │        ├───────────────────►│                         │                │               │
   │        │                    │ [层1] dial vsock.sock   │                │               │
   │        │                    │       CONNECT <port>\n  │                │               │
   │        │                    ├────────────────────────►│                │               │
   │        │                    │                         │ virtqueue 写入  │               │
   │        │                    │                         │ + kick guest   │               │
   │        │                    │                         ├───────────────►│               │
   │        │                    │                         │                │ 层1 通道就绪  │
   │        │                    │ [层2] TypeConnect{tcp, 127.0.0.1:49983} │               │
   │        │                    ├─────────────────────────────────────────►│               │
   │        │                    │                         │                │ dial :49983   │
   │        │                    │                         │                ├──────────────►│
   │        │                    │                         │                │  TCP 连接建立 │
   │        │◄─────────────────────────────────────────────────────────────────────────────-│
   │        │    io.Copy relay 双向字节流（Forwarder ↔ vsock ↔ vsock relay ↔ envd）
   │
   │           [数据面: 用户端口连接建立（port=PORT，sandbox-ctl 不参与）]
   │
   │  node-proxy     mg0→sw-mX (mgmt veth)      cloud-hypervisor   guest user:PORT
   │        │                   │                       │                  │
   │        │ KindTCP:          │                       │                  │
   │        │ dial(floatingip:PORT)                     │                  │
   │        ├──────────────────►│                       │                  │
   │        │                   │ TC DNAT(floatingIP→inner_ip)             │
   │        │                   │ bpf_redirect → sw0-tN tap fd             │
   │        │                   ├──────────────────────►│                  │
   │        │                   │                       │ virtio-net       │
   │        │                   │                       │ virtqueue → guest│
   │        │                   │                       ├─────────────────►│
   │        │                   │                       │                  │ TCP 连接建立
   │        │◄──────────────────────────────────────────────────────────────│
   │        │    TCP splice 双向字节流（Forwarder/vsock 不在此路径）
   │        │    当前: node-proxy 全程 splice；write flowtable + TC hook 内核直通待 FE-7.2 实现
```

#### 4.0.4 关键 IPC 通道一览

**控制面**

| 通道 | 类型 | 端点 | 通信方向 | 鉴权方式 |
|------|------|------|---------|---------|
| config-socket plugin 平面 | UDS h2c 双向流 | `/run/sandbox/node-ctl.socket` | proxy ↔ serve (routesync) | SO_PEERCRED + plugin_pidfile |
| config-socket task 平面 | UDS h2c | `/run/sandbox/node-ctl.socket` | sandbox-ctl → serve | SO_PEERCRED (sandbox-ctl pid) |
| config-socket admin 平面 | UDS h2c | `/run/sandbox/node-ctl.socket` | node-ctl CLI → serve | SO_PEERCRED + socket 0600 |
| resource_listen.socket | UDS 帧化 JSON | `/run/sandbox/resource.sock` | sandbox-ctl → resource controller | UDS socket 权限 |
| node-link | mTLS h2c / plain h2c | cluster-ctl registry host:port | serve ↔ cluster-ctl | mTLS 证书 |
| TAPFD_SOCKET | UDS SCM_RIGHTS | (vswitch-ctl 内部) | vswitch-ctl → cloud-hypervisor（TAP fd 一次性交接）| — |
| D-Bus / sdbus | D-Bus | systemd socket | serve → systemd (StartUnit/StopUnit) | D-Bus policy |
| Prometheus metrics | HTTP loopback | `127.0.0.1:9090` | 监控采集 → proxy | — |

**数据面**

| 通道 | 类型 | 端点 | 通信方向 | 鉴权方式 |
|------|------|------|---------|---------|
| envd.sock / ci.sock | UDS (AF_UNIX stream) | `/run/sandbox/<sid>/envd.sock` | proxy → Forwarder（vsock 反向通道入口）| — (上层 token 鉴权) |
| vsock.sock | UDS (AF_UNIX stream) | `/run/sandbox/<sid>/vsock.sock` | Forwarder → CH virtio-vsock（层1 CONNECT + 层2 TypeConnect）| — (host-only socket) |
| floatingip:PORT | TCP | sandbox floatingip:PORT | **入向**（当前）proxy splice → mg0(root netns) → veth → sw-mX TC DNAT(floatingip→inner_ip) → bpf_redirect → sw0-tN tap → CH virtio-net → guest user app；**回程**（当前）guest → tap → sw0-tN TC SNAT(inner_ip→floatingip) → sw-mX → mg0 → proxy TCP socket；**FE-7.2 入向**：transit NIC TC hook flowtable 命中，改写 dst→floatingIP → mg0 → sw-mX DNAT → tap → guest（绕过 proxy）；**FE-7.2 回程**：guest → tap → sw0-tN SNAT → mg0 → transit NIC TC hook flowtable 反向命中，改写 src→nodeIP → client；sandbox-ctl 不参与 | X-Access-Token (proxy 层) |
| bpffs map_flowtable **[FE-7.2，未实现]** | 内核 BPF map | `/sys/fs/bpf/vswitch/map_flowtable` | proxy 写（入向正向条目 + 回程反向条目）/ TC hook 读（transit NIC ingress 双向命中） | CAP_SYS_ADMIN (root only) |
| MMDS v2 HTTP | HTTP/1.1 loopback | `127.0.0.1:19254` | guest envd → proxy（经 vswitch DNAT）| MMDS session token |

#### 4.0.5 proxy.mode=internal 进程拓扑差异

`proxy.mode=internal` 将 serve 与 proxy 合并为同一个 Go 进程运行，主要用于开发/单机调试场景。与 `external` 模式的差异如下：

**进程拓扑对比**

```
proxy.mode=external（生产）                  proxy.mode=internal（开发/调试）
─────────────────────────────────────────────────────────────────────────────
node-ctl.service                             node-ctl.service
  └── node-ctl serve                           └── node-ctl serve
        │ config-socket plugin plane                  ┌── [内嵌] proxy 逻辑
        │ (routesync IPC)                             │     TLSListener :8443 (proxy.data_listen)
        │                                             │     HostRouter / RouteTable
node-proxy@{1..K}.service × K                        │     Dispatcher / ParkQueue
  └── node-ctl proxy (×K 进程)                        │     MMDS v2 server
        RouteSyncClient → config-socket               │     MetricsServer
        RouteTable (进程内独立副本)               无 routesync IPC，RouteTable 为 serve 内部对象
        TLSListener :8443 (SO_REUSEPORT)         无 node-proxy@N.service 单元
```

**关键差异汇总**

| 维度 | external | internal |
|------|----------|----------|
| 进程数 | 1 serve + K proxy | 1 进程（serve + proxy 合一） |
| `node-proxy@N.service` | 有（K 个独立单元） | 无 |
| routesync IPC | config-socket plugin plane (UDS h2c) | 无（在进程内直接共享 RouteTable） |
| 路由表同步 | 下行帧推送（hello/upsert/delete/bookmark） | in-process 直接访问（无序列化） |
| wake 机制 | proxy 发 wake 帧 → serve OnWake() | Dispatcher 直接调用 orchestrator.Wake(sid) |
| MMDS v2 server | 位于每个 proxy worker 进程内 | 位于 serve 进程内 |
| data_listen 配置 | `--data-listen` CLI 标志（每个 proxy 进程） | `proxy.data_listen` in config.yaml（serve 读取） |
| SO_REUSEPORT | 多 proxy worker 共享 :8443 | 单监听器，无 SO_REUSEPORT 需求 |
| 故障隔离 | serve 崩溃不影响 proxy（数据面继续服务） | serve 崩溃 = 控制面 + 数据面同时不可用 |
| 适用场景 | 生产环境（高可用、多 worker 水平扩展） | 本地开发、单机测试（配置简单） |

**park/wake 差异（internal 模式下无 wake 帧）**

```
external 模式:
  Dispatcher.Dispatch(paused)
    → ParkQueue.Park(sid, conn)
    → WakeSender.Wake(sid)           ← 发 wake 帧到 serve (via routesync)
    → serve.OnWake(sid)              ← serve 恢复 sandbox
    → UnparkAll(sid)                 ← proxy 重新派发挂起连接

internal 模式:
  Dispatcher.Dispatch(paused)
    → ParkQueue.Park(sid, conn)
    → orchestrator.Wake(sid)         ← 进程内直接调用（无 IPC）
    → sandbox 恢复后更新 RouteTable
    → UnparkAll(sid)                 ← 重新派发
```

**注意事项**

- `proxy.mode=internal` 不支持多 worker 水平扩展，:8443 仅有单监听器。
- 生产环境必须使用 `proxy.mode=external`，以实现 serve/proxy 故障隔离与独立重启。
- `proxy.mode=off` 时不启动任何数据面监听，详见 §4.0.6。

#### 4.0.6 proxy.mode=off 进程拓扑说明

`proxy.mode=off` 是纯控制面模式，node-ctl serve 启动后**不构建任何数据面组件**，适用于离线管控、CI 环境预热、FE-2.1 开发阶段先行合入等场景。

**进程拓扑**

```
node-ctl.service
  └── node-ctl serve
        │
        ├── HTTPServer          :443  (TLS / h2c dev)   ← 控制面 API，正常提供
        ├── orchestrator core                            ← 沙箱状态机，正常运行
        ├── resource controller  resource.sock           ← 资源准入，正常运行
        ├── config-socket        node-ctl.socket         ← task/admin/api 平面，正常运行
        │     plugin 平面 ← 无 proxy worker 连接，但平面本身保持监听
        ├── node-link            → cluster-ctl registry  ← 正常上报
        └── SQLite (WAL)         node-ctl.db             ← 正常读写

        ✗  TLSListener :8443            ← 不启动
        ✗  HostRouter / RouteTable      ← 不构建
        ✗  Dispatcher / ParkQueue       ← 不构建
        ✗  MMDS v2 server               ← 不启动（mmds.enabled=true 时启动失败）
        ✗  RouteSyncClient / routesync  ← 不构建（plugin 平面只监听，不主动推路由）
        ✗  FlowTableWriter / bpffs 写入 ← 不执行

无 node-proxy@N.service 单元
无外部入站流量路径
```

**与其他模式的关键差异**

| 维度 | external | internal | off |
|------|----------|----------|-----|
| 数据面监听 :8443 | node-proxy × K | serve 内嵌 | **无** |
| routesync 推送 | 有（向 proxy workers） | 有（in-process） | **无**（plugin 平面保持监听但不推） |
| MMDS v2 server | proxy 进程内 | serve 进程内 | **不允许**（配置校验失败） |
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

1. **FE-2.1 并行开发**：proxy 功能尚未合入时，`proxy.mode=off` 可先行合入控制面代码，沙箱创建/销毁/暂停等生命周期 API 完整可用，仅缺数据面访问能力。
2. **离线管控节点**：节点仅作资源池成员注册与沙箱调度（cluster-ctl 分配），数据面由其他节点承载（跨节点 migration 场景）。
3. **CI/预热环境**：自动化测试只需验证控制面 API，不需要真实流量，避免 TLS 证书与端口占用。
4. **运维排障**：临时禁用数据面入站（如端口冲突、证书失效），保留控制面以便执行 `node-ctl sandbox list/stop` 等运维操作。

### 4.1 整体拓扑与数据流

```
外部 SDK / 用户浏览器
         │ HTTPS  Host: <port>-<sid>.<domain>
         ▼
  cluster-ctl router（集群层，按 Host 分流）
         │ HTTPS :8443
         ▼
┌──────────────────────────────────────────────────────────┐
│  宿主机节点                                               │
│                                                          │
│  node-proxy worker × K（SO_REUSEPORT，:8443）            │
│  ┌────────────────────────────────────────────────────┐  │
│  │ TLS 终止 → Host 解析 → (sid, port) 提取           │  │
│  │ auth: ConstantTimeCompare(X-Access-Token, route.Token) │
│  │                                                    │  │
│  │ state=running:                                     │  │
│  │   port 49983 → envd.sock（UDS splice）             │  │
│  │   port 49999 → ci.sock（UDS splice）               │  │
│  │   其他端口   → floatingip:port（TCP splice         │  │
│  │               + 注册 eBPF flowtable）              │  │
│  │                                                    │  │
│  │ state=paused:                                      │  │
│  │   → per-sid park 队列（挂起，最长 park_timeout）   │  │
│  │   → 上行 wake 帧 → serve                          │  │
│  │                                                    │  │
│  │ state=saved（cluster 深度休眠）:                    │  │
│  │   → 返回 migration_token 触发跨机恢复              │  │
│  │                                                    │  │
│  │ sid 不存在 / dead → 返回 404                       │  │
│  │                                                    │  │
│  │ MMDS v2 sidecar（mmds.enabled=true）:              │  │
│  │   guest PUT /latest/api/token → session token      │  │
│  │   guest GET /  → accessTokenHash                  │  │
│  └──────────────────────────┬─────────────────────────┘  │
│                             │                            │
│  ┌──────────────────────────▼─────────────────────────┐  │
│  │  routesync（config-socket plugin 平面）             │  │
│  │  ← serve 下行：upsert / delete / bookmark / hello  │  │
│  │  → serve 上行：register / wake                     │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                          │
│  ┌──────────────────────────────────────────────────┐    │
│  │  eBPF flowtable（/sys/fs/bpf/vswitch/）          │    │
│  │  {src_ip,src_port,dst_ip,dst_port} → {fp,port}   │    │
│  │  TC hook 内核直通（已建连接）                     │    │
│  └──────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────┘
```

### 4.2 proxy 进程模块划分

#### 4.2.1 核心模块

| 模块 | 职责 |
|------|------|
| `TLSListener` | 监听 `--data-listen`（SO_REUSEPORT），TLS 握手，Accept 循环 |
| `HostRouter` | 解析 HTTP Host header，提取 `(sid, port)`；支持 `<port>-<sid>.<domain>` 和 `E2b-Sandbox-Id` 两种寻址 |
| `RouteTable` | 本地路由表（`sync.Map[sid → RouteEntry]`）；`ApplyUpsert`、`ApplyDelete`、`Lookup` |
| `Dispatcher` | 按 (sid, port, state) 路由到目标：UDS splice / TCP splice / park；e2b profile port 49983/49999 → KindUDS；bare profile 访问此二端口 → KindDeny(501)；其他端口 → KindTCP；UDS 目标为 sandbox-ctl Forwarder 持有的 host 侧 UDS，每条连接经 vsock 反向通道穿透到 guest 内 envd/ci 进程 |
| `RouteTable` | 本地路由表（`sync.Map[sid → RouteEntry]`）；`ApplyUpsert`、`ApplyDelete`、`Lookup`；`Resolve(ctx, sid, port)` 内部实现 park/wake：paused 时挂起在 `waiters[sid] chan struct{}`，同时调 `Table.Wake(sid)` 写 wakeCh；`notifyLocked(sid)` 在 ApplyUpsert/Delete 时解除挂起；`pending map[string]bool` 防止同 sid 重复发 wake 帧 |
| `RouteSyncClient` | config-socket plugin 平面 h2c 客户端；register 帧作为请求 body 发出（首帧），服务端回 hello 后再推全量 upsert + bookmark；增量订阅；指数退避重连 |
| `MMDSServer` | 内嵌 MMDS v2 HTTP handler（`--mmds-listen`）；`ByFloatingIP` 查路由表；token = `sid + "." + hex(HMAC-SHA256(mmds_secret, sid))`，确定性，无存储 |
| `FlowTableWriter` | **[计划中，FE-7.2，当前未实现]** 新建 TCP 连接后写 eBPF flowtable（`bpf_map_update_elem`）；连接关闭后清理 |
| `AuthMiddleware` | `off/log/enforce` 三档；enforce 模式 `ConstantTimeCompare` |
| `MetricsServer` | Prometheus `/metrics` on `--metrics-listen` |

**envd.sock / ci.sock 的本质：vsock 反向通道转发器**

`RouteEntry.EnvdUDS`（`/run/sandbox/<sid>/envd.sock`）和 `CiUDS`（`ci.sock`）并非 envd/code-interpreter 进程本身的 socket，而是 `sandbox-ctl run` 启动时通过 `--connect` 参数在 host 侧创建的 **UDS 监听器**，由 `Forwarder` goroutine 持有。

**端到端数据面完整路径（以 port 49983 envd 为例）：**

```
客户端 SDK  (HTTPS / gRPC-web)
  │  目标: <49983-sid>.domain:443
  ▼
cluster-router  [集群入口，L7 TLS 终止]
  │  转发到节点 node-proxy :8443（HTTP/2 或 HTTP/1.1）
  ▼
node-proxy worker  [node-proxy@N.service，host netns]
  │  TLSListener.Accept
  │  HostRouter.Parse(Host) → (sid, port=49983)
  │  RouteTable.Lookup(sid) → RouteEntry{State:"running",
  │                             EnvdUDS:"/run/sandbox/<sid>/envd.sock"}
  │  AuthMiddleware.Check(X-Access-Token)   [enforce 模式]
  │  Dispatcher: port 49983 → dial("unix", EnvdUDS)
  ▼
sandbox-ctl Forwarder.acceptLoop  [sandbox-ctl run 进程，host netns，父 cgroup]
  │  /run/sandbox/<sid>/envd.sock 上接受连接
  │  层1: OpenForward() → dial /run/sandbox/<sid>/vsock.sock
  │       发送 CONNECT <port>\n  （指定 guest 侧 vsock 监听端口）
  ▼
cloud-hypervisor  [virtio-vsock 设备后端，host 用户态]
  │  收到 CONNECT 请求 → 写入 virtio-vsock virtqueue（共享内存）
  │  通知 guest kernel（类似 virtio-net kick 机制）
  ▼
guest kernel virtio_vsock 驱动
  │  从 virtqueue 读取连接请求 → 交给监听该 vsock port 的进程
  ▼
guest vsock relay  [guest 内，层1 终点]
  │  层1 通道建立完成（双向字节流就绪）
  │  层2: 读 TypeConnect{Network:"tcp", Address:"127.0.0.1:49983"}
  │       dial tcp 127.0.0.1:49983
  ▼
envd 进程  [guest 内，监听 127.0.0.1:49983]

← 双向 io.Copy relay 贯通整条链路：
   建立过程（两层协议）:
     层1: Forwarder dial vsock.sock → CONNECT <port>\n
          → CH virtio-vsock virtqueue → guest kernel virtio_vsock → vsock relay（层1终点）
     层2: Forwarder 发 TypeConnect{tcp, 127.0.0.1:49983}
          → vsock relay dial :49983 → envd（层2终点，连接就绪）
   字节流路径:
     客户端 ↔ node-proxy ↔ Forwarder ↔ [vsock.sock → CH virtio-vsock → vsock relay] ↔ envd
```

**vsock 通道的两层协议**

Forwarder 从 vsock.sock 到 guest envd 的过程涉及两个独立的协议层：

*第一层：vsock 连接建立（Forwarder ↔ CH，通过 virtio-vsock virtqueue）*

vsock.sock 是 CH 实现的 vsock 设备 host 侧 Unix socket 入口。Forwarder dial vsock.sock 后，先发送连接请求（`CONNECT <port>\n`），指定 guest 侧 vsock 监听端口。CH 收到后通过 **virtio-vsock virtqueue**（共享内存，类似 virtio-net 但完全绕过网络栈）将连接请求推给 guest 内核的 `virtio_vsock` 驱动，驱动交给 guest 内监听该端口的 vsock relay 进程。至此 Forwarder 与 guest relay 之间建立了一条双向字节流通道。

*第二层：TypeConnect 帧（Forwarder → guest relay，运行在已建立的 vsock channel 上）*

vsock 通道建立后，Forwarder 发送应用层帧 `TypeConnect{Network:"tcp", Address:"127.0.0.1:49983"}`，这是 sandbox-ctl 自定义的 IPC 协议，告诉 guest relay 在 guest 内部 dial 哪个本地服务。relay 收到后 dial `127.0.0.1:49983`（envd），然后在 vsock channel 与这条 TCP 连接之间做字节 relay。

| 层次 | 参与方 | 协议 | 作用 |
|------|--------|------|------|
| 第一层 | Forwarder ↔ CH vsock.sock ↔ guest kernel virtio_vsock | `CONNECT <port>\n` + virtio-vsock virtqueue | 建立 host→guest 虚拟字节流通道 |
| 第二层 | Forwarder → guest vsock relay（跑在第一层通道内）| `TypeConnect{Network, Address}` | 告知 relay 在 guest 内 dial 哪个本地服务 |

> virtio-vsock 与 virtio-net 的区别：virtio-net 传输 Ethernet 帧，经 tap → TC eBPF 进入网络栈；virtio-vsock 传输纯字节流，彻底绕过 guest 和 host 的网络栈，是 sandbox-ctl ↔ guest 之间的专用 IPC 通道。

关键含义：
- **sandbox-ctl 是数据面上的字节 relay**，不只是控制面组件；每个数据字节都经过 sandbox-ctl 进程的 `io.Copy` goroutine 搬运
- `envd.sock` 是 sandbox-ctl 在 host 侧创建的 UDS 监听器，并非 envd 自身的 socket
- `vsock.sock` 是 cloud-hypervisor 暴露的 vsock 设备入口，sandbox-ctl Forwarder 通过它向 guest 内发起反向连接
- **sandbox-ctl cgroup 隔离**：当前（`--cgroup-adopt`）sandbox-ctl 与 CH 同处 unit cgroup，`memory.high` 节流时存在 relay 链路死锁风险，当前接受该代价；计划（`--cgroup-isolated`，未实现）sandbox-ctl 留在 unit cgroup，CH 移入子 cgroup，彻底消除节流路径

`--connect` 指令由 orchestrator 在 `LaunchSpec` 中注入（`sandboxcfg.ConnectSpecs()`），仅对 e2b profile 生效；bare profile 不暴露控制端口。快照（pause）期间 Forwarder 会先 `Pause()` 阻止新连接，再 `CloseActive()` 折叠活跃 relay，保证快照窗口内无半开连接。

**用户端口（port ≠ 49983/49999）端到端数据面完整路径：**

用户端口走 TCP splice。当前实现中 node-proxy 全程承运（`httputil.ReverseProxy` + CONNECT tunnel io.Copy）。eBPF flowtable 加速（FE-7.2）尚未实现，以下阶段二为设计规格。

阶段一：经过 node-proxy 用户态（当前实现）

```
客户端 SDK  (HTTPS / TCP)
  │  目标: <PORT>-<sid>.domain:443
  ▼
cluster-router  [集群入口，L7 TLS 终止]
  │  转发到节点 node-proxy :8443（SO_REUSEPORT）
  ▼
node-proxy worker  [node-proxy@N.service，host netns]
  │  TLSListener.Accept
  │  HostRouter.Parse(Host) → (sid, port=PORT)
  │  RouteTable.Lookup(sid) → RouteEntry{State:"running",
  │                             FloatingIP:"x.x.x.x"}
  │  AuthMiddleware.Check(X-Access-Token)   [enforce 模式]
  │  Dispatcher: other port → dial TCP(FloatingIP:PORT)
  │  io.Copy 双向 splice 启动（用户态字节搬运，全程经过 node-proxy）
  ▼
mg0  [mgmt veth peer，root netns]
  │  内核路由：floatingIP/20 → mg0（vswitch 启动时安装）
  │  veth pair
  ▼
sw-mX  [mgmt veth switch 侧，switch netns（sw0_vswitch）]
  │  TC hook: tc/ingress_mx
  │  DNAT: dst floatingIP → dst inner_ip
  │  bpf_redirect(slot->ifindex)  → sw0-tN
  ▼
sw0-tN  [tap device，switch netns]
  │  CH 通过 SCM_RIGHTS 持有此 tap fd（vswitch-ctl open-port 交接）
  ▼
cloud-hypervisor  [virtio-net backend，host 用户态]
  │  从 tap fd 读取以太帧 → 经 virtio-net virtqueue 送入 guest
  ▼
guest 网卡 (virtio-net)  [guest netns]
  ▼
用户 app  [guest 内，监听 :PORT]

← 当前入向：客户端 → node-proxy（io.Copy splice）→ mg0 → veth → sw-mX DNAT → tap → CH virtio-net → guest
← 当前回程：guest → CH virtio-net → tap → sw0-tN TC SNAT(inner_ip→floatingIP) → sw-mX → mg0 → node-proxy TCP socket
← FE-7.2 入向：transit NIC TC hook flowtable 命中，改写 dst→floatingIP → mg0 → sw-mX DNAT → tap → guest（绕过 node-proxy）
← FE-7.2 回程：guest → tap → sw0-tN SNAT → mg0 → transit NIC TC hook flowtable 反向命中，改写 src→nodeIP → client
```

阶段二：flowtable 写入后，TC hook 内核直通（绕过 node-proxy）【计划中，FE-7.2】

```
客户端后续数据包  (已建连接)
  │
  ▼
transit NIC (eth1)  [TC hook: prog_tc_ingress]
  │  bpf_map_lookup_elem(map_flowtable, {src,sport,dst,dport}) → 命中
  │  内核改写 dst → FloatingIP:PORT；node-proxy 用户态 splice 循环退出
  ▼
mg0  [mgmt veth peer，root netns]
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
  │  改写 src FloatingIP → nodeIP（还原为客户端所见的 node-proxy 地址）
  ▼
client

← FE-7.2 入向：transit NIC TC hook flowtable 命中，改写 dst→floatingIP → mg0 → sw-mX DNAT → tap → guest（绕过 node-proxy）
← FE-7.2 回程：guest → tap → sw0-tN SNAT → mg0 → transit NIC TC hook flowtable 反向命中，改写 src→nodeIP → client
← 双向均绕过 node-proxy 用户态，吞吐量接近线速
```

连接关闭时，node-proxy 保留一个轻量 goroutine 监听 FIN/RST，收到后执行 `bpf_map_delete_elem` 清理 flowtable 条目，防止 4 元组复用时命中过期条目。**[FE-7.2 实现后生效；当前无 flowtable，连接关闭仅回收 io.Copy goroutine]**

与 envd/ci 路径的关键差异：

| 维度 | envd/ci（port 49983/49999） | 用户端口（其他 port） |
|------|----------------------------|----------------------|
| 转发层 | UDS → vsock（始终经过 sandbox-ctl） | TCP → mg0 → sw-mX TC DNAT → tap（当前全程经过 node-proxy；FE-7.2 后首包经 node-proxy，后续 transit NIC flowtable 内核直通）|
| 数据路径中的进程 | node-proxy + sandbox-ctl（全程字节 relay） | 当前：node-proxy 全程 io.Copy；FE-7.2 后：首包经 node-proxy，后续绕过所有用户态 |
| 网络命名空间穿越 | host UDS → vsock 虚拟设备 | host netns → vswitch netns → guest netns |
| 快照/暂停影响 | Forwarder.Pause() 阻断新连接 | paused 状态下 Resolve() 挂起请求，Table.Wake() 发 wake 帧 |
| 吞吐量上限 | sandbox-ctl io.Copy goroutine 瓶颈 | 当前：node-proxy io.Copy；FE-7.2 后：TC hook 线速转发 |

#### 4.2.2 proxy.mode 三档行为对比

| 模式 | node-proxy 进程 | MMDS 位置 | 生产适用性 |
|------|----------------|-----------|-----------|
| `external` | 独立 K 个 worker，`node-proxy@{1..K}.service` | proxy 进程内（每 worker 均可响应）| ✅ 推荐生产 |
| `internal` | serve 进程内嵌单实例 | serve 进程内 | ⚠️ 开发/低流量场景；serve 崩溃连数据面一并挂 |
| `off` | 不启动 | 无 | 🔧 调试 / 纯控制面验证 |

### 4.3 routesync 协议详细设计

#### 4.3.1 帧格式

```
[4B LE uint32 帧长度][JSON 正文（最大 1 MiB）]
```

#### 4.3.2 握手流程

```
proxy worker 连接 config-socket
    │
    ├─→ PUT /internal/plugin/{id}/register（SO_PEERCRED 鉴权）
    │     register 帧作为请求 body 随 PUT 立即发出（proxy 上行，首帧）
    │   {"type":"register","register":{
    │     "subscribe":{"kind":"route_wake"},
    │     "proxy":{"socket":{"path":"/run/sandbox/proxy-1.sock"}},
    │     "mmds":true}}
    │
    │← hello 帧（serve 读到 register 后下行，含策略）
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
    ├─→ wake（proxy 上行，paused 沙箱触发唤醒）
    │   {"type":"wake","sid":"abc123"}
```

proxy 在收到 `bookmark` 后，删除路由表中**未在本次全量同步中出现**的旧条目（孤儿清理），确保与 serve 路由表严格一致。

#### 4.3.3 RouteEntry 数据结构

```go
type RouteEntry struct {
    Sid            string `json:"sid"`
    Profile        string `json:"profile"`         // e2b | bare
    TemplateID     string `json:"template_id"`
    State          string `json:"state"`           // running | paused | saved
    EnvdUDS        string `json:"envd_uds"`        // 绝对路径，e2b profile 专属
    CiUDS          string `json:"ci_uds"`          // code-interpreter UDS
    FloatingIP     string `json:"floatingip"`      // 宿主机侧 tap 对端 IP
    AccessToken    string `json:"access_token"`    // sb.EnvdAccessToken，hex
    SnapLoc        string `json:"snap_loc"`        // local | remote
    MmdsSecret     string `json:"mmds_secret"`     // HMAC 确定性，hex
    MigrationToken string `json:"migration_token"` // saved 状态下的跨机恢复凭证
    Group          string `json:"group"`
    RouteKey       string `json:"route_key"`
}
```

### 4.4 流量路由决策树

```
收到入站连接
    │
    ├─ 解析 Host → (sid, port)
    │   失败 → 400 Bad Request
    │
    ├─ 查路由表 RouteTable.Lookup(sid)
    │   未命中 → 404 Not Found
    │
    ├─ AuthMiddleware（auth_mode=enforce）
    │   X-Access-Token ≠ route.AccessToken → 401 Unauthorized
    │
    ├─ route.State == "running"
    │   ├─ port == 49983 → dial unix(route.EnvdUDS)
    │   │                   EnvdUDS = /run/sandbox/<sid>/envd.sock  [host 侧 UDS]
    │   │                   └─ sandbox-ctl Forwarder acceptLoop
    │   │                        → vsock 反向通道握手（TypeConnect{127.0.0.1:49983}）
    │   │                        → guest 内 envd 进程 → splice 双向
    │   ├─ port == 49999 → dial unix(route.CiUDS)   （同上，目标 127.0.0.1:49999）
    │   └─ 其他端口    → dial TCP route.FloatingIP:port → io.Copy 双向 splice
    │                   → [FE-7.2 计划] 写 eBPF flowtable（4 元组 → {fp, port}）+ TC hook 接管
    │
    ├─ route.State == "paused"
    │   → RouteTable.Resolve(ctx, sid, port)  内部挂起在 waiters[sid] chan struct{}
    │   → Table.Wake(sid) 写入 wakeCh，RouteSyncClient 读取后发 wake 帧到 serve
    │     serve 触发 resume；resume 完成后 ApplyUpsert(state=running) 调 notifyLocked(sid)
    │   → notifyLocked 解除 waiters，Resolve 返回，重走 running 分支
    │   → park_timeout 到期：ctx 取消，Resolve 返回 err → 503/504
    │   （去重：pending map[string]bool 防止同 sid 重复发 wake 帧）
    │
    └─ route.State == "saved"
        → [计划中，proxy 当前代码尚未实现] 409 + X-Migration-Token: <migration_token>
```

### 4.5 eBPF flowtable 集成（计划中，属 FE-7.2，proxy 当前代码尚未实现）

> **注**：`internal/proxy/` 当前版本中不存在 `FlowTableWriter`、`bpf_map_update_elem` 等实现。以下为设计规格，待 FE-7.2 实现后生效。现阶段用户端口全程由 node-proxy io.Copy 双向 splice 承运。

已建 TCP 连接在 node-proxy 完成第一个 splice 循环后，写入 vswitch 共享 flowtable（`/sys/fs/bpf/vswitch/map_flowtable`）：

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

写入后，vswitch TC hook 在内核直接按 flowtable 转发，node-proxy 退出该连接的 splice 循环（保留 goroutine 监听 FIN/RST 以清理 flowtable 条目）。

**连接关闭时**：node-proxy 监听到 EOF/RST 后，执行 `bpf_map_delete_elem` 清理对应 flowtable 条目，避免旧条目干扰后续重用相同 4 元组的新连接。

### 4.6 park/wake 详细流程

```
proxy（state=paused）                   serve                      sandbox
      │                                   │                           │
      │ ParkQueue.Park(sid, conn)         │                           │
      │ WakeSender.Wake(sid)              │                           │
      ├─── wake 帧 ──────────────────────►│                           │
      │                                   │ OnWake(sid)               │
      │                                   │ per-sid singleflight      │
      │                                   │ resumeIfPaused()          │
      │                                   ├── StartUnit(runner@sid) ──►
      │                                   │                           │ restore snapshot
      │                                   │                           │ envd ready
      │                                   │◄─────────────────────────┤
      │                                   │ routesync upsert(running) │
      │◄─── upsert(state=running) ────────┤                           │
      │ ParkQueue.UnparkAll(sid)          │                           │
      ├── splice ──────────────────────── ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─►│
      │                                                               │

异常路径（sid unknown/dead）：
      │◄─── delete 帧 ────────────────────┤
      │ ParkQueue.Cancel(sid) → 返回 404  │
```

**singleflight 保证**：多个 proxy worker 可能同时收到同一 paused 沙箱的请求，分别发 `wake` 帧；serve 侧对同一 sid 的 `OnWake` 调用用 singleflight 合并，只执行一次 `resumeIfPaused`，防止并发 restore 导致 cloud-hypervisor 状态混乱。

### 4.7 MMDS v2 设计

proxy 在 `--mmds-listen`（默认 `127.0.0.1:19254`）上运行 HTTP/1.1 服务。vswitch 将 guest 内 `169.254.169.254:80` 的流量 DNAT 至此地址。

**mmds_secret 确定性推导**（serve 侧计算，经 routesync 下发）：

```
mmds_secret = HMAC-SHA256(manifest_key_bytes, "kuasar-mmds-v1:" + sid)
```

manifest_key 明文不离开 serve 内存；N 个 proxy worker 均持有相同 mmds_secret（来自 RouteEntry），无需共享状态即可对等响应。

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
     │ 验 SHA512 hash == serve 提前下发的 EnvdAccessToken
     │ envd re-key 完成，后续 RPC 验 token
```

### 4.8 API 设计

#### 4.8.1 对外 API 变更（proxy 数据面入口）

node-proxy 本身不暴露控制面 REST API；其对外接口是 **HTTPS 数据面**，行为变更如下：

| 场景 | 请求格式 | 响应变化 |
|------|---------|---------|
| 正常转发 | `Host: <port>-<sid>.<domain>`，`X-Access-Token: <token>` | 透传后端响应，无附加 header |
| 鉴权失败 | 同上，token 不匹配 | **新增** 401 `{"message":"unauthorized"}` |
| Paused 沙箱唤醒中 | 同上，state=paused | **新增** 连接挂起最长 park_timeout，恢复后透传 |
| Saved 沙箱（跨机）| 同上，state=saved | **新增** 409 + `X-Migration-Token` header |
| 沙箱不存在 | 同上 | 404（现有行为，已有） |

服务级别：P99 连接建立延迟（running 沙箱）< 50 ms；吞吐量峰值受限于宿主机 NIC 带宽（不受 node-proxy 用户态限制）。

#### 4.8.2 周边 API 调用变更

| 调用方向 | 接口 | 变化 |
|---------|------|------|
| proxy → serve | config-socket plugin 平面 `PUT /internal/plugin/{id}/register` | **新增** |
| proxy → serve | routesync wake 上行帧 | **新增** |
| serve → proxy | routesync upsert/delete/hello/bookmark 下行帧 | **新增** |
| proxy → vswitch bpffs | `bpf_map_update_elem` / `bpf_map_delete_elem` on `map_flowtable` | **新增** |
| guest envd → proxy | MMDS v2 HTTP `169.254.169.254:80` → `127.0.0.1:19254`（vswitch DNAT）| 新增 MMDS sidecar，接口协议已有定义 |

**过载风险评估**：routesync 为 push 模型，serve 主动推送增量变更，proxy 本地查表 O(1)，无额外 RPC 调用在热路径上；flowtable 写入为内核调用，延迟 < 1 µs；整体热路径不引入新的外部依赖过载风险。

### 4.9 数据库设计

proxy 进程**无独立数据库**，路由状态完全来自 routesync 内存同步。serve 侧 SQLite `sandboxes` 表中以下字段为 proxy 路由装配的数据源：

| 字段 | 用途 |
|------|------|
| `floatingip` | proxy 转发目标 IP |
| `envd_uds` | port 49983 UDS 路径 |
| `ci_uds` | port 49999 UDS 路径 |
| `traffic_access_token` | proxy 鉴权比对值（明文存储，创建/connect 时 mint） |
| `state` | running/paused 决定 proxy 路由行为 |
| `snapshot_ref` | paused 快照位置（local/remote 决定 snap_loc） |

**无新增 DB 字段**（本特性不扩展 sandboxes schema）。

### 4.10 可观测性设计

#### 4.10.1 指标设计（Prometheus）

```
# 数据面连接计数（per-sid, per-port-type）
node_proxy_connections_total{state, port_type}
# state: routed | parked | rejected_auth | rejected_notfound | rejected_saved
# port_type: envd | ci | user

# 在途连接数（gauge）
node_proxy_connections_active{port_type}

# park/wake 延迟直方图
node_proxy_park_duration_seconds{result}
# result: woken | timeout | cancelled

# routesync 重连次数
node_proxy_routesync_reconnect_total

# routesync 路由表条目数（gauge）
node_proxy_route_table_size{state}
# state: running | paused | saved

# flowtable 写入延迟
node_proxy_flowtable_write_duration_seconds

# MMDS 请求计数
node_proxy_mmds_requests_total{method, result}
```

#### 4.10.2 日志与事件

**关键日志（结构化 JSON）**：

| 事件 | level | 关键字段 |
|------|-------|---------|
| routesync 连接建立 | info | worker_id, serve_addr |
| routesync 重连 | warn | worker_id, attempt, backoff_ms |
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

#### node-proxy@.service 启动标志

| 标志 | 默认值 | 说明 |
|------|--------|------|
| `--config-socket` | `/run/sandbox/node-ctl.socket` | serve config-socket 路径 |
| `--id` | `proxy-1` | worker 唯一标识 |
| `--socket` | `/run/sandbox/proxy-1.sock` | gateway 反代 UDS（可选）|
| `--data-listen` | `:8443` | SO_REUSEPORT 数据面监听 |
| `--tls-cert` | — | 证书 PEM 路径（必填，除非 h2c dev 模式）|
| `--tls-key` | — | 私钥 PEM 路径 |
| `--auth` | `enforce` | off / log / enforce |
| `--park-timeout` | `90s` | 策略到达前初始值；serve hello 帧下推后覆盖 |
| `--mmds-listen` | `127.0.0.1:19254` | MMDS HTTP 监听（mmds.enabled 时必填）|
| `--metrics-listen` | `127.0.0.1:9090` | Prometheus metrics |

### 4.12 sandbox-vswitch 网络层

sandbox-vswitch 是 kuasar-sandbox 的 L2/L3 数据面底座，proxy 的用户端口路由（floatingip:port）和 MMDS 管理流量均建立在它之上。

#### 4.12.1 总体架构

```
┌──────────────────────────────────────────────────────────────┐
│  sw0_vswitch netns（独立网络命名空间）                        │
│                                                              │
│  sw0-tN   tap device（microVM，持久化）                      │
│  sw0-pN   veth switch side（container）                      │
│  sw0-nN   veth sandbox side（container，attach 后移入沙箱 netns）│
│  sw0-mX   mgmt veth（接 mgmt netns，可选）                   │
│  transit  NIC（eth1，接外部网关，GENEVE 隧道出口）            │
│                                                              │
│  每个设备挂载 TC ingress eBPF 程序（shared block）           │
│  BPF map pin 在 bpffs → 控制面进程退出后数据面不中断         │
└──────────────────────────────────────────────────────────────┘
```

控制面（`vswitch serve` / `vswitch-ctl`）只负责配置 BPF map 和管理 netns；一旦 `start` 完成，转发逻辑完全在内核 TC eBPF 程序中执行，控制面进程退出不影响已建立连接。

**用户端口完整数据面路径（tap 模式）**

| 方向 | 路径 |
|------|------|
| 入向 | transit NIC → TC eBPF(GENEVE 解封装) → sw0-tN → tap fd（CH 持有）→ CH virtio-net backend → guest 网卡 → user app |
| 出向 | guest 网卡(virtio-net) → CH virtio-net backend → tap fd → sw0-tN → TC eBPF(GENEVE 封装) → transit NIC → 外部网关 |

cloud-hypervisor 是 vswitch tap 设备与 guest virtio-net 之间的桥接层：持有 tap fd（通过 `vswitch-ctl open-port` SCM_RIGHTS 交接），以用户态读写原始 Ethernet 帧，经 virtio-net virtqueue 送入/收自 guest，无需任何 netns 间通道。

#### 4.12.2 端口模式

##### tap 模式（microVM，kuasar 默认）

```
vswitch-ctl open-port sw0 --port=N
  → 进入 switch netns
  → open(/dev/net/tun) + TUNSETIFF(IFF_TAP|IFF_NO_PI|IFF_VNET_HDR)
  → TUNSETPERSIST(1)          ← tap 设备持久化，进程退出不消失
  → SCM_RIGHTS 把 fd 发到 TAPFD_SOCKET
  → cloud-hypervisor 接收 fd，作为 virtio-net backend
```

**为什么不需要 veth pair**：TAP 设备天生有两端——网络栈端（sw0-tN，位于 switch netns，TC eBPF 挂载于此）和字符设备端（fd，不绑定 netns）。cloud-hypervisor 持有 fd，直接以用户态读写原始 Ethernet 帧，充当 guest virtio-net ↔ host tap 的桥接器，无需任何 netns 间通道。

**tap 模式下用户端口数据面路径**

```
入向:
  transit NIC → TC eBPF(GENEVE 解封装) → sw0-tN（switch netns）
    → tap fd（CH 通过 SCM_RIGHTS 持有）
    → CH virtio-net backend（用户态，读 Ethernet 帧写 virtqueue）
    → guest virtio-net 驱动 → guest 网卡 → user app:PORT

出向:
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
  → node-proxy MMDS server（看到 src = floating_ip）
回程：反向 DNAT，guest 始终看到 src=169.254.169.254
```

**② sandbox ↔ 用户端口（两条子路径：本节点 node-proxy / 跨节点 GENEVE）**

**② a. 本节点 node-proxy → sandbox（mgmt veth 路径）**

入向（node-proxy 发起）：
```
node-proxy  [host root netns]
  │  dial TCP(floatingIP:PORT)
  │  内核路由：floatingIP/20 → mg0（vswitch 启动时安装）
  ▼
mg0  [mgmt veth peer，root netns]
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

回程（sandbox 回复 node-proxy）：
```
guest user app → guest 网卡 (virtio-net)
  → CH virtio-net backend → tap fd → sw0-tN
  │  TC hook: tc/ingress_nx
  │  识别 dst 在 host 范围（非 mgmt_cidrs，非 GENEVE 外部）
  │  SNAT: src inner_ip → src floatingIP
  │  bpf_redirect → sw-mX
  ▼
sw-mX → mg0 → host root netns → node-proxy TCP socket
```

FE-7.2 后入向（transit NIC 内核直通，绕过 node-proxy）：
```
transit NIC (eth1)  [TC hook: prog_tc_ingress]
  │  bpf_map_lookup_elem(map_flowtable, {src,sport,dst,dport}) → 命中
  │  改写 dst → floatingIP:PORT；node-proxy 用户态 splice 循环退出
  ▼
mg0  [mgmt veth peer，root netns]
  │  内核路由：floatingIP/20 → mg0
  │  veth pair
  ▼
sw-mX  [switch netns]
  │  TC DNAT: dst floatingIP → dst inner_ip
  │  bpf_redirect → sw0-tN
  ▼
sw0-tN → tap fd → CH virtio-net backend → guest user app:PORT
```

FE-7.2 后回程（双向均内核直通）：
```
guest user app → CH virtio-net → tap → sw0-tN
  │  TC SNAT: src inner_ip → src floatingIP
  │  bpf_redirect → sw-mX → mg0 → host kernel
  ▼
transit NIC (eth1)  [TC hook: prog_tc_ingress，反向]
  │  bpf_map_lookup_elem(map_flowtable, {src=floatingIP,sport,...}) → 命中
  │  改写 src floatingIP → nodeIP（还原为 node-proxy 地址）
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

> **与 proxy 的关系**：用户端口流量（非 49983/49999）node-proxy 通过 mgmt veth 路径（②a）将 TCP 连接送达 sandbox；sandbox-ctl 不参与此路径；proxy 完成 L7 鉴权后 io.Copy 全程 splice（FE-7.2 flowtable 接管后完全绕过 proxy 用户态）。

**③ tap fd 交接（cloud-hypervisor 拿到虚拟网卡）**

`vswitch-ctl open-port` 是一次性操作：持久 tap 常驻 switch netns；cloud-hypervisor fd 关闭时 tap 的网络栈端仍存在（TUNSETPERSIST），可供下次快照恢复时重新交接。

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
| 用户端口（非 49983/49999）| node-proxy → mg0 → sw-mX TC DNAT → sw0-tN tap → CH virtio-net → guest；回程 sw0-tN SNAT → mg0 → node-proxy | ❌ 不经手 | ✅ mgmt veth + tap | 网络包，走 TC eBPF（DNAT/SNAT）|
| envd/ci 端口（49983/49999）| node-proxy → envd.sock → Forwarder → vsock.sock → CH virtio-vsock → vsock relay → envd | ✅ Forwarder relay | ❌ 不经过 vswitch | virtio-vsock virtqueue，共享内存 |

vsock 由 cloud-hypervisor 实现，走 virtio-vsock virtqueue（共享内存），与 vswitch 的 tap 设备、TC eBPF 程序、GENEVE 隧道完全无关。两者是并行的两套 host↔guest 通道，面向不同用途：vswitch tap 承载对外可路由的网络流量（用户端口，sandbox-ctl/Forwarder 全程不参与），vsock 承载 sandbox-ctl 与 guest 之间的内部 IPC（含 envd/ci 数据面 relay，Forwarder 做两层协议 relay）。

---

## 5. 性能

### 关键路径延迟目标

| 路径 | P50 | P99 | 说明 |
|------|-----|-----|------|
| running 沙箱新连接建立（TLS + 路由查表 + splice 建立）| < 10 ms | < 50 ms | 主要消耗在 TLS 握手 |
| paused 沙箱 wake 延迟（park 开始到 splice 建立）| < 2 s | < 30 s | 受 resume（快照恢复）P99 主导 |
| 路由表查表（sync.Map Lookup）| < 100 ns | < 500 ns | O(1) hash map |
| flowtable 写入（bpf_map_update_elem）| < 1 µs | < 5 µs | 内核 syscall |
| 已建连接 TC hook 转发（字节级吞吐）| 接近 NIC 线速 | — | eBPF 内核旁路，绕过 node-proxy |
| routesync 增量 upsert 应用延迟 | < 1 ms | < 5 ms | JSON 解码 + sync.Map Store |
| MMDS session token 颁发 | < 1 ms | < 5 ms | HMAC 计算（确定性，无存储）|

### 容量分析

| 维度 | 单节点目标 | 说明 |
|------|-----------|------|
| 并发沙箱数 | 4096 | 路由表 4096 条 sync.Map 条目，内存 < 10 MiB |
| 并发 TCP 连接数 | > 10 万 | Go goroutine per-conn，8 KiB 栈 ≈ 800 MiB；受 ulimit 和内存限制 |
| park 队列（全部 paused 同时唤醒）| 4096 × 若干连接 | park goroutine 轻量；超时后自动回收 |
| routesync 全量同步时间（4096 条）| < 500 ms | 单次 JSON 帧约 1 KiB，4 MiB 总量，loopback h2c 传输极快 |

---

## 6. 可靠性

### 6.1 故障管理

| 失效场景 | 数据面影响 | 自愈路径 |
|---------|-----------|---------|
| `node-ctl serve` 崩溃 | 控制面中断；running 沙箱 proxy 凭缓存继续转发；paused 沙箱 wake 挂起至 park_timeout | systemd Restart=on-failure → Reconcile 收养 → proxy 自动重连重同步 |
| `node-proxy worker × 1` 崩溃 | 该 worker 上连接 RST；新连接由其余 K-1 worker 承接 | systemd RestartSec=1 重启 → 重走全量同步 |
| `node-proxy worker × K`（全部）崩溃 | 数据面中断 | systemd 依次重启所有 worker；running 沙箱的 microVM 不受影响（systemd 独立 unit） |
| bpffs `map_flowtable` 不可访问 | flowtable 集成降级；proxy 继续纯用户态 splice 转发（功能不损失，吞吐下降）| 日志告警 `FlowtableUnavailable`；运维检查 vswitch 状态 |
| routesync 通道积压（> 1024 帧）| proxy subscriber 被丢弃；指数退避重连后全量重同步 | 重连间隔：100ms → 200ms → 400ms … 最大 30s |
| serve 与 proxy 版本不匹配 | hello 帧 version 字段检测到不兼容时 proxy 主动关闭连接并告警 | 滚动升级时先升 serve，再滚动重启 proxy worker |

### 6.2 系统防呆

- **park 超时强制释放**：park goroutine 在 `park_timeout` 到期后无论 serve 是否响应都执行 Cancel，返回 404；防止请求永久挂起。
- **ApplyDelete 解除挂起**：收到 `delete` 帧时，`notifyLocked(sid)` 立即解除该 sid 所有 `waiters`，Resolve 返回 err，防止孤儿 goroutine。
- **全量同步孤儿清理**：`bookmark` 收到后，删除本次同步未出现的路由条目，防止路由表与 serve 状态永久分叉。
- **MMDS token 无状态**：token = `sid + "." + hex(HMAC-SHA256(mmds_secret, sid))`，确定性计算，无存储、无 TTL；token 泄露风险由 mmds_secret 轮换（sandbox 销毁）覆盖。
- **Forwarder 快照静默期保护**：快照（pause）开始前 `Forwarder.Pause()` 阻止新连接进入 envd.sock/ci.sock，`CloseActive()` 折叠所有活跃 vsock relay（层1通道 + 层2字节流一并关闭），保证快照窗口内不存在半开的 vsock 连接；快照完成后 `Resume()` 重新开放监听。防止快照期间 relay goroutine 持有 vsock channel 导致 VM 状态不一致。
- **sandbox-ctl cgroup 隔离防死锁**：当前（`--cgroup-adopt`）sandbox-ctl 与 CH 同处 systemd unit cgroup；`memory.high` 节流时 Forwarder relay goroutine（层1 virtqueue 写入 + 层2 io.Copy）可能被一并卡住，造成整条 `node-proxy ↔ Forwarder ↔ CH virtio-vsock ↔ vsock relay ↔ envd` 链路死锁；该风险当前已知并接受。计划（`--cgroup-isolated`，未实现）：sandbox-ctl 留在 unit cgroup，CH 移入子 cgroup `<unit-cgroup>/sandbox-<sid>/`；`memory.high` 仅节流 CH，sandbox-ctl 可随时发出 balloon inflate 命令解压；`KillMode=control-group` 覆盖整个子树，CH 不会因 sandbox-ctl 退出而成为孤儿。
- **tap fd TUNSETPERSIST 防用户端口中断**：用户端口数据面路径（`vswitch tap → CH virtio-net → guest`）与 sandbox-ctl 完全解耦。cloud-hypervisor 崩溃时 tap fd 引用计数归零，字符设备端关闭；但 sw0-tN 因 `TUNSETPERSIST(1)` 仍留在 switch netns，下次 CH 重启时 `vswitch-ctl open-port` 重新交接新 fd，用户端口数据面可自愈，无需重建 vswitch 端口或重写 eBPF flowtable 条目。

### 6.3 过载控制

- **routesync 背压**：serve 侧订阅通道缓冲 1024 帧，积压即丢弃 subscriber（非 block），防止 slow proxy 拖慢 serve 的状态广播。
- **park 队列无硬上限**：per-sid 的 park 连接数由 park_timeout 自然淘汰，不设硬上限（防止误杀合法慢 resume）；极端场景（大量同时唤醒）下 goroutine 数量受 OS 内存限制，建议配合 node-level max_connections 使用。
- **连接速率限制**：未内置（依赖上层 cluster-router 或负载均衡器做连接速率限制）。

### 6.4 冗余设计

- **多 worker 冗余（SO_REUSEPORT）**：生产建议 2 个 worker；任一 worker 崩溃时 OS 将新连接分发至存活 worker，存量连接 RST 后 SDK 重连至存活 worker。
- **eBPF flowtable bpffs pin**：flowtable BPF map pin 至 `/sys/fs/bpf/vswitch/`，proxy 进程崩溃后 map 持久存在，TC hook 继续按已记录条目转发已建 TCP 连接。
- **serve 与 proxy 进程解耦**：运维可独立重启 serve 或 proxy，互不依赖进程生命周期。

### 6.5 资源残留

- **TCP 连接**：proxy 进程退出时 OS 自动关闭所有 socket，客户端收到 RST；无残留。
- **eBPF flowtable 条目**：proxy 监听连接 EOF/RST 后清理对应条目；proxy 异常崩溃时，对应连接的 flowtable 条目将残留至 LRU 淘汰（`BPF_MAP_TYPE_LRU_HASH`，由 vswitch 控制 max_entries，不影响新连接）。
- **park goroutine**：park_timeout 超时后自动回收；正常关闭时 `ParkQueue.Close()` 取消所有 context。

### 6.6 健康检查

proxy worker 不暴露独立健康探针端点；node-proxy@.service 的存活状态由 systemd 监控（`Type=exec`，进程退出即为不健康，自动重启）。

`/metrics` 端点上的 `node_proxy_routesync_reconnect_total` 连续增长可作为代理健康告警依据。

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
| `ProxyAuthRejectionSpike` | `rate(node_proxy_connections_total{state="rejected_auth"}[5m]) > 10` | warning |
| `ProxyRoutesynReconnecting` | `rate(node_proxy_routesync_reconnect_total[5m]) > 0` | warning |
| `ProxyParkTimeoutHigh` | `rate(node_proxy_park_duration_seconds_count{result="timeout"}[5m]) > 1` | critical |
| `ProxyWorkerDown` | `up{job="node-proxy"} == 0`（任一 worker）| critical |
| `ProxyFlowtableWriteError` | `rate(node_proxy_flowtable_write_errors_total[5m]) > 0` | warning |

### 6.8 数据可靠性设计

routesync 路由表为**纯内存状态**，proxy 进程重启后通过全量重同步从 serve 恢复，无持久化需求；路由真相源为 serve 的 SQLite `sandboxes` 表，proxy 不维护独立状态，无数据一致性风险。

---

## 4. FE 与 US 分工

| FE 编号 | US 编号 | 描述 | 负责团队 | 预估工时 |
|--------|--------|------|---------|---------|
| FE-2.1 | US-2.1-1 | `node-ctl proxy` 子命令框架：TLSListener、HostRouter、SO_REUSEPORT 绑定 | orchestrator | M（1w）|
| FE-2.1 | US-2.1-2 | RouteSyncClient：config-socket plugin 平面 h2c 客户端、全量 + 增量同步、指数退避重连 | orchestrator | M（1w）|
| FE-2.1 | US-2.1-3 | RouteTable + Dispatcher：运行态路由查表、三档端口分发（UDS/floatingip/park）| orchestrator | S（3d）|
| FE-2.1 | US-2.1-4 | ParkQueue + WakeSender：park/wake 机制、per-sid singleflight、超时清理 | orchestrator | M（1w）|
| FE-2.1 | US-2.1-5 | FlowTableWriter：新建连接写 eBPF flowtable，关闭时清理；bpffs 不可达时降级 | orchestrator | S（3d）|
| FE-2.1 | US-2.1-6 | AuthMiddleware：off/log/enforce 三档，ConstantTimeCompare | orchestrator | S（2d）|
| FE-2.1 | US-2.1-7 | MMDSServer：内嵌 MMDS v2 HTTP，ByFloatingIP 查路由表，session token TTL | orchestrator | M（1w）|
| FE-2.1 | US-2.1-8 | serve 侧 routesync 广播：StreamAuthority、upsert/delete/bookmark/hello 下行帧实现 | orchestrator | M（1w）|
| FE-2.1 | US-2.1-9 | proxy.mode=internal：serve 内嵌单实例 proxy（复用同模块代码）| orchestrator | S（3d）|
| FE-2.1 | US-2.1-10 | node-proxy@.service systemd 单元模板 + 运维文档（启动标志、TLS 配置、worker 数建议）| infra | S（2d）|
| FE-2.1 | US-2.1-11 | 可观测性：Prometheus metrics、结构化日志、告警规则配置 | orchestrator | S（3d）|
| FE-2.1 | US-2.1-12 | 集成测试：running/paused/saved 场景 e2e 验证，auth 失败验证，worker 故障恢复验证 | QA | M（1w）|

**工时说明**：S=3d，M=1w；US-2.1-1 ～ 2.1-8 为 P0 核心路径，预计 6 周交付；US-2.1-9（internal 模式）和 US-2.1-10/11/12 可并行或紧随推进。
