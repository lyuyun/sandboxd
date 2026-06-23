# node-ctl 组件设计文档

**版本**: v1.0
**日期**: 2026-06-23
**状态**: 草稿

---

# 1. 背景与目标

## 1.1 需求范围

### 设计目标

node-ctl 是 Kuasar Sandbox 平台的**计算节点层编排引擎**，运行在每台宿主机上，向上对接集群控制面(cluster-ctl)，向下管理本节点上所有沙箱与构建任务的完整生命周期。

其核心职责覆盖以下五个维度：

1. **沙箱生命周期管理**：通过 systemd transient unit + cloud-hypervisor + sandbox-ctl 组合，实现沙箱的冷启动(快照恢复)、暂停(内存快照)、恢复、销毁，以及自动挂起(idle auto-suspend)；
2. **数据面路由装配**：维护本节点路由表(sid → floatingip:port)，经 routesync 协议实时推送给 node-proxy worker；node-proxy 基于 eBPF/TC 实现内核态数据转发，控制面重启期间已建立连接不中断；
3. **内存资源管控**：通过 cgroup v2 + virtio-balloon + 准入协议(UDS 消息流)实现 60× 内存超售，四水位联动令牌桶速率限制与 Active Reclaimer 主动回收；
4. **模板构建流水线**：三阶段(import / steps / template)流水线，所有构建步骤在 guest 内执行，宿主侧通过 sandbox-ctl exec 做平台接力，产物上传至 manifest 存储；
5. **密钥生命周期**：manifest_key 在节点内存中以 TTL 租约形式缓存(来自集群 key_put 或本机 manifest_keys 表)，全程不落明文，经确定性 MMDS 协议下发 guest。

### 需求列表(RR)

| 编号 | 需求 | 优先级 |
|------|------|--------|
| RR-01 | | |
| RR-02 | | |
| RR-03 | | |

### 特性范围(FE)

| 编号 | 特性 | 说明 |
|------|------|------|
| **FE-1** | **沙箱生命周期管理** | |
| FE-1.1 | 沙箱 CRUD + 生命周期 | create / list / get / kill / pause / resume |
| FE-1.2 | 自动挂起(auto-suspend) | 按 idle 时长触发 Pause，流量唤醒自动 Resume |
| FE-1.3 | Secure 模式与 CA Bundle 装配 | mmds.enabled、envd token re-key、CA 文件注入 guest |
| FE-1.4 | per-tenant 配额 + 优雅终止 + 入站连接限制 | 三类面向租户的准入/控制能力 |
| **FE-2** | **数据面代理与路由** | |
| FE-2.1 | 数据面路由与转发 | internal/external/off 三种 proxy 模式 |
| **FE-3** | **模板与构建** | |
| FE-3.1 | 模板构建流水线(含构建日志流) | fromImage / fromTemplate，三阶段，COPY/ADD，日志实时流 |
| FE-3.2 | 快照导出 / 导入 / 晋升 | 本机快照 → 远程 manifest，跨机迁移 token |
| **FE-4** | **内存管理与快照加速** | |
| FE-4.1 | 快照序列化与恢复 | 全量 memfile、差分脏页追踪、页去重、快照扇出 |
| FE-4.2 | 内存效率优化 | UFFD 按需加载、异步预取、balloon/hinting、DAX 共享★、内存统一持有★ |
| **FE-5** | **块存储与缓存** | |
| FE-5.1 | 块存储基础能力 | mmap 缓存、稀疏文件、流式加载、块去重、块状态追踪 |
| FE-5.2 | node★ 分层缓存与内容寻址 | L1/L2/L3 分层缓存、收敛加密、Maglev 亲和、EROFS 确定性展平 |
| **FE-6** | **存储集成** | |
| FE-6.1 | 文件存储卷 | SFS Turbo/NFS 命名卷、多卷挂载、跨会话数据持久化 |
| FE-6.2 | 对象存储接入 | S3/OBS 兼容凭证 MMDS 安全下发、运行时凭证刷新 |
| **FE-7** | **网络控制** | |
| FE-7.1 | 沙箱网络基础设施 | 网络隔离、slot 池化、发送速率限制 |
| FE-7.2 | node★ 高性能数据面 | eBPF/TC 内核态转发、ARP 代答、GENEVE 跨宿主隧道 |
| **FE-8** | **安全与鉴权** | |
| FE-8.1 | 密钥管理 | manifest_key TTL 租约、api_key HMAC 验证、MMDS 下发 |
| **FE-9** | **可观测性** | |
| FE-9.1 | 沙箱可观测性 | 运行时 metrics API、日志 API(v1/v2)、OTel trace |
| FE-9.2 | 审计日志持久化 | 结构化审计日志落盘，syslog 转发 |
| **FE-10** | **资源管控** | |
| FE-10.1 | 内存超售与准入 | 四水位、令牌桶、FIFO 队列、Active Reclaimer |
| **FE-11** | **运营与计费** | |
| FE-11.1 | Admin 批量管控 | Admin 强杀团队全部沙箱、Admin 取消团队全部构建 |
| FE-11.2 | 用量计量 | 沙箱启停时间记录、规格 × 时长计量、用量查询 API |
| FE-11.3 | 计费事件推送 | 生命周期事件(创建/暂停/恢复/销毁)结构化推送，Webhook 集成 |
| **FE-12** | **节点集群参与** | |
| FE-12.1 | 集群接入(node-link) | 节点注册、心跳上报、命令受理、路由事件上报 |
| FE-12.2 | 资源 drain | 节点维护模式，停止新准入，等待现有沙箱释放 |

## 1.2 交付边界

### 边界模型说明

**node-ctl 是计算节点上所有能力的统一控制入口**。客户端(SDK、cluster-ctl、平台 agent)只需对接 node-ctl 的 REST API 或 routesync 接口，无需感知节点内部的下游组件。

但 node-ctl 并非所有能力的实现者——它扮演**编排者**角色，将部分特性委托给专属组件执行。理解这一分工有两个意义：
1. 避免将"不在 node-ctl 代码中"误读为"平台不支持"；
2. 明确各组件的故障域，便于定向排查。

下表按"入口在哪里"归类所有平台特性：

### 以 node-ctl 为入口、委托下游组件实现的特性

这些特性的 **API 入口在 node-ctl**(客户端调 REST 或通过数据面访问)，node-ctl 负责参数组装、生命周期协调和状态上报，**实际执行由下游组件完成**：

| 特性 | node-ctl 的入口职责 | 下游实现组件 | 接口 |
|------|-------------------|------------|------|
| microVM 快照 / 恢复(pause / resume) | 调用 sandbox-ctl Pause/Restore；上报状态；路由表 paused 标记 | **sandbox-runtime**(sandbox-ctl + cloud-hypervisor) | CLI（subprocess） |
| 差分快照（增量链式快照） | 传 restore ref 给 sandbox-ctl；SnapshotProvenance 链（from_refs / overlay.base_from_refs）由 sandbox-runtime 内部维护，node-ctl 透明 | **sandbox-runtime** | SandboxConfig YAML |
| UFFD 按需加载、heap 收拢 | cloud-hypervisor 补丁内置能力，sandbox-runtime 透明使用；node-ctl 无感知，无需配置 | **sandbox-runtime**(cloud-hypervisor 补丁) | 无（平台内部） |
| 内存气球回收(balloon) | resource-controller 经 Heartbeat/OOMReport 触发；sandbox-ctl 内 BalloonController 调 CH API `/api/v1/vm.resize`（free_page_reporting intentionally OFF） | **sandbox-runtime**(sandbox-ctl → virtio-balloon → CH API) | UDS resource 协议 |
| DAX 共享 runtime 镜像(virtio-pmem) | `cfg.runtime_e2b` / `cfg.runtime_base` 路径经 sandboxcfg 写入 SandboxConfig `boot.runtime`；sandbox-ctl 以 `--pmem file=<path>,discard_writes=on` + 内核参数 `dax=always` 挂载 | **sandbox-runtime**(cloud-hypervisor virtio-pmem) | SandboxConfig YAML |
| 块存储 L1/L2/L3 分层缓存 | 快照恢复时 sandbox-ctl 向 store-ctl 按需拉取 chunk；node-ctl 无感知 | **sandbox-accelerator**(cache-ctl / store-ctl) | store-ctl 本地 UDS |
| 收敛加密、跨租户 chunk 去重 | 构建收尾调 `sandbox-ctl upload-snapshot`；manifest_key 经 env 传入 | **sandbox-accelerator**(manifest-ctl / store-ctl) | CLI + env |
| 确定性展平(EROFS 字节级相同) | 构建 Phase A/B：export 路径在 guest 内运行（`sandbox-ctl exec /opt/sandbox-runtime/bin/flatten-ctl export`）；镜像元数据读取在宿主侧运行（`flatten-ctl info --json`） | **sandbox-accelerator**(flatten-ctl，guest 内 export + 宿主侧 info) | sandbox-ctl exec stdio / subprocess |
| 网络槽分配(vswitch attach / detach) | 创建沙箱时调 `vswitch-ctl attach`；销毁时 detach；网络配置写入 SandboxConfig | **sandbox-vswitch** | CLI + tapfd UDS |
| 文件存储卷(SFS Turbo/NFS) | `POST /sandboxes` 接收 `volumes[]`；协调 volume-ctl 创建/绑定卷；对接 SFS Turbo 或自建 NFS export(FE-6.1，当前 ❌ 未实现) | **volume-ctl**(独立卷服务) | volume-ctl REST / CLI |
| 对象存储接入(S3/OBS) | 创建时接收 `storage.object_credentials` 字段并加密落盘；支持 S3 兼容协议及华为云 OBS；通过 MMDS v2 安全下发至 guest；支持运行时刷新(FE-6.2，当前 ❌ 未实现) | **MMDS**(node-ctl 内嵌) | MMDS HTTP 169.254.169.254 |
| L2 结构性隔离(ARP 代答)、GENEVE 跨宿主 | vswitch 内核 eBPF 自主运行，node-ctl 无需感知；由 vswitch-ctl start 预先建立 | **sandbox-vswitch**(内核 eBPF/TC，独立生命周期) | 无(透明) |
| 用户端口数据转发(HTTP / WS / gRPC) | node-proxy 按路由表 `floatingip:port` 转发；eBPF flowtable 接管已建连接 | **envd**(guest 内 Connect RPC 进程)+ **sandbox-vswitch**(内核转发) | UDS --connect / floatingip |
| Secure 模式(envd 双重鉴权) | node-proxy 寄宿 MMDS v2；生成 envd_access_token；routesync 下发 mmds_secret | **envd**(guest 内 `/init` re-key；后续 RPC 验 token) | MMDS v2 HTTP + envd Connect RPC |
| 沙箱应用日志(v1/v2 API) | REST 入口在 node-ctl；从 journald 按 unit + tag 读取 | **sandbox-runtime**(sandbox-ctl 宿主进程将 app stdio 直写 journald；guest console 经 virtio-serial 转发) | journalctl |
| 沙箱运行时 metrics(CPU/内存/磁盘) | REST 入口在 node-ctl；直接读 cgroup v2 文件系统(FE-9.1，当前 ❌ 未实现) | **Linux cgroup v2**(内核接口，无需额外进程) | `/sys/fs/cgroup/sandboxes/<sid>/` |

### 不经过 node-ctl 的平台能力

这些特性**客户端不经过 node-ctl REST API**，直接与专属组件交互，或由其他平台层提供：

| 特性 | 负责组件 | 说明 |
|------|---------|------|
| envd 进程执行 API(process.Start/Kill/List) | **envd**(guest 内) | 客户端通过数据面直接访问 `<port>-<sid>.<domain>:49983`，node-ctl 仅负责路由装配 |
| envd 文件系统 API(upload/download/compose) | **envd**(guest 内) | 同上，node-proxy 数据面透传 |
| 用户端口自动发现(port scanner / socat) | **envd**(guest 内) | envd 内置扫描器，无需 node-ctl 参与 |
| 全局跨节点沙箱列表 | **cluster-ctl registry** | 超出单节点边界；node-ctl 仅暴露本节点列表 |
| 集群调度(P2C / shuffle-sharding) | **cluster-ctl scaler** | node-ctl 仅上报负载心跳供 scaler 决策 |
| OAuth2 / SSO 认证 | 外部 auth 服务(当前 ❌ 未实现) | — |
| 用量计量、计费事件推送 | 外部计费服务(当前 ❌ 未实现) | — |
| Webhook 事件推送 | 外部消息服务(当前 ❌ 未实现) | — |
| 桌面 / Computer Use(Xvfb / VNC) | 用户应用层(当前 ❌ 未实现) | — |
| 出站网络策略(nftables / SOCKS5) | **sandbox-vswitch**(当前 ❌ 未实现) | 若实现：入口为 node-ctl 创建参数 → vswitch-ctl attach 附加策略 |
| 运行时 Feature Flag | 外部服务(当前 ❌ 未实现) | — |
| 日志聚合(Loki / ELK) | 基础设施层 | 审计日志 syslog forward 是对接点 |

### 对外接口约束

| 接口 | 协议 | 说明 |
|------|------|------|
| e2b REST API | HTTPS/h2c，JSON | 与 e2b SDK 兼容；api_key 认证 |
| envd(guest 内) | Connect RPC over UDS | sandbox-ctl `--connect` 映射，不改 envd 源码；数据面路由透传 |
| routesync(plugin 平面) | h2c 帧化 JSON(4B LE + JSON body) | node-proxy worker / 路由观察者订阅 |
| node-link(集群) | mTLS 或 plain h2c（可配置），同 routesync 帧格式 | 对端为 cluster-ctl registry，本文仅描述节点侧 |
| vswitch-ctl | CLI / tapfd UDS | attach/detach/open-port；网络槽分配入口 |
| sandbox-ctl | CLI（subprocess）；run-sandbox 路径为 execve | run/snapshot/restore/connect/exec/upload-snapshot |
| resource 协议 | UDS，帧化 JSON | Admit / Settled / RequestBudget / Heartbeat / OOMReport / Release / Reattach |

### 性能与可靠性约束

| 指标 | 目标 |
|------|------|
| 沙箱冷启动 P99(快照缓存命中) | < 500 ms |
| Pause → Resume P99 | < 200 ms |
| 单节点内存超售密度 | ≥ 60× |
| 数据面连接中断率(控制面重启期间) | 0%(eBPF flowtable pin bpffs 保证) |
| 准入排队 P99 延迟 | < 50 ms(green/yellow 水位) |
| Admit 拒绝率(正常负载) | < 5% |
| 节点重启后对账收养时间 | < 30 s(已有活单元) |

---

# 2. 系统设计

## 2.1 设计原理

见 [sandbox-orchestrator/docs/node.md § 1.2 设计原则](sandbox-orchestrator/docs/node.md#12-设计原则)。

## 2.2 部署架构

node-ctl 以 **systemd 单元层** 为沙箱生命周期的地基——每个沙箱以独立的 `sandbox-runner@<sid>.service` 运行，生命周期与 node-ctl serve 进程完全解耦：serve 崩溃后 microVM 继续运行，重启后通过 Reconcile 收养(§2.4.2)。数据面外置(`proxy.mode=external`)：node-ctl serve 承载控制面，独立 node-proxy worker 进程承载数据面，两者通过 config-socket plugin 平面的 routesync 流同步路由。

### 2.2.1 架构概览

```
                                                            e2b SDK / CLI
                                                                 │
                   ┌─────────────────────────────────────────────┴─────────────────────────────────────────────┐
                   │ A Control HTTPS   Host: api.<domain>                                                      │ B Data HTTPS   Host: <port>-<sid>.<domain>
                   ▼                                                                                           ▼
┌──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│  Cluster                                                                                                                                                 │
│                                                                                                                                                          │
│  ┌────────────────────────────────────────────────────────────┐       ┌──────────────────────────────────────────────────────────────────────────────┐   │
│  │  cluster-ctl registry/scaler                               │       │  cluster-ctl router                                                          │   │
│  │                                                            │       │    Host: api.<domain>          →  node-ctl serve :443                        │   │
│  │                                                            │       │    Host: <port>-<sid>.<domain> →  node-ctl proxy :8443                       │   │
│  └──────────────────────────┬─────────────────────────────────┘       └──────────────────────────────────────────────────────────────┬───────────────┘   │
└─────────────────────────────┼────────────────────────────────────────────────────────────────────────────────────────────────────────┼───────────────────┘
                              │ node-link(mTLS)                                                                                        │ HTTPS :8443
                              │ Up: node_register · heartbeat · route_upsert(running|paused)· route_delete · build_event               │
                              │ Down: command(create|connect|delete|key_put|key_drop|build_register)                                   │
                              ▼                                                                                                        ▼
┌──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│  Node                                                                                                                                                    │
│                                                                                                                                                          │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐  ┌─────────────────────────────────────────────┐                    │
│  │  node-ctl serve   api.listen :443                                               │  │  node-ctl proxy worker × K                  │                    │
│  │                                                                                 │  │  --data-listen :8443(SO_REUSEPORT)          │                    │
│  │  ┌──────────────────────────────────────────────────────────────────┐           │  │  --socket proxy-{id}.sock                   │                    │
│  │  │  HTTPServer handler                                              │           │  │                                             │                    │
│  │  │    /sandboxes · /templates                                       │           │  │  ┌──────────────────────────────────────┐   │                    │
│  │  └──────────────────────────────────────────────────────────────────┘           │  │  │  proxy                               │   │                    │
│  │                                                                                 │  │  │    running:                          │   │                    │
│  │  ┌──────────────────────────────────────────────────────────────────┐           │  │  │      49983 → UDS envd.sock           │   │                    │
│  │  │  orchestrator core                                               │           │  │  │      49999 → UDS ci.sock             │   │                    │
│  │  │  ├─ Sandbox state machine(running / paused / dead)               │           │  │  │      others → floatingip:port        │   │                    │
│  │  │  ├─ Reconcile(adopt)                                             │     ┌─Wake│──│──│    paused → park → Wake              │   │                    │
│  │  │  ├─ Reaper(deadline / auto-suspend / deep-idle)                  │     │     │  │  └──────────────────────────────────────┘   │                    │
│  │  │  └─ BuildPool                                                    │     │     │  │                                             │                    │
│  │  └──────────────────────────────────────────────────────────────────┘     │     │  │  ┌──────────────────────────────────────┐   │                    │
│  │                                                                           │     │  │  │  MMDS v2                             │   │                    │
│  │  ┌──────────────────────────────────────────────────────────────────┐     │     │  │  │  --mmds-listen 127.0.0.1:19254       │   │                    │
│  │  │  resource controller  (resource.sock)                            │     │     │  │  │    Resp guest GET 169.254.169.254:80 │   │                    │
│  │  │   State / AdmissionController / Allocator / Active Reclaimer     │     │     │  │  │    mmds_secret (from routetable)     │   │                    │
│  │  │   ◄── sandbox-ctl  Admit/Settled/Heartbeat/OOMReport/Release     │     │     │  │  └──────────────────────────────────────┘   │                    │
│  │  └──────────────────────────────────────────────────────────────────┘     │     │  │                                             │                    │
│  │                                                                           │     │  │                                             │                    │
│  │  ┌──────────────────────────────────────────────────────────────────┐     │     │  │                                             │                    │
│  │  │  config-socket  /run/sandbox/node-ctl.socket                     │     │     │  │  ┌──────────────────────────────────────┐   │                    │
│  │  │  ├─ plugin       ◄───────────────────────────────────────────────│─────┘     │  │  │  routesync                           │   │                    │
│  │  │  │               ────────────────────────────────────────────────│──────────►┤──├─►│    route upsert                      │   │                    │
│  │  │  ├─ launchTask   ◄── node-ctl run-sandbox  launchspec            │           │  │  │    route bookmark                    │   │                    │
│  │  │  ├─ buildTask    ◄── node-ctl run-builder  buildspec             │           │  │  │    route delete                      │   │                    │
│  │  │  ├─ admin        ◄── node-ctl CLI  manifest-key add/remove/list  │           │  │  └──────────────────────────────────────┘   │                    │
│  │  │  └─ api                                                          │           │  │                                             │                    │
│  │  └──────────────────────────────────────────────────────────────────┘           │  └───────────────────────┬─────────────────────┘                    │
│  │                                                                                 │                          │ → sandbox UDS / IP                       │
│  │  ┌──────────────────────────────────────────────────────────────────┐           │                          │                                          │
│  │  │  node Link                                                       │           │                          │                                          │
│  │  │  ├─ handleCommand   ◄── cluster-ctl                              │           │                          │                                          │
│  │  │  ├─ heartBeat       ──► cluster-ctl registry                     │           │                          │                                          │
│  │  │  ├─ events          ──► cluster-ctl registry                     │           │                          │                                          │
│  │  │  └─ routesync       ──► cluster-ctl registry                     │           │                          │                                          │
│  │  └──────────────────────────────────────────────────────────────────┘           │                          │                                          │
│  └──────────────────┬────────────────────────────────────────────────────────┬─────┘                          │                                          │
│                     │ R/W                                                    │ D-Bus / sdbus                  │                                          │
│  ┌──────────────────▼──────────────────────────────────────────────────┐     │                                │                                          │
│  │  SQLite(node-local, WAL)                                            │     │                                │                                          │
│  │  sandboxes · builds · manifest_keys                                 │     │                                │                                          │
│  └─────────────────────────────────────────────────────────────────────┘     │                                │                                          │
│                                                                              │                                │                                          │
│  ┌───────────────────────────────────────────────────────────────────────────▼─────────────────────────┐      │                                          │
│  │  systemd                                                                                            │      │                                          │
│  │  sandbox-runner@<sid>.service  × N  (execve sandbox-ctl)                                            │      │                                          │
│  │  sandbox-builder@<bid>.service × M  (spawns sandbox-ctl per phase)                                  │      │                                          │
│  └────────────────────────────────────────────────────────────────────────────────────┬────────────────┘      │                                          │
│                                                                                       │ execve                │                                          │
│  ┌────────────────────────────────────────────────────────────────────────────────────▼───────────────────────▼────────────────┐                         │
│  │  sandbox-ctl × (N+M)                                                                                                        │                         │
│  │  ├─ cloud-hypervisor                                                                                                        │                         │
│  │  └─ guest: sandbox-init / envd                                                                                              │                         │
│  │     ├─ 49983  envd                                                                                                          │                         │
│  │     └─ 49999  code-interpreter                                                                                              │                         │
│  └─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘                         │
│                                                                                                                                                          │
│  eBPF/TC hook                                                                                                                                            │
│  /sys/fs/bpf/vswitch/                                                                                                                                    │
│  ├─ prog_tc_ingress / prog_tc_egress                                                                                                                     │
│  ├─ map_flowtable  {4 tuple} → {floatingip, port}                                                                                                        │
│  └─ map_conntrack  (SYNACK / ESTABLISHED / CLOSING)                                                                                                      │
└──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

cluster-ctl router 按 Host 分流，两条路径互斥：
- **控制面**(`api.<domain>`)→ node serve `:443`
- **数据面**(`<port>-<sid>.<domain>`)→ node-ctl proxy `:8443`(`cluster.data_endpoint` 配置)

**serve 崩溃期间**：proxy worker 凭本地路由缓存继续转发 running 沙箱流量(paused 沙箱的 Wake 无人应答，请求挂起直至 `park_timeout` 超时)。serve 重启后 worker 自动重连，重新同步路由表。

**单个 proxy worker 崩溃**：SO_REUSEPORT 下其余 K-1 个 worker 继续接收新连接；systemd 重启该 worker 后重新注册即恢复。

**proxy worker 数量建议**：生产节点建议 **2 个**。Go M:N 调度器让单进程天然利用全部 vCPU，多 worker 的价值在于故障隔离——一个崩溃重启期间另一个继续接收新连接。沙箱密度极高或需细粒度滚动重启时可增至 **4 个**；超过 4 个边际收益很低。

### 关键交互路径

> 格式：`发起方 → 接收方  消息/动作`；无外部接收方时为组件内部操作，以 `:` 标注。

#### 沙箱生命周期(控制面)

**创建**
- SDK                → serve `:443`        `POST /sandboxes`（经 cluster-router，`X-API-KEY`）
- serve              :                     `apikey.Parse` 格式校验 → `resolveAllowed`（按指纹查 manifest_keys，HMAC 验证）
- serve              → systemd             D-Bus `StartUnit sandbox-runner@<sid>`
- sandbox-ctl        → serve               `resource.sock` `Admit`（内存准入）
- serve              → envd                `GET /health` 轮询就绪 → `POST /init`（下发 envVars；MMDS 开启时携带 accessToken）
- serve              → proxy workers       routesync `upsert`（`state=running`，含 access_token、mmds_secret、floatingip）
- serve              → cluster-ctl         node-link 上行 `route_upsert`（集群模式）

**暂停**
- SDK                → serve               `POST /sandboxes/{id}/pause`（`ownsSandbox` 验证 api_key ↔ ManifestKey HMAC）
- serve              → sandbox-ctl         `snapshot`（cgroup 冻结 + 内存快照，sandbox-ctl 内部处理）
- serve              → SQLite              `SetSnapshotRef` + `SetState(paused)`
- serve              → systemd             `StopUnit sandbox-runner@<sid>`
- serve              → vswitch-ctl         `detach`
- serve              → proxy workers       routesync `upsert`（`state=paused`）
- serve              → cluster-ctl         node-link 上行 `route_upsert(state=paused)`（集群模式）

**恢复（控制面主动 connect）**
- SDK                → serve               `POST /sandboxes/{id}/connect`（`ownsSandbox` 验证）
- serve              :                     per-sid singleflight → `resumeIfPaused` → `launch`（restore 快照）
- serve              → proxy workers       routesync `upsert`（`state=running`）

**销毁**
- SDK                → serve               `DELETE /sandboxes/{id}`（`ownsSandbox` 验证）
- serve              → systemd             `StopUnit sandbox-runner@<sid>`
- serve              → vswitch-ctl         `detach`
- serve              :                     清理 RunDir/BaseDir → `st.Delete`
- serve              → proxy workers       routesync `delete` 帧

#### 数据面访问与唤醒

**running sandbox 数据面**
- SDK                → proxy `:8443`       （经 cluster-router）解析 Host 提取 `(sid, port)`，查 routetable O(1)
- proxy              :                     `auth=enforce`：`ConstantTimeCompare(X-Access-Token, route.AccessToken)`，不符返回 401
- proxy              → envd.sock           端口 49983（envd Connect RPC）
- proxy              → ci.sock             端口 49999（code-interpreter）
- proxy              → floatingip:port     其他端口；新建连接 splice + 注册 eBPF flowtable；已建连接 TC hook 内核转发

**paused sandbox 数据面触发唤醒**
- proxy              :                     routetable 查到 `state=paused` → 新连接入 per-sid park 队列（最长 park_timeout）
- proxy              → serve               routesync `wake` 上行帧
- serve              :                     `OnWake` per-sid singleflight → `resumeIfPaused` → 广播 `upsert(state=running)`
- proxy              → sandbox             park 队列解除，splice 转发（happy path）
- serve              → proxy              sid 未知/dead 时：`OnWake` 调 `publishDelete`，广播 `delete` 下行帧至所有 proxy
- proxy              :                    `ApplyDelete` 清路由表、释放 park 队列 → 返回 404

**Secure 模式 MMDS 握手（mmds.enabled=true）**
- envd               → proxy MMDS          `PUT /latest/api/token`（169.254.169.254:80 经 nftables 重定向）
- proxy MMDS         :                     `ByFloatingIP` 查 routetable 得 sid → `MmdsSecret(sid)` 取 mmds_secret → 返回 session token
- envd               → proxy MMDS          `GET /`（携带 session token）；MMDS 返回 `accessTokenHash = hex(SHA512(EnvdAccessToken))`
- serve              → envd                `POST /init`（明文 EnvdAccessToken）；envd 验 SHA512 hash → re-key

#### 模板构建

**注册 + 触发**
- SDK                → serve `:443`        `POST /v3/templates`（经 cluster-router，`X-API-KEY`）
- serve              :                     `resolveAllowed` 验证 manifest key → `RegisterBuild()` 分配 templateID + buildID
- serve              → SQLite              写 `status=registered`
- serve              → SDK                 202 `{templateID, buildID}`
- SDK                → serve `:443`        `POST /v2/templates/{tid}/builds/{bid}`（含 fromImage/steps/startCmd/readyCmd）
- serve              :                     `TriggerBuild()` 校验 COPY hash 已上传（如有）
- serve              → SQLite              写 `status=waiting`
- serve              → SDK                 202 立即返回（不阻塞）
- SDK（可选）         → serve               `GET /templates/{tid}/files/{hash}` 取预签名 PUT URL
- SDK（可选）         → 对象存储             直接 PUT COPY 上下文

**BuildPool 调度**
- serve(BuildPool)   :                     2 s 轮询 SQLite，捞 `status=waiting` → CAS 改 `building` → goroutine `executeBuild`
- serve              → vswitch-ctl         `Attach`（分配网络 slot）
- serve              → systemd             D-Bus `StartUnit sandbox-builder@<bid>`（Type=oneshot，`lc.Start` 阻塞至退出）
- sandbox-builder    → serve               config-socket buildTask 平面 `FetchBuildSpec()`（SO_PEERCRED 鉴权）
- sandbox-builder    :                     `builder.Run(spec)`，三阶段串行，复用同一网络 slot

**三阶段流水线**
- sandbox-builder    → sandbox-ctl         **A. import**：启动空沙箱
- flatten-ctl(guest) → 镜像仓库             拉取镜像（租户 creds 仅在 exec env，不落宿主）
- sandbox-ctl        → sandbox-builder     导出 rootfs artifact（stdio 流）
- sandbox-builder    → sandbox-ctl         **B. steps**：以 import 产出为 rootfs 冷启动
- sandbox-builder    → envd                通过 envd exec 依次执行 RUN steps
- flatten-ctl(guest) → sandbox-ctl         导出新 rootfs artifact
- sandbox-builder    → sandbox-ctl         **C. template**：以最终 rootfs 冷启动
- sandbox-builder    → envd                `startCmd` 启动；`readyCmd` 每 2 s 轮询至成功
- sandbox-ctl        :                     `snapshot` 写快照 bundle
- sandbox-ctl        → manifest 存储        `upload-snapshot` 上传全部 artifact，refs 改写 `manifest://`
- sandbox-builder    :                     结果 JSON 写 stdout → unit 捕获为 `<bid>.result`

**构建完成**
- serve(executeBuild) → SQLite             读 `<bid>.result` → 写 `status=ready`，`persist_id=e2b-snp-<64hex>`
- serve              :                     `publishBuildState("ready")`
- SDK                → serve               `GET /templates/{tid}/builds/{bid}/status?logsOffset=N`（轮询进度与日志）
- serve              → cluster-ctl         node-link 上行 `build_event`（集群模式）

#### 集群协作(node-link)

**create command 下发**
- cluster-ctl        → serve               node-link 下行 `command(kind=create)`，含 SID、ManifestKey、TemplateRef
- serve              :                     `HandleCommand`：同步 `precheckCluster` → 立即返回 accept ack；异步 `bootCluster`
- serve              :                     `bootCluster`：`MintToken` → `launch` → `publishUpsert`
- serve              → cluster-ctl         node-link 上行 `route_upsert`（触发 registry Reserve 完成）

**key_put 分发**
- cluster-ctl        → serve               node-link 下行 `command(kind=key_put)`，含 ManifestKey、ExpiresUnix（TTL=3h，每小时）
- serve              :                     manifest_key 写入内存缓存；`precheckCluster` 从缓存匹配租户，无需查 SQLite

**心跳上报**
- serve              → cluster-ctl         node-link 上行 `heartbeat`，含 Zone、Allocated、Pool、Draining
- cluster-ctl(scaler) :                    `Draining=true` 时停止向该节点下发新 create；依据心跳做 P2C 调度

#### 初始化与重启恢复

**proxy worker 启动 / 重连**
- proxy worker       → serve               config-socket plugin 平面注册
- serve              :                     `StreamAuthority`：`Subscribe()` → `Range()`（全量 upsert）→ `bookmark`
- proxy worker       :                     清理未出现的旧条目；后续仅收增量 `upsert`/`delete`
- serve              → proxy worker        订阅通道满（1024 帧）时丢弃该 subscriber
- proxy worker       :                     指数退避重连 → 重走全量同步

**serve 重启后对账收养**
- serve(Reconcile)   → systemd             `ListUnitsByPatterns("sandbox-runner@*.service")` 获取活跃单元
- serve              → SQLite/内存          DB `state=running` 且有活跃单元 → 写内存缓存继续托管；无活跃单元 → 标 `state=dead`
- proxy workers      → serve               重连后自动触发全量路由同步

### 2.2.2 故障域矩阵

| 失效进程 | 数据面影响 | microVM 影响 | 自愈路径 |
|---|---|---|---|
| `node-ctl serve` | 控制面中断；worker 凭缓存继续转发 running 流量 | **无**(systemd 单元独立) | systemd 重启 → Reconcile 收养 → 恢复控制面；paused 沙箱 Wake 挂起至 park_timeout |
| `node-ctl proxy` worker × 1 | 该 worker 上的连接断 | **无** | SO_REUSEPORT 其余 worker 继续；systemd 重启后重连同步路由 |
| `sandbox-runner@<sid>` | 该沙箱不可用 | **仅该沙箱 dead**(`Restart=no`) | Reconcile 标 dead；客户重 create 或从 paused 快照 resume |

### 2.2.3 端口与 TLS 布局

| 配置项 | 值 | 承载内容 | 生产建议 |
|---|---|---|---|
| `api.listen` | `:443` | 控制面(`api.<domain>`)+ 兜底 gateway | 通配 TLS 证书(`*.<domain>` + `api.<domain>`) |
| `--data-listen`(proxy) | `:8443` SO_REUSEPORT | 数据面(`<port>-<sid>.<domain>`) | 与 `api.listen` 同一证书；`cluster.data_endpoint` 指向此端口 |
| `--metrics-listen`(proxy) | `127.0.0.1:9090` | Prometheus `/metrics` | 内网；不对外暴露 |
| `--mmds-listen`(proxy) | `127.0.0.1:19254` | MMDS v2；nftables 将 `169.254.169.254:80` 重定向至此 | loopback；`mmds.enabled=true` 时配 |
| `resource_listen.socket` | `/run/node-ctl/resource.sock` | 资源控制协议(Admit / Settled / Heartbeat / Release) | 本地 UDS；不对外 |
| `paths.config_socket` | `/run/sandbox/node-ctl.socket` | config-socket 四平面(task / admin / plugin / api) | 本地 UDS；0600 权限 |

dev 模式：`api.listen` 不配 TLS 即走 h2c，SDK 指向 `E2B_API_URL` / `E2B_SANDBOX_URL`，无需通配 DNS 与证书。

### 2.2.4 systemd 单元与启动顺序

**node-ctl.service**(运维管理，由部署方预装)

```ini
[Unit]
Description=kuasar node-ctl (e2b host + node resource control)
After=network-online.target cache-ctl.service store-ctl.service
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/bin/node-ctl serve --config /etc/node-ctl/config.yaml
Restart=on-failure
RestartSec=2

[Install]
WantedBy=multi-user.target
```

`serve` 可选 `--proxy internal|external|off` 覆盖 `proxy.mode` 配置。启动顺序：加载配置 → 打开 SQLite DB → 连接 systemd D-Bus → 初始化 Orchestrator → `InstallUnits()`(生成并安装 sandbox-runner@/sandbox-builder@ 模板单元，daemon-reload)→ `Reconcile()`(对账收养孤儿沙箱)→ 启动 Reaper(5 s 间隔)和 BuildPool(2 s 间隔)→ 可选启动资源控制器(`resource_listen.enabled`)→ 绑定 config-socket → 构建数据面(internal/external/off)→ 可选 MMDS 监听 → 绑定 API TLS 监听。

**node-proxy@.service**(external 模式，运维管理，`node-ctl` 不自动安装)

```ini
[Unit]
Description=kuasar node-ctl data-plane proxy %i
After=network-online.target node-ctl.service

[Service]
Type=exec
ExecStart=/usr/bin/node-ctl proxy \
  --config-socket=/run/sandbox/node-ctl.socket \
  --id=proxy-%i \
  --socket=/run/sandbox/proxy-%i.sock \
  --data-listen=:8443 \
  --tls-cert=/etc/node-ctl/tls/fullchain.pem \
  --tls-key=/etc/node-ctl/tls/privkey.pem \
  --auth=enforce \
  --park-timeout=90s \
  --mmds-listen=127.0.0.1:19254 \
  --metrics-listen=127.0.0.1:9090
Restart=always
RestartSec=1
```

主要标志：`--config-socket` / `--id`(每 worker 唯一，如 `proxy-1`)/ `--socket`(gateway 反代 UDS)/ `--data-listen`(SO_REUSEPORT，多 worker 共享同一端口)/ `--auth` / `--park-timeout`(策略到达前回退值，serve 握手后下推)/ `--mmds-listen`(`mmds.enabled=true` 时配)/ `--metrics-listen`。

启用(建议 2 个实例，见 §2.2.1)：`systemctl enable --now node-proxy@{1,2}.service`

**sandbox-runner@.service / sandbox-builder@.service**(node-ctl 自动生成安装)

serve 启动时调用 `InstallUnits()` 生成这两个模板单元写入 `units.dir`(默认 `/etc/systemd/system`)，随后 daemon-reload。每个沙箱/构建对应一个实例：

| 单元 | 触发方 | 启动命令 | 类型 |
|------|--------|---------|------|
| `sandbox-runner@<sid>.service` | `orch.launch()` via D-Bus | `node-ctl run-sandbox --sandbox-id=<sid> ...` | oneshot |
| `sandbox-builder@<bid>.service` | `orch.triggerBuild()` via D-Bus | `node-ctl run-builder --build-id=<bid> ...` | oneshot |

两个单元均通过 `--config-socket` 连回 node-ctl 取 LaunchSpec(task 平面)；sandbox-ctl 运行在单元的 cgroup(`--cgroup-adopt`)，KillMode=control-group 保证整个进程组原子终止。

## 2.3 API & DB 设计

### 2.3.0 模板（Template）核心概念

#### TemplateID 的两种形式

模板 ID 在生命周期中有两种形式，两者作用域完全不同，不可互换：

**1. 瞬态 ID（Transient ID）**

```
transient-<uuidv7>
```

`POST /v3/templates`（注册）时分配，对应数据库 `builds.template_id` 列。**仅用于 URI 路径参数 `{tid}`**，服务端通过 `b.TemplateID == tid` 校验归属，传入 Persist ID 会直接返回 404：

- `POST /v2/templates/{tid}/builds/{bid}` — 触发构建
- `GET /templates/{tid}/builds/{bid}/status` — 查询进度
- `GET /templates/{tid}/files/{hash}` — 检查 COPY 上下文

响应体中的 `templateID` 字段在构建完成前也返回此值，但这是响应字段，与 URI 参数含义不同（见下方）。

**2. 持久 ID（Persist ID）**

```
<profile>-<kind>-<64hex manifest key>

示例：
  e2b-img-a3f1...   e2b profile，冷启动镜像
  e2b-snp-9c2b...   e2b profile，快照模板
  bare-img-ff04...  bare profile（命令行路径）
```

构建完成后写入 `builds.persist_id` 列，Profile 和 Kind 编码在前缀中，自描述。出现在**响应体的 `templateID` 字段**中：

- `GET /templates/{tid}/builds/{bid}/status`（`status=ready` 后）— `templateID` 字段从 Transient 切换为 Persist
- `GET /templates`（只返回已完成模板）— `templateID` 字段始终是 Persist 形式；可通过 `buildID` 与构建流程中的 `bid` 对应
- `GET /sandboxes/{id}` / `GET /v2/sandboxes` — `templateID` 字段返回 Persist 形式

`POST /sandboxes` 的 `templateID` 请求字段接受 Persist ID、Transient ID、名称或别名，服务端通过 `resolveTemplateAlias` 统一解析为 Persist ID 后存入 `sandboxes.template_id`。

**两种形式在 API 中的位置对比**

| | 瞬态 ID `transient-<uuidv7>` | 持久 ID `<profile>-<kind>-<64hex>` |
|---|---|---|
| **URI 路径参数 `{tid}`** | ✓（唯一合法值） | ✗（返回 404） |
| **`POST /v3/templates` 响应 `templateID`** | ✓ | — |
| **`GET …/status` 响应 `templateID`（构建中）** | ✓ | — |
| **`GET …/status` 响应 `templateID`（ready）** | — | ✓ |
| **`GET /templates` 响应 `templateID`** | — | ✓ |
| **`POST /sandboxes` 请求 `templateID`** | ✓（接受，自动解析） | ✓（直接使用） |
| **`GET /sandboxes` 响应 `templateID`** | — | ✓ |

#### Manifest Key（64hex）的构成与确定性

`<profile>-<kind>-<64hex>` 中的 64 个十六进制字符是 **manifest blob 的 SHA-256**（32 字节），由 `manifest-ctl store`（img 产物）或 `sandbox-ctl upload-snapshot`（snp 产物）写入 manifest store 后输出：

```
ManifestKey = hex( SHA-256( codec.Marshal(manifest, sealed_key_table) ) )
```

**manifest blob 的结构**

| 部分 | 内容 |
|------|------|
| 头部 | version、chunk mode（FastCDC / Fixed）、image size、min/max chunk size |
| chunk 记录 | 每个 chunk 的 offset、size、`isZero` 标志、`store_content_key`（`SHA256(ciphertext)`） |
| hole 记录 | 稀疏文件的空洞区间（offset, size） |
| sealed key table | 所有 chunk 明文 key 经 customer key 密封的 AES-256-GCM 密文 |

**决定 manifest key 的四个输入**

| 输入 | 来源 | 说明 |
|------|------|------|
| **镜像内容**（image bytes） | EROFS 镜像（img）或 snapshot bundle（snp）的字节流 | FastCDC 收敛分块 + 收敛加密：相同内容 → 相同 ciphertext → 相同 chunk key → 相同 manifest blob |
| **Sparse 空洞**（holes） | 稀疏文件的 hole 区间 | 直接写入 manifest 头部；空洞位置不同 → manifest 不同 |
| **Generation salt** | `store.GetSalt()` 返回的 32 字节平台盐 | store 每次 salt 轮换后，相同内容产生不同 key；同一 generation 内跨节点一致 |
| **Customer key**（manifest_key） | 租户 32 字节对称密钥（`manifest-key add` 注册） | sealed key table 的 AES-GCM nonce 由 `HMAC-SHA256(customerKey, "ktable-nonce/v1\x00" ‖ aad ‖ keys)` **确定性推导**（非随机），保证相同内容重复上传产生相同 manifest blob |

> builder 调用 `manifest-ctl store` 时不传 `--extra-salt`，salt 只取 generation 盐，无额外偏移量。

**关键性质**

- **构建幂等**：相同源镜像 + 相同 generation + 相同 manifest_key → 完全相同的 PersistID，重复构建不产生新 key。
- **租户隔离**：不同 manifest_key → 相同镜像产生不同 PersistID，跨租户不可碰撞。
- **跨节点一致**：同一 generation 盐在集群内共享，任意节点上传相同内容得到相同 key，是 manifest store 作为内容寻址存储的基础。
- **盐轮换影响**：store generation 盐轮换（极少发生）后，同一镜像的 PersistID 会变化，旧 key 仍可读取（manifest 内已包含所有 chunk key）。

---

#### Profile：决定主进程归属

| Profile | 主进程 | `kuasar-sandbox.launch` | 适用场景 |
|---|---|---|---|
| `e2b` | 固定为 `envd`，提供 e2b API（代码执行、文件系统、code-interpreter 端口 49999） | **不支持**（envd owns launch） | 兼容 e2b SDK 的通用沙箱 |
| `bare` | 由镜像 Entrypoint/Cmd 决定，可通过 `kuasar-sandbox.launch` 覆盖 | **支持** | 自定义运行时、非 e2b 场景 |

**取值来源**：Profile 无 API 字段，不可配置——构建 API（`POST /v3/templates` + trigger）源码硬编码 `ProfileE2B`，只能产出 `e2b`；`bare` 仅由运维人员通过命令行工具写入 manifest store 后手动构造 templateID 指定。

**Guest Runtime EROFS**：Profile 的本质差别体现在 guest runtime 镜像，而非应用 rootfs（templateID 的 manifest key 对应的内容）。每个 sandbox 的 guest 由两层叠加组成：

| 层 | 内容 | e2b | bare |
|---|---|---|---|
| runtime EROFS（平台层） | `sandbox-init`（guest PID 1）+ 平台工具（`flatten-ctl` 等） + **`envd`**（e2b only） | `sandbox-runtime-e2b.erofs`（`boot.runtime_e2b`） | `sandbox-runtime.erofs`（`boot.runtime_base`） |
| 应用 rootfs（租户层） | 租户的 OCI 镜像内容，由 templateID 的 manifest key 寻址 | ← 两种 profile 相同，均来自 manifest store → | |

runtime EROFS 以 virtio-pmem DAX 方式挂载到 guest，多个沙箱共享宿主机同一份 page cache。`sandbox-runtime-e2b.erofs` 由构建脚本 `build-runtime-e2b.sh` 在 base runtime 基础上注入 `envd` 生成。

#### Kind：决定启动方式

| Kind | 启动方式 | 启动速度 | 每次启动状态 |
|---|---|---|---|
| `img` | 冷启动：kernel boot → sandbox-init → init 命令 → launch 主进程 | 较慢 | 全新状态 |
| `snp` | 快照恢复：直接解冻已运行的 VM 内存，主进程已在运行 | 较快（跳过 kernel boot 和 init） | 与黄金快照一致 |

**取值来源**：Kind 无 API 字段，由 trigger 请求（`POST /v2/templates/{tid}/builds/{bid}`）的 `start_cmd` 隐式决定——有 `start_cmd` 则走 boot+snapshot 流程产出 `snp`，无则仅 flatten 产出 `img`；bare profile 同理，`flatten-ctl export --upload` 产出 `img`，`sandbox-ctl upload-snapshot` 产出 `snp`。

#### 构建产物类型

构建 API（`POST /v3/templates` + trigger）只产出 e2b profile 的模板：

| 产物 | 触发条件 | 说明 |
|---|---|---|
| `e2b-img-<key>` | 无 `start_cmd` | 只做镜像压平（flatten），冷启动用 |
| `e2b-snp-<key>` | 有 `start_cmd` | 额外做 boot+snapshot，快照恢复用 |

bare profile 无构建 API 入口，`bare-img` / `bare-snp` 只能通过命令行工具直接写入 manifest store：
- `bare-img`：`flatten-ctl export --upload`（EROFS rootfs → manifest store）
- `bare-snp`：`sandbox-ctl upload-snapshot`（本地快照 bundle → manifest store）

#### 构建流水线与输入类型

构建 API（`POST /v3/templates` + trigger）的三阶段流水线（A import → B steps → C template）由 `TriggerSpec` 的输入字段驱动。

**基础输入（`fromImage` / `fromTemplate` 二选一）**

| 字段 | 触发 Phase | 说明 |
|---|---|---|
| `fromImage`（OCI 镜像 URI） | A | 在空白 builder sandbox 内，`flatten-ctl export` 从 registry 拉取并展平为应用 EROFS；未提供时从 `builder_image_uri_mask` 推导（e2b CLI 预推送镜像到此地址） |
| `fromTemplate`（img 模板 key） | 跳过 A | 直接以该模板的 `manifest://<key>` 作为 Phase B/C 的 `boot.root.base` |
| `fromTemplate`（snp 模板 key） | 跳过 A | 从快照的 `snapshot.cfg` 读取 `base_ref`（应用 EROFS key）、`overlay.base`（积累的 overlay diff chain），并继承 `metadata["e2b.start_cmd"]` / `["e2b.ready_cmd"]`（可被 trigger 的 `startCmd` 覆盖） |

**可选叠加：steps（Phase B）**

只要 `steps` 非空，流水线在 e2b sandbox（挂载上述 base）内顺序执行以下指令类型：

| Step 类型 | 执行方式 | 作用 |
|---|---|---|
| `RUN` | envd exec（guest shell） | 修改 rootfs 文件系统 |
| `COPY` / `ADD` | host 侧从 S3 拉取 context tar → sandbox-ctl exec → flatten-ctl tar extract（guest 内） | 注入文件到 guest rootfs |
| `ENV` / `ARG` / `WORKDIR` / `USER` | host 侧上下文变量，不直接修改 rootfs | 影响后续 RUN 的执行环境和最终 OCI runtime-config |

Phase B 结束后 `flatten-ctl export --skip-mounts` 重新导出完整 rootfs，base + steps 合并为单一 EROFS。

**产物类型由 `startCmd` 决定**

```
startCmd 非空 → Phase C（生产 sandbox 冷启动 → 快照） → 产出 snp 模板
startCmd 为空 → 跳过 Phase C，直接上传 image.img  → 产出 img 模板
```

`fromTemplate(snp)` 会从 `snapshot.cfg.metadata` 继承 `start_cmd`，trigger 不带 `startCmd` 时也自动走 Phase C 生成快照。`fromTemplate(snp)` 且无 steps 且无 `startCmd` 是无效请求（nothing to do）。

**构建输入与产物组合汇总**

| 基础输入 | steps | startCmd | 产物 |
|---|---|---|---|
| `fromImage` | — | 无 | `e2b-img-<key>` |
| `fromImage` | 有 | 无 | `e2b-img-<key>`（steps 已合入 EROFS） |
| `fromImage` | 有/无 | 有 | `e2b-snp-<key>` |
| `fromTemplate(img)` | 有 | 有/无 | `e2b-img` 或 `e2b-snp` |
| `fromTemplate(snp)` | 有/无 | 继承/覆盖 | `e2b-snp-<key>`（必然，继承了 startCmd） |

#### img 与 snp 的 Ingest 差异

两者使用**完全相同**的 `ingest.Ingester` 管线（FastCDC 分块 + 收敛加密 + 相同 generation salt + customer key），差别仅在于喂给 ingester 的 `sparse.Source`。

**img：单次 ingest，一个 source**

```
manifest-ctl store image.img
└─ sparse.Source = fetch.OpenTarStream("image.img")  // 整个 EROFS 镜像 tarstream
```

一次 `ing.Ingest()` 调用，产出一个 manifest key，即 PersistID 中的 64hex。

**snp：多次 ingest，分两阶段**

```
sandbox-ctl upload-snapshot <bundle>
```

*阶段一：子构件各自独立 ingest*

| 子构件 | 说明 | 产出 |
|--------|------|------|
| `boot.root.BaseRef`（EROFS base 镜像） | `sha256` 摘要校验后 `ing.Ingest(OpenTarStream(path))` | 独立 manifest key，写回 snapshot.cfg |
| `boot.root.Base`（root top layer） | overlay diff，同上 | 独立 manifest key |
| `boot.root.Overlay.Base`（root overlay） | 同上 | 独立 manifest key |
| `boot.disks[i].BaseRef / Base / Overlay.Base` | 附加磁盘镜像，同上 | 独立 manifest key |

下层（`base_from_refs`）必须已是 `manifest://`，否则直接报错（local-layer invariant）；下层只验证存在性和 key 一致性，不重新 ingest。

*阶段二：`[memory][ZIP]` 复合 source 一次 ingest*

```
memZipSource（实现 sparse.Source）:
  [0,        memSize)  ← bundle 内存 dump（稀疏，RunAt 透传 bundle.RunAt，Hole 直接记入 manifest）
  [memSize,  total)    ← 重新渲染的 ZIP(config.json, state.json, snapshot.cfg)
                          snapshot.cfg 中已将 file:// 替换为阶段一产出的 manifest:// key
```

该复合 source 经一次 `ing.Ingest()` 产出 PersistID 的 64hex（`e2b-snp-<key>`）。

**差异对比**

| | `img` | `snp` |
|---|---|---|
| **ingest 次数** | 1 | 1（memory+ZIP）+ N（子构件各 1） |
| **PersistID 直接覆盖的内容** | 整个 EROFS 镜像 | `[memory dump][ZIP 元数据]` 复合体 |
| **EROFS 与 PersistID 的关系** | 直接（key 就是 EROFS 内容的 hash） | 间接（EROFS base key 嵌在 ZIP 内的 snapshot.cfg 里） |
| **内存 dump 的 sparse hole** | 无 | `memZipSource.RunAt()` 透传 bundle 的 `RunAt()`，hole 区间直接记入 manifest，不产生 chunk，不占存储 |
| **分块/加密管线** | ← 完全相同的 `ingest.Ingester` → | |

> **实际意义**：`e2b-snp-<key>` 的变化触发条件比 `e2b-img-<key>` 更多——内存状态、snapshot.cfg 中任意子构件 key 的变化（EROFS base、overlay diff、磁盘镜像）均会导致 snp key 改变，即使 EROFS 文件系统内容本身未变（例如以相同镜像用不同 start_cmd 重新快照）。

---

### 2.3.1 REST API(e2b 兼容层)

所有端点在 `api.listen` 上暴露，TLS 可选(dev 模式 h2c)。鉴权：`X-API-KEY: <api_key>` 或 `Authorization: Bearer <api_key>`(两者等价)。未注明必填的字段均为选填。

#### 端点总览

| 方法 | 路径 | 状态码 | 说明 |
|------|------|--------|------|
| `POST`   | `/sandboxes`                              | 201 | 创建沙箱 |
| `GET`    | `/sandboxes/{id}`                         | 200 | 查询单个沙箱 |
| `GET`    | `/v2/sandboxes`                           | 200 | 列出本节点活跃沙箱 |
| `DELETE` | `/sandboxes/{id}`                         | 204 | 销毁沙箱 |
| `POST`   | `/sandboxes/{id}/connect`                 | 200 | 连接沙箱（含自动恢复暂停） |
| `POST`   | `/sandboxes/{id}/pause`                   | 204 | 暂停沙箱 |
| `POST`   | `/sandboxes/{id}/timeout`                 | 204 | 更新沙箱 deadline |
| `POST`   | `/sandboxes/{id}/export`                  | 200 | 快照导出（迁移 token 或晋升模板） |
| `POST`   | `/sandboxes/import`                       | 200 | 快照导入 |
| `POST`   | `/v3/templates`                           | 202 | 注册模板构建任务 |
| `POST`   | `/v2/templates/{tid}/builds/{bid}`        | 202 | 触发三阶段构建流水线 |
| `GET`    | `/templates/{tid}/builds/{bid}/status`   | 200 | 查询构建进度与日志 |
| `GET`    | `/templates/{tid}/files/{hash}`          | 201 | 检查 COPY 上下文；返回预签名 PUT URL |
| `GET`    | `/templates`                              | 200 | 列出当前 manifest-key 下全部已完成模板 |
| `GET`    | `/health`                                 | 204 | 存活探针 |

> **重要**：`POST /sandboxes` 的沙箱资源规格(CPU/内存/网络/挂载/启动命令等)通过 **`metadata`** 字段的命名空间 key 或等价的 **`X-Kuasar-Sandbox-*`** 请求头注入，而非 JSON 顶层字段。Header 优先级高于同名 metadata key。
>
> | Header | metadata key | 值格式(JSON) |
> |--------|-------------|---------------|
> | `X-Kuasar-Sandbox-Resource` | `kuasar-sandbox.resource` | `{"capacity":{"cpu":N,"memory":M}}` |
> | `X-Kuasar-Sandbox-Network`  | `kuasar-sandbox.network`  | `{"hostname":"...","dns":["..."]}` |
> | `X-Kuasar-Sandbox-Launch`   | `kuasar-sandbox.launch`   | `{"start_cmd":"..."}` |
> | `X-Kuasar-Sandbox-Init`     | `kuasar-sandbox.init`     | `[{"exec":"..."}]` |
> | `X-Kuasar-Sandbox-Mounts`   | `kuasar-sandbox.mounts`   | `[{"source":"...","destination":"..."}]` |
> | `X-Kuasar-Sandbox-Files`    | `kuasar-sandbox.files`    | `[{"path":"...","content":"..."}]` |
> | `X-Kuasar-Sandbox-Metadata` | `kuasar-sandbox.metadata` | `{"key":"value"}` |
>
> **各字段的启动场景适用范围**
>
> | metadata key | 冷启动（img 模板） | 快照启动（snp 模板 / resume） | 说明 |
> |---|:---:|:---:|---|
> | `kuasar-sandbox.resource` — `capacity` | ✓ | ✗ | snp restore 以快照内固定的容量为准，创建时写入的 capacity 会被丢弃 |
> | `kuasar-sandbox.resource` — `allocatable` | ✓ | ✓ | 可调度量不受快照约束，两种场景均生效 |
> | `kuasar-sandbox.network` | ✓ | ✓ | 创建时的字段覆盖快照继承的网络字段（`MergeNetwork`，显式值优先） |
> | `kuasar-sandbox.launch` | ✓（仅 bare profile） | ✗ | 主进程定义已固化在快照中，restore 消息不含 Launch 字段 |
> | `kuasar-sandbox.init` | ✓ | ✗ | init 命令在主进程 fork 前执行；快照是 init 完成后的状态，restore 消息不含 Init 字段 |
> | `kuasar-sandbox.mounts` | ✓ | ✗ | guest 侧 mount 由 sandbox-init 在启动前挂载；快照已包含 mount 状态，restore 不重新挂载 |
> | `kuasar-sandbox.files` | ✓ | ✓ | restore 消息携带 Files 字段，在 guest 解冻前重新做 tmpfs+bind 注入；适合注入黄金快照中不含的实例私有 secret / 配置 |
> | `kuasar-sandbox.metadata` | ✓ | ✓ | 透传到 `SANDBOX_CONFIG.metadata`，随快照持久化 |

---

#### POST /sandboxes — 创建沙箱

**请求体**

```json
{
  "templateID": "e2b-snp-<64hex>",   // 必填：模板 ID(profile-kind-key)
  "timeout":    3600,                 // 最大存活秒数；0 或不填 = 使用全局默认
  "envVars": {                        // guest 环境变量(注意字段名：envVars 非 env)；仅冷启动(img 模板)生效，
    "PORT": "3000"                    // snp 模板/resume 恢复快照时进程已存在，env 不重新注入。
  },                                  // 优先级(e2b)：image ENV < envVars
                                      // 优先级(bare)：image ENV < envVars < kuasar-sandbox.launch.env
  "secure": false,                    // true = Secure 模式(MMDS v2 + envd re-key)
  "metadata": {                       // KV 载体；kuasar-sandbox.* 命名空间由平台使用
    "kuasar-sandbox.resource": "{\"capacity\":{\"cpu\":2,\"memory\":512}}",
    "kuasar-sandbox.team":     "team-id"
  }
}
```

> CPU/内存规格、网络策略、挂载、文件注入等均通过 `metadata` 的命名空间 key 或等价 Header 传递(见上方表格)，不在顶层 JSON 字段中出现。

**响应 201 Created**

```json
{
  "sandboxID":          "01934b7a-1234-7xxx-xxxx-xxxxxxxxxxxx", // 沙箱 ID（uuidv7），后续所有沙箱操作的路径参数 {id}
  "templateID":         "e2b-snp-<64hex>",  // Persist ID；请求传入 transient/alias 时，服务端已解析为此形式
  "clientID":           "orchestrator",      // 固定值
  "domain":             "sandbox.example.com", // 数据面域名；envd 访问地址为 49983-{sandboxID}.{domain}
  "envdVersion":        "0.6.1",            // e2b profile 固定 "0.6.1"；bare profile 固定 "0.1.0"（占位）
  "envdAccessToken":    "<token>",          // envd Connect RPC 的 bearer token；Secure 模式下 guest 内 envd 会校验此值
  "trafficAccessToken": "<token>",          // node-proxy 校验此 token 后才转发数据面流量；SDK 在请求头中携带
  "alias":              ""                  // 固定为空字符串（当前版本未实现沙箱别名）
}
```

> 响应不含 `state`、`startedAt`、`endAt`、`metadata` 字段（仅 `GET /sandboxes/{id}` 返回完整详情）。沙箱创建完成时已处于 `running` 状态，envd 就绪后才返回 201，无需客户端额外轮询。

**错误响应**

| HTTP | message 字段 | 触发条件 |
|------|-------------|---------|
| 400 | `bad body` / 验证错误 | 请求体格式错误或 templateID 不合法 |
| 401 | `unauthorized` | api_key 格式无效 |
| 403 | `manifest key not allowed` | manifest_key 不在 allowlist |
| 500 | `internal error` | 其他错误 |

---

#### GET /v2/sandboxes — 列出本节点活跃沙箱

**查询参数**

| 参数 | 说明 |
|------|------|
| `state` | 过滤状态；`running`、`paused`，不填返回全部（dead 状态的沙箱在销毁时从 DB 删除，不会出现） |
| `limit` | 每页最大条数，默认 100 |
| `nextToken` | 分页游标；首次请求不传，后续传上一响应头 `x-next-token` 的值 |

**响应 200**：JSON 数组（非对象包装）。下一页游标在响应头 `x-next-token` 中返回；值为空表示无更多数据。

```json
[
  {
    "sandboxID":   "01934b7a-...",             // 沙箱 ID（uuidv7）
    "templateID":  "e2b-snp-<64hex>",          // Persist ID
    "clientID":    "orchestrator",             // 固定值
    "state":       "running",                  // running | paused（列表不含 dead）
    "cpuCount":    2,                          // 节点级统一配置（api.Resources.VCPU），非沙箱实际申请值，所有沙箱相同
    "memoryMB":    512,                        // 节点级统一配置（api.Resources.MemoryMB），同上
    "diskSizeMB":  1024,                       // 节点级统一配置（api.Resources.DiskMB），同上
    "envdVersion": "0.6.1",                    // e2b profile 固定 "0.6.1"；bare profile 固定 "0.1.0"
    "startedAt":   "2026-06-24T10:00:00Z",     // 沙箱创建时间，ISO-8601（注意：与 GET /sandboxes/{id} 不同，此处是字符串而非 Unix 整数）
    "endAt":       "2026-06-24T11:00:00Z",     // deadline ISO-8601；无 deadline 时等于 startedAt（不为零值，SDK 要求合法时间戳）
    "metadata":    {}                          // 创建时传入的 metadata
  }
]
```

---

#### GET /sandboxes/{id} — 查询单个沙箱

**响应 200**

```json
{
  "sandboxID":          "01934b7a-...",      // 沙箱 ID（uuidv7），创建时分配
  "templateID":         "e2b-snp-<64hex>",  // Persist ID；创建时已从 transient/alias 解析为此形式
  "clientID":           "orchestrator",      // 固定值，标识响应来源为 node-ctl orchestrator
  "domain":             "sandbox.example.com", // 数据面域名，来自节点配置 api.domain
  "envdVersion":        "0.6.1",            // e2b profile 固定 "0.6.1"；bare profile 固定 "0.1.0"（占位，不代表实际版本）
  "envdAccessToken":    "<token>",          // envd API 鉴权令牌；bearer token，用于访问 49983 端口的 Connect RPC
  "trafficAccessToken": "<token>",          // 数据面代理鉴权令牌；node-proxy 校验此 token 后才转发流量
  "alias":              "",                 // 固定为空字符串（当前版本未实现沙箱别名）
  "state":              "running",          // running | paused | dead
  "startedAt":          1719187200,         // 沙箱创建时间，Unix 整数（秒）；注意与 GET /v2/sandboxes 不同，此处不是 ISO 字符串
  "endAt":              1719273600,         // deadline Unix 整数（秒）；0 = 无 deadline
  "metadata":           {}                  // 创建时传入的 metadata（含 kuasar-sandbox.* 命名空间 key）
}
```

---

#### POST /sandboxes/{id}/connect — 连接(含自动恢复暂停沙箱)

**请求体**

```json
{ "timeout": 1800 }   // 重设 deadline 秒数；0 = 保留原 deadline
```

可选 `X-Kuasar-Migration-Token: <token>` 请求头：若 id 不在本节点，则先从 token 自动导入快照再连接（等同 import + connect 原子操作）。

**响应 200**

```json
{
  "sandboxID":          "01934b7a-...",      // 同请求路径中的 {id}
  "templateID":         "e2b-snp-<64hex>",  // Persist ID
  "clientID":           "orchestrator",      // 固定值
  "domain":             "sandbox.example.com",
  "envdVersion":        "0.6.1",
  "envdAccessToken":    "<token>",           // 每次 connect（含 resume）均重新生成；客户端应以此次响应的值为准
  "trafficAccessToken": "<token>",           // 同上，每次 connect 重新生成
  "alias":              ""
}
```

> 响应结构与 `POST /sandboxes` 201 响应完全相同（均来自 `sandboxResp()`），但 `envdAccessToken` 和 `trafficAccessToken` 在每次 connect 时重新 mint——paused 沙箱 resume 后旧 token 失效，客户端须使用本次响应中的新 token。

---

#### POST /sandboxes/{id}/pause — 暂停

**路径参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `id` | string | 沙箱 ID（sandboxID） |

无请求体。

**行为**

1. 查询 sandbox 并验证 `apiKey` 所有权（`ownsSandbox`），不匹配时等同不存在。
2. 若 `state == paused` → 立即返回 409，不重复快照。
3. 调用 `o.snapshot()` 将 guest 内存写入本地 checkpoint 目录（`checkpoint.local_dir`）。
4. 原子更新：`state = paused`，`snapshot_ref = manifest://<key>`（或本地路径）。
5. **Cluster 模式**（`cluster.registry` 非空且 `checkpoint.deep_idle_sec > 0`）：将 `DeadlineUnix` 重置为 `now + deep_idle_sec`，启动深度闲置倒计时；到期后 reaper 自动将快照晋升为 SAVED 远端 manifest。

**响应**

| 状态码 | 含义 |
|--------|------|
| `204 No Content` | 快照已落盘，沙箱进入 paused 状态，无响应体 |
| `404 Not Found` | 沙箱不存在，或 apiKey 与创建时的 ManifestKey 不匹配 |
| `409 Conflict` | 已是 paused 状态（`ErrAlreadyPaused`），无需重试 |
| `500` | 快照失败（磁盘满、cloud-hypervisor 错误等） |

> **注**：pause 是同步操作——handler 等待 `snapshot()` 返回后才写 204，调用方可直接视 204 为快照完成。

---

#### POST /sandboxes/{id}/timeout — 更新 deadline

**路径参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `id` | string | 沙箱 ID（sandboxID） |

**请求体**

```json
{ "timeout": 1800 }
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `timeout` | int | 从**当前时刻**起的相对秒数（非绝对时间戳）；`DeadlineUnix = time.Now() + timeout` |

> `timeout = 0` 将 deadline 设为当前时刻，reaper 在下一轮扫描（≤5 s）时自动触发 pause。

**行为**

- 计算新 deadline：`DeadlineUnix = time.Now().Unix() + timeoutSec`。
- 写入 SQLite：`o.st.SetDeadline(ctx, id, DeadlineUnix)`。
- apiKey 不匹配时 `ok = false`，返回 404（与沙箱不存在的错误行为一致）。

**响应**

| 状态码 | 含义 |
|--------|------|
| `204 No Content` | Deadline 更新成功，无响应体 |
| `404 Not Found` | 沙箱不存在，或 apiKey 不匹配 |
| `500` | 存储写入失败 |

---

#### DELETE /sandboxes/{id} — 销毁沙箱

**路径参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `id` | string | 沙箱 ID（sandboxID） |

无请求体，无查询参数。

**行为**

1. 验证 `apiKey` 所有权（`ownsSandbox`），不匹配时等同不存在（返回 404）。
2. **teardown**（对 running / paused 状态均执行）：
   - `lc.Stop()` → 停止 systemd runner 单元；单元为 `KillMode=control-group`，SIGKILL 覆盖整个 cgroup（cloud-hypervisor 及其子进程一并终止），无需额外等待。
   - `lc.ResetFailed()` → 清除 systemd 失败标记。
   - `vs.Detach()` → 从 vswitch 摘除网络端口，路由立即失效。
   - `os.RemoveAll(sb.RunDir)` → 删除 tmpfs 运行目录（套接字、设备文件等）。
3. `st.Delete()` → 从 SQLite 中删除沙箱记录（记录消失，`GET /sandboxes/{id}` 随即返回 404）。
4. `publishDelete()` → 向 node-proxy 广播路由删除，外部流量立即断连。

> **注**：DELETE 是强制终止（等价 SIGKILL），无优雅关机宽限期；paused 状态沙箱同样可以直接 DELETE，teardown 中 Stop 为空操作。

**响应**

| 状态码 | 含义 |
|--------|------|
| `204 No Content` | 沙箱已销毁，无响应体 |
| `404 Not Found` | 沙箱不存在，或 apiKey 不匹配 |
| `500` | 存储操作失败 |

---

#### POST /sandboxes/{id}/export — 快照导出

沙箱必须处于 `paused` 状态且已有远程快照（`SnapshotRef` 非空）。若快照仍是本地文件，导出时自动晋升为 manifest store 远程引用。

**请求体**

```json
{
  "toTemplate": false,   // true = 晋升为可复用模板（fork）；false = 生成单次迁移 token（move/copy）
  "keepSource": false    // true = 导出后保留源沙箱行（copy 语义）；false = 删除源沙箱行（move 语义，默认）
}
```

**响应 200**

```json
{ "result": "<value>" }
```

`result` 的值取决于 `toTemplate`：

| `toTemplate` | `result` 内容 | 用途 |
|---|---|---|
| `false`（默认） | base64 编码的迁移 token（opaque 字符串） | 传给目标节点的 `POST /sandboxes/import`，或作为 `X-Kuasar-Migration-Token` header 触发 connect 时自动导入 |
| `true` | Persist ID（`<profile>-snp-<64hex>`） | 可直接用于 `POST /sandboxes templateID` 字段，等同一个只读快照模板 |

**错误响应**

| HTTP | 触发条件 |
|------|---------|
| 400 | 沙箱未处于 paused 状态 / 无快照 / 快照晋升失败 |
| 403 | api_key 不属于该沙箱的租户 |
| 404 | 沙箱不存在 |

---

#### POST /sandboxes/import — 快照导入

将迁移 token 重建为本节点上的 `paused` 沙箱行。导入成功后需调用 `POST /sandboxes/{id}/connect` 恢复运行。

**请求体**

```json
{ "token": "<migration-token>" }   // 必填；来自 export 响应的 result 字段（toTemplate=false 时）
```

**响应 200**

```json
{
  "sandboxID": "01934b7a-..."   // 与原沙箱 ID 相同（从 token 中还原），非新分配；此端点不返回完整沙箱对象
}
```

**错误响应**

| HTTP | 触发条件 |
|------|---------|
| 400 | token 格式错误 / id 或 snapshot_ref 缺失 / 该沙箱 ID 已存在于本节点 / 节点 guest runtime 与快照不匹配（`runtime_digest` 校验失败） |
| 403 | token 的 manifest_key 指纹与本节点租户不符 / 租户 key 未在本节点注册 |

---

#### 规划中端点(❌ 未实现)

| 端点 | 说明 | FE |
|------|------|-----|
| `GET /sandboxes/{id}/metrics` | per-sandbox cgroup 实时指标 | FE-9 |
| `GET /sandboxes/{id}/logs` | 应用日志流（v1 简单分页） | FE-9 |
| `GET /v2/sandboxes/{id}/logs` | 应用日志流（v2 cursor + level + search） | FE-9 |
| `PUT /sandboxes/{id}/storage/credentials` | 运行时刷新 S3/OBS 对象存储凭证 | FE-6.2 |

---

#### GET /sandboxes/{id}/logs — 应用日志（v1）❌ 未实现

> FE-9.1。日志来源为 guest 内 envd 采集的结构化 journald 流；底层实现读取 `journald` tag=sandbox 的条目。

**路径参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `id` | string | 沙箱 ID |

**查询参数**

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `start` | int64 | — | 起始时间戳（**毫秒** Unix），只返回 `timestamp ≥ start` 的条目；缺省从最早记录开始 |
| `limit` | int32 | — | 返回条目上限；缺省由服务端决定 |

**响应 200**

```json
{
  "logs": [
    { "line": "server started on :8080", "timestamp": "2026-06-27T10:00:00Z" }
  ],
  "logEntries": [
    {
      "level": "info",
      "message": "server started on :8080",
      "timestamp": "2026-06-27T10:00:00Z",
      "fields": { "pid": "42" }
    }
  ]
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `logs` | array | 非结构化日志列表；每项含 `line`（原始文本）和 `timestamp`（ISO-8601） |
| `logEntries` | array | 结构化日志列表；每项含 `level`、`message`、`timestamp`、`fields`（任意 key-value） |
| `logEntries[].level` | string | `debug \| info \| warn \| error` |

> `logs` 与 `logEntries` 条目一一对应，内容相同；SDK 要求同时返回两种形式以向后兼容。

---

#### GET /v2/sandboxes/{id}/logs — 应用日志（v2）❌ 未实现

> FE-9.1。在 v1 基础上增加翻页方向、级别过滤和全文搜索，满足 US-9.1-2。

**路径参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `id` | string | 沙箱 ID |

**查询参数**

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `cursor` | int64 | — | 游标时间戳（**毫秒** Unix）；`forward` 方向：返回 `timestamp > cursor` 的条目；`backward` 方向：返回 `timestamp < cursor` 的条目 |
| `limit` | int32 | — | 返回条目上限 |
| `direction` | string | `forward` | `forward`（时间正序，向后翻页）或 `backward`（时间倒序，向前翻页） |
| `level` | string | — | 最低日志级别过滤：`debug \| info \| warn \| error`；低于该级别的条目被排除 |
| `search` | string | — | 大小写敏感的子字符串匹配，作用于 `message` 字段 |

**响应 200**

```json
{
  "logs": [
    {
      "level": "error",
      "message": "connection timeout",
      "timestamp": "2026-06-27T10:05:00Z",
      "fields": { "remote": "10.0.0.1:443" }
    }
  ]
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `logs` | array | 结构化日志列表（同 v1 `logEntries` 结构） |
| `logs[].level` | string | `debug \| info \| warn \| error` |
| `logs[].message` | string | 日志消息正文 |
| `logs[].timestamp` | string | ISO-8601 时间戳 |
| `logs[].fields` | object | 任意扩展字段（key: string → value: string） |

> 翻页示例：取最新 50 条 → `?direction=backward&limit=50`；继续向前 → `?direction=backward&limit=50&cursor=<最旧条目的 timestamp ms>`。

**错误码**

| 状态码 | 含义 |
|--------|------|
| `404` | 沙箱不存在或 apiKey 不匹配 |
| `500` | 日志读取失败（journald 不可达等） |

---

#### GET /sandboxes/{id}/metrics — 运行时指标 ❌ 未实现

> FE-9.1。直接读取宿主 cgroup v2 文件系统（`/sys/fs/cgroup/sandboxes/<sid>/`），延迟 < 1 s（US-9.1-1）。

**路径参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `id` | string | 沙箱 ID |

**查询参数**

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `start` | int64 | — | 区间起始时间戳（**Unix 秒**）；缺省返回当前单点快照 |
| `end` | int64 | — | 区间结束时间戳（**Unix 秒**）；与 `start` 共同指定历史区间 |

> 不传 `start`/`end` 时，服务端采集一次当前值并以单元素数组返回。

**响应 200**

```json
[
  {
    "cpuCount": 2,
    "cpuUsedPct": 12.5,
    "memTotal": 536870912,
    "memUsed":  104857600,
    "memCache":  20971520,
    "diskTotal": 10737418240,
    "diskUsed":   2147483648,
    "timestampUnix": 1751020800,
    "timestamp": "2026-06-27T10:00:00Z"
  }
]
```

响应为 `[]SandboxMetric` 数组（区间查询时按时间升序排列）。

| 字段 | 类型 | cgroup v2 来源 | 说明 |
|------|------|----------------|------|
| `cpuCount` | int32 | `cpuset.cpus.effective` 核数 | 分配给该沙箱的 vCPU 数量 |
| `cpuUsedPct` | float32 | `cpu.stat usage_usec` 差值 / 采样间隔 | CPU 使用率百分比（0–100×cpuCount） |
| `memTotal` | int64 | `memory.max`（字节） | 分配的内存上限 |
| `memUsed` | int64 | `memory.current`（字节） | 当前实际用量（含页缓存） |
| `memCache` | int64 | `memory.stat file`（字节） | 页缓存部分（`memUsed - memCache` ≈ RSS） |
| `diskTotal` | int64 | overlay 挂载点 `statvfs.f_blocks × bsize`（字节） | 沙箱磁盘总容量 |
| `diskUsed` | int64 | overlay 挂载点 `(f_blocks - f_bfree) × bsize`（字节） | 沙箱磁盘已用量 |
| `timestampUnix` | int64 | 采样时刻 Unix 秒 | 首选字段 |
| `timestamp` | string | 同上，ISO-8601 | 已标记废弃，兼容保留 |

> **OOM 计数**（`memory.events oom`）对应 US-9.1-1 中的 `oom_count`，计划以扩展字段形式加入，当前接口版本不包含。

**错误码**

| 状态码 | 含义 |
|--------|------|
| `404` | 沙箱不存在或 apiKey 不匹配 |
| `500` | cgroup 文件读取失败（沙箱已终止、cgroup 已回收等） |

---

#### 模板构建端点(e2b v2/v3)

| 方法 | 路径 | 状态码 | 说明 |
|------|------|--------|------|
| POST | `/v3/templates` | 202 | 注册模板构建任务（分配 templateID/buildID） |
| POST | `/v2/templates/{tid}/builds/{bid}` | 202 | 触发三阶段构建流水线 |
| GET  | `/templates/{tid}/builds/{bid}/status` | 200 | 查询构建进度与日志 |
| GET  | `/templates/{tid}/files/{hash}` | 201 | 检查 COPY 上下文是否已上传；返回预签名 PUT URL |
| GET  | `/templates` | 200 | 列出当前 manifest-key 下全部已完成模板 |

---

##### POST /v3/templates — 注册模板构建任务

预先分配 `templateID`（临时 handle）和 `buildID`，记录 template 名称、标签与资源规格。随后调用 trigger 端点启动实际构建。

**请求体**（所有字段均可选）

```json
{
  "name":      "my-template",    // 模板名称；构建完成后写入 names[]，可通过名称在 POST /sandboxes 中引用
  "tags":      ["v1", "latest"], // 别名列表；构建完成后写入 aliases[]，同样可在 POST /sandboxes 中引用
  "cpuCount":  2,                // 或 cpu_count（snake_case 别名）；模板默认 CPU 核数
  "memoryMB":  512               // 或 memory_mb（snake_case 别名）；模板默认内存 MB
}
```

**请求头**（均可选，与 trigger 端点共享相同语义）

| Header | 命名空间 key（存入 `Build.Metadata`） | 说明 |
|--------|--------------------------------------|------|
| `X-Kuasar-Sandbox-Resource` | `kuasar-sandbox.resource` | 完整资源规格 JSON（含 `capacity` 和 `allocatable`）；与请求体 `cpuCount`/`memoryMB` 共存时，**请求体字段优先** |
| `X-Kuasar-Sandbox-Network` | `kuasar-sandbox.network` | 模板默认网络配置（`hostname`、`dns`）；`inner_ip`/`transit_*` 为节点管理字段，模板层设置无效 |
| `X-Kuasar-Sandbox-Mounts` | `kuasar-sandbox.mounts` | guest 内额外挂载列表（bind / tmpfs 等），直接透传为 `SANDBOX_CONFIG.mounts` |
| `X-Kuasar-Sandbox-Files` | `kuasar-sandbox.files` | guest 内虚拟文件（tmpfs+bind 注入），如证书、配置文件 |
| `X-Kuasar-Sandbox-Init` | `kuasar-sandbox.init` | guest 内 sidecar 进程列表，与主进程并行启动 |
| `X-Kuasar-Sandbox-Launch` | `kuasar-sandbox.launch` | 主进程配置（**仅 bare profile**；e2b profile 拒绝） |
| `X-Kuasar-Sandbox-Metadata` | `kuasar-sandbox.metadata` | 任意透传 key-value，随快照迁移 |

> - `cpuCount`/`memoryMB` 写入 `kuasar-sandbox.resource` capacity，存储在 `builds.metadata_json`，作为沙箱的默认规格
> - 请求体字段全部省略时仍会分配 `templateID` 和 `buildID`，资源规格可在 trigger 时通过请求体或 header 补充（trigger 的同名 namespace key 以整体覆盖 register 时的值）
> - 此处设置的所有 namespace key 在 `POST /sandboxes` 时作为默认值，create 请求的同名 key 整体覆盖模板值

**响应 202**

```json
{
  "templateID": "transient-<uuidv7>",   // 瞬态 ID，仅用于后续 {tid} 路径参数；构建完成前不可用于 POST /sandboxes
  "buildID":    "<build-uuidv7>",       // 构建任务 ID；用于 {bid} 路径参数，以及在 GET /templates 列表中关联 Persist ID
  "public":     false,                  // 固定为 false（当前版本不支持公开模板）
  "names":      [],                     // 注册时为空；构建完成后写入 name 字段值（若有）
  "tags":       [],                     // 注册时为空；构建完成后写入 tags 字段值（若有）
  "aliases":    []                      // 注册时为空；构建完成后与 tags 相同（两者实际指向同一数组）
}
```

---

##### POST /v2/templates/{tid}/builds/{bid} — 触发构建流水线

传入基础镜像/模板、构建步骤和启动命令，驱动三阶段流水线（A import → B steps → C template）。`tid` / `bid` 来自 register 响应；构建异步执行，通过 `/status` 轮询进度。

**请求体**

```json
{
  "fromImage":    "ghcr.io/owner/repo:tag",  // OCI 镜像 URI；与 fromTemplate 互斥
  "fromTemplate": "e2b-snp-<64hex>",         // 基础模板 persist id；与 fromImage 互斥
  "fromImageRegistry": {                     // fromImage 的 registry 凭证（可选）
    "username": "user",
    "password": "pass"
  },
  "steps": [                                 // 构建步骤列表（Phase B），可选
    { "type": "RUN",  "args": ["pip install flask"] },
    { "type": "COPY", "args": ["src/", "/app/"], "filesHash": "<sha256>" },
    { "type": "ENV",  "args": ["PORT=3000"] },
    { "type": "ARG",  "args": ["VERSION=1.0"] },
    { "type": "WORKDIR", "args": ["/app"] },
    { "type": "USER",    "args": ["nobody"] }
  ],
  "startCmd":  "python /app/main.py",              // 快照构建触发命令（有值则走 Phase C 生成 snp 模板）
  "start_cmd": "python /app/main.py",              // snake_case 别名，与 startCmd 等价
  "readyCmd":  "curl -sf http://localhost:3000/",  // 就绪探针（Phase C 轮询间隔 2s）
  "ready_cmd": "...",                              // snake_case 别名
  "cpuCount":  2,    // 或 cpu_count；追加 / 覆盖 register 时的资源规格
  "memoryMB":  512   // 或 memory_mb
}
```

> **COPY step 前置条件**：`filesHash` 对应的 context tar 必须已通过 `/templates/{tid}/files/{hash}` 端点上传至 `builder.files_storage`，否则返回 400；`files_storage` 未配置时返回 501。

**请求头**

| Header | 命名空间 key（存入 `Build.Metadata`） | 说明 |
|--------|--------------------------------------|------|
| `X-Kuasar-Pull-Token` | — | 平台 registry pull token，优先级高于 `fromImageRegistry` 字段 |
| `X-Kuasar-Sandbox-Resource` | `kuasar-sandbox.resource` | 资源规格 JSON；与请求体 `cpuCount`/`memoryMB` 共存时，**请求体字段优先**（`SetCapacity` 覆盖 header 的 capacity） |
| `X-Kuasar-Sandbox-Network` | `kuasar-sandbox.network` | 模板默认网络配置（`hostname`、`dns`）；`inner_ip`/`transit_*` 为节点管理字段，模板层设置无效 |
| `X-Kuasar-Sandbox-Mounts` | `kuasar-sandbox.mounts` | guest 内额外挂载列表（bind / tmpfs 等），直接透传为 `SANDBOX_CONFIG.mounts` |
| `X-Kuasar-Sandbox-Files` | `kuasar-sandbox.files` | guest 内虚拟文件（tmpfs+bind 注入），如证书、配置文件 |
| `X-Kuasar-Sandbox-Init` | `kuasar-sandbox.init` | guest 内 sidecar 进程列表，与主进程并行启动 |
| `X-Kuasar-Sandbox-Launch` | `kuasar-sandbox.launch` | 主进程配置（**仅 bare profile**；e2b profile 拒绝，envd 固定接管） |
| `X-Kuasar-Sandbox-Metadata` | `kuasar-sandbox.metadata` | 任意透传 key-value，随快照迁移 |

trigger 时传入的 header 以**命名空间 key 为粒度覆盖** register 时的同名 key（整个 namespace value 替换，无字段级深度合并）。

**资源规格说明**

```
X-Kuasar-Sandbox-Resource: {"capacity":{"cpu":4,"memory":"2048MiB"},"allocatable":{"cpu":3.5,"memory":"1800MiB"}}
```

`capacity` 设置 VM 实际分配规格；`allocatable` 设置 cgroup 限额（可低于 capacity，用于超售）。

> **snp 模板约束**：`cpuCount`/`memoryMB`/`X-Kuasar-Sandbox-Resource` 对 `snp` 模板只影响 Phase C 黄金快照的构建规格。快照一旦生成，内存镜像中的规格即固定，runtime 拒绝不匹配的 capacity，orchestrator 在渲染 SANDBOX_CONFIG 时静默丢弃 snp 的 capacity 覆盖。如需修改 snp 的规格，必须重建模板。

**挂载示例**

```
X-Kuasar-Sandbox-Mounts: [
  {"type":"tmpfs","destination":"/tmp/fast","options":["size=256m"]},
  {"type":"bind","source":"/host/data","destination":"/data","options":["ro"]}
]
```

**配置优先级（创建沙箱时）**

```
Build.Metadata（模板默认值，register + trigger 层叠）
    ↓  MergeMetadata，以 namespace key 为粒度，create 覆盖模板
POST /sandboxes { metadata } + X-Kuasar-Sandbox-* headers（create-time 覆盖）
```

**响应**：`202 No Content`（空体）；构建异步在 `sandbox-builder@<bid>.service` 中执行。

**错误响应**

| HTTP | 触发条件 |
|------|---------|
| 400 | `fromImage` 与 `fromTemplate` 同时提供 / COPY context 未上传 / `fromTemplate` 无 steps 且无 startCmd |
| 404 | `tid` 或 `bid` 不存在，或不属于当前 api_key |
| 501 | 使用了 COPY step 但 `builder.files_storage` 未配置 |

---

##### GET /templates/{tid}/builds/{bid}/status — 查询构建进度

**路径参数**

| 参数 | 说明 |
|------|------|
| `{tid}` | 瞬态 ID（`transient-<uuidv7>`），来自 `POST /v3/templates` 响应的 `templateID`；传 Persist ID 返回 404 |
| `{bid}` | 构建 ID，来自 `POST /v3/templates` 响应的 `buildID` |

**查询参数**

| 参数 | 说明 |
|------|------|
| `logsOffset` | 已读取的日志行数（整数）；首次传 `0`，后续传上一次响应 `logEntries` 的累计长度；服务端返回从该偏移起的新增条目 |

**响应 200**

```json
{
  "templateID": "transient-<uuidv7>",              // 构建中为瞬态 ID；status=ready 后切换为 Persist ID（<profile>-<kind>-<64hex>）
  "buildID":    "<build-uuidv7>",                  // 与注册时返回的 buildID 相同
  "status":     "building",                        // registered | building | ready | error
  "logs":       ["import: pulling...", "..."],      // 自 logsOffset 起的新增日志文本行；与 logEntries[].message 同步
  "logEntries": [                                  // 结构化日志条目；与 logs 内容一一对应
    {
      "timestamp": "2026-06-24T10:00:00.000Z",     // RFC3339Nano UTC
      "level":     "info",                         // debug | info | warn | error
      "message":   "import: pulling..."
    }
  ],
  "reason":  { "message": "error detail..." },     // 仅 status=error 时出现
  "names":   ["my-template"],                      // 仅 status=ready 时出现；注册时 name 字段的值
  "aliases": ["my-template"]                       // 仅 status=ready 时出现；注册时 tags 字段的值
}
```

**`status` 状态流转**

```
registered  →  building  →  ready
                         ↘  error
```

| 值 | 含义 |
|---|---|
| `registered` | 已通过 `POST /v3/templates` 登记，尚未调用 trigger 端点 |
| `building` | 三阶段流水线执行中（`sandbox-builder@<bid>.service` 运行中） |
| `ready` | 构建完成；`templateID` 切换为 Persist ID，可用于 `POST /sandboxes` |
| `error` | 流水线失败；`reason.message` 包含错误详情 |

**`logsOffset` 增量拉取示例**

```
首次请求：GET …/status?logsOffset=0
  → logEntries 共 5 条，累计 offset = 5

下次请求：GET …/status?logsOffset=5
  → 返回第 6 条起的新增条目
```

---

##### GET /templates/{tid}/files/{hash} — 检查 COPY 上下文

用于 COPY step 的 context tar 预上传流程：客户端在 trigger 构建前调用此端点，检查 context 是否已存在；若不存在则向返回的预签名 PUT URL 上传，再调用 trigger。构建时 orchestrator 用 `filesHash` 生成预签名 GET URL 供 build sandbox 拉取。

**路径参数**

| 参数 | 说明 |
|------|------|
| `{tid}` | 瞬态 ID（`transient-<uuidv7>`），来自 `POST /v3/templates` 响应的 `templateID`；同时作为 S3 对象路径的一部分，确保 context 与当前构建绑定 |
| `{hash}` | COPY context tar 的内容哈希（SHA256 hex），与 `steps[].filesHash` 字段值相同；服务端以此作为 S3 对象 key 的一部分，实现内容寻址去重 |

**响应 201**

```json
{
  "present": false,                           // true = context 已在存储中，可跳过上传；false = 需要 PUT
  "url":     "https://obs.../presigned-put-url" // 预签名 PUT URL；present=true 或 false 时均返回，有效期由 files_storage.presign_expiry 配置
}
```

**客户端上传流程**

```
GET /templates/{tid}/files/{hash}
  → present=false → PUT <url>（直接上传 context tar 原始字节，无需特定 Content-Type）
  → present=true  → 跳过上传

POST /v2/templates/{tid}/builds/{bid}   ← 所有 COPY step 的 context 上传完成后再 trigger
  steps[].filesHash = {hash}            ← trigger 前校验：hash 未上传则返回 400
```

**错误响应**

| HTTP | 触发条件 |
|------|---------|
| 404 | `tid` 不存在或不属于当前 api_key |
| 501 | `builder.files_storage` 未配置 |

---

##### GET /templates — 列出已完成模板

返回当前 manifest-key 下所有 `status=ready` 的模板（`buildStatus` 使用 SDK 字段名）。只返回已完成的模板，构建中的条目不出现在此列表中。

**响应 200**

```json
[
  {
    "templateID":  "e2b-snp-<64hex>",   // Persist ID；永远是 <profile>-<kind>-<64hex> 形式，可直接用于 POST /sandboxes
    "buildID":     "<build-uuidv7>",     // 与 POST /v3/templates 返回的 buildID 相同，用于关联构建任务
    "names":       ["my-template"],      // 注册时 name 字段的值；构建完成时写入，可为空数组
    "aliases":     ["my-template"],      // 注册时 tags 字段的值；与 names 实际指向同一数组，内容相同
    "public":      false,                // 固定为 false（当前版本不支持公开模板）
    "buildStatus": "ready",              // 固定为 ready（此列表只含已完成模板）
    "createdAt":   1719187200            // 构建注册时间，Unix 整数（秒）
  }
]
```

> **典型用法**：构建完成后无需查此列表——`GET …/status` 在 `status=ready` 时响应体的 `templateID` 已是 Persist ID，直接使用即可。此端点更适合**发现与管理**场景：列出账号下全部可用模板，或在 `buildID` 遗失时通过 `names` / `aliases` 重新找回对应的 `templateID`。

---

#### 密钥管理(CLI + admin 控制套接字，非 REST 端点)

manifest_key allowlist 通过 `node-ctl manifest-key` CLI 管理，底层走本地控制套接字(UDS)的 admin plane(`/internal/admin/manifest-keys`)，不暴露为公共 HTTP 端点。

```bash
# 添加(--ttl 0 = 永不过期)
node-ctl manifest-key add --label prod-2026 --ttl 720h <MANIFEST_KEY>
# 撤销
node-ctl manifest-key remove <MANIFEST_KEY>
# 列出(仅输出指纹，不输出密钥)
node-ctl manifest-key list
```

输出：`<24-hex-fingerprint>  <label>  created=<RFC3339>  expires=<RFC3339|never>`

---

#### Admin 批量管控(❌ HTTP 端点未实现，FE-11.1 规划中)

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/admin/teams/{tid}/sandboxes/kill` | 强制销毁该团队全部沙箱 |
| POST | `/admin/teams/{tid}/builds/cancel` | 取消该团队全部进行中构建 |
| GET  | `/admin/teams/{tid}/usage?start=&end=` | 团队用量明细(FE-11.2) |
| GET  | `/admin/teams/{tid}/volumes` | 团队命名卷列表(FE-6.1) |

当前节点级 admin 能力通过 `node-ctl resource {drain|grant|reclaim}` CLI 操作，同样走本地 UDS。

---

#### 错误响应格式

所有 4xx/5xx 响应体(`Content-Type: application/json`)：

```json
{ "message": "human-readable description" }
```

常见 HTTP 状态码：`400` 请求格式错误、`401` 鉴权失败、`403` 无权限(manifest_key 不在 allowlist)、`404` 资源不存在、`500` 内部错误。

---

#### 系统端点

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/health` | 存活探针，响应 `204 No Content`(空体) |
| GET | `/metrics` | Prometheus text format，在独立的 `metrics_listen` 地址暴露，不在 `api.listen` |

### 2.3.2 数据库设计(sqlite，单文件 WAL)

**sandboxes 表**

```sql
CREATE TABLE IF NOT EXISTS sandboxes (
    id                   TEXT PRIMARY KEY,           -- uuidv7，即 sid
    template_id          TEXT NOT NULL,
    state                TEXT NOT NULL,              -- running | paused | dead
    deadline_unix        INTEGER NOT NULL DEFAULT 0, -- 0 = 无 deadline
    run_dir              TEXT NOT NULL,              -- /run/sandbox/<sid>/
    base_dir             TEXT NOT NULL,
    envd_uds             TEXT NOT NULL DEFAULT '',   -- UDS 路径，--connect 映射
    ci_uds               TEXT NOT NULL DEFAULT '',
    floatingip           TEXT NOT NULL DEFAULT '',
    vswitch_port         TEXT NOT NULL DEFAULT '',
    inner_ip             TEXT NOT NULL DEFAULT '',
    port_mac             TEXT NOT NULL DEFAULT '',
    manifest_key_hash    TEXT NOT NULL,              -- hex(SHA256(mk)[:12])，非唯一前缀索引
    manifest_key_enc     TEXT NOT NULL,              -- AES-256-GCM 密文
    snapshot_ref         TEXT NOT NULL DEFAULT '',   -- 本机 bundle 路径或 manifest://
    envd_access_token    TEXT NOT NULL DEFAULT '',
    traffic_access_token TEXT NOT NULL DEFAULT '',
    metadata_json        TEXT NOT NULL DEFAULT '{}', -- kuasar-sandbox.* 命名空间 KV
    env_json             TEXT NOT NULL DEFAULT '{}',
    created_unix         INTEGER NOT NULL
);
CREATE INDEX IF NOT EXISTS idx_sandboxes_state  ON sandboxes(state);
CREATE INDEX IF NOT EXISTS idx_sandboxes_mkhash ON sandboxes(manifest_key_hash);
```

> 规划列(❌ 未实现)：`volumes_json TEXT`(FE-6.1 文件存储卷)、`object_creds_enc TEXT`(FE-6.2 对象存储凭证)将在对应特性落地时 `ALTER TABLE` 追加。

**builds 表**(兼任模板登记)

```sql
CREATE TABLE IF NOT EXISTS builds (
    build_id          TEXT PRIMARY KEY,          -- 构建任务 id(uuidv7)
    template_id       TEXT NOT NULL,             -- 临时 id(触发和状态查询用)
    persist_id        TEXT NOT NULL DEFAULT '',  -- 构建完成后写入 <profile>-<kind>-<key>
    manifest_key_hash TEXT NOT NULL,
    manifest_key_enc  TEXT NOT NULL,             -- AES-256-GCM 密文
    profile           TEXT NOT NULL,             -- e2b | bare
    kind              TEXT NOT NULL,             -- img | snp
    from_image        TEXT NOT NULL DEFAULT '',
    from_template     TEXT NOT NULL DEFAULT '',
    start_cmd         TEXT NOT NULL DEFAULT '',
    ready_cmd         TEXT NOT NULL DEFAULT '',
    steps_json        TEXT NOT NULL DEFAULT '[]',
    status            TEXT NOT NULL,             -- registered | waiting | building | ready | error
    reason            TEXT NOT NULL DEFAULT '',
    names_json        TEXT NOT NULL DEFAULT '[]',
    aliases_json      TEXT NOT NULL DEFAULT '[]',
    created_unix      INTEGER NOT NULL,
    registry_auth_enc TEXT NOT NULL DEFAULT '',  -- AES-256-GCM docker config.json
    metadata_json     TEXT NOT NULL DEFAULT '{}'
);
CREATE INDEX IF NOT EXISTS idx_builds_status ON builds(status);
CREATE INDEX IF NOT EXISTS idx_builds_mkhash ON builds(manifest_key_hash);
```

**manifest_keys 表**(create/build allowlist)

```sql
CREATE TABLE IF NOT EXISTS manifest_keys (
    key_hash          TEXT NOT NULL,              -- hex(SHA256(mk)[:12])，非唯一
    key_enc           TEXT NOT NULL,              -- AES-256-GCM 密文
    label             TEXT NOT NULL DEFAULT '',
    created_unix      INTEGER NOT NULL,
    expires_unix      INTEGER NOT NULL DEFAULT 0, -- 0 = 永不过期；>0 = Unix 过期时间
    registry_auth_enc TEXT NOT NULL DEFAULT ''    -- catch-all 镜像拉取凭据(docker config.json)
);
CREATE INDEX IF NOT EXISTS idx_manifest_keys_hash ON manifest_keys(key_hash);
```

> 注：`sandbox_quota`(并发沙箱数限制)尚未实现，不在当前 schema 中。

**volumes 表**(❌ FE-6.1，规划中)

```sql
CREATE TABLE volumes (
    name              TEXT NOT NULL,
    manifest_key_hash TEXT NOT NULL,
    provider          TEXT NOT NULL,   -- sfs_turbo | nfs
    export_endpoint   TEXT NOT NULL,   -- SFS Turbo 挂载点或 NFS server:path
    size_bytes        INTEGER,
    created_unix      INTEGER NOT NULL,
    PRIMARY KEY (name, manifest_key_hash)
);
```

**usage_events 表**(❌ FE-11.2，规划中)

```sql
CREATE TABLE usage_events (
    id                INTEGER PRIMARY KEY AUTOINCREMENT,
    sandbox_id        TEXT NOT NULL,
    manifest_key_hash TEXT NOT NULL,
    event_type        TEXT NOT NULL,   -- created | paused | resumed | dead
    ts_unix           INTEGER NOT NULL,
    cpu_cores         REAL,
    memory_mb         INTEGER,
    template_id       TEXT
);
CREATE INDEX idx_usage_events_key_ts ON usage_events(manifest_key_hash, ts_unix);
```

FE-11.2 落地后，每次沙箱状态变更在同一事务内写入 `usage_events`，保证计量与状态变更原子一致。

**设计说明**

- `*_key_enc` / `*_creds_enc` 字段格式：`keytag(4B) ‖ nonce(12B) ‖ ciphertext‖tag`（AES-256-GCM Seal 一体输出，hex 编码后存 TEXT）；keytag = `SHA256(rawKey)[:4]`，用于轮转时选择解密密钥；
- `*_key_hash` 是指纹(`hex(SHA256(mk)[:12])`)，允许哈希碰撞(非唯一约束)，精确匹配通过解密后明文比对；
- `builds` 表写入语义：`PutBuild()` 使用 `INSERT ... ON CONFLICT(build_id) DO UPDATE SET ...`（UPSERT）；状态推进由 `CASBuildStatus()` 完成，执行 `UPDATE builds SET status=? WHERE build_id=? AND status=?`，保证多 worker 竞争构建任务的安全性；`waiting` 是独立的 DB 状态（触发后入队等待 pool slot），`SDKStatus()` 将 registered/waiting/building 均映射为 `"building"` 上报 SDK；
- 当前 `sandboxes` 有两个独立索引(`idx_sandboxes_state`、`idx_sandboxes_mkhash`)，分别覆盖状态扫描和 manifest_key 定位；FE-11.1 的 `/admin/teams/{tid}/sandboxes/kill` 需要复合索引 `(manifest_key_hash, state)`，届时追加；
- `expires_unix` 为整数(`DEFAULT 0`)，0 表示不过期；`AllowedManifestKeysByHash` 过滤条件：`expires_unix=0 OR expires_unix>now()`。

## 2.4 组件模块与运行态设计

### 2.4.1 Process Topology

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│  host  (default netns)                                                            │
│                                                                                   │
│  ┌─────────────────────────────────────────────────────────────────────────────┐ │
│  │  node-ctl serve  [systemd service, single process]                           │ │
│  │                                                                               │ │
│  │  config-socket  (UDS, h2c, 4 planes multiplexed):                            │ │
│  │    task   plane  <── sandbox-ctl      /internal/task/launchspec              │ │
│  │    admin  plane  <── node-ctl CLI     /internal/admin/*                      │ │
│  │    plugin plane  <── node-proxy       PUT /internal/plugin/{id}/register     │ │
│  │    api    plane  <── external clients /sandboxes  /templates  ...            │ │
│  │                                                                               │ │
│  │  resource_listen.socket  (UDS, resource control plane):                      │ │
│  │    <── sandbox-ctl  (Admit / Settled / Heartbeat / Release)                  │ │
│  │                                                                               │ │
│  │  MMDS HTTP  [proxy.mode=internal only]  127.0.0.1:19254                      │ │
│  │    <── guest envd via 169.254.169.254:80 redirect in vswitch                 │ │
│  └──────┬──────────────────────────────┬──────────────────────────────────────┘ │
│         │ D-Bus / sdbus                 │ vswitch-ctl attach/detach (subprocess) │
│         v                               v                                         │
│  ┌──────────────────┐   ┌──────────────────────────────────────────────────────┐ │
│  │  systemd         │   │  sandbox-vswitch  (vswitch-ctl serve)                │ │
│  │                  │   │                                                        │ │
│  │  sandbox-runner  │   │  [SWITCH_NETNS]  [PORT_NETNS]  [MGMT_NETNS]          │ │
│  │    @<sid> x N    │   │                                                        │ │
│  │  sandbox-builder │   │  eBPF/TC hooks pinned to bpffs                        │ │
│  │    @<bid> x M    │   │  /sys/fs/bpf/vswitch/{prog_tc_ingress, ...}           │ │
│  │                  │   │                                                        │ │
│  │  sandbox-runner  │   │  TAP fd -> cloud-hypervisor  (SCM_RIGHTS via          │ │
│  │    .slice        │   │             vswitch-ctl open-port + TAPFD_SOCKET)      │ │
│  │  sandbox-builder │   └──────────────────────────────────────────────────────┘ │
│  │    .slice        │                                                             │
│  └──────┬───────────┘                                                             │
│         │ execve                                                                   │
│         v                                                                         │
│  ┌─────────────────────────────────────────────────────────────────────────────┐ │
│  │  sandbox-ctl x (N+M)  [host netns]                                           │ │
│  │    └─ cloud-hypervisor  [host netns, child process]                          │ │
│  │         ┌───────────────────────────────────────────────────────────────┐    │ │
│  │         │  guest VM  (isolated netns managed by VM kernel)               │    │ │
│  │         │    sandbox-init  -->  envd  (Connect RPC, :49983 / :49999)     │    │ │
│  │         └───────────────────────────────────────────────────────────────┘    │ │
│  └─────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                   │
│  ┌─────────────────────────────────────────────────────────────────────────────┐ │
│  │  node-proxy worker x K  [host netns, SO_REUSEPORT, external mode only]      │ │
│  │    MMDS HTTP  [proxy.mode=external only]                                     │ │
│  │    --> envd UDS  (port 49983 / 49999, e2b profile)                          │ │
│  │    --> floatingIP:port  (user ports, TCP forwarded by vswitch)               │ │
│  └─────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                   │
│  ┌─────────────────────────────────────────────────────────────────────────────┐ │
│  │  cache-ctl  [host netns]        store-ctl  [host netns]                      │ │
│  │  (sandbox-accelerator; node-ctl.service After= hard dependency)              │ │
│  └─────────────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────────────┘
```

**config-socket plane auth**  (all based on SO_PEERCRED, no HTTP-level token):

| plane  | path pattern                         | auth                                          |
|--------|--------------------------------------|-----------------------------------------------|
| task   | `/internal/task/launchspec`          | SO_PEERCRED pid == pidfile (sandbox-ctl pid)  |
| admin  | `/internal/admin/*`                  | SO_PEERCRED pid in admin_pidfile, or socket 0600 |
| plugin | `PUT /internal/plugin/{id}/register` | SO_PEERCRED pid in plugin_pidfile             |
| api    | all other paths                      | `X-API-KEY` or `Authorization: Bearer`        |

### 2.4.2 沙箱生命周期状态机

**持久化状态**(DB `sandboxes.state`)只有三种：`running`、`paused`、`dead`。CREATING / PAUSING / RESUMING / KILLING 均为瞬态执行阶段，不写入数据库；Kill() 直接删除行(不经 dead 状态)。

```
NONE
 │ POST /sandboxes
 │ → 生成 uuidv7 sid，DB 写 state=running，异步执行 launch()
 ▼
[state=running] ◄─────────────────────────────────────────────┐
 │                                                              │
 │ 启动中(瞬态)：systemd StartUnit → sandbox-ctl Admit       │
 │   → cloud-hypervisor restore → 等待 envd ready             │
 │                                                              │
 │ idle > suspend_after(auto-suspend)                        │
 │ 或 POST /sandboxes/{id}/pause                               │
 │ → DB 立即写 state=paused                                    │
 ▼                                                              │
[state=paused]                                                 │
 │                                                              │
 │ 暂停中(瞬态)：sandbox-ctl snapshot → 停止 unit            │
 │   → detach vswitch → routesync 广播 upsert(state=paused)   │
 │                                                              │
 │ 流量到达 / proxy Wake 帧                                     │
 │ → resume() 克隆私有副本 nb，nb.state=running，执行 launch() │
 │   成功后 DB 写 state=running，routesync 广播 upsert         │
 └──────────────────────────────────────────────────────────────┘

 DELETE /sandboxes/{id} / deadline 到期
 → teardown()(StopUnit/detach vswitch/清理目录)+ st.Delete()
 → 行直接从 DB 删除(不经 state=dead)

[state=dead](仅 Reconcile() 对账写入)：
 → serve 重启后扫描 systemd 单元，DB running 但无活跃单元
 → 标 state=dead，由 Reaper 后续清理

集群深度休眠(SAVED，由 cluster-ctl 触发)：
 [state=paused] + cluster-ctl 晋升远程快照
 → snap_loc 由 "local" 变为 "remote"，routesync RouteEntry.state = "saved"
 → Wake → 跨机迁移恢复(携带 migration_token)，非本机 resume
```

**关键实现细节**：

- **Create 写 running 时机**：`Create()` 在调用 `launch()` 之前就将 `state=running` 写入 DB，`launch()` 失败则由上层 Kill() 清理；
- **pauseSandbox 写 paused 时机**：`pauseSandbox()` 先拍快照再写 `state=paused`，随后停止 unit、detach vswitch、routesync 广播；
- **resume 私有副本**：`resume()` 在私有 `nb := *sb` 上操作，成功后再原子发布到 in-memory 缓存，避免并发读写竞争；
- **per-sid singleflight**：同一 sid 的并发 Wake 帧合并为单次 resume，防止多 worker 同时触发 cloud-hypervisor restore；
- **KillMode=control-group**：systemd StopUnit 原子杀死 sandbox-ctl + cloud-hypervisor 整个进程组，无孤儿进程；
- **Reconcile()**：以 `ListUnitsByPatterns("sandbox-runner@*.service")` 的 active/activating 状态为存活权威，DB running 但无对应活跃单元 → 标 `state=dead`；
- **Reaper()**：每 5 s 扫描 running 沙箱，超过 deadline → 调 `pauseSandbox()`；同时触发 `reapDeepIdle()`(cluster SAVED 晋升)和 `PruneExpiredManifestKeys()`。

### 2.4.3 模板构建三阶段流水线运行态

**资源调度**

构建任务由 `sandbox-builder@<bid>.service`(oneshot 单元)承载：

- `builder.max_concurrent` 信号量(默认 2)控制并发数；
- `builds` 表 CAS 抢占防多实例重复构建；
- 整条流水线在单元 cgroup(`sandbox-builder.slice`)内，`CPUQuota` / `MemoryMax` 对阶段 VM 生效。

**三阶段数据流**

```
Phase A：Import(有 fromImage)
──────────────────────────────
宿主: StartUnit(sandbox-builder@<bid>) → run-builder
guest(空盘 + builder runtime):
    flatten-ctl export --output - <fromImage>  [使用租户 FLATTEN_* 凭据]
    → tarstream via exec stdio → 宿主 workdir/image.img

Phase B：Steps(有 steps)
──────────────────────────
guest(base 镜像 + builder runtime + 大可写 upper):
    for step in steps:
        RUN  → envd process.Start(cmd) [bash -l -c <cmd>，按 ENV/WORKDIR/USER 上下文]
        COPY → sandbox-ctl exec + flatten-ctl tar extract --dense(不依赖 tar/gzip)
        ENV/ARG/WORKDIR/USER → 更新宿主侧 BuildSpec 上下文
    宿主: 累积上下文叠回 base config
    guest: flatten-ctl export --skip-mounts → 宿主 workdir/image.img(更新)

Phase C：Template(有 startCmd)
──────────────────────────────
guest(最终镜像 + 生产 e2b runtime，runtime_ref 冻入快照):
    envd 启动 startCmd(默认用户 user，/home/user)
    宿主: 轮询 readyCmd(2s 间隔，至多 ready_timeout_sec)
    就绪: sandbox-ctl snapshot --output <bundle>
    上传: sandbox-ctl upload-snapshot <bundle>(manifest_key 经 env，不过控制面)
    结果: {snapshot_key, start_cmd, ready_cmd} → stdout → <bid>.result
```

**COPY 三段式(对象存储直传)**：

1. 客户端调 `GET /templates/{tid}/files/{hash}`，响应 `{present, url}`；
2. 若 `present=false`，客户端直传 gzip(tar) 到 presigned PUT URL(字节不过控制面)；
3. 构建期：`http.Get(presigned_url)` → 宿主 gunzip → `sandbox-ctl exec flatten-ctl tar extract --dense <rule>`。

**构建日志流**

```
写入路径:
  run-builder 里程碑  ──► go-systemd/journal(纯 Go，CGO-free)
  envd RUN/startCmd 输出 ──► run-builder 流回 ──► journal
  阶段 app stdio  ──► sandbox-ctl --stdout-to/--stderr-to journald=build
  flatten-ctl 进度 ──► sandbox-ctl exec --stderr-to journald=build

读取路径:
  GET /status?logsOffset=N
  → journalctl _SYSTEMD_UNIT=sandbox-builder@<bid>.service
              SYSLOG_IDENTIFIER=build --output=json
  → 切片 [N:] → {logs[], logEntries[{timestamp, level, message}]}
```

### 2.4.4 数据面路由装配与 eBPF 转发

**routesync 广播**

serve 是本节点路由权威。沙箱状态变化时，实时向所有 plugin 平面订阅者(node-proxy workers)广播 routesync 消息。传输层为 h2c 双向流(config-socket 的 plugin 平面)，帧格式：

```
帧格式：[4B LE uint32 帧长度][JSON 正文(最大 1 MiB)]

// 握手：权威 → 订阅者，含策略
{"type":"hello","hello":{"version":1,"policy":{"domain":"...","auth_mode":"off|log|enforce","park_timeout_ms":5000}}}

// 订阅者注册(上行第一帧)
{"type":"register","register":{"subscribe":{"kind":"route_wake"},"proxy":{"socket":{"path":"/run/node-proxy/proxy.sock"}},"mmds":true}}

// 路由新增/更新(含全量同步期间的 Upsert)
{"type":"upsert","route":{"sid":"abc123","profile":"e2b","template_id":"e2b-snp-<64hex>",
  "state":"running","envd_uds":"/run/sandbox/abc123/envd.sock",
  "ci_uds":"/run/sandbox/abc123/ci.sock","floatingip":"10.x.x.x",
  "access_token":"<hex>","snap_loc":"local","mmds_secret":"<hex>",
  "group":"","route_key":"","migration_token":""}}

// SAVED 状态(cluster 深度休眠)
{"type":"upsert","route":{"sid":"abc123","state":"saved","snap_loc":"remote","migration_token":"<tok>",...}}

// 路由删除
{"type":"delete","sid":"abc123"}

// 全量快照结束标记
{"type":"bookmark"}

// 订阅者请求唤醒(上行，route_wake 订阅者专用)
{"type":"wake","sid":"abc123"}
```

**订阅者生命周期**：

1. 订阅者连接 config-socket，`PUT /internal/plugin/{id}/register`，上行首帧发 `register`；
2. 服务端发 `hello`(策略)，再流式发 `upsert` × 全量(running + paused)，最后发 `bookmark`；
3. 之后仅发增量 `upsert`/`delete`；
4. 订阅者事件通道缓冲为 **1024**，积压满时直接关闭通道(订阅者被丢弃)；订阅者需自行指数退避重连，重连后重走全量同步(`BeginSync` → 全量 `upsert` + `bookmark`，清理未出现的旧条目)；
5. 无 rev / 循环缓冲机制：每次重连必须全量重同步。

**eBPF 数据面(external 模式)**

```
外部请求(port 443/80)
    │
    ▼
[node-proxy worker × K，SO_REUSEPORT]
    │ 解析 Host / E2b-Sandbox-Id header
    │ 查本地路由表(sync.Map，O(1))
    │
    ├── e2b 内置端口(49983/49999)→ UDS(envd.sock/ci.sock)
    │                               via sandbox-ctl --connect 映射
    │
    └── 其他端口 → floatingip:port
                  ├── ESTABLISHED 连接：eBPF TC hook 转发(< 50 µs)
                  └── 新建连接：node-proxy splice + 注册 flowtable

/sys/fs/bpf/vswitch/
  ├── prog_tc_ingress   # attach 到 eth0 ingress
  ├── prog_tc_egress    # attach 到 eth0 egress
  ├── map_flowtable     # {src_ip,src_port,dst_ip,dst_port} → {floatingip,port}
  └── map_conntrack     # 连接状态(SYNACK / ESTABLISHED / CLOSING)
```

**PAUSED 沙箱的 park/wake**

路由条目 `state="paused"` 时，node-proxy 将新连接加入 per-sid park 队列(最长等待 `park_timeout_ms`，默认 30 s)：

1. 向 serve 发 `wake` 上行帧(routesync route_wake 订阅者专用)；
2. serve `OnWake()` 通过 per-sid singleflight 触发 `resume()`，成功后广播 `upsert(state=running)`；
3. park 队列中所有连接解除，splice 到 floatingip:port；
4. 若 sid 未知或已 dead，serve 广播 `delete`，node-proxy 返回 404/410。

**MMDS 确定性密钥**

node-proxy 在 FC MMDS v2 端点响应 guest 的 `GET /latest/meta-data/` 请求。mmds_secret 在 serve 侧计算：

```go
// keys.MmdsSecret(manifestKeyHex, sandboxID) → 32 字节密钥
mmds_secret = HMAC-SHA256(manifest_key_bytes, "kuasar-mmds-v1:" + sid)
```

公式确定性，serve 将结果写入 `RouteEntry.mmds_secret`(hex)，经 routesync 下发给所有 worker；N 个 worker 无需共享状态即可对等响应，manifest_key 明文不离开 serve 内存。

**Secure 模式(mmds.enabled=true)**

创建沙箱时指定 `mmds_enabled: true` 激活安全姿态，node-ctl 与 node-proxy 协作完成两段式鉴权：

```
创建阶段：
  node-ctl 铸 envd_access_token(随机 32 B hex)
  → sandbox 行写 envd_access_token(加密存储)
  → routesync RouteEntry.mmds_secret 携带 mmds_key(供 MMDS 响应)

启动阶段(guest sandbox-init 就绪后)：
  node-proxy MMDS v2 → sandbox-init 取 mmds_key
  → sandbox-init 调 POST /init(经 UDS)携带 accessToken=envd_access_token
  → envd re-key：此后所有 envd RPC 需在 Authorization 头携带该 token

数据面鉴权(非 e2b 内置端口的 user 端口)：
  proxy.auth=enforce 时：请求需携带 traffic_access_token(X-Access-Token 头)
  token 不匹配 → 401；无 Secure 模式时 proxy.auth=passthrough
```

**CA Bundle 注入**

CA Bundle(PEM 格式)通过 `kuasar-sandbox.files` 配置命名空间注入，而非 create 请求的顶层字段：

```json
// 创建时 metadata 或 X-Kuasar-Sandbox-Files 请求头
{
  "kuasar-sandbox.files": "[{\"path\":\"/etc/ssl/certs/kuasar-custom-ca.crt\",\"content\":\"<PEM>\"}]"
}
```

sandbox-ctl 在 `launch` 和 `restore` 时均将 files 注入 guest，guest 内证书验证库自动信任该 CA。

**互联网访问开关**

通过 `kuasar-sandbox.network` 命名空间控制，而非顶层 API 字段：

```json
{"kuasar-sandbox.network": "{\"internet\": false}"}
```

`internet: false` → vswitch attach 附加 `--no-nat` 标志，guest 无公网出站。

**per-sandbox 入站并发连接限制**(❌ FE-1.4，规划中)

计划通过 `kuasar-sandbox.network` 命名空间的 `max_connections` 字段控制，node-proxy 维护 per-sid 活跃连接计数器：

```
新连接建立时：
  if counter[sid] >= max_connections: 返回 503
  else: counter[sid]++
连接关闭时: counter[sid]--
```

当前 `RouteEntry` 不含 `max_connections` 字段，该能力为规划中特性。

**友好错误页**

node-proxy 对以下场景返回结构化 JSON 错误(`Content-Type: application/json`)，而非通用 502/503：

| 场景 | HTTP 状态 | 错误码 |
|------|---------|--------|
| sid 不存在 | 404 | `sandbox_not_found` |
| 沙箱已 dead | 410 | `sandbox_terminated` |
| park 等待 wake 超时 | 503 | `sandbox_wake_timeout` |
| 入站连接超限 | 503 | `connection_limit_exceeded` |
| 沙箱正在 CREATING | 503 | `sandbox_not_ready` |

### 2.4.5 内存资源管控运行态

**Reservation 生命周期阶段**

sandbox-ctl 与 resource-controller 交互过程中，Reservation 经历以下阶段(`Stage` 字段)：

| 阶段 | 进入时机 | 说明 |
|------|---------|------|
| `admitted` | Admit 成功 | 已准入，扣减 startup_pool |
| `creating` | sandbox-ctl 开始创建 microVM | 仍在 pre-settled 集合中 |
| `startup` | microVM 启动中 | 仍在 pre-settled 集合中 |
| `restoring` | 从快照恢复中 | 仍在 pre-settled 集合中 |
| `settled` | Settled 消息到达 | startup_pool 归还；allocatable_now 收缩 |
| `burst` | RequestBudget 触发扩容 | 高于 settled 基线的临时配额 |
| `recover` | Active Reclaimer 回收中 | 气球施压，缩减 allocatable_now |
| `released` | Release 消息到达 | 准备从 Reservations 中移除 |

**pre-settled 集合**(admitted/creating/startup/restoring)对应的 `effective_startup_budget` 计入 `startup_pool`，Settled 后归还。

**水位区与准入决策**

```
allocatable_pool = 节点总内存 - host_reserved - operational_margin
startup_pool     = allocatable_pool × startup_factor

绿区 (alloc < low_watermark):    正常准入，直接授予 budget
黄区 (low ≤ alloc < high):       进入 FIFO 准入队列等待
红区 (high ≤ alloc < pool-emerg): 停止新准入，Active Reclaimer 激进模式
                                  RequestBudget urgency≠high 被阻止
危急 (alloc ≥ pool-emerg):        触发紧急气球缩减 + OOM 风险告警
```

(以上水位阈值均为可配置的 `Watermarks` 参数：`high_factor`/`low_factor`/`emergency_factor`。)

**准入协议消息流**

```
sandbox-ctl                     resource-controller (resource_listen.socket)
    │                                        │
    │──── Admit(sid, startup_mem) ──────────►│ 检查水位/令牌桶/startup_pool
    │◄─── AdmitResponse(ok|reject, budget) ──│ stage: admitted
    │                                        │
    │  [microVM 启动完成，内存 RSS 稳定]      │
    │──── Settled(token, current_rss) ───────►│ stage: settled
    │                                        │ startup_pool 归还 effective_startup_budget
    │                                        │ allocatable_now = max(rss, floor)
    │                                        │ 唤醒准入等待队列
    │  [运行中 burst 申请]                   │
    │──── RequestBudget(delta, urgency) ─────►│ urgency: normal|high|low
    │◄─── BudgetResponse(granted) ───────────│ 红/危急区阻止非 high urgency
    │                                        │
    │  [定期心跳，~30s]                      │
    │──── Heartbeat(current_rss) ────────────►│ 更新 LastReportedRSS
    │◄─── HeartbeatAck(allocatable_now) ─────│ 同步当前可用配额给客户端
    │                                        │
    │──── OOMReport ─────────────────────────►│ 触发紧急气球回收
    │──── Release ───────────────────────────►│ 归还全部配额，移除 Reservation
    │                                        │
    │  [控制器重启后，sandbox-ctl 重连]       │
    │──── Reattach(token) ───────────────────►│ 重绑 Reservation
    │◄─── Ack(new_allocatable) ──────────────│
    │                                        │
    │  [管理员强制操作(admin 平面)]          │
    │  AdminGrant(token, delta) ─────────────►│ 强制扩容，绕过水位/令牌桶
    │  AdminReclaim(token, target) ──────────►│ 强制缩容(target < current)
    │  AdminDrain(token) ─────────────────────►│ 优雅释放
    │                                        │
```

**Active Reclaimer**

后台 goroutine，每 10 s 扫描所有 settled reservation：

```
for each sid in settled_reservations:
    target = max(floor_memory, current_rss × reclaim_margin)  // 默认 margin=1.25（黄区 1.10，红区 1.05）
    if allocatable_now(sid) > target:
        delta = allocatable_now(sid) - target
        通过 virtio-balloon 向 guest 施压(归还 delta MB 物理帧)
        更新 allocatable_now(sid) = target
        state.json 批量 flush(5s 间隔)
```

**令牌桶**

每秒最多授予 `allocatable_pool × grant_rate_factor`(默认 5%)的新内存配额。令牌桶为空时，Admit 头部阻塞于队列，Worker 设 one-shot `AfterFunc(eta, pushWake)` 在令牌补充后精确唤醒。

## 2.5 配置设计

### 2.5.1 node-ctl 主配置文件

`node-ctl serve --config <path>` 读取一个 YAML 文件，对应 `internal/config.Config` 结构体。配置按关注点分组，外部二进制(`sandbox-ctl`/`vswitch-ctl`/`flatten-ctl`/`manifest-ctl`)不在此配置——自动从与 `node-ctl` 同目录查找，再退回 PATH。

**顶层字段**

```yaml
api:          # 北向控制面 + TLS
proxy:        # 数据面代理
paths:        # 节点本地目录与 socket
units:        # systemd 单元管理
sandbox:      # 沙箱实例默认值(resources / network / boot)
builder:      # 构建实例配置
resource_listen:  # 可选：内嵌资源控制器
checkpoint:   # 暂停状态分级策略
mmds:         # 可选：FC MMDS v2 服务
cluster:      # 可选：接入集群注册中心
encryption_key:   # AES-256 密钥(或 NODE_CTL_ENCRYPTION_KEY 环境变量)
manifest_config:  # 远程 manifest store 配置文件路径
```

**api**

```yaml
api:
  domain: sandboxes.example.com      # 必填；e2b SDK 的 E2B_DOMAIN
  listen: ":443"                      # 默认 :443；dev 用 :3000(plain h2c)
  tls:
    cert: /etc/node-ctl/tls/fullchain.pem  # 通配 *.<domain> 和 api.<domain>
    key:  /etc/node-ctl/tls/privkey.pem
```

**proxy**

```yaml
proxy:
  mode: internal          # internal(默认)| external | off
  auth: enforce           # off | log | enforce(默认)；验证 X-Access-Token
  data_listen: ":8443"    # 数据面监听地址(SO_REUSEPORT，external 模式)
  park_timeout: 30s       # 等待路由/唤醒的最长时间
  metrics_listen: ""      # Prometheus 文本端点；"" = 关闭
```

- `proxy.mode=internal`：代理在 serve 进程内，单一二进制无额外组件；
- `proxy.mode=external`：代理 worker 通过 config-socket plugin 平面自注册，node-ctl 向 live 注册集推路由；
- `mmds.enabled=false` 时必须 `proxy.auth=enforce`(envd 以非安全模式运行，proxy 是唯一数据面门卫)。

**paths**

```yaml
paths:
  run_root:      /run/sandbox                   # 默认；tmpfs 运行态(整机重启后清空)
  base_root:     /var/lib/sandbox               # 默认；持久状态
  db_path:       /var/lib/sandbox/node-ctl.db   # 默认 = <base_root>/node-ctl.db
  config_socket: /run/sandbox/node-ctl.socket   # 默认；单 UDS 四平面(h2c)
  admin_pidfile:  ""   # 可选：admin 平面 PID 白名单文件；"" = 依赖 socket 0600 权限
  plugin_pidfile: ""   # 可选：plugin 平面 PID 白名单文件；同上
```

**sandbox**

```yaml
sandbox:
  timeout_sec: 300          # 默认 TTL(秒)
  capacity: 0               # 本节点最大准入沙箱数；0 = 不限

  resources:
    vcpu: 2                 # 默认 vCPU 数
    memory: 2GiB            # 默认内存容量(支持 GiB/MiB/G/M 后缀)
    control_socket: ""      # 资源控制器 UDS；"" = 静态 cgroup 模式(opt-in)

  network:
    switch: sw0             # vswitch 名称
    hostname: sandbox       # guest hostname(sethostname + /etc/hosts)
    dns: [169.254.169.253]  # 注入 /etc/resolv.conf 的 nameserver 列表
    e2b:                    # e2b profile(envd port-forward 需 /30 网段)
      inner_ip: 169.254.0.21/30
      nexthop:  169.254.0.22
    bare:                   # bare profile
      inner_ip: 169.254.1.1/31
      nexthop:  169.254.1.0

  boot:
    kernel:                /opt/sandbox/kernel/6.1/vmlinux
    runtime_e2b:           /opt/sandbox/runtime/v1/sandbox-runtime-e2b.erofs
    runtime_base:          /opt/sandbox/runtime/v1/sandbox-runtime.erofs
    overlay_diff_template: /opt/sandbox/overlay-templates/basic-1G.ext4  # img 冷启动 overlay upper 模板
```

**builder**

```yaml
builder:
  max_concurrent: 2                   # 并发构建数上限(信号量)
  runtime_builder: /opt/sandbox/runtime/v1/sandbox-runtime-builder.erofs
  diff_template:   /opt/sandbox/overlay-templates/builder-8G.ext4
  vcpu: 2
  memory: 4GiB
  cpu_quota: ""                        # -> sandbox-builder.slice CPUQuota
  memory_max: ""                       # -> sandbox-builder.slice MemoryMax
  insecure_registry: false             # 从 plain HTTP 拉取 base image(开发用)
  platform: ""                         # e.g. linux/amd64；"" = 宿主默认
  image_uri_mask: ""                   # e.g. "docker.example.com/e2b/{templateID}:{buildID}"
  pull_timeout_sec:  600
  step_timeout_sec:  600
  ready_timeout_sec: 120
  total_timeout_sec: 1800
  files_storage:                       # COPY 构建上下文对象存储(不配则 COPY 返回 501)
    endpoint:         ""               # S3/OBS endpoint；"" = AWS 默认
    region:           cn-north-4
    bucket:           kuasar-build-files    # 必填
    prefix:           ""
    access_key:       ""               # "" = AWS 默认链(env / 实例角色)
    secret_key:       ""
    force_path_style: false            # versitygw/minio 需 true
    presign_expiry:   1h               # 上传 URL 有效期
```

**checkpoint / mmds / cluster**

```yaml
checkpoint:
  mode:        local                   # local(默认，节点绑定)| remote(可迁移，等同模板)
  local_dir:   /var/lib/sandbox-saved  # 本地快照存储目录
  deep_idle_sec: 0                     # PAUSED → SAVED 晋升等待(集群模式；0=关闭)

mmds:
  enabled: false          # true 启用 FC MMDS v2；要求 proxy.mode != off
  listen:  127.0.0.1:19254

cluster:
  registry:           ""               # "" = standalone；host:port = 接入集群
  node_id:            ""               # "" = hostname
  labels: {}                           # zone/pool/slot 等 nodeSelector 标签
  data_endpoint:      ""               # 集群路由器转发数据面的 host:port
  heartbeat_interval: 10s
  tls_cert: ""                         # node-link 客户端证书(mTLS)；"" = plain h2c
  tls_key:  ""
  tls_ca:   ""                         # 验证 registry 服务端证书的 CA

resource_listen:
  enabled: false          # true = 内嵌资源控制器；需配合 sandbox.resources.control_socket
  socket:  ""             # "" = 内置默认路径
  config:  ""             # "" = 内置默认参数(详见 2.4.5)
```

**encryption_key**

```yaml
encryption_key: "<64-hex>"   # manifest_key 静态加密主密钥
                              # ":"-分隔多密钥，[0] 为活跃加密，其余只解密
                              # 可用 NODE_CTL_ENCRYPTION_KEY 环境变量覆盖
manifest_config: /opt/sandbox/manifest.yaml  # 远程 manifest store 配置
```

**校验规则**

| 条件 | 错误 |
|------|------|
| `api.domain` 为空 | 启动失败 |
| `encryption_key`(含 env)为空 | 启动失败 |
| `checkpoint.mode` 不在 `local\|remote` | 启动失败 |
| `proxy.mode` 不在 `internal\|external\|off` | 启动失败 |
| `mmds.enabled=false` 且 `proxy.auth≠enforce` | 启动失败 |
| `mmds.enabled=true` 且 `proxy.mode=off` | 启动失败 |
| `builder.files_storage` 存在但 `bucket` 为空 | 启动失败 |

### 2.5.2 沙箱运行时配置命名空间

创建沙箱时，租户可通过以下两种等效方式注入逐实例配置覆盖：

1. **`metadata` 字段**(JSON body 的 `metadata` map)：键为命名空间，值为 JSON 字符串；
2. **`X-Kuasar-Sandbox-*` 请求头**：与 metadata 键等价，**请求头优先**(覆盖 metadata 中同键值)。

| 请求头 | metadata 键 | 值类型 |
|--------|------------|--------|
| `X-Kuasar-Sandbox-Resource` | `kuasar-sandbox.resource` | JSON object |
| `X-Kuasar-Sandbox-Network` | `kuasar-sandbox.network` | JSON object |
| `X-Kuasar-Sandbox-Launch` | `kuasar-sandbox.launch` | JSON object(仅 bare profile) |
| `X-Kuasar-Sandbox-Init` | `kuasar-sandbox.init` | JSON array |
| `X-Kuasar-Sandbox-Mounts` | `kuasar-sandbox.mounts` | JSON array |
| `X-Kuasar-Sandbox-Files` | `kuasar-sandbox.files` | JSON array |
| `X-Kuasar-Sandbox-Metadata` | `kuasar-sandbox.metadata` | JSON object |
| —(集群内部) | `kuasar-sandbox.cluster` | JSON object |

**各命名空间说明**

```
kuasar-sandbox.resource
  capacity:    {cpu: N, memory: "NMiB"}   覆盖 vCPU/内存容量(仅 img 模板有效；snp 快照固定)
  allocatable: {cpu: F, memory: "NMiB"}   初始可用量(资源控制器起点)

kuasar-sandbox.network
  hostname: "name"                         guest hostname
  dns: ["8.8.8.8"]                         /etc/resolv.conf nameserver 列表
  inner_ip: "169.254.0.21/30"              guest NIC CIDR
  nexthop:  "169.254.0.22"                 默认路由网关
  transit_gateway_ip / transit_geneve_vni / transit_mac   集群 overlay 网络字段

kuasar-sandbox.launch   (仅 bare profile；e2b 拒绝，envd 拥有 launch)
  exec: "/app/server"
  args: ["--port","8080"]
  workdir: "/app"
  user: "1000:1000"
  restart: "always"
  stop_signal: "SIGTERM"
  env: {"KEY":"val"}
  plugin: [...]

kuasar-sandbox.init     []InitConfig    sandbox-init 在 workload 前执行的初始化步骤
kuasar-sandbox.mounts   []MountConfig   额外挂载(virtiofs / block 等)
kuasar-sandbox.files    []FileConfig    注入 guest 的文件(path/content/mode)
                        示例：CA Bundle 注入
                        [{"path":"/etc/ssl/certs/custom-ca.crt","content":"-----BEGIN CERTIFICATE-----\n...","mode":"0644"}]

kuasar-sandbox.metadata {key:val}       透传到 SANDBOX_CONFIG.metadata(如 e2b.start_cmd)

kuasar-sandbox.cluster  {group:"g1", route_key:"k1"}  集群路由分组与分片标识(cluster-ctl 内部)
```

**配置优先级**(从低到高)

```
节点默认(config.yaml sandbox.*)
  ↓ 覆盖
模板构建时 metadata(builds 表 metadata_json)
  ↓ 覆盖
创建请求 metadata 字段(kuasar-sandbox.<ns> 键)
  ↓ 覆盖
创建请求 X-Kuasar-Sandbox-* 请求头(同键时头优先)
```

创建时的配置命名空间经 `sandboxcfg.MergeMetadata(templateMeta, createMeta)` 合并后，写入 `sandboxes.metadata_json` 持久化；resume 时从 DB 读回重放，保证配置快照跨节点可还原。

### 2.5.3 资源控制器配置(DaemonConfig)

当 `resource_listen.enabled=true` 时，`resource_listen.config` 指向一个独立 YAML 文件(`DaemonConfig`)，默认路径由 `ResourceListenConfig.Socket` 隐含；不配则使用内置默认值。

```yaml
# 资源控制器配置(resource_listen.config 指向此文件)
listen:     /run/sandbox-resource.sock   # resource_listen.socket 覆盖此值
state_path: /run/node-ctl/state.json     # 内存状态镜像(tmpfs，进程重启后仍在)
cgroup_scan_paths:
  - /sys/fs/cgroup/sandboxes             # 重启对账扫描路径

resources:
  physical_memory: auto                  # "auto" = /proc/meminfo MemTotal；或 "512GiB"
  physical_cpu:    auto                  # "auto" = runtime.NumCPU()；或 "128"
  host_reserved:
    memory: 16GiB                        # 宿主预留内存
    cpu:    1.5                          # 宿主预留 CPU(核)

watermarks:
  operational_margin_factor: 0.10        # allocatable_pool = (physical-host_reserved)×(1-margin)
  high_factor:               0.85        # 绿→黄 转换水位
  low_factor:                0.70        # 黄→绿 转换水位
  emergency_factor:          0.05        # (pool - pool×emerg) 即危急触发点
  startup_factor:            0.50        # startup_pool = allocatable_pool × this

rate_limits:
  memory_grant_per_sec_factor: 0.05      # 每秒最多授予 allocatable_pool×5% 的新配额

admission:
  rate:            4                     # 令牌桶速率(admission/s)
  burst:           16                    # 令牌桶突发容量
  startup_ttl:     30s                   # pre-settled 阶段超时(未 Settled 则超期清理)
  queue_ttl:       30s                   # 准入等待队列单个请求超时
  queue_max_depth: 256                   # 准入队列最大深度

dampening:
  recover_duration: 60s                  # Active Reclaimer 进入 recover 后稳定期
  cooldown_periods: 10                   # 令牌桶冷却轮数

logging:
  level:      info                       # debug | info | warn | error
  audit_path: /run/node-ctl/audit.log    # 准入/释放审计日志
```

**水位计算公式**

```
physical_pool = physical_memory - host_reserved.memory
allocatable_pool = physical_pool × (1 - operational_margin_factor)
startup_pool     = allocatable_pool × startup_factor

绿区阈值 = allocatable_pool × low_factor
黄区阈值 = allocatable_pool × high_factor
危急阈值 = allocatable_pool × (1 - emergency_factor)

令牌桶速率(字节/s)= allocatable_pool × memory_grant_per_sec_factor
```

## 2.6 运维操作

### 2.6.1 二进制与子命令

`node-ctl` 是单一静态二进制，通过子命令切换角色。外部依赖二进制(`sandbox-ctl`、`vswitch-ctl`、`flatten-ctl`、`manifest-ctl`)从 `node-ctl` 同目录自动发现，再退回 PATH，**不可配置**。

```
node-ctl <subcommand> [flags]

serve           主守护进程(API + 代理 + 资源控制 + 集群接入)
proxy           外部数据面代理 worker(proxy.mode=external 时使用)
run-sandbox     sandbox-runner@<sid>.service 内置启动器(非人工调用)
run-builder     sandbox-builder@<bid>.service 内置启动器(非人工调用)
config          校验/渲染配置文件，或输出带注释的骨架
manifest-key    租户根密钥白名单管理(通过 admin 平面 UDS)
export-sandbox  导出暂停沙箱快照(晋升模板或铸造迁移 token)
import-sandbox  从迁移 token 在本节点重建沙箱
resource        资源控制器运维(status/list/drain/grant/reclaim)
version         打印版本号
```

**config**

```
node-ctl config --config <file>    # 加载 + 校验 + 重新序列化(规范化)
node-ctl config --template [-o f]  # 输出带注释的骨架 YAML(同 orchConfigSkeleton)
```

**manifest-key**

```
node-ctl manifest-key add    <manifest-key>... [--label L] [--ttl <duration>]
                             [--registry-auth <file>] [--registry-username U --registry-password P]
node-ctl manifest-key remove <manifest-key>...
node-ctl manifest-key check  <manifest-key>...
node-ctl manifest-key list
  [--socket <uds>]  # 默认读 NODE_CTL_SOCKET env，再退回 /run/sandbox/node-ctl.socket
```

- `<manifest-key>`：租户 manifest key 的 64-hex 原始密钥（非 `e2b_...` 格式的 API key）；也可通过 `MANIFEST_KEY` 环境变量传入
- `--ttl <duration>`：密钥过期时长（如 `24h`、`30d`）；0 或省略 = 永不过期；重复 add 同一密钥会刷新过期时间
- `remove` 同样传入 hex 原始密钥（服务端自动计算 fingerprint 匹配行）

所有操作经 config-socket admin 平面(SO_PEERCRED 鉴权)，不直接访问 DB。

**export-sandbox / import-sandbox**

```
node-ctl export-sandbox <sid> [--to-template] [--keep-source] [--socket S]
  # --to-template  晋升为可复用模板(打印 template_id)
  # 不加           铸造单次迁移 token(base64，打印到 stdout)
  # --keep-source  保留源沙箱(copy)；默认移除(move)
  # 需 E2B_API_KEY 环境变量

node-ctl import-sandbox <token> [--socket S]
  # 在本节点从迁移 token 重建暂停沙箱，打印新 sandbox_id
  # 需 E2B_API_KEY 环境变量；节点须已 manifest-key add 租户密钥
```

**resource**

```
node-ctl resource status [--state /run/node-ctl/state.json]
  # 读 state.json(tmpfs)，打印 zone/pool/allocated/utilization/reservations 计数

node-ctl resource list   [--state ...]
  # 打印全部 reservations JSON

node-ctl resource drain  [--socket <uds>] [--disable]
  # 启用/禁用 drain 模式(drain=true 时新 Admit 被拒绝)

node-ctl resource grant   <sid> --memory <size> [--socket <uds>]
  # 强制扩容(绕过水位/令牌桶)，用于人工干预

node-ctl resource reclaim <sid> --memory <target> [--socket <uds>]
  # 强制缩容(target < current allocatable)
```

`status`/`list` 直接读 `/run/node-ctl/state.json`(只读，无需连接 UDS)；`drain`/`grant`/`reclaim` 经 resource_listen UDS(`nodectl.DefaultSocket`)。

**环境变量**

| 变量 | 用途 |
|------|------|
| `NODE_CTL_ENCRYPTION_KEY` | 覆盖 `encryption_key` 配置(优先) |
| `NODE_CTL_SOCKET` | CLI 子命令默认 config-socket 路径 |
| `E2B_API_KEY` | `export-sandbox`/`import-sandbox` 鉴权 |

### 2.6.2 典型运维操作

**节点上线**

```bash
# 1. 安装二进制 + 依赖
install -m 755 node-ctl sandbox-ctl vswitch-ctl flatten-ctl manifest-ctl /usr/bin/

# 2. 写配置
install -m 640 config.yaml /etc/node-ctl/config.yaml

# 3. 生成加密主密钥(或由 secret 管理系统注入 NODE_CTL_ENCRYPTION_KEY)
openssl rand -hex 32

# 4. 注册租户根密钥(manifest key，64-hex 原始密钥；非 e2b_ 格式的 API key)
node-ctl manifest-key add <64-hex-manifest-key>

# 5. 启动
systemctl enable --now node-ctl.service
```

**密钥轮转(滚动，需一次重启)**

```bash
# 1. 在 encryption_key 前追加新密钥(新密钥:旧密钥)
# 2. systemctl restart node-ctl.service   # 进程不支持 SIGHUP/reload，需重启加载新配置
# 3. 重启后所有新写入(manifest_key add / Admit 等)自动使用新密钥；
#    旧密文按 keytag 匹配旧密钥解密，无需重加密，多密钥共存即可
# 4. 确认业务稳定后，移除旧密钥并再次重启
```

**节点下线 / 软件升级**

```bash
# 停止新准入
node-ctl resource drain

# 等现有沙箱到期或手动迁移(export-sandbox → import-sandbox 至其他节点)
# 确认节点已无运行沙箱(任选其一)：
#   node-ctl resource status   → 查看 reservations 计数降为 0
#   GET /v2/sandboxes          → 列出节点活跃沙箱

# 升级二进制，重启
systemctl restart node-ctl.service

# 恢复准入
node-ctl resource drain --disable
```

**紧急扩/缩容(bypass 水位)**

```bash
# 强制给某沙箱扩 256 MiB(绕过令牌桶)
node-ctl resource grant <sid> --memory 256MiB

# 强制将某沙箱缩回 128 MiB
node-ctl resource reclaim <sid> --memory 128MiB
```

**快照迁移(跨节点移动沙箱)**

```bash
# 在源节点(沙箱已 PAUSED 或自动 pause)
token=$(node-ctl export-sandbox <sid>)

# 在目标节点(需已 manifest-key add 同一租户密钥)
node-ctl import-sandbox "$token"
```

---

## 2.7 公有云集成方案

### 2.7.1 API Key 多租户管理

#### 当前架构的公有云适配缺口

现有模型中，平台方持有 manifest key，客户拿到的是从 manifest key 派生的 api key（`e2b_` 格式 HMAC token）。在公有云多租户场景下，存在以下差距：

| 维度 | 当前机制 | 公有云需要 |
|------|----------|-----------|
| 密钥所有权 | 平台持有 manifest key | 租户感知自己的密钥边界 |
| 吊销粒度 | 整个 group 或重启节点 | 单个 api key 即时吊销 |
| 轮转代价 | 需重启 node-ctl | 零停机轮转 |
| 审计 | 无 key usage 日志 | 每次鉴权可溯源 |
| 多租户隔离 | group 级别 | 租户内还需 project/app 级隔离 |

#### 密钥分层模型

公有云场景建议引入三层密钥，与现有两层校验对齐：

```
租户 root key（平台托管，存 KMS）
    └── group manifest key（每个租户的 group，现有层）
            └── api key（租户自助创建，e2b_ 派生 token，现有层）
```

- **root key**：租户注册时由 KMS（AWS KMS / GCP Cloud KMS / Vault）生成并持有，用于加密 group manifest key。租户不直接接触 root key，只能通过平台 API 触发密钥操作。
- **group manifest key**：与现有 secretbox 加密方案对齐，存 DB，用 root key 加密。平台在 node-ctl 侧持有解密后的 manifest key，继续走现有 `ownsSandbox()` 校验。
- **api key**：租户通过平台 API 自助创建，对应现有 `manifest-key add --ttl --label`。面向租户开放，支持多 key 共存、按单 key 吊销。

#### 即时吊销

现有 `manifest-key remove` 依赖命令行操作和本地 allowlist。公有云需改为：registry 维护全局吊销列表，router 的 `authorize()` 每次校验时查询（利用现有 `auth_cache_ttl` 短 TTL 缓存机制）。吊销写 registry 后推送失效事件到所有节点，不需要重启。

#### 零停机轮转

现有 secretbox 已通过 keytag 机制支持多密钥共存，group manifest key 轮转流程：新建 key → 双写新旧 → 逐步迁移 sandbox → 删旧 key，全程不重启。encryption key 本身的轮转仍需一次 `systemctl restart`（见 2.6.2），公有云场景推荐通过 KMS 在线解密规避（见 2.7.3）。

#### 审计日志

在 cluster-router `authorize()` 和 node-ctl `ownsSandbox()` 两处各打结构化日志：

```json
{"time":"...","event":"auth","result":"ok|fail","api_key_fingerprint":"<sha256[:8]>","sandbox_id":"...","group":"...","layer":"router|node"}
```

fingerprint 取 `SHA256(manifestKey)[:12]`（12 字节 = 24 hex 字符，与 api key 内嵌 fp 一致）可溯源，不暴露原始 key。

#### 双层校验在公有云下的分工

cluster 部署下两层校验粒度不同，**不可互相替代**：

| 层 | 校验粒度 | 实现 | 是否可省略 |
|----|---------|------|-----------|
| cluster-router | group 级准入（api_key 属于该 group） | `GET /op/verify-key?group=` → registry | 否（防未授权准入） |
| node-ctl | sandbox 级归属（api_key 与该 sandbox 的 manifest key HMAC 匹配） | `ownsSandbox()` → `verifyKey()` | **否**（防同 group 内跨沙箱越权） |

公有云多租户下同一 group 内可能承载多个租户 project 的沙箱，node 层的 per-sandbox 校验从"纵深防御"升级为**强制隔离**。

---

### 2.7.2 云服务认证凭据管理

node-ctl serve 在运行时持有四类凭据，各自的存储方式和公有云安全要求不同，下面逐一说明。

---

#### encryption_key — 租户 manifest key 的静态加密主密钥

**是什么**：所有租户的 manifest key 在写入 SQLite 前都用这把 AES-256-GCM 主密钥加密（secretbox）。这是整个系统安全边界的根密钥，泄露后攻击者可解密所有租户的 manifest key。

**当前存储方式**：写在配置文件的 `encryption_key` 字段，或通过 `NODE_CTL_ENCRYPTION_KEY` 环境变量覆盖（环境变量优先）：

```bash
NODE_CTL_ENCRYPTION_KEY=<64-hex-key1>:<64-hex-key2>   # 冒号分隔，首个为活跃密钥
```

**公有云风险**：明文写在配置文件或 systemd EnvironmentFile 里，一旦配置文件泄露，所有租户数据暴露。

**公有云推荐做法**：将主密钥本身交由 KMS 保管（AWS KMS / 华为云 DEW / HashiCorp Vault），节点启动时用实例 IAM Role 调 KMS 解密接口，解密结果只注入内存：

```bash
# ExecStartPre 脚本示例，将 KMS 解密结果写入内存环境变量
export NODE_CTL_ENCRYPTION_KEY=$(aws kms decrypt \
  --ciphertext-blob fileb:///etc/node-ctl/enc-key.bin \
  --output text --query Plaintext | base64 -d | xxd -p -c 64)
exec node-ctl serve
```

节点实例只持有 `kms:Decrypt` 权限，KMS 侧留有每次解密的审计日志，主密钥明文不落磁盘。

---

#### MANIFEST_KEY — 单租户快照加密密钥

**是什么**：每个租户有一个 manifest key（64-hex 原始密钥），用于加密该租户在 manifest store 里的快照数据。它加密存在 DB 里（由 encryption_key 保护），使用时由 node-ctl 解密后传给子进程。

**当前传递方式**：按场景分两种：

| 场景 | 调用命令 | 传递方式 |
|------|---------|---------|
| 沙箱暂停 → 上传已有本地快照（`promote()`） | `sandbox-ctl upload-snapshot` | node-ctl 通过 `cmd.Env` 显式注入 `MANIFEST_KEY=<hex>` |
| 沙箱运行中直接远端快照（`snapshotRemote()`） | `sandbox-ctl snapshot --upload` | 沙箱启动时配置已写入，sandbox-ctl 从沙箱自身读取，node-ctl 不注入 |

两种场景下明文均只存在于子进程环境变量或沙箱运行时内存中，不落磁盘。

**公有云说明**：MANIFEST_KEY 的安全边界取决于 encryption_key——DB 里的密文由 encryption_key 保护，解密后在内存中按需传递。只要 encryption_key 通过 KMS 管控，MANIFEST_KEY 的安全强度随之提升，无需额外处理。

---

#### files_storage AK/SK — 构建上下文对象存储访问凭据

**是什么**：`builder.files_storage` 配置项中的 `access_key/secret_key`，用于对接 S3/OBS 兼容对象存储（存放模板构建时的 COPY 上下文文件）。node-ctl 用这组凭据生成 presign URL，实际上传/下载由客户端或构建沙箱直接与对象存储交互，字节不过 node-ctl。

**当前存储方式**：明文写在配置文件。

**公有云推荐做法**：

```yaml
builder:
  files_storage:
    access_key: ""   # 留空 → SDK 自动走实例 IAM Role / 云厂商凭据链
    secret_key: ""
```

留空时，AWS SDK 会按默认凭据链顺序查找（环境变量 → 实例元数据服务 IMDSv2 → 配置文件），自动获取临时 STS token，无需硬编码 AK/SK。若因合规要求必须使用静态 AK/SK，应通过 Secret Manager（AWS Secrets Manager / 华为云 CSMS）在启动时动态注入，不写入配置文件。

所需最小权限：`s3:PutObject`（presign 客户端上传）、`s3:GetObject`（presign 构建侧下载）、`s3:HeadObject`（判断文件是否已存在）。

---

#### cluster mTLS 证书 — 节点与 registry 的双向认证

**是什么**：集群模式下，node-ctl 作为 node-link 客户端向 cluster-ctl registry 发起连接，双方通过 mTLS 互认身份（`cluster.tls_cert/tls_key/tls_ca`）。

**当前存储方式**：证书和私钥路径写在配置文件，文件由运营人员手动部署到节点。

```yaml
cluster:
  tls_cert: /etc/node-ctl/node-link.crt   # 节点客户端证书（公钥）
  tls_key:  /etc/node-ctl/node-link.key   # 私钥，权限应设 0400
  tls_ca:   /etc/node-ctl/registry-ca.crt # 用于验证 registry 服务端证书的 CA
```

**公有云推荐做法**：私钥不纳入配置管理仓库（Ansible/Terraform state），推荐通过 cert-manager 或节点 Bootstrap PKI 自动签发并定期轮转。

---

#### 最小权限矩阵

| 主体 | 目标资源 | 所需权限 |
|------|---------|---------|
| node-ctl 实例 Role | files_storage bucket | `s3:PutObject`、`s3:GetObject`、`s3:HeadObject` |
| node-ctl 实例 Role | KMS 主密钥 | `kms:Decrypt`（限定指定 key ARN） |
| 租户（客户端） | files_storage bucket | 无直接权限，通过 presign URL 单次授权 |
| cluster registry | store-ctl | 内部网络直连，不走公有云 IAM |

---

### 2.7.3 私有镜像仓库凭据管理

#### 现有凭据优先级链路

`resolveBuildCreds()` 按以下优先级选取本次构建的拉取凭据，高优先级命中即停：

```
[1] X-Kuasar-Pull-Token（kpt_ token，per-build，manifest-key 封装）
[2] fromImageRegistry.username/password（SDK cleartext，TLS 保护传输）
[3] cluster transient（集群模式，registry 经 node-link Command 下发，不落盘）
[4] manifest_keys.registry_auth_enc（租户默认，docker config.json，secretbox 加密存 DB）
```

**链路[1]：kpt_ 封装 token（最安全，推荐公有云）**

由客户端调 `e2b-key-ctl seal-pull-token` 用 manifest key 封装成 `kpt_<base64url-AES-256-GCM>` 令牌，经 `X-Kuasar-Pull-Token` header 传入；node-ctl 在 `resolveBuildCreds()` 中用 `regcreds.Open(manifestKey, token)` 解封，得到 `Creds{Username, Password, Token}`。封装密钥派生自 `SHA256("kuasar-pull-token-v1:" + manifestKeyHex)`，与租户 manifest key 绑定，其他租户无法伪造。适合短有效期 bearer token（如 ECR 12h token），客户侧每次构建前刷新后重新封装。

**链路[2]：SDK cleartext**

`fromImageRegistry.username/password` 明文在请求体，TLS 保护传输，不单独加密。适合静态 Basic Auth（如 Harbor、自建 registry）。

**链路[3]：cluster transient creds（集群模式专属）**

registry 在下发 `Command{Kind: build}` 时携带 `registry_auth`（docker config.json 格式），node-ctl 存入 `o.clusterBuilds[buildID]`，构建终止（终态）时立即 `delete`，**原始 docker config.json 不写入 DB**（`cluster.md §7.5`）。`CredsForImage()` 按 registry host 匹配（精确 → `"*"` catch-all → first entry）。

**链路[4]：租户默认（DB 存储）**

通过 `node-ctl manifest-key add` 写入 `manifest_keys.registry_auth_enc`（secretbox 加密）：
```bash
# 从 docker config.json 文件导入（多 registry 支持）
node-ctl manifest-key add --registry-auth ~/.docker/config.json <manifest-key>

# 或简单 username/password（组装 catch-all "*" 条目）
node-ctl manifest-key add --registry-username <user> --registry-password <pass> <manifest-key>
```
静态凭据，适合长期有效的静态 Basic Auth；**不适合** ECR/GCR 等短有效期 token。

**凭据到 flatten-ctl 的注入路径**：

```
resolveBuildCreds()
  → Creds JSON 明文赋给 b.RegistryAuth
  → PutBuild() 以 secretbox 加密写入 builds.registry_auth_enc（DB 持久化）
  → runBuildUnit() 将 b 写入内存 o.pend[bid]
  → BuildSpecFor() 从 pend[bid].build.RegistryAuth 读取明文 JSON（json.Unmarshal，非解密），调 c.FlattenEnv()
  → BuildSpec.Env["FLATTEN_REGISTRY_TOKEN"|"FLATTEN_REGISTRY_USERNAME/PASSWORD"]
  → configsock 下发给 run-builder
  → builder.buildPipeline 注入 guest flatten-ctl 的执行环境（不写磁盘）
```

---

#### 公有云适配缺口

| 缺口 | 影响 | 当前能否绕过 |
|------|------|------------|
| ECR/GCR/SWR token 12h 自动过期 | 链路[4]静态 token 过期后构建失败 | 链路[1]客户侧每次刷新后重新封装 kpt_，可绕过但运维负担重 |
| bearer token 无过期检测 | `kpt_` 或链路[4]的 token 过期不提前报错，直到 flatten-ctl 拉取失败才暴露 | 无 |
| 集群模式 ECR token 刷新时序 | registry 侧需在每次 build Command 下发前主动刷 ECR token（12h TTL 内可能多次构建） | 需 registry 侧实现 |
| 多 registry 匹配局限 | `CredsForImage` 不支持前缀/正则匹配，pull-through cache 等场景需手动配 `"*"` catch-all | 可用 `"*"` 兜底但失去精确控制 |
| 凭据审计 | 哪次构建用了哪组凭据，是否拉取成功，无日志 | 无 |

---

#### 设计方案

##### 方案 A：平台侧 token 代理（集群模式，推荐）

cluster registry 在下发 build Command 前，由平台侧实现 token 代理逻辑，node-ctl 无需改动：

```
租户构建请求 → registry
  ↓
registry 解析 fromImage，提取 registry host
  ↓
检测 host 类型（ECR / GCR / SWR / Harbor / …）
  ↓
[ECR] 调 GetAuthorizationToken（registry 侧 IAM Role，无需 AK/SK）
[GCR] 调 generateAccessToken（Workload Identity）
[SWR] 调 SWR GetPermanentAccessKey 或临时 token
  ↓
封装为 docker config.json:
  {"auths":{"<registry-host>":{"token":"<fresh-bearer>"}}}
  ↓
随 Command.RegistryAuth 下发节点（transient，不落盘）
```

优点：
- node-ctl 代码零改动，复用链路[3]
- token 在 registry 侧统一刷新，节点上不留任何仓库凭据
- registry 侧集中审计（哪个 group/build 拉了哪个镜像）

##### 方案 B：node-ctl 感知已知云仓库，自动刷新（standalone 模式补充）

对无 cluster registry 的 standalone 部署，在 `BuildSpecFor()` 下发 `BuildSpec` 前插入刷新逻辑：

```go
// 伪代码：build.go BuildSpecFor() 路径
creds, _ := regcreds.Open / json.Unmarshal(b.RegistryAuth)
creds, _ = regcreds.RefreshIfNeeded(creds, b.FromImage, o.cfg)
// RefreshIfNeeded：识别 ECR/GCR/SWR host → 用 Instance Role 换临时 token
```

`regcreds.RefreshIfNeeded` 的识别规则：

| registry host 模式 | 刷新方式 |
|---|---|
| `*.dkr.ecr.*.amazonaws.com` | `aws ecr get-login-password`（AWS SDK，走 Instance Role） |
| `gcr.io`、`*.pkg.dev` | `gcloud auth print-access-token`（或 Workload Identity metadata） |
| `*.myhuaweicloud.com/swr*` | `swr token API`（华为云 Agency） |

刷新结果只注入当次 BuildSpec 的 Env，不回写 DB，token 生命周期与构建进程一致。

##### bearer token 过期检测（两方案共用）

在 `BuildSpecFor()` 读取 `RegistryAuth` 后，解析 bearer token 的 JWT `exp` 字段（或 ECR token 的 base64 前缀）做提前检测：

```go
if c.Token != "" && regcreds.TokenExpired(c.Token) {
    return nil, "", false,
        fmt.Errorf("build %s: registry token expired; re-seal a fresh token via kpt_ or trigger refresh", bid)
}
```

提前在 `BuildSpecFor()` 阶段失败，比等 flatten-ctl 拉取失败的错误信息更明确。

##### 凭据审计日志

在 `resolveBuildCreds()` 出口打结构化日志（不记录凭据明文，只记录来源和 registry host）：

```json
{
  "event": "build_creds_resolved",
  "build_id": "...",
  "registry_host": "123456789.dkr.ecr.us-east-1.amazonaws.com",
  "creds_source": "pull_token|sdk_cleartext|cluster_transient|tenant_default|anonymous",
  "token_type": "bearer|basic|none"
}
```

##### 多 registry 精确匹配扩展

`CredsForImage` 当前匹配顺序：exact host → `"*"` → first entry。公有云场景补充**前缀匹配**，处理 ECR pull-through cache（`public.ecr.aws/`）等情形：

```go
// 在 exact 命中和 "*" 之间插入前缀匹配
for prefix, entry := range d.Auths {
    if strings.HasPrefix(host, prefix) { return entryCreds(entry) }
}
```

这是纯 `regcreds` 包内改动，不影响其他路径。

---

#### 凭据生命周期汇总

> 注：四条链路的 `resolveBuildCreds()` 解析结果（Creds JSON）均经 `PutBuild()` 以 secretbox AES-256-GCM 加密写入 `builds.registry_auth_enc`；下表"原始凭据保护"列描述的是该凭据**到达 node-ctl 之前**的安全特征。

| 凭据来源 | 原始凭据保护 | 解析后 Creds 持久化 | 公有云推荐度 |
|---------|------------|-------------------|------------|
| kpt_ pull token（链路[1]） | AES-256-GCM（manifest key 派生封装） | secretbox 加密存 builds.registry_auth_enc | ★★★★★ |
| SDK cleartext（链路[2]） | 仅 TLS 传输保护，请求体明文 | secretbox 加密存 builds.registry_auth_enc | ★★☆☆☆ |
| cluster transient（链路[3]） | node-link TLS；原始 docker config.json 仅存内存（clusterBuilds map），不落盘 | 解析后 Creds secretbox 加密存 builds.registry_auth_enc | ★★★★☆ |
| 租户默认 DB（链路[4]） | manifest_keys.registry_auth_enc（secretbox AES-256-GCM） | secretbox 加密存 builds.registry_auth_enc | ★★☆☆☆（不适合短效 token） |
| 方案 A：platform proxy | 同链路[3]，原始 docker config.json 不落盘 | 解析后 Creds secretbox 加密存 builds.registry_auth_enc | ★★★★★ |
| 方案 B：node-ctl 自动刷新 | 刷新 token 只注入当次 BuildSpec.Env，不回写 DB | secretbox 加密存 builds.registry_auth_enc（刷新前来源决定） | ★★★★☆ |

---

### 2.7.4 快照远端存储

#### 存储架构概览

快照涉及两条独立的存储路径：

| 路径 | 用途 | 后端 | 配置项 |
|------|------|------|--------|
| 沙箱快照（pause/export） | microVM 内存+磁盘状态，以 `manifest://<key>` 寻址 | manifest store（store-ctl daemon） | `manifest_config`（yaml 文件路径） |
| COPY 构建上下文 | 模板构建时 Dockerfile COPY 指令的文件包 | S3/OBS 兼容对象存储 | `builder.files_storage` |

两条路径字节均不过 node-ctl 控制面（manifest store 由 sandbox-ctl 直接写；files_storage 由客户端通过 presign URL 直传）。

#### 沙箱快照存储（manifest store）

**Checkpoint 模式**由 `checkpoint.mode` 控制：

```yaml
checkpoint:
  mode: local        # local（默认）| remote
  local_dir: /var/lib/sandbox-saved   # mode=local 时的本地目录
  deep_idle_sec: 0   # >0 时：PAUSED 状态超过此秒数自动 promote 为 SAVED（remote；集群模式专用）
```

- **local**（默认）：`sandbox-ctl snapshot --output <dir>` 写节点本地目录，快照只能在本节点恢复（node-bound）。调用 `export-sandbox` 时按需 promote 为 remote。
- **remote**：`sandbox-ctl snapshot --upload` 直接上传 manifest store，`SnapshotRef` 记为 `manifest://<64-hex-key>`，可跨节点迁移。

**manifest store 配置**（由 `manifest_config` 指向的 YAML 文件描述）：

```yaml
store:
  endpoint: "store-ctl.internal:9100"   # store-ctl daemon gRPC 地址
  pool: 8
  timeout: "10s"
cache:
  endpoint: "cache-ctl.internal:9101"   # 可选；空则 store-as-cache
chunker:
  mode: cdc       # "cdc"（默认，FastCDC 内容定义分块）| "fixed"
crypto:
  chunk: aes      # 唯一合法值："aes"（AES-256-CTR 收敛加密 + AES-256-GCM key table）
  manifest: aes
```

node-ctl 本身不感知 store-ctl 的后端；store-ctl 可接入任意对象存储（见下）。

**快照生命周期**：

```
pause ──► snapshotLocal()  ──► /var/lib/sandbox-saved/<sid>/  (local)
      └──► snapshotRemote() ──► sandbox-ctl snapshot --upload ──► store-ctl ──► 对象存储
                                                                   └── manifest://<key>

export-sandbox (PAUSED + local ref)
  └──► promote() ──► sandbox-ctl upload-snapshot ──► store-ctl ──► manifest://<key>
  └──► mintSandboxToken() ──► base64 SandboxToken{snapshot_ref: "manifest://..."}

deep_idle_sec 超时 ──► PAUSED → SAVED auto-promote（集群场景）
```

**跨节点迁移校验**：`importSandboxWithKey()` 在恢复前校验 `tok.RuntimeDigest`（目标节点 guest runtime erofs 的 SHA256），runtime 不匹配则拒绝导入，防止 ABI 不兼容导致的恢复失败。

#### COPY 构建上下文存储（files_storage）

对接 S3/OBS 兼容对象存储，控制面只生成 presign URL，字节不过控制面：

```yaml
builder:
  files_storage:
    endpoint: "https://obs.cn-north-4.myhuaweicloud.com"   # 留空则用 AWS 默认端点
    region: "cn-north-4"
    bucket: "my-build-contexts"
    prefix: "node-ctl/"                 # 可选键前缀
    access_key: ""                      # 留空 → AWS/Huawei 默认凭据链（推荐）
    secret_key: ""
    force_path_style: false             # minio/versitygw 需设为 true
    presign_expiry: "1h"                # PUT presign 有效期
```

未配置 `files_storage` 时，COPY 步骤返回 501；本地开发可指向 versitygw/minio 网关（设 `force_path_style: true`）。

#### 对接云文件存储服务（NFS/SFS Turbo）

对于需要在多节点共享快照目录（如 `checkpoint.local_dir` 或 `paths.base_root` 下的模板层缓存）的场景，可将 NFS/SFS Turbo 挂载为节点本地路径，node-ctl 无感知：

```
# /etc/fstab 示例（华为云 SFS Turbo）
<sfs-turbo-endpoint>:/  /var/lib/sandbox-saved  nfs  vers=3,timeo=600,nolock  0 0
```

注意：NFS 挂载路径下的快照不具备 content-addressable 寻址（与 manifest store 不同），只适合"单节点 local 快照 + 跨节点共享 NFS 挂载"的简单迁移场景；**大规模多节点场景推荐使用 manifest store 远端模式（`checkpoint.mode: remote`）。**

---

# 3. DFX 设计

## 3.1 性能

### 关键路径延迟目标

| 路径 | P50 | P99 | 瓶颈 |
|------|-----|-----|------|
| 沙箱冷启动(快照缓存命中) | < 100 ms | < 500 ms | microVM restore IOPS |
| 沙箱冷启动(快照未缓存) | < 800 ms | < 3 s | manifest store 拉取 |
| PAUSED → RUNNING(Resume) | < 80 ms | < 200 ms | 内存快照恢复 |
| 路由查找(in-memory map) | < 1 µs | < 5 µs | mutex + map O(1) |
| eBPF TC 转发增量延迟 | < 50 µs | < 200 µs | 内核 TC hook |
| 用户态 ReverseProxy 转发延迟 | < 200 µs | < 500 µs | httputil.ReverseProxy splice |
| 准入决策(绿区) | < 1 ms | < 5 ms | UDS RTT |
| 准入排队等待(黄区) | — | < 50 ms | 令牌桶补充或 Settled 唤醒 |
| 构建日志分页拉取 | < 10 ms | < 50 ms | journalctl 子进程 |

### 性能设计细节

**内存超售密度计算**：

```
单节点 512 GiB
host_reserved.memory = 16 GiB(默认)，operational_margin_factor = 0.10
allocatable_pool = (512 - 16) × (1 - 0.10) ≈ 446 GiB
startup_pool     = allocatable_pool × startup_factor(0.50) ≈ 223 GiB

每沙箱 capacity = 8 GiB，空闲态 RSS floor ≈ 128 MiB
空闲态理论密度 = 446 GiB / 128 MiB ≈ 3 570 个
实际瓶颈为 startup_pool(每次启动约消耗 effective_startup_budget ≈ 1 GiB)：
  → 最多约 223 个沙箱并发处于 pre-settled 阶段

稳态密度目标 ≥ 400 个(60× 基于 capacity=8 GiB 基准)
```

**in-memory 路由缓存(internal 模式)**：

`Orchestrator.reg`(`map[string]*types.Sandbox`，`sync.Mutex` 保护)是数据面热路径的一级缓存。缓存条目不可变(copy-on-write)：任何字段修改均通过 `mutateCached()` 在 mutex 内复制并替换指针，持有旧指针的读者永远不会观察到撕裂写：

```go
func (o *Orchestrator) mutateCached(id string, fn func(*types.Sandbox)) *types.Sandbox {
    o.mu.Lock(); defer o.mu.Unlock()
    cur := o.reg[id]; if cur == nil { return nil }
    nb := *cur          // 值拷贝
    fn(&nb)
    o.reg[id] = &nb     // 原子替换指针
    return &nb
}
```

`Route()` 先查 `reg`(O(1)，hot path)；cache miss 或 state≠running 时走 singleflight resume。

**per-sid singleflight(resume 并发去重)**：

`flightGroup` 是本地实现的 singleflight(`internal/orch/singleflight.go`，无外部依赖)，对同一 sid 的并发 `Do()` 调用：第一个执行 fn，其余等待 `done` channel；所有调用共享同一结果。保证 N 个并发 Wake 帧或数据面请求对同一暂停沙箱只触发一次 cloud-hypervisor restore：

```go
o.sf.Do(sid, func() error { return o.resumeIfPaused(ctx, sid) })
```

**external 模式路由表(proxy worker 侧)**：

`routetable.Table`(`sync.Mutex + map[string]RouteEntry`)持有 routesync 下发的完整路由快照。`Resolve()` 是数据面热路径：

```
Resolve(ctx, sid):
  1. waitSynced()  — 首次全量同步完成(syncedCh 已 close)后立即返回，否则等待
  2. Lookup(sid)   — mutex + map O(1)；若 state=running 立即返回
  3. Wake(sid)     — 发 Wake 帧(FNV 去重，wakeCh 容量 1024)
  4. waitRunning() — 挂载 per-sid waiter channel；ApplyUpsert/Delete notifyLocked() 唤醒
```

park 超时由 Policy.ParkTimeoutMS(orchestrator 推送)或 `--park-timeout` 兜底控制。

**proxy 转发(httputil.ReverseProxy)**：

```go
tr := &http.Transport{
    DialContext:         dialRoute,        // 实际 dial(UDS / TCP)在 ctx 里携带 Route
    ForceAttemptHTTP2:  false,            // envd h1/h2c 均可；避免强制 h2
    MaxIdleConnsPerHost: 64,              // 每个 poolKey 最多 64 个 keep-alive 连接
    ResponseHeaderTimeout: 0,            // 无超时(流式 RPC)
}
rp := &httputil.ReverseProxy{
    Director:      setPoolKey,           // req.URL.Host = poolKey(route) 控制连接池分区
    Transport:     tr,
    FlushInterval: -1,                   // 立即 flush(streaming-safe，无内部缓冲)
}
```

`poolKey(route)` 按实际后端路径分区(TCP 为 `floatingip:port`，UDS 为 path 摘要)，防止不同沙箱/端口之间复用 keep-alive 连接导致请求路由到错误后端。

**external 模式 gateway(FNV-32a 亲和分发)**：

external 模式下 node-ctl 的 API listener 上落的数据面请求经 gateway 转发给已注册的 proxy worker。worker 通过 config-socket plugin 平面动态注册，**无静态 socket 列表**。worker 选取策略：

```go
h := fnv.New32a()
h.Write([]byte(sid))
sock := targets[h.Sum32() % uint32(len(targets))]
```

相同 sid 总是命中同一 worker(连接亲和)，保证 per-sid parking waiter 不分散；worker 增减时亲和自动重分布。CONNECT 隧道走独立 `relayConnect()` 路径(httputil.ReverseProxy 不支持 CONNECT 隧道)。

**SO_REUSEPORT 水平扩展(external 模式)**：

K 个 proxy worker 进程共享同一 `--data-listen` 端口，由 `listenReusePort()` 在 socket 创建时设 `SO_REUSEPORT`：

```go
unix.SetsockoptInt(int(fd), unix.SOL_SOCKET, unix.SO_REUSEPORT, 1)
```

内核按四元组哈希在 worker 间负载均衡，TLS 握手 CPU 线性扩展，无单点瓶颈。每个 worker 维护独立路由缓存(routesync 独立订阅)，worker 增减只影响自身缓存不影响其他。

**Prometheus metrics 设计**：

`internal/metrics.M` 为依赖最小化的纯计数器注册表(无 histogram，无客户端库)，每个 counter 一个 `atomic.Int64`，写路径仅一次原子操作。counter 名支持内联标签(`data_requests_total{result="ok"}`)。`/metrics` handler 在导出时加 mutex 快照所有值后排序输出，避免长时间持锁。当前实际导出的 counter 包括：

```
data_requests_total{result}        # result: ok | notfound | unauthorized | upstream_error | ...
gateway_forward_total{result}      # external 模式 gateway 转发结果
```

**routesync 重连**：

订阅者指数退避重连(初始 200 ms，上限 5 s)；重连必须走全量重同步(`BeginSync` → Upsert × 全量 + `Bookmark`，清除未出现的旧条目)。重同步期间旧路由表仍对外服务(不清空)，保证路由可用性不因重连产生短暂全空。节点路由规模通常在百量级，全量同步耗时毫秒级，重连开销可控。

## 3.2 可靠性

### 控制面与数据面生命周期解耦

```
故障场景                       数据面影响                         控制面恢复路径
────────────────────────────────────────────────────────────────────────────────
node-ctl serve 崩溃/重启      内部模式(proxy in-process)：       systemd 自动重启
(含资源控制器，同进程)       已建连接中断；新连接排队             → Reconcile() 收养活跃沙箱
                               外部模式(worker 独立进程)：        孤儿 running 行标 dead
                               worker 继续服务已建连接；
                               新连接 park 至 serve 恢复(30 s)

proxy worker 崩溃             该 worker 所有连接断                 systemd 重启 worker
(external 模式)              其他 worker 继续服务                worker 重连 → BeginSync 重同步

sandbox-ctl / 沙箱进程崩溃   该沙箱连接断，其他沙箱不受影响       Reconcile 对账：无 systemd 单元
                                                                   → teardown + SetState(dead)

routesync 断流                worker 路由表停止更新                指数退避重连(200 ms→5 s)
                               已建连接按旧路由继续服务             + BeginSync 全量重同步

node-link 断流(集群)        registry 暂失本节点路由视图           节点重连(200 ms→5 s 退避)
                               本节点沙箱不受影响                   重注册 + 全量路由上报
```

### 节点重启对账(Reconcile)

`serve` 在绑定监听之前调用 `Reconcile(ctx)`，以 `ListUnitsByPatterns("sandbox-runner@*.service")` 的结果为存活权威(`cmd/node-ctl/main.go`)：

```go
// internal/orch/orch.go — 精简逻辑
units, _ := lc.List(ctx, "sandbox-runner@*.service")
alive := map[string]bool{} // ActiveState=="active" || "activating"

st.RangeByState(ctx, StateRunning, func(sb) {
    if alive[sb.ID] {
        o.cache(sb)   // 收养：重挂路由，TTL 继续
    } else {
        dead = append(dead, sb)
    }
})
for _, sb := range dead {
    o.teardown(ctx, sb)                 // 停单元 / 释放网络 / 删运行目录
    st.SetState(ctx, sb.ID, StateDead)  // 行保留，state 更新为 dead
}
```

- **paused 行不受影响**：无运行单元属正常，不计入 dead；
- **整机重启**：`run_root` 是 tmpfs，所有 running 行无对应单元 → 全部标 dead；paused 行保留(快照持久化在 `checkpoint.local_dir`)。

### Reaper — TTL 自动暂停

`go core.Reaper(ctx, 5*time.Second)` 随 serve 启动为后台 goroutine：

- **每 5 s** 扫描 `state=running` 且 `deadline_unix ≤ now` 的沙箱，调用 `pauseSandbox()`(TTL 到期 → 自动暂停，不 kill)；
- 同轮触发 `reapDeepIdle()`：paused 且深度闲置超时(cluster 选项)→ 提升为 SAVED 快照；
- 同轮裁剪过期的 manifest key 行(`PruneExpiredManifestKeys`)。

### 资源控制器重启恢复

资源控制器(`internal/nodectl`)**随 serve 进程内启动**(`startResourceController`，`cmd/node-ctl/resource_daemon.go`)，不是独立 systemd 单元。serve 重启时完整重建。

**Persister — 进程级持久化**

`Persister` 将 `State`(含全部 `Reservations`)原子写入 `/run/node-ctl/state.json`(write+fsync+rename tmp，`internal/nodectl/persister.go`)：

- `/run` 是 tmpfs → 进程重启后文件**仍存在**；整机重启后文件**消失**(冷启动)；
- `Persister.Load()` 返回 `(nil, nil)` 表示文件不存在(冷启动)，非致命错误；
- 每次状态变更(Admit / Settled / Release / IdleSweeper sweep)均调用 `Flush()`。

**重启恢复流程**

```
serve 重启
  └─ startResourceController()
       ├─ persister.Load() → 恢复 Reservations(Conn=nil)
       ├─ Server.Listen() + Serve()   // 开始接受连接
       ├─ go sweeper.Run()
       └─ go reclaimer.Run()

sandbox-ctl(微 VM 仍运行)
  └─ 检测 resource_listen.socket 连接断
       └─ 重拨 → TypeReattach(token)
            └─ handleReattach(): res.Conn = conn  // 重新绑定
```

**IdleSweeper — 未重连清理**(每 10 s，`internal/nodectl/server.go`)

```
IsPreSettled(stage) && now - StageEnteredAt > StartupTTL (30 s)
  → 创建阶段超时未 Settled，释放 reservation

res.Conn == nil && now - LastHeartbeatAt > 3 × Heartbeat (30 s) = 90 s
  → sandbox-ctl 彻底离线，释放 reservation
  → Admission.PushWake() 解阻队列 + Persister.Flush()
```

预置阶段(IsPreSettled)：`admitted / creating / startup / restoring`；其余阶段(settled / burst / recover)使用心跳阈值。

### SQLite 持久层

SQLite 以 WAL 模式打开(DSN: `?_pragma=journal_mode(WAL)&_pragma=busy_timeout(5000)`)，支持并发读：

| 操作 | 写入语义 |
|------|---------|
| Create | `Put()` INSERT OR CONFLICT DO UPDATE(upsert)；state=running |
| Pause | `SetState(StatePaused)` |
| Kill | `st.Delete()` 完整删除行 |
| Reconcile 孤儿 | `SetState(StateDead)` 行保留，state 更新为 dead |
| 配额检查 | `SELECT COUNT(*) WHERE state IN ('running','paused')` 与 Create 同步执行 |

**仅 `running` / `paused` / `dead` 三态持久化**；CREATING / PAUSING / RESUMING / KILLING 均为内存瞬态。

### 幂等操作

| 操作 | 幂等保证 |
|------|---------|
| 沙箱 create | 单元名含 sid，systemd StartUnit 重复调用幂等 |
| Admit(token) | resource controller 按 token 查 State，重复请求走 Reattach 路径 |
| 构建 trigger | builds 表 CAS(`status='registered'`)，重试安全 |
| snapshot chunk 上传 | manifest-ctl 内容寻址(SHA256)，重复上传幂等 |
| routesync upsert | 路由表按 sid 覆盖，重复 upsert 无副作用 |
| Reconcile() | `cache()` 操作幂等；dead 清理前置 `teardown` 可重入 |

## 3.3 安全

### api_key 鉴权

**Wire format**(76 chars，`internal/apikey/apikey.go`)：

```
api_key = "e2b_" + hex( fp(12) ‖ ts(4) ‖ nonce(4) ‖ mac(16) )

fp(12)   = SHA256(manifest_key)[:12]                  // 指纹：manifest_keys 表非唯一索引
ts(4)    = big-endian uint32(unix_seconds)             // mint 时间，嵌入 HMAC，不独立校验
nonce(4) = CSPRNG 4 bytes                              // 同一 key 可 mint 多个不同 api_key
mac(16)  = HMAC-SHA256(manifest_key, fp‖ts‖nonce)[:16]
```

**验证流程**(`resolveAllowed` / `ownsSandbox`)：

```
1. apikey.Parse(apiKey)    → 提取 fp / ts / nonce / mac(结构校验)
2. AllowedManifestKeysByHash(hex(fp))
   → SELECT key_enc FROM manifest_keys
       WHERE key_hash=? AND (expires_unix=0 OR expires_unix>now)
   → box.DecryptString(key_enc) → manifest_key 明文候选集
3. 对每个候选 manifest_key：
   apikey.Verify(p, raw) = hmac.Equal(recomputed_mac, p.MAC)  // constant-time
4. 未找到匹配 → resolveAllowed 返回 "" → Create/Build 返回 403
   找到匹配   → 返回 manifest_key 明文，用于沙箱/构建授权
```

per-resource 所有权：`ownsSandbox(sb, apiKey) = verifyKey(apiKey, sb.ManifestKey)`——每行携带自己的 manifest key，不需要再查表。

**manifest_key 许可列表**(`manifest_keys` 表，`internal/store/store.go`)：

| 列 | 含义 |
|----|------|
| `key_hash` | hex(SHA256(rawKey)[:12])，非唯一索引，O(1) 查 fp |
| `key_enc` | AES-256-GCM 密文，`box.Encrypt()` |
| `expires_unix` | 0 = 永不过期；Reaper 定期 `PruneExpiredManifestKeys` 清理 |
| `registry_auth_enc` | 加密的租户默认镜像仓库凭据(构建用) |

管理操作经 admin 平面 UDS(`/internal/admin/manifest-keys`)：`op=add|remove|check`；每次操作只返回 fingerprint(24 hex)，不回显 key 明文。

### manifest_key 静态加密(secretbox)

密文格式(`internal/secretbox/secretbox.go`，hex 编码存 TEXT 列)：

```
keytag(4) ‖ gcm_nonce(12) ‖ AES-256-GCM(ciphertext+tag)

keytag = SHA256(rawKey)[:4]   // 选取解密密钥，支持多密钥并存
```

- AES-256-GCM：`cipher.NewGCM(aes.NewCipher(key))`，随机 nonce，每次加密结果不同；
- `encryption_key` 配置为 `:` 分隔多 64-hex 密钥，`[0]` 为活跃加密密钥，其余仅解密(零停机轮转)；
- `NODE_CTL_ENCRYPTION_KEY` 环境变量覆盖配置文件值(部署级密钥注入)。

### manifest_key 全链路不落明文

| 环节 | 存在形式 | 生命周期 |
|------|---------|---------|
| 数据库(manifest_keys / sandboxes / builds 表) | secretbox 密文(keytag‖nonce‖ct‖tag) | 永久加密，按密钥版本轮转 |
| serve 进程内存 | 解密后明文(Go heap，查询时临时) | 进程重启清除 |
| config-socket task 平面(LaunchSpec.Env) | 明文 env frame，仅经 UDS 传输，不落磁盘 | exec-replace 后生命周期结束 |
| guest(MMDS 获取) | `mmds_key`(派生值)，**非** manifest_key | guest 进程生命周期 |
| 磁盘任意位置 | **从不存储明文** | — |

`mmds_key = HMAC-SHA256(manifest_key, "kuasar-mmds-v1:"+sid)`(`internal/keys/keys.go`)——单向派生，guest 无法反推 manifest_key。

### 数据面访问 token(X-Access-Token)

Create 时生成：`envd_access_token = keys.MintToken()` = random 32-byte hex(`internal/keys/keys.go`)。

**代理鉴权**(`internal/proxy/auth.go`)：

```go
// 每个数据面请求
tok := r.Header.Get("X-Access-Token")
subtle.ConstantTimeCompare([]byte(tok), []byte(route.AccessToken)) == 1
```

- `proxy.auth` 三模式：`off`(不校验)/ `log`(校验失败仅记日志)/ `enforce`(默认，返回 401)；
- `mmds.enabled=false` 时配置验证强制要求 `proxy.auth=enforce`(proxy 是唯一数据面门)；
- 豁免：bare profile(`route.AccessToken=""`，无 envd 无 token)；`?signature=` 查询参数(envd 预签名文件 URL，envd 自行验签)。

### MMDS 安全(内置 Firecracker MMDS v2)

MMDS 服务(`internal/mmds/mmds.go`)使用两阶段流程，让 guest 在不持有 manifest_key 的情况下安全获取 accessTokenHash：

```
PUT /latest/api/token (guest 请求，source IP 即 floatingIP)
  → 按 floatingIP 解析 sid(parking 至路由同步，最长 30s)
  → mmdsSecret = HMAC-SHA256(manifest_key, "kuasar-mmds-v1:"+sid)
  → session_token = sid + "." + hex(HMAC-SHA256(mmdsSecret, sid))
  → 返回 session_token(对 guest 不透明，无法伪造)

GET / (X-metadata-token: session_token)
  → verifyToken: Cut sid.sig → MmdsSecret(sid) → hmac.Equal(sig, sign(sid,secret))
  → 返回 {instanceID: sid, envID: templateID,
           accessTokenHash: hex(SHA512(envd_access_token))}
```

所有 proxy worker 用相同算法独立派生 `mmdsSecret`——无共享状态，无单点。

### config-socket 平面鉴权(SO_PEERCRED)

config-socket(`/run/sandbox/node-ctl.socket`，`0600`)用 `SO_PEERCRED` 在 accept 时捕获对端 pid，per-plane 执行不同策略(`internal/configsock/server.go`)：

| 平面 | 路径 | 鉴权方式 |
|------|------|---------|
| task | `POST /internal/task/launchspec` | `SO_PEERCRED pid == readPID(sb.RunDir/sid.pid)` |
| admin | `/internal/admin/manifest-keys` | `pid ∈ AdminPidfile` 或(Pidfile 未配置时)socket 0600 权限 |
| plugin | `PUT /internal/plugin/{id}/register` | `pid ∈ PluginPidfile` 或 socket 0600 权限 |
| api | 其余所有路径 | `X-API-KEY` HMAC 验证(由 api handler 内部执行) |

task 平面是 manifest_key 的唯一出口(`LaunchSpec.Env`)，pid 校验防止任意进程读取密钥。

### 构建安全边界

- `MANIFEST_KEY` 仅经 config-socket UDS(task 平面)传入 LaunchSpec.Env，不落磁盘，不经 HTTP 控制面；
- guest 仅通过 MMDS 获取 `accessTokenHash`(SHA512 派生值)，无法反推 `envd_access_token` 或 `manifest_key`；
- 租户镜像凭据(`FLATTEN_REGISTRY_*`)经 execve env 进入 guest 运行时，与 `MANIFEST_KEY` 分离，不在 HTTP 控制面传输；
- presigned 对象存储 URL 仅对属主 template_id 签名，桶私有，SDK 客户端无持久凭据。

### 网络隔离

- vswitch 通过 iptables + BPF 实现 guest 间 L2 隔离，guest 只能经 floatingip 访问宿主服务；
- 控制面(`api.listen`)与数据面(external worker 数据口)可分离监听，减少攻击面；
- node-link(集群接入)生产走 mTLS，独立于 `api.listen` 的 TLS 配置。

## 3.4 可运维

### 变更方式

node-ctl **不支持**配置热重载(无 SIGHUP 处理)——大多数配置变更需重启 `serve`。例外(无需重启)：

| 变更类型 | 操作 | 说明 |
|----------|------|------|
| manifest key 许可列表 | `node-ctl manifest-key add/remove` | admin 平面 UDS，即时生效 |
| 密钥轮转(静态加密) | 向 `encryption_key` 前置新 64-hex key，重启 serve | 旧密钥保留解密能力，存量密文无需迁移 |
| 节点排水(停止新准入) | `node-ctl resource drain` | resource controller admin UDS，即时生效 |
| 恢复准入 | `node-ctl resource drain --disable` | 同上 |
| 软件升级 | `resource drain` → 等存量沙箱到期/迁移 → 升级 → `systemctl restart node-ctl` | paused 沙箱快照保留在磁盘 |
| proxy.mode 变更 | 修改 config → 重启 serve | 需停止当前所有连接 |

### 结构化日志

serve 使用 `log/slog`(Go 标准库)输出到 stderr，由 systemd 收集到 journald：

```
# TextHandler 输出格式(默认，key=value 风格)：
time=2026-06-24T10:00:00Z level=INFO msg="sandbox admitted" sid=abc123 template=e2b-snp-<key>

# 资源控制器模块日志前缀(log.Printf)：
[node-ctl resource] admit ab12cd34 sid=abc123 initial_alloc=536870912
[node-ctl resource admit] ...
[node-ctl resource sweep] sweep: token ab12cd34 sid=abc123 exceeded startup TTL, releasing
[node-ctl resource reclaim] reclaim sid=abc123 zone=high rss=... alloc=...→... delta=...
```

查看日志：
```bash
journalctl -u node-ctl.service -f
journalctl -u node-proxy@0.service -f   # external 模式 proxy worker
```

### Prometheus Metrics

opt-in：配置 `proxy.metrics_listen: ":9900"`，`GET http://:9900/metrics` 返回 Prometheus text 格式。

`internal/metrics.M` 仅支持 counter(`atomic.Int64`，无 histogram/gauge)。当前注册的 counter：

```
# 数据面请求(proxy.go / connect.go)
data_requests_total{result="ok|badrequest|notfound|denied|unauthorized|upstream_error|route_error|off"}

# 外部模式网关转发(gateway.go)
gateway_forward_total{result="ok|no_worker|error"}

# 路由器请求(router/router.go)
router_requests_total{plane="control|data"}
router_requests_total{plane="data",result="bad_gateway"}
router_requests_total{result="auth_reject"}
```

proxy worker(`node-ctl proxy`)通过独立的 `--metrics-listen` 标志启用同一端点。

### 资源控制器审计日志

`Auditor`(`internal/nodectl/audit.go`)向 `logging.audit_path`(默认 `/run/node-ctl/audit.log`，**tmpfs**，宿主重启后丢失)追加写入，格式为一行纯文本：

```
<RFC3339Nano> <event> <key>=<val> ...
```

覆盖事件：

| 事件 | 触发时机 | 关键字段 |
|------|---------|---------|
| `admit_queued` | Admit 进排队 | `sid`, `pos`(队列位置), `block`(阻塞原因)|
| `admit` | Admit 完成 | `token`, `sid`, `initial_alloc`, `cap_mem`, `floor_mem`, `effective_startup` |
| `release` | sandbox-ctl Release | `token`, `sid`, `reason`, `alloc_at_release`, `pre_settled` |
| `reclaim` | ActiveReclaimer 回收 | `sid`, `zone`, `rss`, `alloc` 前后值, `delta` |

> 注：Auditor 仅覆盖资源控制器事件；控制面 CRUD(create/kill/pause)无专用审计日志，由 journald(slog)记录。

### 构建日志 API

`GET /builds/{tid}/{bid}/status?logsOffset=N` 返回构建进度日志(SDK 轮询)，后端实现(`internal/orch/buildlogs.go`)：

```bash
journalctl \
  _SYSTEMD_UNIT=sandbox-builder@<bid>.service \
  SYSLOG_IDENTIFIER=build \
  --output=json --no-pager \
  --output-fields=MESSAGE,PRIORITY,__REALTIME_TIMESTAMP
```

- 日志写入方：`run-builder` 将里程碑 + RUN 步骤输出 + flatten 进度写入 journald(`journal.Send(..., SYSLOG_IDENTIFIER=build)`)；
- guest console(`SYSLOG_IDENTIFIER=console`)和 sandbox-ctl 自身日志**不**进入 SDK 视图；
- `priorityToLevel`：PRIORITY ≤ 3 → `"error"`，4 → `"warn"`，≥ 7 → `"debug"`，其余 → `"info"`；
- journalctl 失败(非 systemd 环境、单元未启动)→ 返回空列表，不影响构建状态本身。

### 诊断命令

```bash
# 资源控制器状态(读 state.json，输出 zone/pool/utilization/reservations 数量)
node-ctl resource status [--state /run/node-ctl/state.json]

# 所有 Reservations 详情(JSON)
node-ctl resource list [--state /run/node-ctl/state.json]

# 排水 / 恢复准入
node-ctl resource drain [--socket /run/node-ctl/resource.sock]
node-ctl resource drain --disable

# 手动调整单个 sandbox 内存配额(仅 resource controller 启用时有效)
node-ctl resource grant  <sid> --memory 256MiB
node-ctl resource reclaim <sid> --memory 128MiB

# manifest key 许可列表管理
node-ctl manifest-key add    [--label L] [--ttl 24h] <MANIFEST_KEY>
node-ctl manifest-key remove <MANIFEST_KEY>
node-ctl manifest-key check  <MANIFEST_KEY>
node-ctl manifest-key list

# 配置验证 / 生成模板
node-ctl config --config /etc/node-ctl/config.yaml   # normalize + validate, re-emit
node-ctl config --template                            # 输出带注释的配置骨架

# 版本信息
node-ctl version

# 构建日志(直接查 journald)
journalctl _SYSTEMD_UNIT=sandbox-builder@<bid>.service SYSLOG_IDENTIFIER=build -f

# 沙箱单元状态
systemctl status sandbox-runner@<sid>.service

# 强制重新对账(重启 serve 自动触发 Reconcile)
systemctl restart node-ctl
```

---

# 4. FE与US分工

## FE-1：沙箱生命周期管理

### FE-1.1：沙箱 CRUD + 生命周期管理

**范围**：create / list / get / kill / pause / resume / timeout 端点，systemd unit 编排，状态机实现，重启对账。

| US | UserStory | 验收标准 |
|----|-----------|---------|
| US-1.1-1 | 作为 AI 应用开发者，我能在 500ms 内从快照模板启动一个沙箱并获得连接 URL | P99 < 500ms，从 POST /sandboxes 到 state=running |
| US-1.1-2 | 作为平台运维，我能查看节点上所有沙箱的状态(包括 paused) | GET /sandboxes 返回完整列表，含 paused 行 |
| US-1.1-3 | 作为 AI 应用开发者，我能设置沙箱的存活 deadline，到期自动清理 | DELETE 不触发时，deadline 到期后沙箱标 dead，资源释放 |
| US-1.1-4 | 作为平台运维，serve 重启后已运行沙箱自动收养，客户端连接不断 | 收养时间 < 30s，eBPF 数据面连接无中断 |

### FE-1.2：自动挂起与按需唤醒

**范围**：idle 检测、auto-suspend 流程、park 队列、PAUSED 路由标记、single-flight Resume。

| US | UserStory | 验收标准 |
|----|-----------|---------|
| US-1.2-1 | 作为平台，空闲超过阈值的沙箱自动暂停，释放宿主内存 | idle > suspend_after → state=paused，RSS 释放可观测 |
| US-1.2-2 | 作为 AI 应用开发者，向 paused 沙箱发请求时，沙箱透明唤醒，请求不丢失 | Resume P99 < 200ms，park 队列中请求在 wake 后正常转发 |
| US-1.2-3 | 作为平台，多个并发请求唤醒同一沙箱时，只触发一次 Resume | singleflight 合并，cloud-hypervisor restore 只调用一次 |

### FE-1.3：Secure 模式与 CA Bundle 装配

**范围**：mmds.enabled 激活 Secure 姿态、MMDS v2 两段式 re-key、envd access token 铸造与下发、CA Bundle 注入 guest `/etc/ssl/certs/`、互联网访问开关(`network.internet=false`)。

| US | UserStory | 验收标准 |
|----|-----------|---------|
| US-1.3-1 | 作为企业管理员，我能启用 Secure 模式，使未携带 sandbox token 的 envd 请求被拒绝 | `mmds_enabled=true` 创建后，无 X-Access-Token 的 envd RPC 返回 401 |
| US-1.3-2 | 作为企业管理员，我能注入企业 CA Bundle，使沙箱内可访问内网 HTTPS 服务 | guest 内 `curl https://internal.corp` 证书验证通过 |
| US-1.3-3 | 作为平台安全工程师，我能将沙箱的出站互联网访问关闭，沙箱只能访问平台内部服务 | `network.internet=false` 创建后，guest 内 `curl https://google.com` 连接拒绝 |

### FE-1.4：per-tenant 配额 + 优雅终止 + 入站连接限制

**范围**：per-tenant sandbox_quota(`node-ctl manifest-key add` CLI 配置，超限 429)；DELETE ?timeout=N 优雅终止(SIGTERM + TimeoutStopSec → SIGKILL)；node-proxy per-sandbox 入站并发连接计数(超限 503 + 友好 JSON 错误页)。

| US | UserStory | 验收标准 |
|----|-----------|---------|
| US-1.4-1 | 作为平台运营，我能为租户设置最大沙箱数配额，超限时明确拒绝并说明原因 | 超配额后 create 返回 HTTP 429 `{"error":"quota_exceeded","limit":N,"current":M}`；当前数减少后 create 立即恢复 |
| US-1.4-2 | 作为 AI Agent，我能在删除沙箱时指定宽限时间，让 guest 进程有机会完成清理 | `DELETE ?timeout=30`：guest 进程收到 SIGTERM，30s 内自行退出则无 SIGKILL；超时后强制 SIGKILL |
| US-1.4-3 | 作为平台，沙箱入站并发连接超过上限时，新连接立即收到可读错误，而非通用 502 | 超限连接收到 HTTP 503 `{"error":"connection_limit_exceeded"}`；存量连接不受影响 |
| US-1.4-4 | 作为开发者，当沙箱不存在或已终止时，我收到明确可读的错误，而非通用网关错误 | sandbox_not_found → 404；sandbox_terminated → 410；sandbox_wake_timeout → 503，均含 JSON error 字段 |

## FE-2：数据面代理与路由

### FE-2.1：数据面路由与转发

**范围**：internal/external/off 三种 proxy.mode，routesync 广播，eBPF flowtable，CONNECT 隧道，兜底网关。

| US | UserStory | 验收标准 |
|----|-----------|---------|
| US-2.1-1 | 作为平台，node-proxy worker 崩溃时，其他 worker 继续服务新连接 | SO_REUSEPORT 下，单 worker 崩溃不影响新连接建立 |
| US-2.1-2 | 作为平台，serve 重启期间，已建立的 TCP 连接不中断 | eBPF flowtable pin bpffs，存活期内连接持续 |
| US-2.1-3 | 作为 AI 应用开发者，我能通过 WebSocket 访问沙箱内服务 | CONNECT 隧道(Hijack/h2 stream splice)正常工作 |

## FE-3：模板与构建

### FE-3.1：模板构建流水线(含构建日志流)

**范围**：三阶段(import/steps/template)，COPY/ADD 对象存储直传，构建日志实时流，fromImage/fromTemplate，构建资源池。

| US | UserStory | 验收标准 |
|----|-----------|---------|
| US-3.1-1 | 作为 AI 应用开发者，我能提交 Dockerfile steps 构建自定义沙箱镜像 | 三阶段均通过，产物为 snp 或 img 模板 |
| US-3.1-2 | 作为 AI 应用开发者，我能通过 SDK on_build_logs 实时看到构建进度 | 日志延迟 < 2s，按 logsOffset 分页幂等 |
| US-3.1-3 | 作为平台，两个并发构建不超过 max_concurrent 限制 | 第三个构建阻塞在 waiting 状态，前两个完成后才开始 |
| US-3.1-4 | 作为 AI 应用开发者，构建中的 COPY 指令无需镜像自带 tar/gzip | flatten-ctl tar extract 支持 scratch/distroless 镜像 |

### FE-3.2：快照导出 / 导入 / 晋升

**范围**：export-sandbox、import-sandbox、migration token 铸造、本机快照晋升远程、PAUSED→SAVED 两阶段(节点侧)。

| US | UserStory | 验收标准 |
|----|-----------|---------|
| US-3.2-1 | 作为平台，将本机 paused 沙箱的快照晋升为远程 manifest，使其可在其他节点恢复 | export-sandbox 成功后，源沙箱保持 paused(默认 move，move 后删本机)，迁移 token 有效 |
| US-3.2-2 | 作为平台，在目标节点通过迁移 token 恢复沙箱，校验 runtime 摘要一致 | import 校验 token 指纹与本机 runtime erofs 摘要，不一致则拒绝 |
| US-3.2-3 | 作为 AI SDK，通过带 X-Kuasar-Migration-Token 的 connect 调用，一步完成迁移 | connect 自动 import + resume，SDK 无感知跨节点迁移 |

## FE-4：内存管理与快照加速

### FE-4.1：快照序列化与恢复

**范围**：microVM 全量/差分内存快照序列化、脏页追踪、页去重、快照扇出(同一快照并发起多个沙箱)。

| US | UserStory | 验收标准 |
|----|-----------|---------|
| US-4.1-1 | 作为平台，微 VM 暂停后能序列化为全量内存快照，重启后从快照完整恢复 | pause 后 memfile 落盘，resume 时 cloud-hypervisor 从 memfile 恢复，guest 进程状态连续 |
| US-4.1-2 | 作为平台，同一模板快照可同时扇出起多个沙箱，快照文件只读共享不被修改 | N 个沙箱从同一 snp/img 并发启动，快照文件 SHA256 前后一致 |
| US-4.1-3 | 作为平台，差分快照仅记录自父快照以来的脏页，减少快照体积 | 差分快照体积 < 全量快照体积的 50%(标准内存工作集测试用例) |

### FE-4.2：内存效率优化

**范围**：UFFD 按需加载、历史缺页顺序异步预取、virtio-balloon/hinting 空闲页上报、DAX 共享 runtime 镜像(★)、memfd + 外部 uffd 内存统一持有(★)。

| US | UserStory | 验收标准 |
|----|-----------|---------|
| US-4.2-1 | 作为平台，UFFD 按需加载使恢复延迟不受快照体积影响，首次可运行时间稳定 | 从 restore 触发到 guest 首次用户态可运行 < 200ms，与快照文件大小解耦 |
| US-4.2-2 | 作为平台，异步预取按历史缺页顺序提前拉取，热路径应用缺页率显著降低 | 同工作负载热身后预取命中率 > 80% |
| US-4.2-3 | 作为平台，balloon hinting 主动上报 guest 空闲页，宿主在 idle 后可回收物理内存 | guest 标记空闲后，宿主 RSS 在 5s 内下降可观测(cgroup memory.current 下降) |
| US-4.2-4 | 作为平台，DAX 共享 virtio-pmem 使多沙箱共享同一 runtime erofs，无需各自映射副本 | N 个沙箱挂载同一 DAX 设备时，宿主 page cache 不因沙箱数增加而线性增长 |

## FE-5：块存储与缓存

### FE-5.1：块存储基础能力

**范围**：mmap 块缓存、稀疏文件(空洞 punch)、流式异步块加载、块级内容寻址去重、块状态追踪(dirty/zero/cached)。

| US | UserStory | 验收标准 |
|----|-----------|---------|
| US-5.1-1 | 作为平台，多沙箱引用同一块时，物理内存只保留一份(mmap 共享) | 同一块被 N 个沙箱引用，宿主 RSS 中该块仅出现一份 |
| US-5.1-2 | 作为平台，稀疏文件未写入区域不占磁盘空间 | 新建 1GiB 稀疏文件，`du -sh` 实际占用 < 1MiB |
| US-5.1-3 | 作为平台，沙箱启动时块可在后台流式加载，不阻塞 guest 可运行 | 远程块未完全到达时，沙箱可访问已到达块；缺失块触发按需异步拉取，guest 不崩溃 |

### FE-5.2：node★ 分层缓存与内容寻址

**范围**：L1 本地 / L2 集群 / L3 OBS 三层缓存、收敛加密(相同内容 → 相同密文)、Maglev shuffle-sharding 缓存亲和、EROFS 确定性展平(字节级相同产物)。

| US | UserStory | 验收标准 |
|----|-----------|---------|
| US-5.2-1 | 作为平台，收敛加密使相同内容的块跨租户只存储一份密文 | 两个不同租户上传相同内容，对象存储中只存一份密文块(content-addressed key 相同) |
| US-5.2-2 | 作为平台，EROFS 产物字节级确定性，相同 Dockerfile 构建两次产物 SHA256 一致 | 同 Dockerfile 构建两次，产物文件 SHA256 完全相同 |
| US-5.2-3 | 作为平台，Maglev 亲和使同一沙箱模板的块优先命中同一节点的 L1 缓存 | 相同模板重复启动时，L1 缓存命中率 > 95%(非冷启动) |

## FE-6：存储集成

### FE-6.1：文件存储卷

**范围**：沙箱创建时通过 `volumes[]` 参数挂载外部文件存储卷(SFS Turbo 或自建 NFS)，卷数据跨沙箱会话持久化；node-ctl 协调独立 volume-ctl 服务完成 export 创建与绑定，并将挂载配置注入 sandbox-ctl launch 参数。

| US | UserStory | 验收标准 |
|----|-----------|---------|
| US-6.1-1 | 作为开发者，我能在创建沙箱时挂载命名卷，销毁后用同名卷重新创建沙箱，数据仍存在 | `POST /sandboxes` 携带 `volumes:[{name:"my-data",mount:"/data"}]`；写入文件后销毁，同名卷重新挂载后文件可读 |
| US-6.1-2 | 作为开发者，我能将多个卷挂载到沙箱内不同路径，卷之间数据完全隔离 | 单沙箱支持挂载 ≥ 4 个命名卷，每卷指定独立挂载路径；不同卷数据互不可见 |
| US-6.1-3 | 作为平台，卷挂载失败(存储服务不可达或卷不存在)时，沙箱创建明确报错，不留半初始化状态 | volume-ctl 不可达或 export 失败时，`POST /sandboxes` 返回 503；沙箱不进入 running 状态，无资源泄漏 |
| US-6.1-4 | 作为平台运维，我能通过 API 列出指定团队的全部命名卷及其当前挂载情况 | `GET /admin/teams/{tid}/volumes` 返回卷名、大小、当前挂载的沙箱 ID 列表 |

### FE-6.2：对象存储接入

**范围**：创建沙箱时传入对象存储凭证(AK/SK、endpoint、bucket)，支持 S3 兼容协议及华为云 OBS；凭证经 MMDS 安全下发至 guest 进程，guest 内代码无需硬编码凭证；支持运行时凭证刷新(如 STS / IAM Agency Token 临时凭证续期)，无需重建沙箱。

| US | UserStory | 验收标准 |
|----|-----------|---------|
| US-6.2-1 | 作为开发者，我能在创建沙箱时注入对象存储凭证(S3 或 OBS)，沙箱内进程可直接读写指定桶 | `POST /sandboxes` 携带 `storage.object_credentials:{type,endpoint,ak,sk,bucket}`；guest 内通过 AWS CLI(S3)或 obsutil(OBS)访问桶正常返回列表 |
| US-6.2-2 | 作为平台，凭证仅通过 MMDS 路径下发到 guest，不出现在 node-ctl 日志、审计日志或任何 API 响应体中 | 完整日志链路扫描无明文 AK/SK；MMDS endpoint 仅 guest 内 `169.254.169.254` 可达，宿主外不可访问 |
| US-6.2-3 | 作为开发者，我能在沙箱运行时刷新对象存储凭证(如临时 STS token 即将过期)，无需重建沙箱 | `PUT /sandboxes/{sid}/storage/credentials` 更新凭证后，MMDS 内新凭证 ≤ 5s 内生效；旧凭证同时失效 |

## FE-7：网络控制

### FE-7.1：沙箱网络基础设施

**范围**：沙箱网络命名空间隔离、沙箱间无横向通信、网络 slot 池化管理、网络接口发送速率限制。

| US | UserStory | 验收标准 |
|----|-----------|---------|
| US-7.1-1 | 作为平台，每个沙箱独立网络命名空间，沙箱间无法直接通信 | 沙箱 A 内 ping/connect 沙箱 B 的内网 IP 超时无响应 |
| US-7.1-2 | 作为平台，网络 slot 提前池化，沙箱创建时零延迟绑定网络接口 | 从 slot 池分配网络接口耗时 < 10ms |
| US-7.1-3 | 作为平台，网络接口发送速率限制可配置，防止单沙箱独占带宽 | 配置 `network.tx_rate_limit` 后，沙箱出口速率不超过设定值(iperf3 验证) |

### FE-7.2：node★ 高性能数据面

**范围**：eBPF/TC 纯内核态数据转发、ARP 代答(L2 结构性隔离)、GENEVE 隧道跨宿主传输。

| US | UserStory | 验收标准 |
|----|-----------|---------|
| US-7.2-1 | 作为平台，eBPF/TC 内核态转发避免用户态拷贝，高负载下 CPU 开销低 | 10Gbps 饱和流量下单连接 CPU 占用 < 5%(对比用户态 proxy) |
| US-7.2-2 | 作为平台，ARP 代答隔离 L2 广播，宿主不暴露沙箱 MAC 给其他沙箱 | 沙箱发送 ARP 广播，仅收到 node-proxy 的代答，不收到其他沙箱的响应 |
| US-7.2-3 | 作为平台，GENEVE 隧道支持跨宿主机流量转发，迁移后路由透明切换 | 沙箱跨节点迁移后，原 floatingip:port 可路由到新节点，已有 TCP 连接恢复转发 |

## FE-8：安全与鉴权

### FE-8.1：密钥管理与安全

**范围**：manifest_key 加密存储、TTL 租约、api_key HMAC 验证、MMDS 确定性密钥、keytag 多密钥轮转。

| US | UserStory | 验收标准 |
|----|-----------|---------|
| US-8.1-1 | 作为平台安全负责人，manifest_key 在任何情况下不以明文存磁盘 | 审计 manifest_keys 表，所有 key_enc 均为 AES-256-GCM 密文格式 |
| US-8.1-2 | 作为 AI 应用开发者，我能轮转 manifest_key 而不影响已运行的沙箱 | 新 key 添加后 api_key 验证通过，旧 key 并行有效至 expires_unix |
| US-8.1-3 | 作为平台，N 个 node-proxy worker 均能正确响应同一 sid 的 MMDS 请求 | 确定性公式：任意 worker 计算结果一致，guest 取到正确 mmds_key |

## FE-9：可观测性

### FE-9.1：沙箱可观测性

**范围**：`GET /sandboxes/{id}/metrics`(cgroup v2 实时读取)、`GET /sandboxes/{id}/logs`(v1 分页 / v2 方向+cursor+level+搜索)、OTel span 埋点(create / admit / restore / proxy.route / build.phase)。

| US | UserStory | 验收标准 |
|----|-----------|---------|
| US-9.1-1 | 作为 AI 应用开发者，我能通过 API 查询沙箱的实时 CPU 和内存用量 | `GET /sandboxes/{id}/metrics` 返回 `rss_bytes`、`usage_usec`、`oom_count`，数据延迟 < 1s |
| US-9.1-2 | 作为开发者，我能通过 API 分页读取沙箱应用日志，并按日志级别和关键词过滤 | v2 接口 `?level=error&search=timeout` 在 1s 内返回匹配条目，cursor 分页幂等 |
| US-9.1-3 | 作为 SRE，我能在 Jaeger/Tempo 中看到从 API 请求到 microVM 就绪的完整 trace | create 链路包含 admit / restore span，每个 span 有耗时和状态，P99 < 3s 的 SLO 可在 trace 中验证 |

### FE-9.2：审计日志持久化

**范围**：append-only 结构化审计日志(`/var/log/kuasar/audit.jsonl`)、日志滚动(100 MB/10 files)、syslog forward(可选)。覆盖 sandbox.create / kill / pause / export / import 与 manifest_key.add / revoke / build.trigger。

| US | UserStory | 验收标准 |
|----|-----------|---------|
| US-9.2-1 | 作为合规官，所有控制面操作产生带 operator、action、result 的结构化审计记录，节点重启后历史完整保留 | 重启后 `/var/log/kuasar/audit.jsonl` 历史日志完整；每条记录含 `operator_fp, action, sid, result, trace_id` |
| US-9.2-2 | 作为合规官，审计日志可通过 syslog forward 实时接入 Splunk / Elasticsearch | 配置 `audit.syslog_target` 后，操作日志在 5s 内出现在目标系统 |

## FE-10：资源管控

### FE-10.1：内存超售与准入

**范围**：四水位、令牌桶、startup_pool、FIFO 准入队列、Active Reclaimer、virtio-balloon、cgroup PSI。

| US | UserStory | 验收标准 |
|----|-----------|---------|
| US-10.1-1 | 作为平台，在正常负载下(绿区)，沙箱启动不被资源延迟阻塞 | 准入延迟 P99 < 5ms(绿区) |
| US-10.1-2 | 作为平台，高密度超售时，idle 沙箱内存被主动回收，宿主不 OOM | Active Reclaimer 使 idle 沙箱 RSS 收缩到 floor，宿主内存 < red 水位 |
| US-10.1-3 | 作为平台，resource-controller 重启后，活跃沙箱自动 Reattach，预算记账恢复 | 重建时间 < 30s，无沙箱因控制器重启被 OOM-kill |

## FE-11：运营与计费

### FE-11.1：Admin 批量管控

**范围**：Admin 强制销毁指定团队全部沙箱、Admin 取消指定团队全部进行中构建。

| US | UserStory | 验收标准 |
|----|-----------|---------|
| US-11.1-1 | 作为平台运营，我能通过 Admin API 强制销毁指定团队的全部沙箱 | `POST /admin/teams/{tid}/sandboxes/kill` 后，该团队所有沙箱在 30s 内进入 dead 状态，资源释放 |
| US-11.1-2 | 作为平台运营，我能通过 Admin API 取消指定团队所有进行中的构建任务 | `POST /admin/teams/{tid}/builds/cancel` 后，所有 BUILDING 状态构建置为 CANCELLED，构建进程退出 |

### FE-11.2：用量计量

**范围**：按沙箱 ID 记录启动时间(created_at)、停止时间(dead_at)及规格(CPU 核数、内存大小)，提供用量查询 API，供外部计费服务消费。

| US | UserStory | 验收标准 |
|----|-----------|---------|
| US-11.2-1 | 作为计费服务，我能通过 API 查询指定团队在某时间范围内的沙箱用量明细 | `GET /admin/teams/{tid}/usage?start=&end=` 返回每个沙箱的 sid、template_id、started_at、ended_at、cpu_cores、memory_mb，精度 1 秒 |
| US-11.2-2 | 作为平台，沙箱状态每次变更(created / paused / resumed / dead)时，计量记录自动追加一条时间戳事件 | 状态变更后 1s 内记录落盘；记录含 sid、event_type、ts、cpu_cores、memory_mb |
| US-11.2-3 | 作为平台运维，计量数据在本地持久化，node-ctl 重启后历史记录不丢失 | 重启后查询历史用量与重启前一致，误差 ≤ 1 条事件 |

### FE-11.3：计费事件推送

**范围**：沙箱生命周期事件(created / paused / resumed / dead)以结构化格式推送到外部计费 / Webhook 服务；支持 HTTP Webhook 和消息队列两种出口。

| US | UserStory | 验收标准 |
|----|-----------|---------|
| US-11.3-1 | 作为外部计费系统，我能通过 Webhook 实时接收沙箱生命周期事件，用于按需计费 | 沙箱状态变更后 5s 内 Webhook endpoint 收到事件；Payload 含 sid、team_id、event_type、ts、spec(cpu/mem)；失败自动重试 3 次(指数退避) |
| US-11.3-2 | 作为平台运维，我能配置 Webhook URL 和签名密钥，确保事件推送安全可信 | 通过 `BILLING_WEBHOOK_URL` 与 `BILLING_WEBHOOK_SECRET` 配置；Payload 携带 HMAC-SHA256 签名头 `X-Kuasar-Signature` |
| US-11.3-3 | 作为平台，Webhook 推送失败不影响沙箱正常运行，未送达事件写入本地队列待补发 | Webhook 不可达时沙箱创建/销毁正常返回；积压事件持久化，服务恢复后自动补发，最大补发窗口 24h |

## FE-12：节点集群参与

### FE-12.1：集群接入(node-link)

**范围**：节点向 cluster-ctl registry 注册与心跳上报；受理下行命令(create / connect / delete / key_put / key_drop / build_register)；沙箱生命周期事件上行(created / paused / resumed / dead)；node-link 断线重连与全量重报。

| US | UserStory | 验收标准 |
|----|-----------|---------|
| US-12.1-1 | 作为平台，节点启动后自动注册到 cluster-ctl registry，并以心跳维持存活 | 启动后 10s 内 registry 可查到本节点，心跳间隔 `node_link.heartbeat_interval`(默认 5s)，超时 3 次 registry 标节点不可用 |
| US-12.1-2 | 作为平台，registry 通过 node-link 下发 create 命令时，节点正常创建沙箱并回传 created 事件 | create 命令处理路径与本地 REST /sandboxes 同代码路径；事件含 sid、floatingip、port |
| US-12.1-3 | 作为平台，registry 下发 key_put 后，manifest_key 在内存 TTL 内有效；key_drop 后立即失效，新 create 拒绝，存量沙箱不影响 | key_drop 后：内存 TTL 清空；新 POST /sandboxes 返回 401；已运行沙箱数据面继续正常转发 |
| US-12.1-4 | 作为平台，node-link 连接断线后节点自动重连，重连后上报全量沙箱状态供 registry 对账 | 指数退避重连(最大 30s)；重连后发 full_sync 帧，包含所有 running/paused 沙箱的最新状态 |

### FE-12.2：资源 drain

**范围**：节点维护模式，停止新准入，等待现有沙箱释放；drain 状态经 node-link 心跳携带 `schedulable=false` 同步至 registry。

| US | UserStory | 验收标准 |
|----|-----------|---------|
| US-12.2-1 | 作为平台运维，节点进入 drain 模式后，停止新准入，存量沙箱不受影响 | drain 后 Admit 返回 drained，现有沙箱继续运行 |
| US-12.2-2 | 作为平台，进入 drain 后 registry 停止向本节点调度新沙箱，现有沙箱正常运行到期 | drain 状态经 node-link 心跳携带 `schedulable=false`；registry 侧不再向该节点发 create 命令 |

---

# 附录 A：与 e2b Infra 竞争力对比分析

> 数据基准：e2b-dev/infra 源码分析(2026-06-23)。
> 标注：✅ 已支持 · 🔶 部分支持 · ❌ 不支持 · ★ node-ctl 独有(e2b infra 无)

## A.1 完整特性对比清单

> 仅列 node-ctl 交付边界内的特性，包含以 node-ctl 为入口委托下游实现的能力。envd 进程/文件系统 API、集群调度策略等由其他组件独立提供的能力不在此列，详见 §1.2 交付边界。
> 有 FE 对应的域在标题后标注(FE-x)，十二个域与 FE-1 至 FE-12 一一对应。

### 一、沙箱生命周期管理(FE-1)

**基础生命周期**

| 特性 | e2b infra | node-ctl |
|------|-----------|-------------------|
| 从模板创建沙箱 | ✅ | ✅ |
| 快照恢复启动(snp 模板) | ✅ | ✅ |
| 冷启动(img 模板) | ✅ | ✅ |
| 获取沙箱详情 | ✅ | ✅ |
| 列出沙箱(分页 + 状态过滤) | ✅ | 🔶 仅本节点 |
| 终止沙箱 | ✅ | ✅ |
| 优雅终止(grace period) | ✅ | ❌ |

**暂停与恢复**

| 特性 | e2b infra | node-ctl |
|------|-----------|-------------------|
| 暂停(全内存快照) | ✅ | ✅ |
| 暂停(仅文件系统快照) | ✅ | ✅ |
| 快照前内存回收(fstrim/sync/drop_caches) | ✅ | ✅ |
| 快照前文件系统冻结 | ✅ | ✅ |
| 快照前 heap 收拢 | ✅ | ✅ |
| Resume(内存快照恢复) | ✅ | ✅ |
| Resume(冷启动恢复) | ✅ | ✅ |
| Connect(保活或自动恢复) | ✅ | ✅ |
| 自动暂停(TTL 到期) | ✅ | ✅ |
| 自动恢复(数据面触发，用户无感知) | ✅ | ✅ |
| TTL 刷新(KeepAlive) | ✅ | ✅ |
| TTL 动态修改 | ✅ | ✅ |
| 暂停沙箱跨节点迁移 | ✅ | ✅ |
| 恢复时超时配置(park_timeout) | ✅ | ✅ |

**沙箱配置**

| 特性 | e2b infra | node-ctl |
|------|-----------|-------------------|
| 注入环境变量 | ✅ | ✅ |
| 指定用户自定义 metadata | ✅ | ✅ |
| 创建时指定 CPU 数 | ✅ | ✅ |
| 创建时指定内存 MB | ✅ | ✅ |
| 创建时指定磁盘大小 | ✅ | 🔶 通过模板固定 |
| 启用 Secure 模式(envd 双重鉴权) | ✅ | ❌ |
| 指定 AutoPause 策略 | ✅ | ✅ |
| 指定 AutoResume 策略 | ✅ | ✅ |
| CA Bundle 注入 | ✅ | ❌ |
| 运行时在线调整规格 | ❌ | ❌ |

### 二、数据面代理与路由(FE-2)

node-ctl(node-proxy)负责协议转发；envd 进程/文件系统 API 由 guest 内 envd 直接提供，node-proxy 透传，不在此列。

| 特性 | e2b infra | node-ctl |
|------|-----------|-------------------|
| 双层代理(外部入口 + 节点内) | ✅ | ✅ |
| Host-based 路由(\<port\>-\<sid\>.\<domain\>) | ✅ | ✅ |
| HTTP/HTTPS 反向代理 | ✅ | ✅ |
| WebSocket 透明代理(101 Upgrade) | ✅ | ✅ |
| H2C / gRPC 支持 | ✅ | ✅ |
| 连接池(per-sandbox) | ✅ | ✅ |
| 连接重试(线性退避) | ✅ | ✅ |
| 暂停沙箱自动恢复(数据面触发) | ✅ | ✅ |
| 数据面 worker 水平扩展(SO_REUSEPORT) | 🔶 有 LB 层 | ✅ ★ |
| routesync 增量路由同步 | ❌ | ✅ ★ |
| 兜底网关(worker 不可用时) | ❌ | ✅ ★ |
| 友好错误页(sandbox_not_found 等) | ✅ | ❌ |

### 三、模板与构建系统(FE-3)

| 特性 | e2b infra | node-ctl |
|------|-----------|-------------------|
| 注册模板(分配 templateID) | ✅ | ✅ |
| 从 Docker 镜像构建模板(fromImage) | ✅ | ✅ |
| 从已有模板构建(fromTemplate) | ✅ | ✅ |
| RUN/ENV/ARG/WORKDIR/USER 步骤 | ✅ | ✅ |
| COPY 步骤(文件直传 OBS) | ✅ | ✅ |
| startCmd(预热快照) | ✅ | ✅ |
| readyCmd(就绪探测) | ✅ | ✅ |
| 构建日志实时流(logsOffset 分页) | ✅ | ✅ |
| 构建状态查询 | ✅ | ✅ |
| 快照模板(snp) | ✅ | ✅ |
| 镜像模板(img) | ✅ | ✅ |
| 模板列表 | ✅ | ✅ |
| 模板版本 / 人类可读标签 | ✅ | ❌ |
| 模板 alias 管理 | ✅ | ❌ |
| 构建层级增量缓存(cache-from) | ❌ | 🔶 相同镜像可跳过 |
| 构建并发数控制 | ✅ | ✅ |
| 构建磁盘用量限制 | ✅ | ✅ |
| 构建镜像安全扫描 | ❌ | ❌ |

### 四、内存管理与快照加速(FE-4)

| 特性 | e2b infra | node-ctl |
|------|-----------|-------------------|
| 全内存快照(memfile 序列化) | ✅ | ✅ |
| 差分快照(脏页追踪) | ✅ | ✅ |
| 内存页去重(对比父快照) | ✅ | ✅ |
| UFFD 按需内存加载 | ✅ | ✅ |
| 异步内存预取(按历史缺页顺序) | ✅ | ✅ |
| 内存气球(balloon，空闲页上报) | ✅ | ✅ |
| Balloon hinting(主动提示空闲页) | ✅ | ✅ |
| Huge pages 支持 | ✅ | ✅ |
| 快照扇出(同一快照起多沙箱) | ✅ | ✅ |
| DAX 共享 runtime 镜像(virtio-pmem) | ❌ | ✅ ★ |
| 内存统一持有(memfd + 外部 uffd) | ❌ | ✅ ★ |

### 五、块存储与缓存(FE-5)

| 特性 | e2b infra | node-ctl |
|------|-----------|-------------------|
| 内存映射块缓存(mmap) | ✅ | ✅ |
| 稀疏文件支持(空洞 punch) | ✅ | ✅ |
| 流式块加载(后台异步拉取) | ✅ | ✅ |
| 块级去重(content-addressed) | ✅ | ✅ |
| 块状态追踪(dirty/zero/cached) | ✅ | ✅ |
| 分层缓存(L1 本地 / L2 集群 / L3 OBS) | ❌ | ✅ ★ |
| 收敛加密(相同内容 → 相同密文) | ❌ | ✅ ★ |
| Maglev shuffle-sharding 缓存亲和 | ❌ | ✅ ★ |
| 确定性展平(EROFS 字节级相同) | ❌ | ✅ ★ |

### 六、存储集成(FE-6)

| 特性 | e2b infra | node-ctl |
|------|-----------|-------------------|
| SFS Turbo/NFS 命名卷(跨会话持久化) | ✅ | ❌ |
| 多卷挂载 | ✅ | ❌ |
| 对象存储接入(S3/OBS)★ | ❌ | ❌ |
| 对象存储凭证运行时刷新★ | ❌ | ❌ |

### 七、网络控制(FE-7)

**基础网络**

| 特性 | e2b infra | node-ctl |
|------|-----------|-------------------|
| 沙箱网络隔离 | ✅ | ✅ |
| 沙箱间无横向通信 | ✅ | ✅ |
| 网络 slot 池化管理 | ✅ | ✅ |
| 互联网访问开关 | ✅ | ❌ |
| MMDS 沙箱元数据服务 | ✅ | ✅ |
| 网络接口发送速率限制 | ✅ | ✅ |
| eBPF/TC 纯内核数据面 | ❌ | ✅ ★ |
| ARP 代答(L2 结构性隔离) | ❌ | ✅ ★ |
| GENEVE 隧道跨宿主传输 | ❌ | ✅ ★ |

**出站网络策略**

| 特性 | e2b infra | node-ctl |
|------|-----------|-------------------|
| 出站允许规则(域名/CIDR) | ✅ | ❌ |
| 出站拒绝规则 | ✅ | ❌ |
| 出站代理(BYOP，自定义 SOCKS5/HTTP) | ✅ | ❌ |
| 出站规则运行时热更新 | ✅ | ❌ |
| nftables 防火墙(原子规则更新) | ✅ | ❌ |
| TCP 出站经 SOCKS5 用户态处理 | ✅ | ❌ |
| 非 TCP 出站 allow/deny | ✅ | ❌ |

**入站与流量控制**

| 特性 | e2b infra | node-ctl |
|------|-----------|-------------------|
| 入站公网访问开关 | ✅ | ❌ |
| 流量 access token(用户端口) | ✅ | ✅ |
| 请求 Host 掩码 | ✅ | ❌ |
| 域名级请求头变换规则 | ✅ | ❌ |
| 并发连接数限制(per-sandbox) | ✅ | ❌ |

### 八、安全与鉴权(FE-8)

| 特性 | e2b infra | node-ctl |
|------|-----------|-------------------|
| API Key 认证 | ✅ | ✅ |
| HMAC 派生 api_key(无查表验证) | ❌ | ✅ ★ |
| OAuth2 / OIDC 认证 | ✅ | ❌ |
| 沙箱 per-sandbox access token | ✅ | ✅ |
| Secure 模式(envd 双重鉴权) | ✅ | ❌ |
| CA Bundle 自定义注入 | ✅ | ❌ |
| 多租户数据隔离(跨租户返回 404) | ✅ | ✅ |
| AES-256-GCM 密钥加密存储 | ✅ | ✅ |
| 密钥轮换(keytag 多密钥) | ❌ | ✅ ★ |
| 集群密钥 mTLS 分发 | ❌ | ✅ ★ |
| API 请求体大小限制 | ✅ | 🔶 |
| 租户封禁(team blocking) | ✅ | ❌ |
| Admin API key 管理 | ✅ | ✅ |
| 构建凭据隔离(不入 guest) | ✅ | ✅ |
| 运行时数据静态加密(overlay 写层) | ❌ | ❌ |

### 九、可观测性(FE-9)

| 特性 | e2b infra | node-ctl |
|------|-----------|-------------------|
| 沙箱运行时指标(CPU/内存/磁盘) | ✅ | ❌ |
| 沙箱启动延迟指标 | ✅ | 🔶 内部 startupStats，未暴露 API |
| 节点级资源指标 | ✅ | ✅ Prometheus 端点 |
| Prometheus 完整指标清单 | ✅ | ❌ |
| OpenTelemetry 分布式追踪 | ✅ | ❌ |
| 沙箱日志 API(v1) | ✅ | ❌ |
| 沙箱日志 API(v2：方向/cursor/level/搜索) | ✅ | ❌ |
| 构建日志流 | ✅ | ✅ |
| 审计日志(控制面操作) | ✅ | ❌ |
| 审计日志持久化 | ✅ | ❌ |
| 结构化日志聚合(Loki/ELK 兼容) | ✅ | 🔶 syslog forward 为对接点 |
| ClickHouse 指标存储 | ✅ | ❌ |
| SLO/SLA 定义 | ✅ | ❌ |

### 十、资源管控(FE-10)

| 特性 | e2b infra | node-ctl |
|------|-----------|-------------------|
| 沙箱内存超分(四水位仲裁) | ❌ | ✅ ★ |
| 动态 burst grant(令牌桶) | ❌ | ✅ ★ |
| PSI 反压退避(非 OOM kill) | ❌ | ✅ ★ |
| 主动内存回收(Active Reclaimer) | ❌ | ✅ ★ |
| 启动预算池隔离(防惊群) | ❌ | ✅ ★ |
| 紧急水位强制回收 | ❌ | ✅ ★ |
| per-tenant quota(最大沙箱数) | ✅ | ❌ |
| per-tenant 并发构建数限制 | ✅ | ❌ |
| per-tenant API 请求限流 | ✅ | ❌ |
| cgroup v2 资源隔离 | ✅ | ✅ |
| 磁盘 I/O 速率限制 | ✅ | ✅ |
| 网络发送速率限制 | ✅ | ✅ |
| 熵源设备速率限制 | ✅ | ✅ |

### 十一、运营与计费(FE-11)

| 特性 | e2b infra | node-ctl |
|------|-----------|-------------------|
| 用量计量(启停时间 × 规格) | ✅ | ❌ |
| 计费事件推送(外部系统集成) | ✅ | ❌ |
| Webhook 事件推送(状态变更等) | ✅ | ❌ |
| Admin 强杀团队全部沙箱 | ✅ | ✅ |
| Admin 取消团队全部构建 | ✅ | ✅ |
| 节点滚动升级 | ❌ | ❌ |

### 十二、节点集群参与(FE-12)

集群调度决策(P2C、亲和性、全局列表)由 cluster-ctl 负责，不在 node-ctl 范围内；此处仅列节点侧实现。

| 特性 | e2b infra | node-ctl |
|------|-----------|-------------------|
| 节点注册与心跳上报 | ✅ | ✅ |
| 节点排空(drain) | ✅ | ✅ |
| 沙箱快照导出 / 跨节点迁移 token | ✅ | ✅ |

## A.2 特性数量概览

> 仅统计 node-ctl 交付边界内的特性；envd 进程/文件系统 API、集群调度策略、Feature Flag 等由外部组件提供，不纳入比较(见 §1.2)。

| 特性域 | e2b infra | node-ctl 已支持 | node★ 独有 | 待补齐项 |
|--------|-----------|--------------------------|-----------|---------|
| 沙箱生命周期 | 31 | 27 | 0 | 4(优雅终止 · Secure 模式 · CA Bundle · 运行时调整规格〔双方均缺失〕) |
| 数据面代理与路由 | 9 | 11 | 3 | 1(友好错误页) |
| 模板与构建 | 16 | 14 | 0 | 3(版本/标签 · alias · 安全扫描〔双方均缺失〕) |
| 内存管理与快照加速 | 9 | 11 | 2 | 0 |
| 块存储与缓存 | 5 | 9 | 4 | 0 |
| 网络控制 | 18 | 9 | 3 | 12(出站全部 · 入站 4 项 · 互联网访问 · 并发限制) |
| 安全与鉴权 | 11 | 10 | 3 | 5(OAuth2 · Secure 模式 · CA Bundle · 租户封禁 · 静态加密〔双方均缺失〕) |
| 可观测性 | 13 | 4 | 0 | 9(运行时指标 · Prometheus · OTel · 日志 v1/v2 · 审计日志 · 审计持久化 · ClickHouse · SLO) |
| 资源管控 | 7 | 10 | 6 | 3(并发构建限流 · API 限流 · per-tenant quota) |
| 运营与计费 | 5 | 2 | 0 | 4(计量 · 事件推送 · Webhook · 节点滚动升级〔双方均缺失〕) |
| 节点集群参与 | 3 | 3 | 0 | 0 |
| 存储集成 | 2 | 0 | 0 | 4(SFS Turbo/NFS 命名卷 · 多卷挂载 · 对象存储接入 · 凭证刷新〔对象存储双方均缺失〕) |
| **合计** | **~129** | **~110** | **21** | **~46** |

在 node-ctl 边界范围内，e2b infra 覆盖 ~129 项，node-ctl 当前已支持 ~110 项，待补齐 ~46 项(其中 6 项为双方均缺失的前沿能力)。node★ 独有 21 项集中在资源管控、块存储、网络控制、数据面代理与路由、安全五个域，构成结构性技术壁垒；主要待补齐方向为可观测性(9 项)、出站网络策略(含入站共 12 项)、存储集成(5 项)、安全认证(5 项)。Desktop / Computer Use 属于上层 Agent 应用服务，不在 node-ctl 交付边界内，不纳入本比较。

## A.3 node-ctl 领先领域

### A.3.1 单节点内存密度(最大成本竞争力)

e2b infra 无内存超售设计，沙箱按声明内存独占分配。node-ctl 的超售体系是结构性差异：

| 机制 | 说明 |
|------|------|
| 四水位联动(green/yellow/red/critical) | 负载感知准入，黄区排队而非直接拒绝 |
| 令牌桶速率控制(5%/s) | 防止突发准入引发内存雪崩 |
| startup_pool 独立预算 | 启动期与运行期内存竞争隔离 |
| Active Reclaimer + virtio-balloon | 空闲沙箱 RSS 主动压缩到 floor(默认 128 MiB) |
| DAX virtio-pmem 共享 runtime | 所有沙箱共享同一 runtime 镜像的 page cache |

**量化优势**：相同硬件(512 GiB 节点)下，e2b infra 可运行约 64 个 8 GiB 沙箱；node-ctl 空闲态可维持 400+ 个，稳态密度约为 e2b infra 的 6 倍。单位算力成本约为 e2b infra 的 1/4 至 1/6。

### A.3.2 数据面可靠性(控制面重启零断流)

node-ctl 的 eBPF/TC hook 和 BPF map 全部 pin 到 bpffs(`/sys/fs/bpf/vswitch/`)，node-ctl 重启期间内核转发持续，**已建立 TCP 连接零中断**。e2b infra 基于 iptables + 用户态 proxy，控制面重启会造成连接断开。这对有 SLA 承诺的 B 端客户是量化可感知的差异。

### A.3.3 块存储与快照加速

| 能力 | e2b infra | node-ctl |
|------|-----------|-------------------|
| L1/L2/L3 三级分层缓存 | ❌ | ✅ 本地 RocksDB → 集群 EC 节点 → OBS |
| 收敛加密(相同内容→相同密文) | ❌ | ✅ 跨租户 chunk 去重在加密状态下仍有效 |
| EROFS 确定性展平 | ❌ | ✅ 字节级相同，构建结果完全幂等 |
| Maglev shuffle-sharding 缓存亲和 | ❌ | ✅ 迁移后 L2 命中率不降 |

收敛加密是关键差异：不同租户使用相同基础镜像(如 `python:3.12`)时，存储层只存一份密文 chunk。随租户规模增长，存储边际成本趋向零。

### A.3.4 安全基础设施

| 能力 | e2b infra | node-ctl |
|------|-----------|-------------------|
| HMAC 派生 api_key(鉴权无 DB 查询) | ❌ | ✅ 任意 worker 无状态验证 |
| keytag 热密钥轮转 | ❌ | ✅ 在线轮转，零停机，旧密文按标签解密 |
| mTLS 集群密钥分发 | ❌ | ✅ node-link 通道，密钥不经 HTTP |
| manifest_key 全程不落磁盘明文 | 🔶 | ✅ AES-256-GCM + 内存 TTL 租约 |

## A.4 e2b infra 领先领域

### A.4.1 出站网络策略(企业合规的关键缺口)

e2b infra 拥有完整的出站管控体系，node-ctl 当前**全部缺失**：

| e2b infra 能力 | node-ctl 状态 |
|---------------|----------------------|
| 出站允许/拒绝规则(域名/CIDR) | ❌ |
| BYOP(用户自定义 SOCKS5/HTTP 代理) | ❌ |
| nftables 防火墙(原子规则更新) | ❌ |
| 出站规则运行时热更新 | ❌ |

对于需要防止 C2 回连、数据外传审计(DLP)的金融、医疗、政府类客户，这是**准入门槛级缺失**。

### A.4.2 持久卷与对象存储(长时 Agent 的核心诉求)

e2b infra 内置 NFS 命名卷、多卷挂载、跨会话数据持久化，作为平台一体化能力提供。**node-ctl 已将存储集成纳入 FE-6 交付范围**，补齐此差距：

| 能力 | e2b infra | node-ctl |
|------|-----------|-------------------|
| SFS Turbo/NFS 命名卷(跨会话持久化) | ✅ | ❌ → **FE-6.1 规划中** |
| 多卷挂载 | ✅ | ❌ → **FE-6.1 规划中** |
| 对象存储接入(S3/OBS) | ❌ | ❌ → **FE-6.2 规划中** |
| 对象存储凭证运行时刷新 | ❌ | ❌ → **FE-6.2 规划中** |

FE-6.1 实现路径：独立 volume-ctl 服务对接 SFS Turbo 或自建 NFS export；node-ctl `POST /sandboxes` 增加 `volumes[]` 字段；sandbox-ctl launch 时通过 9p 协议挂载，挂载失败沙箱创建报错不留脏状态。FE-6.2 复用已有 MMDS 基础设施，支持 S3 兼容接口及华为云 OBS，实现成本较低。

对于需要在多次沙箱会话间保留中间结果(数据集、模型 checkpoint)的研究类 Agent，FE-6.1 落地后即可消除此差距；对象存储场景(读训练集、写 checkpoint 到华为云 OBS 或 S3 兼容桶)通过 FE-6.2 零凭证硬编码方式覆盖，安全性优于 e2b infra 的环境变量注入方案。

### A.4.3 企业认证与运营基础设施

| 能力 | e2b infra | node-ctl |
|------|-----------|-------------------|
| OAuth2 / OIDC SSO | ✅ | ❌ |
| 用量计量与计费事件推送 | ✅ | ❌ |
| Webhook 状态变更推送 | ✅ | ❌ |
| 模板版本/人类可读标签 | ✅ | ❌ |

## A.5 综合竞争力定位

| 场景 | 胜出方 | 关键理由 |
|------|--------|---------|
| 高密度 AI 代码执行(RL 训练、批量推理) | **node-ctl 显著领先** | 60× 超售 + DAX + L1/L2/L3 缓存，单节点成本约为 e2b infra 的 1/5 |
| 数据面高可用(SLA ≥ 99.9%) | **node-ctl 领先** | eBPF bpffs 控制面重启零断流；e2b infra 重启断连 |
| 存储成本(大量共享基础镜像) | **node-ctl 领先** | 收敛加密 + 跨租户 chunk 去重；e2b infra 无此机制 |
| 私有化 / 自托管部署 | **node-ctl 领先** | 无 Nomad/PG/Redis/CH 依赖，运维复杂度低 |
| 企业合规安全(出站管控、SSO) | **e2b infra 领先** | 完整出站策略 + OIDC；node-ctl 两项均缺失 |
| 有状态长时 Agent(跨会话持久化) | **当前 e2b infra 领先** | NFS 持久卷已上线；node-ctl FE-6.1 规划中，落地后差距消除 |
| 大规模 SaaS 运营(计费、Webhook) | **当前 e2b infra 领先** | 内置计量和事件推送；node-ctl FE-11.2/11.3 规划中，落地后差距消除 |

---

# 附录 B：缺口补齐优先级建议

以**战略价值 × 实现可行性**为两轴排序，分三个梯队：

## B.1 P0：阻断性缺口(优先补齐，当前是准入门槛)

### B.1.1 出站网络策略

**战略价值**：金融、医疗、政府类企业客户的硬性合规要求。缺失直接导致此类客户无法采购。
**实现路径**：sandbox-vswitch 已有 eBPF/TC 架构基础，vswitch-ctl 扩展 `--egress-policy` 参数；node-ctl 在创建参数中接收策略配置并透传。nftables 规则由 vswitch 管理，热更新通过 vswitch-ctl 实现。
**交付形式**：`POST /sandboxes` 新增 `network.egress_policy` 字段；支持 domain/CIDR 白名单和 BYOP SOCKS5 代理地址。

### B.1.2 OAuth2 / OIDC 认证

**战略价值**：大型企业要求 SSO 对接，API Key 模式无法通过安全审查。
**实现路径**：在 node-ctl 的认证层(当前仅 api_key)增加 OIDC JWT 验证，颁发 short-lived session token；manifest_key 绑定关系通过 OIDC `sub` claim 映射。

## B.2 P1：重要差距(影响客户留存和扩展，3-6 个月补齐)

### B.2.1 文件存储卷(SFS Turbo/NFS)

**战略价值**：长时 Agent、数据科学 Agent 的必需能力。缺失导致客户只能用 e2b infra 处理有状态场景，形成分裂使用。
**对应特性**：FE-6.1 文件存储卷。
**实现路径**：独立 volume-ctl 服务对接 SFS Turbo API 或自建 NFS export；node-ctl `POST /sandboxes` 增加 `volumes[]` 字段；sandbox-ctl launch 时通过 9p 协议挂载。

### B.2.2 模板版本 / 人类可读标签

**战略价值**：CI/CD 集成体验的直接影响因素。当前只有 content key，团队无法像管理 Docker tag 一样管理沙箱环境。
**实现路径**：builds 表增加 `names_json` / `aliases_json` 字段(设计文档中已有字段占位)，增加 `POST /templates/{id}/aliases` 和 `GET /templates` 端点，alias 解析在 create 时展开为 persist_id。低悬果实，builds 表字段已预留。

### B.2.3 Webhook 事件推送

**战略价值**：Agent 编排系统需要被动感知沙箱状态变更(created/paused/killed/build.completed)，当前只能轮询 API。
**实现路径**：node-ctl 在状态变更时向配置的 webhook endpoint 发 HTTP POST；支持指数退避重试；可先实现为 node 本地推送，后续统一到集群事件总线。

### B.2.4 per-tenant API 请求限流

**战略价值**：防止单一失控租户打满节点控制面，是多租户 SaaS 的基础保障。
**实现路径**：node-ctl HTTP handler 层增加 per-manifest_key_hash 的令牌桶(golang.org/x/time/rate)，超限返回 429 + `Retry-After`。

## B.3 P2：中长期投资(开拓新市场)

### B.3.1 用量计量与计费基础设施

**战略价值**：商业化运营的必要条件，无计量则无法对外收费。
**实现路径**：node-ctl 在沙箱 create/pause/kill 时 emit 计量事件(sid, team, start_at, stop_at, vcpu, memory_mb)到消息队列(Kafka/Pulsar)；独立计量服务消费并写入 ClickHouse；REST API 暴露用量查询。

### B.3.2 构建层级增量缓存(cache-from)

**战略价值**：迭代构建体验。当前修改一个 pip 包需重跑所有 steps，层级缓存可将增量构建时间从分钟级压到几十秒。
**实现路径**：在 Phase B(steps)中引入 step 级别的 content hash 检查(inputs + 上下文 hash)；命中则复用上次 flatten 产物跳过执行。

### B.3.3 运行时静态加密(overlay 写层)

**战略价值**：企业数据安全合规要求(SOC2 Type II 等)。
**实现路径**：overlay upper 层通过 dm-crypt/LUKS 或 fscrypt 加密，key 经 manifest_key 派生，随沙箱启动挂载、随销毁丢弃。

### B.3.4 对象存储接入(S3/OBS)

**战略价值**：AI Agent 常见场景——读取训练数据集、写回模型 checkpoint、访问用户私有桶——当前需要将凭证硬编码在 Dockerfile 或 env 中，存在泄露风险；平台侧注入可实现零凭证暴露。
**对应特性**：FE-6.2 对象存储接入。
**实现路径**：`POST /sandboxes` 新增 `storage.object_credentials` 字段(type=s3|obs，加密落盘，不写 API 响应)；node-ctl MMDS server 在 `/latest/meta-data/storage/object` 路径暴露凭证；guest 内 AWS SDK(S3)或华为云 SDK(OBS)通过 IMDS provider 自动拾取，无需代码改造。运行时刷新通过 `PUT /sandboxes/{sid}/storage/credentials` 热更新 MMDS 响应。

## B.4 优先级汇总

| 优先级 | 缺口 | 关键判断 |
|--------|------|---------|
| P0 | 出站网络策略 | 企业合规硬门槛，缺失直接失单 |
| P0 | OAuth2 / OIDC | 大型企业 SSO 准入要求 |
| P1 | 模板版本/alias | 低成本，builds 表字段已预留，强烈建议穿插完成 |
| P1 | per-tenant API 限流 | 低成本，多租户 SaaS 基础保障 |
| P1 | Webhook 事件推送 | Agent 编排生态必需 |
| P1 | 文件存储卷(FE-6.1) | 有状态 Agent 场景的核心诉求 |
| P2 | 用量计量与计费 | 商业化运营前置条件 |
| P2 | 对象存储接入(FE-6.2) | AI Agent 数据访问安全基础，零硬编码凭证 |
| P2 | 构建层级增量缓存 | 开发体验提升 |
| P2 | 运行时静态加密 | SOC2 Type II 合规要求 |

> **建议**：P0 两项决定客户能否采购，应与本设计并行立项。P1 中模板 alias 和 API 限流实现成本最低(builds 表字段已预留、handler 层加令牌桶)，可在本设计开发期间穿插完成，快速拉近特性差距。FE-6.2 对象存储凭证注入复用已有 MMDS 基础设施，实现成本低，建议与 FE-6.1 文件存储卷一并纳入 P1/P2 交付计划。

---

*文档结束*
