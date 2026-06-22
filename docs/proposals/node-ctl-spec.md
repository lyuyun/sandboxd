# node-ctl 特性规格设计文档

**版本** v1.2 · **日期** 2026-06-22  
**基准文档** node.md / node-proxy.md / node-resource.md / cluster.md / sandbox.md / vswitch.md / manifest.md / flatten.md / cache.md / store.md / kuasar-sandbox.md / deployment.md

---

## 目录

1. [背景与目标](#1-背景与目标)
2. [逻辑架构分层设计](#2-逻辑架构分层设计)
3. [API 设计](#3-api-设计)
4. [Use Case](#4-use-case)
5. [User Story](#5-user-story)
6. [可靠性设计](#6-可靠性设计)
7. [性能设计](#7-性能设计)
8. [安全设计](#8-安全设计)
9. [设计规格](#9-设计规格)
10. [设计约束及限制](#10-设计约束及限制)

附录：[A 关键配置项速查](#附录-a关键配置项速查) · [B 沙箱状态机](#附录-b沙箱状态机) · [C 上下文组件职责边界](#附录-c上下文组件职责边界) · [D 数据结构与格式规格](#附录-d数据结构与格式规格)

---

## 1. 背景与目标

### 1.1 业务问题

kuasar-sandbox 平台面向大规模 Agent 在线服务、Serverless 与 RL 训练三类负载，在同一个 microVM 沙箱栈上提供亚秒级冷启/快照恢复、跨镜像内容去重、单节点数千沙箱密度的能力。

平台沙箱栈（sandbox-ctl / vswitch-ctl / flatten-ctl / manifest-ctl / cache-ctl）各自提供原语级 CLI：起一台 microVM、展平镜像、绑端口、上传快照。这些原语没有一个统一的北向聚合层。客户持有 e2b SDK/CLI 与 api key，需要无感知地执行：

- 创建、执行代码、暂停、恢复、销毁沙箱（完整生命周期）
- 构建自定义模板（从容器镜像到可复用快照）
- 在多节点集群中路由会话、按需恢复沙箱

**node-ctl** 补这一聚合层，刻意选择 **e2b 协议兼容**：e2b SDK 生态直接可用，零改造；协议契约清晰，兼容性可用真实 SDK/CLI 端到端验收。

### 1.2 设计目标

| 目标 | 说明 |
|---|---|
| **e2b 协议兼容** | Python/JS e2b SDK、e2b CLI 零改造直接指向本机 |
| **独立 + 集群两用** | 单机可独立承载；配 `cluster.registry` 接入集群由 registry/router/scaler 编排 |
| **依赖面薄，经 CLI 组合** | 经子进程 CLI 驱动兄弟组件（sandbox-ctl / vswitch-ctl / flatten-ctl / manifest-ctl）；不 import 兄弟仓内部包；纯 Go，`CGO_ENABLED=0` |
| **密钥不落明文** | manifest_key 仅密文落盘，运行期只在内存与 env 帧 |
| **高密度承载** | 节点级资源动态仲裁（超分），典型 60× 密度提升 |

### 1.3 产物边界

| 产物 | 职责 |
|---|---|
| `node-ctl` | daemon（serve）+ 启动器（run-sandbox/run-builder）+ 管理 CLI（manifest-key / resource / export-sandbox 等）|
| `e2b-key-ctl` | 纯密钥派生工具；无 DB / config / 编排状态 |

---

## 2. 逻辑架构分层设计

### 2.1 平台总体上下文

node-ctl 处于 kuasar-sandbox 平台的第二层（编排层），向下驱动沙箱运行时栈，向上对接 e2b SDK/CLI 或集群控制面。

```
┌─────────────── 平台管理面（Platform mgmt plane，平台外）────────────────────────────────────┐
│  sandbox 管理平台 / image registry / platform-agent（桥接 node-ctl）                        │
└─────────────────────────────────────────────────────────────────────────────────────────────┘
                                          ↕ e2b API / GroupConfigProvider
┌─────────────── 集群控制面（Cluster Control Plane，sandbox-orchestrator）───────────────────┐
│  cluster-ctl registry（注册表 + 通道枢纽）                                                  │
│  cluster-ctl router  （e2b 兼容统一入口，热路径本地缓存直转）                                │
│  cluster-ctl scaler  （放置调度器，P2C + shuffle-sharding）                                 │
└─────────────────────────────────────────────────────────────────────────────────────────────┘
                                          ↕ node-link (mTLS routesync)
┌─────────────── 计算节点（Compute Node）─────────────────────────────────────────────────────┐
│                                                                                             │
│  ┌── node-ctl serve ──────────────────────────────────────────────────────────────────┐    │
│  │  （本文核心，sandbox-orchestrator 仓）                                               │    │
│  └────────────────────────────────────────────────────────────────────────────────────┘    │
│            │ systemd D-Bus          │ CLI 子进程                │ UDS 协议                  │
│            ▼                        ▼                           ▼                           │
│  sandbox-runner@<sid>    sandbox-ctl run              vswitch-ctl attach/detach            │
│  sandbox-builder@<bid>   sandbox-ctl snapshot         flatten-ctl export（guest 内）        │
│            │ execve       sandbox-ctl exec             manifest-ctl store（构建收尾）        │
│            ▼                        │                                                       │
│  cloud-hypervisor（VMM）   ←─────────┘                                                      │
│  + guest: sandbox-init               ↕ UDS vhost-user-blk / uffd / ctl.sock               │
│    + envd（e2b profile）             │                                                      │
│                                     │ loopback gRPC                                        │
│  cache-ctl tiered（L1 RocksDB）      ↔   store-ctl（OBS 代理）                             │
│            │ wire（L2 miss）                   │ gRPC                                       │
│            ▼                                  ▼                                             │
│  L2 cache cluster（AZ 级）          OBS 桶（Region 级，L3 持久层）                           │
└─────────────────────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 node-ctl 内部分层

```
┌──────────────────────── 北向接入层 ──────────────────────────────────────┐
│  e2b SDK / CLI（E2B_DOMAIN + E2B_API_KEY）                              │
│  cluster-ctl router（注入 E2b-Sandbox-Id + X-Access-Token 转发）         │
│  按 Host 分流：api.<domain> → 控制面；<port>-<sid>.<domain> → 数据面      │
└──────────────────────────────────────────────────────────────────────────┘
                              │ HTTPS / h2c :443
┌──────────────────────── 控制面层 ────────────────────────────────────────┐
│  e2b REST API Handler（api.<domain>）                                   │
│  ├─ 沙箱生命周期：create / get / list / kill / pause / connect / timeout  │
│  ├─ 模板构建：register / trigger / status / files                         │
│  ├─ 导出/导入扩展：export / import（迁移）                                 │
│  └─ api_key MAC 鉴权 + 白名单准入 + 归属校验                               │
└──────────────────────────────────────────────────────────────────────────┘
                              │
┌──────────────────────── 主机编排层 ──────────────────────────────────────┐
│  生命周期编排                                                               │
│  ├─ systemd D-Bus：StartUnit / StopUnit / ResetFailed / ListUnits         │
│  │   模板单元：sandbox-runner@<sid> / sandbox-builder@<bid>               │
│  ├─ run-sandbox 启动器：pidfile 锁 → config-socket → execve sandbox-ctl   │
│  ├─ run-builder 启动器：pidfile 锁 → config-socket → 三阶段构建流水线      │
│  ├─ vswitch-ctl attach → {floatingip, port, mac} / detach                │
│  └─ sandbox-ctl snapshot / upload-snapshot（pause 快照）                  │
└──────────────────────────────────────────────────────────────────────────┘
                              │
┌──────────────────────── 本机控制 socket 层（UDS 0600）────────────────────┐
│  /run/sandbox/node-ctl.socket（h2c，SO_PEERCRED 鉴权）                    │
│  ├─ task 平面：launchspec / buildspec（启动器取工作规约）                   │
│  ├─ admin 平面：manifest-key 白名单管理                                    │
│  ├─ plugin 平面：routesync 订阅（proxy worker / 路由观察者）                │
│  └─ api 平面：e2b 控制面 handler 副本（export / import 扩展）               │
└──────────────────────────────────────────────────────────────────────────┘
              │                                    │
┌─── 数据面层 ──────────────────┐     ┌─── 集群接入层（node-link）──────────┐
│ proxy（internal/external）   │     │  routesync 引擎复用                  │
│ ├─ 路由判定（sid, port）      │     │  ├─ 拨 registry channel.listen       │
│ ├─ X-Access-Token 校验       │     │  ├─ 反向注册为路由权威                │
│ ├─ UDS(envd) / floatingip   │     │  ├─ 上报：register / heartbeat /      │
│ ├─ routesync 广播路由表       │     │  │       sandbox / build 事件         │
│ ├─ auto-resume 单飞           │     │  └─ 受理：create / connect /          │
│ └─ MMDS（mmds.enabled）      │     │          delete / key_put/drop 命令   │
└──────────────────────────────┘     └──────────────────────────────────────┘
              │
┌─── 资源管控层（resource_listen 内置 或 独立 daemon）────────────────────────┐
│  节点资源控制器（对接 sandbox-runtime pkg/resource 协议）                    │
│  ├─ Admission Controller：准入校验 / 令牌桶限速 / FIFO 排队                 │
│  ├─ Memory Allocator：水位仲裁 / burst grant / PSI 反压等待                 │
│  ├─ Reclaim Scheduler：settled 期主动回收（per-sandbox cgroup 收缩）         │
│  └─ State Persister：/run/node-ctl/state.json（tmpfs，原子写）              │
└──────────────────────────────────────────────────────────────────────────┘
              │
┌─── 状态存储层 ────────────────────────────────────────────────────────────┐
│  sqlite WAL（node-ctl.db，0600）                                          │
│  ├─ sandboxes 表：生命周期状态 + 快照引用 + 密文 manifest_key              │
│  ├─ builds 表：模板构建状态（兼任模板登记）                                  │
│  └─ manifest_keys 表：白名单（AES-256-GCM 密文 + SHA256 指纹索引）          │
└──────────────────────────────────────────────────────────────────────────┘
```

### 2.3 上下文组件详述

#### 2.3.1 sandbox-runtime（`sandbox-ctl` + `sandbox-init`）

**角色**：单沙箱生命周期引擎，host 端 + guest 端双进程承载。

**与 node-ctl 的交互**：

| 交互点 | 方向 | 说明 |
|---|---|---|
| `sandbox-ctl run --config <sid>.yaml --cgroup-adopt` | node-ctl → sandbox-ctl | 启动器 run-sandbox execve 替换；LaunchSpec 含 MANIFEST_KEY env |
| `sandbox-ctl snapshot --output / --upload` | node-ctl → sandbox-ctl | pause 时封快照；`--output` 本地，`--upload` 远程 manifest store |
| `sandbox-ctl run --restore <ref>` | node-ctl → sandbox-ctl | resume / auto-resume；ref 为本地 bundle 路径或 `manifest://<hex>` |
| `sandbox-ctl exec --stdin-from / --stdout-to journald=build` | run-builder → sandbox-ctl | 构建流水线平台接力（flatten-ctl 调用、配置注入、工件流） |
| `sandbox-ctl upload-snapshot <bundle>` | run-builder → sandbox-ctl | 构建收尾：本地快照提升为远程 manifest 快照 |
| `/run/sandbox/<sid>/ctl.sock` | 双向 | snapshot 命令连接运行中 sandbox-ctl 的控制 socket |
| pkg/resource 资源控制协议（UDS） | sandbox-ctl → node-ctl resource_listen | Admit / Settled / RequestBudget / Heartbeat / Release / Reattach |

**关键设计点**：

- **内存统一持有**：sandbox-ctl 创建并持有 memfd，CH 复用同一 inode；快照时 sandbox-ctl 直接 SEEK_DATA/HOLE 扫驻留页，无需 CH 中转
- **统一 cgroup**：`--cgroup-adopt` 让 sandbox-ctl 采纳自身所在 systemd 单元 cgroup，`KillMode=control-group` 保证 StopUnit 连 CH 一并 SIGKILL
- **块设备按需加载**：vhost-user-blk backend 在 sandbox-ctl 侧，I/O 按需拉取，配合 userfaultfd 内存按需加载
- **进程模型**：一个 `sandbox-ctl run` = 一个沙箱完整生命周期（类 runc run），退出 = 沙箱销毁

**sandbox-init（guest PID=1）**：

- 单一静态 Go 二进制，无 systemd/busybox
- 三阶段启动：早期挂载 + 并发取 LaunchSpec → overlayfs 组装 → exec 用户 app
- 承载 `sandbox-runtime.erofs`，通过 virtio-pmem + DAX 跨沙箱共享同一份 host page cache（~10 MiB 实际驻留）
- `/opt/sandbox-runtime/` 为平台发布件根（envd 所在），bind 进新 root 后对用户 app 只读可见

#### 2.3.2 cloud-hypervisor（VMM，sandbox-deps 定制补丁）

**角色**：KVM hypervisor，`sandbox-ctl` 的子进程。

**与 node-ctl 的交互**：

| 交互点 | 方向 | 说明 |
|---|---|---|
| `--memory-zone fd=<memfd>` | sandbox-ctl → CH | 外部传入的内存后端（统一持有模型） |
| vhost-user-blk socket | sandbox-ctl ↔ CH | blk0（base EROFS 镜像）/ blk1（overlay ext4 写层）|
| CH API UDS（ch.sock）| sandbox-ctl → CH | `GET /vm.snapshot` / `PUT /vm.resume` 等快照/恢复操作 |
| uffd socket | CH → sandbox-ctl | 缺页通道：CH 进程内创建 uffd，经 SCM_RIGHTS 传给 sandbox-ctl handler |
| vsock（CID=2:5000）| CH ↔ sandbox-init | 应用 stdio MUX + 控制面握手（launch/ping/app_started/app_exited）|

**三处定制补丁（~395 行，最小化）**：

1. **外部缺页处理器**：CH 进程内创建 uffd 后经 SCM_RIGHTS 传给 sandbox-ctl，使内存按需加载成立
2. **外部传入内存后端**：`--memory-zone fd=` 参数，复用 sandbox-ctl 持有的 memfd
3. **快照跳过外部托管内存区**：保存/恢复时不经 VMM 处理宿主托管内存，留给 uffd 按需触发

**路径**：`/opt/sandbox/bin/cloud-hypervisor`（或 `sandbox-ctl` 同目录 / PATH 自动发现）

#### 2.3.3 sandbox-vswitch（`vswitch-ctl`）

**角色**：基于 eBPF/TC 的虚拟交换机，单节点最多 4096 端口，纯内核转发。

**与 node-ctl 的交互**：

| 交互点 | 方向 | 说明 |
|---|---|---|
| `vswitch-ctl attach <sw> --inner-ip [--transit-*]` | node-ctl → vswitch-ctl | create 时为沙箱分配端口，返回 `{port, floatingip, mac}` |
| `vswitch-ctl detach <sw> --port=N` | node-ctl → vswitch-ctl | kill / pause 后释放端口 |
| `open-port` / `tapfd` 交接（TAPFD_SOCKET）| vswitch-ctl → cloud-hypervisor | sandbox-ctl 启动时取 tap fd，配置 virtio-net；经 SCM_RIGHTS 一次性交付 |
| `--mgmt-service 169.254.169.254:80:<mmds.listen>` | vswitch 配置 → node-ctl proxy | MMDS VIP eBPF 直译到 node-ctl mmds.listen；无 iptables，无特权端口 |

**关键设计点**：

- **纯内核数据面**：TC ingress eBPF 程序承载全部转发；控制面进程（vswitch-ctl）在 `attach`/`detach` 时操作 BPF map 后退出，数据路径无用户态进程
- **强隔离**：无 port→port 转发路径；ARP 全代答；MAC 在出口强制改写
- **tapfd 交接协议**：tap fd 经 SCM_RIGHTS + 元数据（inner_ip/mac/mtu）一次性交付 VMM，免去 VMM 按名打开的竞态
- **floatingip 路由**：每个沙箱一个唯一浮动 IP，node-ctl proxy 通过 `floatingip:port` 访问沙箱用户端口

#### 2.3.4 sandbox-accelerator（存储加速栈）

**架构**：

```
┌─ flatten-ctl ──────────┐     ┌─ manifest-ctl ──────────┐
│  OCI → EROFS 确定性展平  │────►│  分块 + 收敛加密 + 去重   │
│  Referrers 幂等回写      │     │  → store-ctl gRPC Put   │
└────────────────────────┘     └─────────────────────────┘
                                           │
                               ┌─ store-ctl ─────────────┐
                               │  fs / OBS 后端抽象        │
                               │  唯一写者 + generation 管理│
                               └─────────────────────────┘
                                           ↑
                               ┌─ cache-ctl tiered ───────┐
                               │  L1 RocksDB + EC client  │
                               │  → L2 shard cluster      │
                               └─────────────────────────┘
```

**各组件与 node-ctl 的交互**：

| 组件 | 交互 | 方向 | 说明 |
|---|---|---|---|
| **flatten-ctl** | `export --output - <fromImage>` | run-builder（guest 内）→ 宿主 | A 阶段：拉取镜像并展平为 tarstream 工件，经 exec stdio 流回宿主 |
| **flatten-ctl** | `export --skip-mounts --output - /` | run-builder（guest 内）→ 宿主 | B 阶段：steps 完成后导出增量镜像工件 |
| **flatten-ctl** | `mountpoint /.kuasar-build` | guest 内 | 构建 steps 导出前创建自 bind 挂载点，防自吞 |
| **flatten-ctl** | `tar extract --dense --stdin <tar>` | sandbox-ctl exec → guest | COPY step：宿主下载 tar → sandbox-ctl exec 管道 → guest 内 flatten-ctl 解包 |
| **manifest-ctl** | `store image.img`（stdout = 64-hex key） | run-builder → manifest-ctl | img-only 构建收尾上传，manifest key 经 stdout 回收 |
| **sandbox-ctl** | `upload-snapshot <bundle>` | run-builder → sandbox-ctl | snp 构建收尾：本地快照提升为远程快照（snapshot.cfg 引用改写为 manifest://）|
| **cache-ctl** | `127.0.0.1:7070`（wire 协议） | sandbox-ctl → cache-ctl | 块设备按需加载：chunk 缓存命中则 <100µs 返回，miss 穿透 L2/L3 |
| **store-ctl** | `127.0.0.1:7100`（gRPC Put/Get）| manifest-ctl / cache-ctl → store-ctl | chunk 持久化写入 / 读取；唯一写者保证去重一致性 |

**关键特性**：

- **确定性展平**：同一镜像每次展平字节级相同，保证 content-key 跨次稳定，去重率可达 85%+
- **收敛加密**：相同内容 → 相同密文，密文层面跨租户去重；全程零明文存储
- **按需加载**：sandbox-ctl vhost-user-blk backend 拦截 I/O → cache-ctl → store-ctl → OBS；实际访问量约 6.4%（镜像）/ 20-40%（内存快照）
- **分层缓存**：L1 RocksDB（节点本地 SSD，典型 1 TiB）→ L2 EC 集群（AZ 级，RS 4+1）→ L3 OBS；L2 命中率目标 99.9%
- **Referrers 幂等**：flatten-ctl 支持 OCI Referrers 回写，同一镜像已展平则直接复用 manifest id，跳过重导

#### 2.3.5 envd（guest 内，e2b profile 专用）

**角色**：guest 内原版 e2b envd，e2b SDK 数据面协议的实现方。

**与 node-ctl 的交互**：

| 交互点 | 方向 | 说明 |
|---|---|---|
| `sandbox-ctl --connect <host-uds:ip:port>` | sandbox-ctl | 将 guest 内 envd :49983 / :49999 映射为 host UDS（envd.sock / ci.sock）|
| `GET /health`（UDS）| node-ctl → envd | create 时等待 envd 就绪，60s 超时上限 |
| `POST /init`（UDS） | node-ctl → envd | 置 envVars / 默认用户 / 工作目录；mmds.enabled 时携带 accessToken |
| `POST /process.Process/Start`（UDS）| run-builder → envd | 构建流水线驱动 steps / startCmd / readyCmd（Connect-RPC，手写最小客户端） |
| proxy 透传 49983/49999（UDS）| client → node-ctl proxy → envd | e2b SDK 数据面：exec / 文件读写 / 运行代码 |

**配置**：

```
e2b profile launch:
  exec: /opt/sandbox-runtime/bin/envd
  args: [-isnotfc, -port, "49983"]   # mmds.enabled=true 时去 -isnotfc
  restart: always
  user: "0:0"                        # 需要 CAP_SETUID/SETGID 以目标用户跑工作负载
```

**envd 版本**：从 sandbox-deps `make envd` 构建（e2b-dev/infra 发布 tag，默认 `2026.22`），注入 `sandbox-runtime-e2b.erofs`，经 virtio-pmem DAX 共享。

#### 2.3.6 整体数据通路（create 全路径）

```
POST /sandboxes (e2b SDK)
  │ api_key MAC 验证 + 白名单准入
  │ 建 /run/sandbox/<sid>/ (tmpfs) + /var/lib/sandbox/<sid>/ (disk)
  ↓
vswitch-ctl attach sw0 --inner-ip 169.254.x.x
  → BPF map CAS Free→IP → 分配 floatingip + port + mac
  → open-port → tap fd 经 TAPFD_SOCKET 待 sandbox-ctl 取
  ↓
写 <sid>.yaml（非密：boot/network/resources/launch）
登记 sqlite sandboxes 行（manifest_key_enc = AES-GCM 密文）
  ↓
StartUnit(sandbox-runner@<sid>)
  └─ run-sandbox 启动器：
       pidfile fcntl 锁（防重入）
       → 拨 config-socket 取 LaunchSpec（含 MANIFEST_KEY env）
       → execve → sandbox-ctl run（接管单元 cgroup）
           → 接收 tap fd（tapfd 交接）→ 启动 cloud-hypervisor
           → 内存 memfd 统一持有 + uffd 注册
           → guest boot：sandbox-init 三阶段
               → overlayfs 组装（blk0=base EROFS，blk1=overlay ext4）
               → bind /opt/sandbox-runtime（envd 等发布件）
               → 并发取 LaunchSpec + switch-root
               → exec envd（e2b profile）
           → sandbox-ctl 轮询 envd /health（host UDS，60s 上限）
  ↓
POST /init（host UDS → envd）
  设 envVars / 默认用户 / workdir
  ↓
起 TTL（reaper 5s 周期，deadline 触发 auto-suspend）
  ↓
回 201 {sandboxID, envdAccessToken, domain, envdVersion, ...}
```

#### 2.3.7 构建流水线（三阶段，所有执行在 guest 内）

```
POST /v2/templates/{tid}/builds/{bid}（trigger）
  ↓
serve：建 workdir → vswitch-ctl attach（构建复用一个网络槽）
StartUnit(sandbox-builder@<bid>)
  └─ run-builder 驻留（Type=oneshot）：
  
  【A import 阶段】（有 fromImage）
    空盘沙箱（builder runtime），guest 内：
    flatten-ctl export --output - <fromImage>  （拉取 + 展平，FLATTEN_* env 提供凭据）
      └─ → cache-ctl L1 → L2 → OBS（层 blob 按需拉取）
    exec stdio 流回宿主 → workdir/image.img

  【B steps 阶段】（有 steps）
    以 base 镜像为 root + builder runtime + 大可写 upper：
    envd 作 app（-isnotfc，不 /init，无 token）
    for each step：
      RUN  → process.Start（UDS，/bin/bash -l -c <cmd>，按用户身份）
      ENV/ARG/WORKDIR/USER → 更新累积上下文
      COPY → sandbox-ctl exec → flatten-ctl tar extract（宿主下载 tar，guest 解包）
    步完 → flatten-ctl export --skip-mounts --output - /（导出增量工件）

  【C template 阶段】（有 startCmd）
    生产 e2b runtime 冷启最终镜像：
    envd 作 app（正常姿态，/init 预置）
    → startCmd 经 envd 启动（e2b 默认用户）
    → readyCmd 轮询（2s 间隔）
    → sandbox-ctl snapshot --output <bundle>
    → sandbox-ctl upload-snapshot <bundle>（上传所有工件到 manifest store）
    
  结果 JSON → stdout → <bid>.result
  ↓
serve 读 .result → 终态落库，持久 templateID = e2b-<snp|img>-<manifest_key>
```

### 2.4 部署拓扑

**节点常驻进程**（按启动顺序）：

```
1. vswitch-ctl serve      → eBPF 交换机起好，数据面就绪
2. store-ctl              → OBS sidecar :7100
3. cache-ctl tiered       → L1 RocksDB :7070 / :7071
4. node-ctl serve         → 控制面 :443 + 内置 resource_listen
                           + node-link（接入集群）
5. node-ctl proxy（可选） → external 模式数据面 worker
```

**关键端口与 socket**：

| 进程 | 地址 | 协议 | 用途 |
|---|---|---|---|
| `node-ctl` | `:443`（可配） | HTTPS/h2c | 对外：e2b 控制面 + 数据面 proxy |
| `node-ctl` | `/run/sandbox/node-ctl.socket` | UDS h2c（0600）| config-socket（task/admin/plugin/api 四平面）|
| `node-ctl resource_listen` | `/run/sandbox-resource.sock` | UDS 帧化 JSON | sandbox-ctl 资源控制协议 |
| `cache-ctl` | `127.0.0.1:7070` | wire（自定义 TCP）| chunk 数据面（sandbox-ctl 拉块设备数据）|
| `cache-ctl` | `127.0.0.1:7071` | gRPC | 健康检查 / info |
| `store-ctl` | `127.0.0.1:7100` | gRPC | Put/Get/GetSalt |
| `sandbox-ctl` | `/run/sandbox/<sid>/*.sock` | UDS | ch.sock / blk{0,1}.sock / uffd.sock / ctl.sock / vsock.sock |

---

## 3. API 设计

### 3.1 控制面 REST API（e2b 兼容）

**基址**：`https://api.<domain>`  
**鉴权**：`X-API-KEY: e2b_<72hex>` 或 `Authorization: Bearer <token>`

api_key 派生自 manifest_key，服务端通过 MAC 校验无需查表：

```
api_key = "e2b_" + hex(fp(12) ‖ ts(4) ‖ nonce(4) ‖ mac(16))
fp  = SHA256(manifest_key)[:12]
mac = HMAC-SHA256(manifest_key, fp‖ts‖nonce)[:16]
```

#### 3.1.1 沙箱生命周期

| 操作 | 方法 + 路径 | 状态码 | 请求体 | 响应体 |
|---|---|---|---|---|
| **create** | `POST /sandboxes` | 201 | `{templateID, timeout, metadata, envVars}` | `{sandboxID, templateID, clientID, domain, envdVersion, envdAccessToken, trafficAccessToken, alias}` |
| **get** | `GET /sandboxes/{id}` | 200 | — | `+state, startedAt, endAt, metadata, cpuCount, memoryMB` |
| **list** | `GET /v2/sandboxes` | 200 | query: `state, limit, nextToken` | 数组 + header `x-next-token` |
| **kill** | `DELETE /sandboxes/{id}` | 204 | — | — |
| **pause** | `POST /sandboxes/{id}/pause` | 204 / 409 | — | — |
| **resume** | `POST /sandboxes/{id}/connect` | 200 | `{timeout}` | 同 create 响应 |
| **timeout** | `POST /sandboxes/{id}/timeout` | 200 | `{timeout}` | — |

**错误语义**：

- `404`：资源不存在或非本租户（不泄露他租户存在性）
- `403`：manifest_key 不在白名单（create/build/import 时）
- `409`：操作冲突（如对已暂停沙箱再次 pause）
- `501`：数据面不支持（bare profile 的 49983/49999 端口，或未配 files_storage 时 COPY）

#### 3.1.2 模板构建 API（e2b v2 build system）

| 操作 | 方法 + 路径 | 状态码 | 说明 |
|---|---|---|---|
| **register** | `POST /v3/templates` | 202 | 分配 `transient-<uuidv7>` templateID + buildID；body: `{name, tags, cpuCount, memoryMB}` |
| **trigger** | `POST /v2/templates/{tid}/builds/{bid}` | 202 | body: `{fromImage, fromTemplate, steps[], startCmd, readyCmd}`；覆盖 register 的资源配置 |
| **status** | `GET /templates/{tid}/builds/{bid}/status` | 200 | `{status∈{building,ready,error}, logs[], logEntries[], reason?}`；`?logsOffset` 分页 |
| **files** | `GET /templates/{tid}/files/{hash}` | 201 | COPY 上传协商：`{present, url}`；url 为 presigned PUT，字节直传 OBS |
| **list** | `GET /templates` | 200 | 本租户 ready 模板，templateID 为持久 id |

**templateID 格式**：

```
持久 id:  <profile>-<kind>-<key>
           profile ∈ {e2b, bare}; kind ∈ {img, snp}; key = 64-hex manifest content key
临时 id:  transient-<uuidv7>     (构建注册期，build complete 后废弃)
```

**步骤支持**：`RUN / ENV / ARG / WORKDIR / USER / COPY`  
**steps / startCmd 约定**：使用 bash，镜像须有 bash（e2b 同款约束）

#### 3.1.3 导出/导入扩展（迁移）

| 操作 | 方法 + 路径 | 说明 |
|---|---|---|
| **export** | `POST /sandboxes/{id}/export` | 回单行 base64 迁移 token；`?to_template=true` 晋升持久 templateID；`?keep_source=true` 保留源行 |
| **import** | `POST /sandboxes/import` | body: `{token}`；校验 runtime 摘要；插 paused 行，回 `{sandboxID}` |

#### 3.1.4 沙箱配置传递链（metadata 命名空间）

客户端通过 `metadata["kuasar-sandbox.<ns>"]` 或 `X-Kuasar-Sandbox-<Ns>` 头注入配置（同名头胜出）：

| 命名空间 | 映射到 | 适用接口 |
|---|---|---|
| `resource` | `resources.{capacity, allocatable}` | create / register / trigger |
| `network` | hostname / nexthop（guest）/ inner_ip / transit_*（vswitch）/ dns | create |
| `launch` | exec / args / env / workdir / restart / user | create（仅 bare profile）|
| `init` / `mounts` / `files` | 直透 sandbox-ctl 对应字段 | create |
| `cluster` | group / route_key（集群路由身份）| create（node-link 命令注入）|

**优先级**：`节点默认 ⊕ 模板配置 ⊕ create 配置`（create 按命名空间胜）

**网络随快照**：渲染时注入 `SANDBOX_CONFIG.metadata["kuasar-sandbox.network"]`，随 snapshot.cfg 落盘并跨 restore 继承。

### 3.2 本机控制 socket 协议（UDS 0600，h2c）

| 平面 | 路径 | 鉴权机制 | 调用方 |
|---|---|---|---|
| **task** | `POST /internal/task/launchspec` / `buildspec` | peer-pid == pidfile | run-sandbox / run-builder |
| **admin** | `/internal/admin/manifest-keys` | peer-pid in admin_pidfile 或 socket 0600 同 uid | `node-ctl manifest-key`；集群 node-link key 命令 |
| **plugin** | `PUT /internal/plugin/{id}/register` | peer-pid in plugin_pidfile 或 socket 0600 同 uid | external proxy worker；路由观察者（platform-agent）|
| **api** | 其余路径 | `X-API-KEY` MAC | `export-sandbox` / `import-sandbox` CLI |

**LaunchSpec**（task 平面，run-sandbox 取用）：

```json
{
  "exec": "sandbox-ctl",
  "args": ["run", "--sandbox-id", "<sid>",
           "--config", "<rundir>/<sid>.yaml",
           "--manifest-config", "<shared manifest.yaml>",
           "--run-root", "/run/sandbox",
           "--cgroup-adopt",
           "--connect", "<uds:ip:port>",   // e2b profile
           "--stdout-to", "journald=sandbox",
           "--stderr-to", "journald=sandbox",
           "--console",  "journald=console"],
  "env": {"MANIFEST_KEY": "<hex>"}         // 密钥仅经 env 传递，不落文件
}
```

### 3.3 routesync 协议（plugin 平面，帧化 JSON over h2c）

双向全双工，单帧 ≤ 1 MiB：

| 方向 | 消息 | 载荷 |
|---|---|---|
| 订阅者→serve | `register` | `{subscribe{kind: route\|route_wake\|registry}, proxy{socket{path}}, mmds, resume_from?}` |
| serve→订阅者 | `hello` | `{version:1, policy{domain, auth_mode, park_timeout_ms}}` |
| serve→订阅者 | `upsert` | `RouteEntry{sid, profile, template_id, state, envd_uds, ci_uds, floatingip, access_token, snap_loc, mmds_secret, rev}` |
| serve→订阅者 | `delete` | `{sid, rev}` |
| serve→订阅者 | `bookmark` | — （初始全量完成标志）|
| 订阅者→serve | `wake` | `{sid}` （请求 resume，仅 route_wake）|

每条 upsert/delete 携带单调 `rev`，支持断线 `resume_from` 增量重放（留存窗口内），超出退回逐条全量重同步。

`RouteEntry.snap_loc` 为 `"local"` / `"remote"` / `""`，供 platform-agent 识别可迁移沙箱；`mmds_secret` 为 `HMAC-SHA256(manifest_key, "kuasar-mmds-v1:"+sid)` 确定性派生，多 worker 无需共享。

### 3.4 节点资源控制协议（UDS，LEN+JSON）

由 `sandbox-runtime/pkg/resource` 权威定义，node-ctl 为参考实现对端：

| 消息 | 发送方 | 说明 |
|---|---|---|
| `Admit(capacity, floor, startup_budget, cgroup_path)` | sandbox-ctl | 准入申请 |
| `AdmitResponse(status, token, granted_initial_alloc, reason?)` | 控制器 | admitted / rejected |
| `Settled(token, current_rss, current_cpu_usec)` | sandbox-ctl | 进入 settled，归还 startup_pool |
| `RequestBudget(token, requested_delta, urgency)` | sandbox-ctl | burst 申请 |
| `BudgetResponse(granted_delta, new_allocatable, cooldown_ms)` | 控制器 | 仲裁结果 |
| `OOMReport(token, oom_count, killed_pid)` | sandbox-ctl | OOM 紧急上报 |
| `Heartbeat(token, current_rss, current_cpu_usec)` | sandbox-ctl | 30s 周期 |
| `HeartbeatAck(new_allocatable)` | 控制器 | 传播 reclaim 调整 |
| `Release(token, reason)` | sandbox-ctl | 沙箱退出 |
| `Reattach(token)` | sandbox-ctl | 断线重连重绑 reservation |

### 3.5 命令行接口（node-ctl / e2b-key-ctl）

**node-ctl**：

| 子命令 | 用途 |
|---|---|
| `serve [--config] [--proxy internal\|external\|off]` | 启动 daemon |
| `proxy --config-socket=<uds> --id=<name> --socket=<uds>` | 外置数据面 worker |
| `run-sandbox / run-builder` | systemd 单元内启动器（内部，非人用）|
| `resource status/list/drain/grant/reclaim` | 资源控制器运维 |
| `config [--template] [-o]` | 配置规范化/输出骨架 |
| `manifest-key add/remove/check/list [--ttl] [--registry-auth]` | 白名单管理 |
| `export-sandbox <sid> [--to-template] [--keep-source]` | 导出迁移 token |
| `import-sandbox <token>` | 导入 paused 行 |
| `version` | 版本 |

**e2b-key-ctl**：

| 子命令 | 用途 |
|---|---|
| `gen-key` | 随机 32B manifest key（64-hex）|
| `gen-apikey [<MK>]` | 派生 e2b api key（`e2b_`+72hex，76字符）|
| `fingerprint [<MK>]` | 24-hex 指纹（与白名单/库内索引一致）|
| `seal-pull-token [<MK>] …` | 封装镜像拉取令牌（`kpt_` 前缀，AES-GCM）|

---

## 4. Use Case

### UC-01：独立节点接入（运维人员）

**目标**：新计算节点快速接入，租户能用 e2b SDK 直连。

**主流程**：

1. 节点上预先启动：`vswitch-ctl serve` → `store-ctl` → `cache-ctl tiered` → `node-ctl serve`
2. 运维生成租户根密钥：`MK=$(e2b-key-ctl gen-key)`
3. 注册白名单：`node-ctl manifest-key add "$MK" --label tenant-a`
4. 派生 SDK 凭据：`export E2B_API_KEY=$(e2b-key-ctl gen-apikey "$MK")`
5. 租户配置 `E2B_DOMAIN=sandboxes.example.com`，e2b SDK 直连本机
6. 运行 `sandbox.create(template_id="e2b-snp-<key>")` → 快照恢复拉起沙箱

**后置条件**：客户端持 api_key，能全生命周期管理沙箱；manifest_key 密文存 sqlite

---

### UC-02：集群模式接入（运维人员）

**目标**：节点加入 cluster-ctl 管理的机群，接受 registry 编排。

**主流程**：

1. 配置 `cluster.registry`、`cluster.node_id`、`cluster.labels`（zone/pool/slot）、`cluster.tls`
2. `node-ctl serve` 启动后自动拨 registry，以 routesync 反向注册为路由权威
3. registry 经 `key_put` 命令下发租户 manifest_key 租约（TTL 3h，每 1h 续租）
4. registry 经 `build_register` 预配构建任务（resources / image_repo / registry_auth）
5. create 请求：router → registry.ReserveSandbox → PlaceSandbox 建议 → 提交 RESERVED → `create` 命令 → 本节点执行 create 流程 → 上报 sandbox 事件 → READY
6. 本节点心跳报 `{zone, allocated, pool, build_alloc, draining}` 给 scaler P2C 放置

---

### UC-03：沙箱完整生命周期（e2b SDK 用户）

**主流程**：

1. `POST /sandboxes` → 201 → sandboxID + envdAccessToken
2. exec / 文件读写 / 运行代码（49983/49999 经 proxy → envd UDS）
3. TTL 到期 → reaper 触发 auto-suspend：`sandbox-ctl snapshot --output / --upload` → StopUnit → sqlite 标 paused
4. 用户再次访问 → proxy 命中 paused → auto-resume 单飞 → StartUnit（--restore）→ 等 running → 解挂
5. `DELETE /sandboxes/{id}` → StopUnit + ResetFailed + vswitch-ctl detach + 删运行目录 + sqlite 删行

---

### UC-04：模板构建（e2b CLI / SDK）

**主流程**：

1. `POST /v3/templates` → 202 → transient templateID + buildID
2. （可选）客户端 `docker build && docker push`，trigger 不带 fromImage（由 `image_uri_mask` 推导）
3. `POST /v2/templates/{tid}/builds/{bid}` → trigger，body 含 fromImage / steps / startCmd
4. 有 COPY：`GET /templates/{tid}/files/{hash}` → presigned PUT → 客户端直传 OBS
5. serve 启 sandbox-builder@<bid>（oneshot）→ run-builder 三阶段
6. 轮询 status + logs（journald → journalctl → serve → SDK on_build_logs）
7. ready → 持久 templateID = `e2b-snp-<key>`

**构建环境隔离**：拉取凭据（FLATTEN_REGISTRY_*）仅经 exec env 入 guest；MANIFEST_KEY 永不入 guest

---

### UC-05：跨机沙箱迁移

**目标**：将 paused 沙箱移到另一节点（节点维护 / 再平衡）。

**一步迁移流程**：

1. 源节点：`node-ctl export-sandbox <sid>` → 确保远程快照（sandbox-ctl upload-snapshot）→ 输出 base64 token
2. 目标节点：须共享同一 manifest store，且已 `manifest-key add "$MK"`
3. 目标节点 connect API：`Sandbox.connect(<sid>, api_headers={"X-Kuasar-Migration-Token": <token>})` → import + restore，一次 SDK 调用收敛

**集群自动迁移**（registry 驱动）：

1. registry 检测 PAUSED 沙箱需迁移 → 下发 `delete{sid}` 到源节点（SAVED 两阶段：源保留直至 token 持久化）
2. ReserveSandbox → PlaceSandbox → `create{sid, migration_token}` 到目标节点 → 一步迁移 → running 事件 → READY

---

### UC-06：节点资源超分与动态仲裁（自动）

**前提**：`resource_listen` 已配置，沙箱 `sandbox.resources.control_socket` 非空。

**主流程**：

1. `sandbox-ctl run` 发 `Admit(capacity=8G, floor=128M, startup=1G)` → 控制器评估水位 → admitted
2. 稳态：实际 RSS=128M，控制器 `allocatable_now=128M`，cgroup memory.high 限在此
3. 用户请求触发 burst：`RequestBudget(urgency=normal, delta=+4G)` → 令牌桶限速批准 → cgroup memory.high 放开到 4.128G
4. 请求处理完：RSS 回落 → `Heartbeat(current_rss=128M)` → Active Reclaimer 下发 `new_allocatable=128M` → cgroup 收缩
5. 多沙箱同时 burst：grant 速率受限（allocatable_pool × 0.05 / s），未获批在 PSI 反压下退避重试

**节点水位上报**（集群 P2C 信号）：node-link 心跳 `{allocated, pool}` → scaler PlaceSandbox 过滤 red/critical 区节点

---

### UC-07：external 数据面 worker 部署（运维）

**目标**：数据面与控制面进程隔离，数据面可独立扩缩。

**流程**：

1. 配置 `proxy.mode=external`，node-ctl serve 不绑数据口
2. 起 N 个 `node-ctl proxy --id=worker-N --socket=/run/.../worker-N.sock --data-listen=:443`（SO_REUSEPORT）
3. 每个 worker 经 plugin 平面注册 `{subscribe:route_wake, proxy:{socket}}`
4. serve 向所有 worker 广播路由（初始全量 upsert + bookmark + 实时增量）
5. worker 独立：路由判定 + token 校验 + 转发 + Wake 上行（paused 沙箱唤醒）
6. worker 崩溃：其余 worker 继续；重启后自动重注册重同步；serve 兜底网关按 FNV(sid) 哈希转到活跃 worker

---

## 5. User Story

### US-01：e2b SDK 零改造接入

> **作为** AI 应用开发者，已用 e2b Python/JS SDK 开发代码解释器工作流，  
> **我希望** 只修改 `E2B_DOMAIN` 和 `E2B_API_KEY`，  
> **就能** 将工作流对接到自托管节点，不修改任何代码。

**验收标准**：
- `sandbox.create()` / `sandbox.exec()` / `sandbox.pause()` / `code_interpreter.run_code()` 正常工作
- `envdVersion` 响应为 `0.6.1`（SDK 版本校验通过）
- `X-Access-Token` 自动携带与校验
- 数据面延迟无明显劣化（2 个网络跳：SDK → proxy → envd UDS）

---

### US-02：安全的多租户隔离

> **作为** 平台运维人员，  
> **我希望** 每个租户有独立根密钥，租户 A 的沙箱对租户 B 完全不可见，  
> **并且** 密钥永远不以明文形式落盘。

**验收标准**：
- 跨租户操作（get/kill/pause）返回 **404**（不泄露存在性）
- sqlite `manifest_key_enc` 列为 AES-256-GCM 密文（`encryption_key` 加密）
- api_key 由 manifest_key HMAC 派生，服务端只存密文，运行期只在内存 + env 帧
- 集群模式：manifest_key 经 mTLS 通道以 TTL 租约下发，节点落密文

---

### US-03：沙箱 TTL 与透明自动恢复

> **作为** AI Agent 平台，  
> **我希望** 空闲沙箱自动暂停释放节点资源，用户再次访问时无感知恢复，  
> **且** 暂停/恢复过程对用户 API 完全透明。

**验收标准**：
- TTL 到期 → auto-suspend（快照封存）→ 路由表标 paused（不删除）
- 下次数据面请求触发 auto-resume（单飞，并发去重），等待不超过 `park_timeout`（默认 30s）
- `POST /sandboxes/{id}/timeout` 可随时续期
- `POST /sandboxes/{id}/connect` 明确 resume，可携带 `timeout` 顺带续期

---

### US-04：自定义模板构建（实时日志）

> **作为** AI 平台租户，  
> **我希望** 用 `e2b template build` CLI 构建自定义环境，  
> **并且** 构建日志实时可见，不需要等构建结束才看到结果。

**验收标准**：
- `status` API 的 `logs[]` / `logEntries[]` 按 `?logsOffset` 分页，SDK `on_build_logs` 流式输出
- 每行有 `{timestamp, level, message}`，level 从 PRIORITY 映射（≤3=error, 4=warn, ≥7=debug）
- flatten 拉取进度（`pull: N/M layers`）、RUN 输出、flatten 导出进度均可见
- 失败时 `status=error`，`reason.message` 指引查看日志详情；详情已在日志流中

---

### US-05：单节点高密度超分承载（60×）

> **作为** 基础设施团队，  
> **我希望** 单节点承载 capacity=8GiB × 256 个沙箱（总 capacity 2TiB，超过物理内存），  
> **且** 活跃 burst 的沙箱不被 OOM kill，节点不发生 host OOM。

**验收标准**：
- guest OOM 频次 = 0 / 节点 / 小时（burst 沙箱获批 budget 后 cgroup 放开）
- host OOM 频次 = 0 / 节点 / 小时（water mark 保证 node_allocated ≤ allocatable_pool）
- 多沙箱同时 burst：grant 速率受令牌桶限制，不同时放开撞墙
- `node-ctl resource status` 可查水位区 / 利用率 / reservation 数

---

### US-06：节点维护安全腾空

> **作为** 运维人员，  
> **我希望** 节点维护前能安全排空，让集群停止分配新沙箱，  
> **且** 已运行的沙箱能迁移而不是强杀。

**验收标准**：
- `node-ctl resource drain` → 控制器不再 admit 新沙箱
- 心跳 `draining=true` → scaler 排除该节点（不下发 drain 命令，节点侧自主）
- platform-agent 订阅 `snap_loc` 路由字段，识别 `"local"` paused 沙箱自动铸造迁移 token 并完成迁移
- `"remote"` paused 沙箱可直接在新节点 resume，无需源节点在线

---

### US-07：集群模式会话亲和（cluster-ctl 视角）

> **作为** cluster-ctl registry，  
> **我希望** node-ctl 节点通过 node-link 接入，上报沙箱状态，接受放置命令，  
> **且** 节点控制面逻辑（create/pause/kill/build）不变，集群只是多一层命令通道。

**验收标准**：
- `node-ctl serve` 启动后自动连接 registry，routesync 反向注册，初始 bookmark 完成
- registry `create{migration_token}` 触发一步快照迁移拉起沙箱
- TTL pause 后发 `sandbox{state:paused, snap_loc:"local"}` 事件，registry 状态收敛到 PAUSED
- node-link 断线后指数退避重连，`resume_from` 增量重报（留存窗口内只补增量）

---

## 6. 可靠性设计

### 6.1 故障域与自愈矩阵

| 故障 | 影响范围 | 自愈机制 |
|---|---|---|
| **node-ctl serve 崩溃/重启** | 控制面暂断；internal 数据面暂断；沙箱 microVM 不受影响 | systemd 自动重启 → 重启对账（§6.2）→ 收养 running 单元；external worker 本地路由表继续服务 running 流量 |
| **external proxy worker 崩溃** | 该 worker 连接断；其余 worker（SO_REUSEPORT）继续 | systemd 重启 → 重注册 + 全量重同步（逐条 upsert + bookmark），无路由空窗 |
| **runner 单元/cloud-hypervisor 崩溃** | 该沙箱死（`Restart=no`，有状态不重试）| 对账标 dead；客户重 create 或从 paused 快照 resume |
| **资源控制器崩溃** | sandbox-ctl 断连；新 admit 失败；已运行沙箱降级独立运行 | systemd 重启 → 读 state.json + 扫 cgroup_scan_paths → 交叉对账 → 等 Reattach；cgroup 是真相之源 |
| **routesync 断流** | worker 路由表停更 | 指数退避重连（0.2s 起/5s 封顶）→ 重注册 + 逐条 upsert + bookmark；重同步全程旧路由表仍在服务 |
| **node-link 断流（集群）** | registry 暂失本节点视图 | 节点指数退避重连，`resume_from` 增量重报；本节点沙箱不受影响 |
| **sandbox-ctl/cloud-hypervisor 崩溃** | 该沙箱死 | 对账标 dead；vswitch-ctl detach；下次 create/resume 重建 |
| **cache-ctl 崩溃** | 块设备 / 快照加载变慢（L1 miss 穿透 L2/OBS）| 重启恢复 RocksDB；沙箱降级 L2/L3 继续运行（延迟上升但不中断）|
| **vswitch-ctl 数据面异常** | 沙箱网络中断（eBPF TC 程序仍运行）| vswitch-ctl 重启后重挂接 bpffs；数据面 eBPF 程序持久化在 bpffs，与用户态进程解耦 |
| **sqlite 损坏** | 控制面不可用 | 文件级备份/重建；沙箱单元仍可由 ListUnits 发现，运维处置 |
| **整机重启** | 所有沙箱消失（tmpfs /run 清空）| paused 行（磁盘快照）保留；重启后 running 行标 dead；paused 可被重新 resume |
| **host OOM** | 内核 OOM killer 选目标 | 水位机制保证正常不触发；controller / cache-ctl OOM score 高于 sandbox（先被杀，sandbox 优先）|

### 6.2 重启对账（serve 重启核心流程）

```
serve 重启后：
  1. ListUnitsByPatterns("sandbox-runner@*.service") 获取存活权威集
  2. 对账：
     单元 active/activating && sqlite running → 收养（重挂内存路由、TTL 继续、推给 worker/registry）
     sqlite running && 无对应活单元       → 清理（StopUnit + detach + 删运行目录 + 标 dead）
     run_root 为 tmpfs                    → 整机重启后所有 running 判 dead
  3. paused 行（磁盘快照）保留，可被 connect/auto-resume 拉起
  4. 集群下：node-link 重连 → resume_from 增量重报本节点沙箱集
```

### 6.3 资源控制器可靠性（重启恢复流程）

```
控制器重启（host 未重启）：
  1. 读 /run/node-ctl/state.json（tmpfs，成功则得 reservation 快照）
  2. 扫 cgroup_scan_paths（/sys/fs/cgroup/sandboxes/）得活沙箱列表
  3. 交叉对账：
     cgroup 在 && state.json 有 → 标"待重连"
     cgroup 在 && state.json 无 → 临时收纳（孤儿 cgroup，等 Reattach）
     state.json 有 && cgroup 不在 → 沙箱已死，丢弃 reservation
  4. 接受新 Admit；已知活 reservation 暂无连接，等 sandbox-ctl Reattach(token) 重绑
  5. IdleSweeper：心跳 90s 静默则视为死亡释放
```

**不变量**：cgroup 是真相之源，state.json 是性能优化；`write→fsync→rename` 原子写防截断。

### 6.4 状态一致性保证

| 不变量 | 实现机制 |
|---|---|
| systemd 单元是 running 存活权威 | 对账以 ListUnits 为准，sqlite 为辅 |
| manifest_key 不明文落盘 | AES-256-GCM 密文列；运行期仅内存 + env 帧 |
| 单元 cgroup = 沙箱资源 cgroup | `--cgroup-adopt`；`KillMode=control-group` |
| auto-resume 幂等 | per-sid single-flight；并发 Wake 合并为一次 StartUnit |
| builds 表 CAS 抢占 | waiting→building 用 CAS，重启/多实例安全 |
| SAVED 两阶段不丢态 | 源节点保留本机快照直至 registry 持久化 token + 下发 delete |

---

## 7. 性能设计

### 7.1 热路径（数据面转发）

| 路径 | 延迟来源 | 优化 |
|---|---|---|
| running 沙箱 e2b 端口（49983/49999）| 网络 RTT + UDS 直达 | `ReverseProxy{FlushInterval:-1}` 零缓冲；envd UDS 无 floatingip 跳；无每请求控制面往返 |
| running 沙箱用户端口 | 网络 RTT + floatingip:port | 直接 dial floatingip，eBPF 数据面转发 |
| 路由表查询 | O(1) | 内存 hash map，worker 本地缓存（无锁，无远程往返）|
| X-Access-Token 校验 | 常数时间 | `hmac.Equal` |
| 集群热路径（router 本地缓存命中）| 零控制面往返 | router 订阅路由流本地缓存，已建立会话直转 |

### 7.2 冷路径延迟分解

| 场景 | 延迟来源 | 优化手段 |
|---|---|---|
| **auto-resume（本机快照）** | 快照恢复（亚秒）+ 单元启动 + envd 就绪 | 单飞合并并发 wake；paused 路由保留（无 404 gap）；本机 bundle 避免远程拉取 |
| **集群 PAUSED 恢复** | 免放置（同节点 connect）+ 快照恢复 | ReserveSandbox 直走 connect 命令，跳过 PlaceSandbox |
| **集群 SAVED 恢复（shuffle-sharding）** | 放置建议（scaler 本地 O(1)）+ 快照恢复 | shuffle-sharding 缓存局部性：group 钉 n 个 slot，chunk 在固定节点常热，免远程拉取 |
| **create（snp 模板快照恢复）** | 控制往返（UDS）+ 快照恢复（亚秒）+ envd 就绪（≤60s）| snp 模板比冷启快数量级；密钥预置（无 key 推送延迟）|
| **create（img 模板冷启）** | microVM boot + 镜像按需加载 + envd 就绪 | vhost-user-blk 按需加载（仅访问 ~6.4%）；L1 cache 命中 <100µs；DAX 共享 sandbox-runtime |

### 7.3 高密度超分指标

| 指标 | 目标 | 关键机制 |
|---|---|---|
| guest OOM 频次 | 0 / 节点 / 小时 | burst grant 保证有预算沙箱可拉到 capacity |
| host OOM 频次 | 0 / 节点 / 小时 | water mark（allocatable_pool × 85% 触发拒绝）|
| burst P99 应用延迟 | < 100ms 或基线 × 1.2 | 令牌桶限速防惊群；PSI 反压等待而非强杀 |
| budget-grant P99 | < 50ms | 控制器 RPC 计时 |
| Admit 拒绝率（green/yellow 区）| < 5% | startup_pool 自然限流（按内存而非个数）|
| 节点预算利用率 | > 75%（内存时间均值）| Active Reclaimer 主动收回 settled 沙箱 |
| 密度提升倍数 | ~60×（`physical / typical_floor`）| 超分 + 动态仲裁 + balloon 回收 |

**防惊群三重机制**：

1. **令牌桶限速**：每秒总 grant ≤ `allocatable_pool × memory_grant_per_sec_factor`（默认 5%）
2. **startup_pool 隔离**：admission 阶段按 `effective_startup_budget` 消耗 startup_pool，自然限流并发创建数
3. **PSI 反压**：未获批的 burst 在 cgroup memory.high PSI 反压下退避等待，不堆积大量 RequestBudget

### 7.4 扩展性

| 维度 | 方案 | 瓶颈上限 |
|---|---|---|
| 数据面水平扩 | external + SO_REUSEPORT：N 个对等 worker 共享数据口 | 内核连接分发，无主从 |
| routesync 重连 | 逐条 upsert + bookmark + resume_from 增量重放 | 高密度下内存有界（无全量巨帧）|
| 构建并发 | `builder.max_concurrent` 计数信号量 + sandbox-builder.slice cgroup 上限 | CPU/内存上限由 slice 约束 |
| 集群 router 扩 | 无状态 N 副本置 LB，断线增量重放 | cache 同步带宽有界 |
| 集群 registry 扩 | SandboxStore/BuildStore 按 group 分片；NodeStore 全局 | 接口为 etcd/raft 预留 |

### 7.5 构建流水线性能

- **阶段复用**：构建复用一个 vswitch 网络槽（attach 一次，各阶段顺序交接 tapfd）
- **工件流传**：镜像工件经 exec stdio 流回，宿主不落大文件中转（内存+管道直传）
- **Referrers 幂等**：flatten-ctl 支持 OCI Referrers 回写，同一镜像已展平则跳过拉取+展平+上传（最快路径）
- **COPY 直传桶**：文件不经控制面，presigned PUT 直接上传 OBS，触发时 HEAD 确认

---

## 8. 安全设计

### 8.1 密钥模型

```
manifest_key（32B/64-hex）              ← 每租户根密钥，永不明文落盘
    │ HMAC-SHA256 派生
    ▼
api_key = "e2b_" + hex(fp‖ts‖nonce‖mac)   ← e2b SDK 格式校验 /^e2b_[0-9a-f]+$/
    │ SHA256[:12] → fp（O(1) 库内预筛）
    │ 解密 manifest_key_enc → HMAC 全校验（杜绝 hash 碰撞串租户）
    ▼
sqlite manifest_key_enc                 ← AES-256-GCM（keytag‖nonce‖ct+tag）
    │ env 帧（LaunchSpec / BuildSpec）
    ▼
MANIFEST_KEY env（sandbox-ctl / manifest-ctl 进程）  ← 仅内存，不落文件
```

**密钥轮换**：`encryption_key` 支持 `:` 分隔多键；`[0]` 为活动密钥，其余解旧记录；`keytag` 字段标记使用哪把密钥，轮换时重加密存量记录即可。

### 8.2 认证与授权体系

| 位置 | 机制 | 保护对象 |
|---|---|---|
| **控制面 api_key** | fp 预筛 + HMAC-SHA256 MAC 全校验 | 所有 e2b REST 操作 |
| **归属校验** | api_key → manifest_key → 资源行 manifest_key 比对 | 跨租户访问：统一返回 404 |
| **白名单门** | manifest_key 须在 manifest_keys 表（create/build/import）| 防止未授权沙箱创建 |
| **本机 socket（task）** | `SO_PEERCRED` peer-pid == pidfile | LaunchSpec/BuildSpec 取用（仅启动器）|
| **本机 socket（admin）** | peer-pid in admin_pidfile 或 socket 0600 同 uid | 白名单写操作 |
| **本机 socket（plugin）** | peer-pid in plugin_pidfile 或 socket 0600 同 uid | proxy worker / 路由观察者注册 |
| **数据面 token** | `X-Access-Token` per-sandbox 随机 token，常数时间比较 | 数据面请求（proxy.auth=enforce）|
| **node-link mTLS** | 证书 SAN / 指纹背书 node_id | registry 命令（manifest_key 下发、create/delete）|
| **构建凭据** | AES-GCM 密文存 builds.registry_auth_enc，exec env 传入 guest | 镜像拉取凭据隔离 |

### 8.3 数据面纵深防御

```
客户端
  │ X-Access-Token
  ▼
proxy（enforce：不符 → 401）          ← 第一道闸门（proxy.auth=enforce）
  │ host UDS（envd.sock）
  ▼
envd（mmds.enabled=true 时）           ← 第二道闸门（MMDS re-key + 每身份 token）
  │
  ▼ 业务逻辑
```

| 姿态 | 数据面闸门 | 适用场景 |
|---|---|---|
| `mmds.enabled=false`（默认）| proxy 单闸门（强制 enforce）| 简单部署；快照扇出无 envd token 错配（envd 非 secure）|
| `mmds.enabled=true` | proxy + envd 纵深防御 | 高安全要求；FC 兼容；快照扇出子沙箱数据面可用（MMDS re-key）|

**MMDS 安全机制**：

- session token = `<sid>.<HMAC>`，HMAC 密钥 = `HMAC-SHA256(manifest_key, "kuasar-mmds-v1:"+sid)`
- 确定性派生：N 个 worker 无主从，PUT 与 GET 落到不同 worker 也一致
- VIP（169.254.169.254）经 vswitch eBPF mgmt-extract 直译到 mmds.listen，无特权端口

### 8.4 网络隔离

- **envd 控制端口不走 floatingip**：49983/49999 仅经 proxy host-UDS（`--connect` 映射）可达，沙箱间无通路
- **沙箱 eBPF 强隔离**：无 port→port 转发路径；MAC 出口强制改写；ARP 代答；转发判定基于 slot_id 算术，不信任沙箱报文源地址
- **vswitch 数据面持久**：控制面进程崩溃不断流（TC eBPF 程序 + BPF map 持久化在 bpffs）

### 8.5 镜像构建安全边界

| 边界 | 保护措施 |
|---|---|
| 拉取凭据不入 guest 文件 | 仅经 exec env（`FLATTEN_REGISTRY_*`）传入；workdir 里的阶段 yaml 无秘密 |
| MANIFEST_KEY 不入 guest | exec env 中 MANIFEST_KEY 不转发给 guest |
| COPY 隔离 | 文件不过控制面（presigned PUT 直传 OBS）；桶私有，客户端无凭据；端点归属校验只签属主路径 |
| 构建沙箱隔离 | 每次构建独立 microVM；KVM 强隔离；no-steps+no-startCmd 拒绝（无事可做）|
| runtime 摘要校验 | 迁移 token 含 runtime erofs 摘要；目标机摘要不匹配拒绝导入 |

### 8.6 迁移 Token 安全

- token 携带沙箱自有 env 与数据面 token，按"沙箱级敏感"处理
- 默认 move（回收源行）；`--keep-source` 为显式 copy
- **按需铸造，绝不随 routesync 路由广播**（`snap_loc` 可广播；migration_token 仅在 export 时铸造）
- 集群 SAVED 两阶段：registry 持久化 token 后才下发 delete，token 消费后源行回收

### 8.7 admin 平面访问控制

| 配置 | 效果 |
|---|---|
| `paths.admin_pidfile` 非空 | admin 平面 peer-pid 须在多行 PID 白名单中（`#` 注释行） |
| `paths.plugin_pidfile` 非空 | plugin 平面（proxy/agent）同上 |
| 均为空 | 仅靠 UDS 0600 文件权限（同 uid / root）保护 |

**信任模型**：host root / daemon uid 可信；guest 内代码在 microVM 内，KVM 隔离，无法触及 host UDS。

### 8.8 审计与可观测性

| 对象 | 机制 |
|---|---|
| 资源仲裁高频事件 | `resource_listen.logging.audit_path`（tmpfs，原子写，不写磁盘）|
| 构建日志 | journald 单汇（SYSLOG_IDENTIFIER=build），不含平台密钥信息 |
| `manifest-key list` | 只显示指纹（24-hex），绝不回显 key 本体 |
| 数据面 Prometheus | `data_requests_total{result=ok\|unauthorized\|notfound\|denied}`；`gateway_forward_total{result}` |
| 构建日志流 | 对 SDK `on_build_logs` 实时可见；guest 内核 dmesg（tag=console）刻意过滤，不进 SDK |

---

## 9. 设计规格

本章描述 node-ctl 各核心特性的行为契约，包含触发条件、前置/后置条件、选择逻辑、错误行为与幂等性保证，作为实现与验收的依据。

### 9.1 沙箱生命周期管理规格

#### 9.1.1 create

**前置条件**：
- api_key MAC 校验通过，且 manifest_key 指纹存在于白名单
- templateID 非空时，须符合持久格式（`<profile>-<kind>-<key>`）或为 `transient-*`

**启动模式选择逻辑**：

```
templateID 含 "-snp-"  →  快照恢复（sandbox-ctl run --restore manifest://<key>）
templateID 含 "-img-"  →  冷启（sandbox-ctl run，block device = 对应 manifest 镜像）
templateID 为空或为默认 →  冷启（使用 sandbox.default_template 配置）
```

**资源分配顺序**（顺序不可调换，后步失败须回滚前步）：

1. Admit（resource_listen）→ 拒绝则返回 503，不继续
2. `vswitch-ctl attach` → 分配 floatingip + port + tapfd
3. 写 `<rundir>/<sid>.yaml`（非密配置）和 sqlite 行（state=running，manifest_key_enc 密文）
4. `systemd StartUnit(sandbox-runner@<sid>)`

**就绪条件**：
- e2b profile：envd `/health` 返回 200，超时 60s 则视为 create 失败
- bare profile：systemd unit 进入 active 状态

**后置不变量**：sqlite `state=running`，routesync upsert 已广播，TTL reaper 已登记 `deadline_unix`

**失败回滚**：步骤 N 失败，逆序回滚已完成步骤；回滚本身失败时记 error 日志，不阻塞响应，沙箱行标 dead

#### 9.1.2 pause

**前置条件**：sid 存在且 `state=running`；`state=paused` → 返回 409；不存在 → 404

**快照目标选择**：

```
checkpoint.mode=local   →  sandbox-ctl snapshot --output <bundle_path>
checkpoint.mode=remote  →  sandbox-ctl snapshot --upload（直传 manifest store）
```

**执行顺序**：

1. `sandbox-ctl snapshot`（经 ctl.sock 发送 snapshot 命令）
2. `systemd StopUnit(sandbox-runner@<sid>)`
3. sqlite 更新：`state=paused`，`snapshot_ref=<local:path 或 manifest://key>`
4. routesync upsert 广播（state=paused，snap_loc=local 或 remote）

**路由表行为**：pause 后路由条目**保留**（state 改为 paused），proxy 可识别并触发 auto-resume；不删除

#### 9.1.3 connect（resume）

**前置条件**：sid 存在

**按 state 分支**：

```
state=running  →  只更新 TTL（deadline_unix = now + timeout），不重启，直接返回当前沙箱信息
state=paused   →  执行恢复流程（见下）
state=dead     →  404（已不可恢复）
```

**恢复流程**：

1. 单飞判定（per-sid `singleflight`）：并发 connect 同一 paused sid 合并为单次执行
2. `systemd StartUnit(sandbox-runner@<sid>, --restore <snapshot_ref>)`
3. 等待 envd `/health`（e2b profile，上限 60s）
4. sqlite 更新：`state=running`，`snapshot_ref` 清空，`deadline_unix` 更新
5. routesync upsert 广播（state=running）

**失败行为**：StartUnit 失败或 envd 超时 → sqlite 保持 paused，向所有等待者返回 503，路由表保持 paused

#### 9.1.4 kill

**前置条件**：sid 存在（任意 state）；不存在 → 直接 204（幂等）

**执行顺序**（严格顺序，各步骤失败均记录但不中止后续）：

1. `systemd StopUnit(sandbox-runner@<sid>)`
2. `systemd ResetFailed(sandbox-runner@<sid>)`（清除 failed 状态，防再次 Start）
3. `vswitch-ctl detach --port=<vswitch_port>`
4. `rm -rf /run/sandbox/<sid>/`（tmpfs，快速）
5. sqlite 删行（或标 dead，取决于是否需要 audit trail）
6. routesync delete 广播

**原子性保证**：sqlite 删行在所有资源回收完成后执行；删行失败不影响实际资源已回收的事实

#### 9.1.5 TTL 管理

- reaper 5s 周期扫描 `deadline_unix ≤ now()` 的 `state=running` 行，触发 pause 流程（auto-suspend）
- `state=paused` 行不受 reaper 扫描（paused 不自动销毁）
- `POST /sandboxes/{id}/timeout` 更新 `deadline_unix = now() + timeout`；精度 ±5s（下一个 reaper 周期生效）
- TTL 最小值 1s，无上限；`timeout=0` 表示立即暂停（等同 pause）

---

### 9.2 模板构建流水线规格

#### 9.2.1 阶段触发条件

| 阶段 | 触发条件 | 跳过条件 |
|---|---|---|
| **A import** | `fromImage` 非空 XOR `fromTemplate` 非空 | 二者均空 |
| **B steps** | `steps` 列表非空 | 列表为空 |
| **C template** | `startCmd` 非空 | `startCmd` 为空 |

三阶段均跳过（无 fromImage、无 steps、无 startCmd）→ trigger 返回 400（无事可做）

`fromImage` 与 `fromTemplate` 互斥，同时非空 → 400

#### 9.2.2 幂等性

- 同一 `buildID` 重复 trigger：
  - `state=waiting/building` → 返回当前 state，不重新触发
  - `state=ready/error` → 返回终态（幂等，不重试）
- builds 表写入用 CAS（`waiting → building`），并发 trigger 只有一次成功，其余按已有 state 返回
- Referrers 幂等：`fromImage` 若 OCI registry 已有同 digest 展平 Referrers，A 阶段跳过直接复用，content key 不变

#### 9.2.3 构建沙箱隔离规格

- 每次构建独立 `sandbox-builder@<bid>` microVM，与运行沙箱完全隔离
- 构建 sandbox 共用同一节点的 vswitch 端口（attach 一次，各阶段顺序复用）
- 构建完成（ready/error）后 sandbox-builder oneshot 退出，运行目录随即清理
- 宿主不直接执行任何构建命令（RUN / flatten-ctl / envd steps 全在 KVM 隔离内）

#### 9.2.4 产物规格

| 条件 | kind | templateID 格式 | 产物 |
|---|---|---|---|
| 无 `startCmd` | img | `<profile>-img-<64hex>` | image.img（EROFS + overlay）|
| 有 `startCmd` | snp | `<profile>-snp-<64hex>` | 快照 bundle（memory + disk + config）|

- `<64hex>` 为 manifest content key，由内容确定性派生：相同输入跨时间跨节点产生相同 key
- 成功后临时 `transient-<uuidv7>` 被持久 templateID 替换（builds 表 + 返回响应）

#### 9.2.5 构建日志规格

- 每条日志条目：`{timestamp: RFC3339Nano, level: "info"|"warn"|"error"|"debug", message: string}`
- `level` 从 journald `PRIORITY` 映射：≤3 → error，4 → warn，5–6 → info，≥7 → debug
- `logsOffset` 为字节偏移（非行号）；同一 `(bid, offset)` 多次请求返回完全相同内容（无副作用）
- 内核 dmesg（`SYSLOG_IDENTIFIER=console`）不进日志流，仅 `SYSLOG_IDENTIFIER=build` 可见
- 日志不得含 `MANIFEST_KEY` / registry 密码 / presigned URL 完整签名（启动器侧脱敏）

---

### 9.3 数据面代理规格

#### 9.3.1 路由决策

每个入站请求按以下优先级路由（任一步骤失败即停止）：

```
1. 解析 sid：
   - X-Kuasar-Sandbox-Id 头（集群 router 注入）
   - Host: <port>-<sid>.<domain>（e2b 标准格式）
   → 无法解析 → 404

2. 路由表查 sid：
   → 不存在           → 404
   → state=dead       → 404
   → state=paused     → 触发 auto-resume（单飞），等待 park_timeout
                        成功 → 继续步骤 3；超时 → 503
   → state=running    → 继续步骤 3

3. 端口路由（e2b profile）：
   → 目标端口 49983 或 49999 → 转发到 envd_uds（host UDS）
   → 其余端口               → 转发到 floatingip:<port>

4. bare profile：
   → 所有端口               → 转发到 floatingip:<port>
   → 49983 / 49999          → 501（无 envd）
```

#### 9.3.2 鉴权规格（proxy.auth=enforce）

- 鉴权在路由判定之后、连接建立之前执行
- `X-Access-Token` 与 `RouteEntry.access_token` 恒时比较（防 timing 侧信道）
- 不匹配 → 401；头缺失 → 401
- 鉴权发生在路由判定之后（先有 404 才有 401），不泄露 sid 存在性

#### 9.3.3 auto-resume 单飞规格

- 粒度：per-sandboxID（`singleflight.Group`），key=sid
- 并发 wake 同一 paused sid：只有一个 goroutine 执行 StartUnit + 健康等待，其余共享同一 future
- 超时（park_timeout，默认 30s）：所有等待者同时收到 503，路由表条目保持 paused
- 成功：所有等待者同时获得 running 路由，继续转发

#### 9.3.4 external worker 同步规格

- worker 通过 plugin 平面注册 `{subscribe: route_wake}`
- serve 向所有已注册 worker 广播增量 upsert/delete；初始全量 upsert + bookmark 后进入增量模式
- worker 崩溃重启后：重注册 → serve 重放全量（逐条 upsert + bookmark），重同步期间旧路由表继续服务
- serve 侧兜底网关：无 worker 或 worker 均不可用时，serve 自身处理请求（按 FNV(sid) 哈希转 worker 或自处理）

---

### 9.4 集群集成（node-link）规格

#### 9.4.1 节点注册

- `cluster.registry` 非空时，serve 启动后自动拨出，以 `{kind: registry, node_id, labels}` 连接
- registry 反向成为本节点路由权威订阅者（接收本节点全量 + 增量沙箱事件）
- 初始同步：遍历 sqlite sandboxes + builds，逐条 upsert，末尾发 bookmark
- 断线后：指数退避重连（1s → 60s，±20% jitter）；重连后以 `resume_from` 请求增量重报；超出留存窗口则全量重报

#### 9.4.2 命令处理规格

registry 向节点下发的命令经 node-link 通道传递，语义等同对应本地 API：

| 命令 | 语义等同 | 特殊处理 |
|---|---|---|
| `create{sid, spec}` | `POST /sandboxes` | sid 已存在且 running → 幂等返回 |
| `create{sid, spec, migration_token}` | 迁移导入 + restore | 验证 runtime 摘要后一步完成 import + resume |
| `connect{sid, timeout}` | `POST /sandboxes/{sid}/connect` | — |
| `delete{sid}` | `DELETE /sandboxes/{sid}` | — |
| `key_put{fingerprint, key_enc, expires_at}` | 写 manifest_keys 表 | 解密验证完整性后存入 |
| `key_drop{fingerprint}` | 删 manifest_keys 行 | 立即生效，后续鉴权失败 |
| `build_register{bid, spec}` | `POST /v3/templates` + trigger | — |

#### 9.4.3 事件上报规格

节点主动上报以下事件至 registry（经 routesync 引擎，kind=registry）：

| 事件 | 触发时机 | 携带字段 |
|---|---|---|
| sandbox upsert | state 变更（running/paused/dead）| RouteEntry 全量 |
| sandbox delete | kill 完成后 | sid, rev |
| build upsert | state 变更（building/ready/error）| build_id, state, result_manifest_key |
| node heartbeat | 每 30s | allocated, pool, build_alloc, draining, zone |

**节点心跳超时**：registry 90s 未收到心跳 → 标节点 suspect，停止下发新 create 命令；节点重连后恢复

---

### 9.5 密钥管理规格

#### 9.5.1 白名单准入规格

- create / trigger / import 三类操作，执行前必须通过白名单门
- 校验流程（见下）失败时，无论哪步失败均返回 403，不泄露具体原因
- 白名单条目 `expires_at > 0` 且已超期 → 403（与不存在等同对待）
- reaper 定期清理已超期白名单条目（不阻塞请求路径）

#### 9.5.2 api_key 校验流程

```
步骤 1  格式校验：/^e2b_[0-9a-f]{72}$/  →  不符 → 401

步骤 2  指纹预筛：fp = api_key[4:28]（前 24-hex）
        查 manifest_keys WHERE fingerprint=fp
        →  无命中 → 401

步骤 3  解密：AES-GCM 解密 key_enc → manifest_key（明文仅存在于调用栈）

步骤 4  MAC 全校验：
        取 api_key 中 ts[28:36]、nonce[36:44]
        重算 mac = HMAC-SHA256(manifest_key, fp‖ts‖nonce)[:16]
        constant-time 比较 api_key[44:76] == hex(mac)  →  不符 → 401

步骤 5  归属校验（涉及具体资源时）：
        操作目标行 manifest_key_hash == fp  →  不符 → 404（不泄露存在性）
```

#### 9.5.3 密钥轮换规格

- `encryption_key` 支持多键，`:`分隔，index 0 为活动密钥，其余仅解旧
- `keytag` 字段（4B LE）标记加密该记录时用的密钥 index
- 新写入记录：用 index 0 加密，keytag=0
- 读取旧记录：按 keytag 定位密钥解密，透明兼容
- 主动轮换：`node-ctl encryption-key rotate` 扫库批量重加密（后台可并行，原子写单行）

#### 9.5.4 集群密钥分发规格

- registry 通过 `key_put` 命令下发，节点写入 `manifest_keys`，`expires_at = now + 3h`
- registry 每 1h 下发 `key_put` 续租，节点更新 `expires_at`
- registry 撤销通过 `key_drop{fingerprint}`，节点立即删行，生效时间 ≤ 下一次请求
- 节点本地手工添加（独立模式）：`node-ctl manifest-key add <MK> [--ttl=<dur>]`，仅本节点生效

---

### 9.6 资源管理规格

#### 9.6.1 准入仲裁规格

两个独立拒绝条件，任一满足即返回 `AdmitResponse{status: rejected}`：

| 条件 | 拒绝原因 | 说明 |
|---|---|---|
| `node_startup_used + effective_startup > allocatable_pool × startup_fraction` | `startup_pool_exhausted` | 启动预算池耗尽（并发创建流控）|
| `node_allocated + floor > allocatable_pool × high_watermark` | `high_watermark` | 节点已分配量超高水位 |

- `effective_startup = min(floor + 512MiB, capacity × 0.25)`：大沙箱不独占启动预算池
- 拒绝时 node-ctl 向 create API 返回 503（`{"code":503, "message":"node resource exhausted: <reason>"}`）

#### 9.6.2 内存 burst 仲裁规格

- 令牌桶容量 = `allocatable_pool × 20%`；填充速率 = `allocatable_pool × 5% / s`
- `urgency=critical`（OOM 防护）：绕过令牌桶，直接授权至 `capacity`（上限）
- `urgency=normal`：grant = `min(requested, bucket_tokens, allocatable_pool × 0.8 − node_allocated)`
- `cooldown_ms` 返回桶恢复可再次 grant 的预计时间，sandbox-ctl 据此退避重试，不轮询

#### 9.6.3 回收调度规格

每次 Heartbeat 驱动回收计算（settled 阶段沙箱）：

```
target_alloc = settled_rss × 1.3          // 保留 30% burst 余量
if current_allocatable > target_alloc:
    new_allocatable = max(floor, current_allocatable × 0.9)
    → HeartbeatAck(new_allocatable)        // 通知 sandbox-ctl 调低 cgroup memory.high
    node_allocated -= (current_allocatable − new_allocatable)
```

紧急回收（emergency_watermark 触发，可用内存 < 5%）：

- 主动扫全部 settled 沙箱，不等 Heartbeat 周期
- 强制下调所有 settled 沙箱 `new_allocatable = floor`（最小化占用）

#### 9.6.4 控制器重启恢复规格

1. 读 `/run/node-ctl/state.json`（tmpfs，原子 write-fsync-rename，若存在则得 reservation 快照）
2. 扫 `cgroup_scan_paths` 获取活跃 cgroup 列表（真相之源）
3. 交叉对账：
   - cgroup 在 && state.json 有：标"待重连"，等 sandbox-ctl Reattach(token)
   - cgroup 在 && state.json 无：孤儿 cgroup，临时收纳，等 Reattach
   - state.json 有 && cgroup 不在：沙箱已死，丢弃 reservation，不计入 node_allocated
4. IdleSweeper：Reattach 超时 90s 静默的待重连条目视为死亡，释放 reservation

---

### 9.7 e2b 协议兼容性规格

#### 9.7.1 兼容端点

| 端点 | 兼容级别 | 说明 |
|---|---|---|
| `POST /sandboxes` | 完全兼容 | 支持全部 e2b v1 字段 |
| `GET /sandboxes/{id}` | 完全兼容 | state 含 paused（e2b 扩展字段）|
| `GET /v2/sandboxes` | 部分兼容 | 仅返回本节点；分页 `x-next-token` 兼容 |
| `DELETE /sandboxes/{id}` | 完全兼容 | |
| `POST /sandboxes/{id}/pause` | 完全兼容 | |
| `POST /sandboxes/{id}/connect` | 完全兼容 | |
| `POST /sandboxes/{id}/timeout` | 完全兼容 | |
| `POST /v3/templates` | 完全兼容 | |
| `POST /v2/templates/{tid}/builds/{bid}` | 完全兼容 | |
| `GET /templates/{tid}/builds/{bid}/status` | 完全兼容 | `logEntries[]` 为扩展字段 |
| `GET /templates/{tid}/files/{hash}` | 完全兼容 | presigned PUT 协议 |

#### 9.7.2 不支持的操作（返回 501）

- bare profile 沙箱的 49983 / 49999 端口请求（无 envd）
- bare profile 沙箱的 `/init` 请求
- COPY step（未配置 `builder.files_storage`）
- 未来 e2b API 新增端点（默认 404，需显式实现后升级）

#### 9.7.3 响应字段兼容性要求

- `envdVersion`：固定为 `"0.6.1"`，与 SDK 内置版本校验匹配，升级须同步更新 runtime 镜像
- `domain`：格式 `<port>-<sid>.<api.domain>`，e2b SDK 据此构造数据面 URL，不可改变
- `X-Access-Token`：请求头名称不可改（SDK 硬编码）
- 错误响应：`{"code": <int>, "message": <string>}`（e2b 统一格式，不可加包装层）

---

## 10. 设计约束及限制

### 10.1 运行环境约束

| 约束 | 规格要求 | 原因 |
|---|---|---|
| **操作系统** | Linux；内核 ≥ 5.15 | eBPF TC（`BPF_PROG_TYPE_SCHED_CLS`）/ cgroup v2 / io_uring |
| **cgroup 版本** | cgroup v2（unified hierarchy）| cgroup v1 不支持 `memory.high` PSI 反压 |
| **KVM** | `/dev/kvm` 可写，CPU VMX/SVM 开启 | cloud-hypervisor 要求 |
| **systemd** | ≥ v247（`Delegate=yes` + `cgroup v2`）| `--cgroup-adopt` + D-Bus `StartUnit`/`StopUnit` |
| **BPF filesystem** | 已挂载 `/sys/fs/bpf` | vswitch-ctl eBPF map 持久化 |
| **Go 版本** | ≥ 1.22 | `CGO_ENABLED=0`，纯静态二进制 |
| **架构** | x86-64（amd64）| cloud-hypervisor 仅支持 x86-64 |

### 10.2 架构约束（不可逾越）

| 约束 | 描述 |
|---|---|
| **跨仓不 import 内部包** | node-ctl 仅经子进程 CLI + UDS 协议与兄弟组件通信，不 import sandbox-runtime / sandbox-vswitch / sandbox-accelerator 仓内部 package。跨仓接口变化只影响 CLI 参数与 socket 消息格式，不引入编译耦合。|
| **manifest_key 永不明文落盘** | AES-256-GCM 密文存 sqlite；内存中的 plaintext 生命周期限于单次操作调用栈。|
| **MANIFEST_KEY 永不入 guest** | LaunchSpec / BuildSpec 中的 `MANIFEST_KEY` env 由 run-sandbox / run-builder 消费后即刻从进程 env 中清除，不经任何路径传入 microVM（vhost-user-blk、vsock、MMDS 均不携带）。构建阶段入 guest 的只有 `FLATTEN_REGISTRY_*` 凭据。|
| **sqlite 单写者** | 单节点只能运行一个 `node-ctl serve` 实例。不支持多 serve 进程共享同一 sqlite 文件（WAL 写者锁 + pidfile 互斥）。控制面水平扩容只能通过 cluster-ctl 实现（多节点，非单节点多进程）。|
| **构建只在 guest 内执行** | 所有 RUN / flatten-ctl / envd step 在 microVM KVM 隔离内运行，宿主不直接执行构建命令。|
| **e2b api_key 格式不可改** | SDK 侧正则 `/^e2b_[0-9a-f]+$/` 硬编码，node-ctl 密钥派生必须产出此格式，否则 SDK 校验失败。|

### 10.3 接口兼容约束

| 约束 | 描述 |
|---|---|
| **e2b REST 语义兼容** | 所有已支持的 e2b REST 端点响应结构、状态码、字段名须与 e2b 官方 SDK 测试套件兼容；禁止在同名字段返回不兼容类型。|
| **envd 版本锁定** | `sandbox-runtime-e2b.erofs` 内注入的 envd 版本须与 SDK 声明的 `envdVersion`（`"0.6.1"`）匹配；版本升级须同步更新 runtime 镜像。|
| **bash 依赖** | steps 和 startCmd 均经 `/bin/bash -l -c <cmd>` 执行。fromImage / fromTemplate 指向的基础镜像须包含 bash；缺失时构建失败，无降级路径。|
| **config-socket 协议版本** | task / admin / plugin / api 平面之间使用帧化 JSON over h2c；serve 重启后 socket 文件重建，旧 worker 需重连。socket 路径变更须同步更新所有 pidfile / unit 配置。|

### 10.4 功能限制

| 限制 | 影响 | 备注 |
|---|---|---|
| **bare profile 无 envd** | `/init` API、49983/49999 数据面端口均不可用，返回 501 | 设计如此；bare profile 只暴露 floatingip 用户自定义端口 |
| **COPY step 依赖 files_storage** | 未配置 `builder.files_storage` 时 COPY step 返回 501 | 需要 OBS 或兼容 S3 桶；小型部署可跳过 COPY 能力 |
| **snapshot 恢复须共享 manifest store** | 跨机迁移（`checkpoint.mode=remote`）要求目标节点和源节点共享同一 store-ctl / OBS 桶 | 完全隔离的孤岛节点只能独立运行，无法接收跨机迁移快照 |
| **LIST 仅返回本节点** | `GET /v2/sandboxes` 仅返回当前节点上的沙箱；集群全局视图须通过 registry API 聚合 | 独立部署不存在此限制 |
| **整机重启后 running 行丢失** | `/run/sandbox/`（tmpfs）清空，所有 running 行标 dead；paused 行（磁盘快照）保留 | microVM 内存非持久；需 pause 再恢复的设计预期行为 |
| **运行时不支持在线扩缩容** | 无法对 running 沙箱动态调整 CPU/内存配额 | 需 pause → 修改模板配置 → resume 完成规格变更 |
| **构建 fromImage 须经 flatten-ctl 支持的 registry** | 不支持直接引用本地 OCI layout 目录 | flatten-ctl 仅支持远程 registry；本地构建需先 `docker push` 到私有 registry |
| **mmds auto-resume 后 accessToken 变化** | `mmds.enabled=true` 时快照恢复后 mmds token 重新派生；SDK 须重取 `accessToken` | 可预期行为；mmds.enabled=false 时 token 随快照持久化，无此问题 |
| **proxy auth=permissive 无数据面鉴权** | 数据面请求无需 `X-Access-Token`，任意请求可访问任意 sid | 仅适合单租户内网；生产必须 `enforce` |
| **external proxy 重同步期间路由短暂不一致** | worker 重连后初始全量 upsert 期间，新 sid 可能未在路由表，请求返回 404 | 窗口时间取决于节点沙箱数量；留存窗口内 `resume_from` 可减少重同步量 |
| **构建并发上限** | `builder.max_concurrent`（默认 2）限制单节点并发构建数；超额请求需排队 | 构建机资源（CPU / 网络 / vswitch slot）有限；不推荐单节点大量并发构建 |
| **node-link 降级** | `cluster.tls` 为空时 node-link 使用 plain h2c；manifest_key 在传输中无额外加密 | 生产集群必须配置 mTLS；plain 模式仅适合同主机 UDS 或实验环境 |

### 10.5 可观测性范围约束

| 约束 | 描述 |
|---|---|
| **资源仲裁审计日志不落磁盘** | `resource_listen.logging.audit_path` 强制写 tmpfs；目的是避免高频写影响节点 I/O，但重启后丢失 |
| **构建日志不含平台密钥** | journald 日志流（SDK `on_build_logs`）严格过滤：不得输出 MANIFEST_KEY / registry 密码 / presigned URL 完整签名；违规须在启动器侧脱敏 |
| **guest 内核 dmesg 不进 SDK 日志流** | `console=journald=console` 日志流与 `SYSLOG_IDENTIFIER=build` 分离；SDK 仅可见 build identifier 日志，内核 dmesg 不可见 |
| **manifest-key list 只显示指纹** | `node-ctl manifest-key list` 输出 24-hex 指纹，不回显 key 明文；管理员无法通过 CLI 导出 key 原文 |
| **Prometheus 指标不含 sandboxID** | 数据面指标按 `{result, profile}` 聚合；`sandboxID` 仅出现在 trace / log 中，不进 metric label（防高基数）|

### 10.6 已知技术风险与缓解

| 风险 | 严重度 | 缓解措施 |
|---|---|---|
| **sqlite 文件损坏** | 高（控制面不可用）| 定期备份（推荐 `sqlite3 .backup`）；paused 沙箱快照仍在磁盘，可手工重建 paused 行 |
| **vswitch eBPF 程序加载失败** | 高（所有新建沙箱无网络）| 节点启动健康检查；bpffs 持久化保证重启不重加载，仅首次或升级需重加载 |
| **encryption_key 丢失** | 极高（所有 manifest_key_enc 不可解密）| 密钥须存入密钥管理系统（KMS）并备份；配置文件不含主密钥时通过 `NODE_CTL_ENCRYPTION_KEY` env 注入 |
| **resource_listen 持续不可用** | 中（新沙箱不受资源管控，无法超分保护）| 资源控制协议为可选（`control_socket` 为空则静态 cgroup）；serve 与 resource_listen 一体时不存在 |
| **node-link 长时间断连** | 中（节点对 registry 不可见，不接受新 create 命令）| 节点本地运行不受影响；断线期间本地 TTL reaper 继续工作；重连后增量重报 |
| **构建磁盘写满** | 中（builder 写层无限增长）| `sandbox.data_root` 磁盘使用率监控；`builder.max_disk_gb` 限制单次构建上限；构建 sandbox oneshot 完成即清理 |

---

## 附录 A：关键配置项速查

| 字段 | 默认 | 必填 |
|---|---|---|
| `api.domain` | — | **必填** |
| `encryption_key` | — | **必填**（或 `NODE_CTL_ENCRYPTION_KEY` env）|
| `proxy.mode` | `internal` | 否 |
| `proxy.auth` | `enforce` | 否（mmds.enabled=false 时强制）|
| `sandbox.resources.control_socket` | 空（静态 cgroup）| 否 |
| `cluster.registry` | 空（独立模式）| 集群必填 |
| `cluster.node_id` | — | 集群必填 |
| `cluster.tls` | 空 | 生产必填 |
| `mmds.enabled` | `false` | 否 |
| `builder.max_concurrent` | `2` | 否 |
| `builder.files_storage` | 空（COPY 回 501）| 有 COPY step 时必填 |
| `checkpoint.mode` | `local` | 否（跨机迁移需 `remote`）|

---

## 附录 B：沙箱状态机

```
            create (img=冷启 / snp=快照恢复)
                 │
                 ▼              TTL 到期 / pause API
   ┌──────► running ────────────────────────────────► paused ──────────────────┐
   │             │                                    │（本机快照，snap_loc=local）│ 深空闲/显式
   │             │ kill                                │ PAUSED→SAVED              │
   │             ▼                                    ▼（远程快照，snap_loc=remote）│
   │          (行删除)                               saved                       │
   │                                                 │（不绑节点，migration_token）│
   └──────◄── connect(resume) / data-plane wake ───◄──┘                         │
                                                    ◄──── 迁移 import + restore ──┘
```

**节点内 sqlite state**：`running` | `paused` | `dead`  
**集群 registry state**：`NONE` | `RESERVED` | `READY` | `PAUSED` | `SAVED`

---

## 附录 C：上下文组件职责边界

| 组件 | 仓库 | node-ctl 经何驱动 | 反向接口 |
|---|---|---|---|
| `sandbox-ctl` | sandbox-runtime | systemd 单元 execve（run-sandbox）| ctl.sock（snapshot 客户端）；pkg/resource（Admit/…）|
| `cloud-hypervisor` | sandbox-deps | sandbox-ctl 子进程 | CH API UDS（ch.sock）；vhost-user-blk socket |
| `sandbox-init` | sandbox-runtime | guest PID=1（由 CH 启动）| vsock 5000（launch 握手；envd 控制）|
| `envd` | sandbox-deps（注入 runtime-e2b）| `/health` 轮询 + `/init` 配置 | host-UDS（proxy 数据面透传）|
| `vswitch-ctl` | sandbox-vswitch | CLI 子进程（attach/detach）| tapfd 交接给 sandbox-ctl（tap fd）|
| `flatten-ctl` | sandbox-accelerator | guest 内 sandbox-ctl exec 驱动 | exec stdio（tarstream 工件流）|
| `manifest-ctl` | sandbox-accelerator | CLI 子进程（store image.img）| stdout manifest key |
| `cache-ctl` | sandbox-accelerator | 由 sandbox-ctl 直接拨号（wire 协议）| L1/L2/L3 chunk 数据 |
| `store-ctl` | sandbox-accelerator | 由 cache-ctl / manifest-ctl 拨号（gRPC）| OBS/fs 后端持久化 |
| `cluster-ctl registry` | sandbox-orchestrator | node-link 拨出 → 反向注册 | create/connect/delete/key_put/key_drop/build_register 命令 |

---

## 附录 D：数据结构与格式规格

### D.1 标识符格式

| 标识符 | 格式 | 约束 |
|---|---|---|
| **manifest_key** | 64-hex（32 字节随机）| 大小写不敏感；存库统一小写 |
| **api_key** | `e2b_` + 72-hex（76 字符）| SDK 正则 `/^e2b_[0-9a-f]+$/`；fp=前24hex，ts=24–31hex，nonce=32–39hex，mac=40–71hex |
| **api_key 指纹（fp）** | `SHA256(manifest_key)[:12]` = 24-hex | 库内预筛 `manifest_keys.fingerprint` 索引 |
| **templateID（持久）** | `<profile>-<kind>-<key>` | profile ∈ {`e2b`,`bare`}；kind ∈ {`img`,`snp`}；key = 64-hex manifest content key |
| **templateID（临时）** | `transient-<uuidv7>` | register→complete 期间有效；complete 后废弃 |
| **sandboxID / buildID** | UUIDv4（36 字符，含连字符）| 全节点唯一；集群下由 scaler 保证全局唯一 |
| **manifest content key** | 64-hex | `SHA256(encrypted_chunk_stream)`；相同内容跨时间相同 |
| **mmds session token** | 24-hex | `HMAC-SHA256(manifest_key, "kuasar-mmds-v1:"+sid)[:12]`；确定性派生 |
| **envd access token** | `<sid>.<HMAC-suffix>` | 随 create/connect 随机派生；proxy 常数时间比较 |
| **migration token** | base64(JSON) | 含 sid / snap_ref / envd_token / runtime_digest 等；按沙箱级敏感处理 |

### D.2 行为时序常量

| 行为 | 值 |
|---|---|
| reaper 周期 | 5s（deadline_unix 精度 ±5s）|
| auto-resume 最大等待 | `park_timeout`（默认 30s，routesync hello 策略下发）|
| envd /health 轮询间隔 / 超时 | 200ms / 60s |
| Heartbeat 周期 / 静默超时 | 30s / 90s（超时 IdleSweeper 回收）|
| routesync 重连退避 | 200ms → 5s（指数）|
| node-link 重连退避 | 1s → 60s（指数，±20% jitter）|
| routesync 留存窗口 | `routesync.retention_window`（建议 ≥ 5min；超出则全量重同步）|
| routesync 单帧上限 | 1 MiB |
| manifest_key TTL（集群） | 3h；每 1h 续租 |
| build 单步超时 / 总超时 | `builder.step_timeout`（默认 10min）/ `builder.build_timeout`（默认 60min）|
| COPY presigned URL 有效期 | `builder.files_presign_ttl`（默认 1h）|

### D.3 资源预算常量

| 参数 | 默认值 | 含义 |
|---|---|---|
| `high_watermark` | 85% | 超过 allocatable_pool×85% 拒绝新 Admit |
| `low_watermark` | 70% | 低于此恢复 burst grant |
| `emergency_watermark` | 5%（可用余量）| 触发紧急主动 reclaim |
| `startup_fraction` | 50% | startup_pool 最大消耗比例 |
| `burst_headroom_ratio` | 30% | settled 沙箱预留 burst 余量 |
| `reclaim_step` | 0.9 | 每次 reclaim 步长（乘因子）|
| `memory_grant_per_sec_factor` | 5% | grant 令牌桶填充速率 |

**预算计算公式**：

```
physical_total  = /proc/meminfo MemTotal
host_reserved   = max(2GiB, physical_total × 0.05)
allocatable_pool = physical_total − host_reserved − node_budget × 0.10

effective_startup = min(floor + 512MiB, capacity × 0.25)
```

### D.4 sqlite 数据库结构

文件 `<data_dir>/node-ctl.db`（0600），WAL 模式，`synchronous=NORMAL`。

**sandboxes 表**：

| 列 | 类型 | 说明 |
|---|---|---|
| `id` | TEXT PK | sandboxID |
| `template_id` | TEXT | 持久或临时 templateID |
| `state` | TEXT | `running` \| `paused` \| `dead` |
| `deadline_unix` | INTEGER | Unix 秒，reaper 检测 |
| `manifest_key_hash` | TEXT | 24-hex 指纹 |
| `manifest_key_enc` | BLOB | AES-256-GCM 密文 |
| `snapshot_ref` | TEXT | `local:<path>` \| `manifest://<64hex>` \| 空 |
| `envd_access_token` | TEXT | per-sandbox 随机 token |
| `floatingip` | TEXT | vswitch 浮动 IP |
| `vswitch_port` | INTEGER | detach 用 |
| `profile` | TEXT | `e2b` \| `bare` |
| `metadata` | TEXT | JSON |
| `created_at` / `updated_at` | INTEGER | Unix 秒 |

**builds 表**：

| 列 | 类型 | 说明 |
|---|---|---|
| `id` | TEXT PK | buildID |
| `template_id` | TEXT | 关联 templateID |
| `state` | TEXT | `waiting` \| `building` \| `ready` \| `error` |
| `manifest_key_hash` | TEXT | 24-hex 指纹 |
| `manifest_key_enc` | BLOB | AES-256-GCM 密文 |
| `registry_auth_enc` | BLOB | 拉取凭据密文（nullable）|
| `spec` | TEXT | BuildSpec JSON |
| `result_manifest_key` | TEXT | 成功后 64-hex content key（nullable）|
| `result_kind` | TEXT | `img` \| `snp`（nullable）|
| `reason` | TEXT | 失败原因（nullable）|
| `created_at` / `updated_at` | INTEGER | Unix 秒 |

**manifest_keys 表**：

| 列 | 类型 | 说明 |
|---|---|---|
| `id` | INTEGER PK AUTOINCREMENT | — |
| `fingerprint` | TEXT UNIQUE | 24-hex，预筛索引 |
| `key_enc` | BLOB | AES-256-GCM 密文 |
| `label` | TEXT | 运维标注（nullable）|
| `expires_at` | INTEGER | Unix 秒（0=永不过期）|
| `created_at` | INTEGER | — |

**AES-GCM 密文格式**：`keytag(4B LE uint32) ‖ nonce(12B random) ‖ ciphertext+tag`

### D.5 RouteEntry 结构

```
RouteEntry {
  sid          string   // sandboxID，主键
  profile      string   // "e2b" | "bare"
  template_id  string   // 持久 templateID
  state        string   // "running" | "paused" | "dead"
  envd_uds     string   // host UDS 路径（e2b profile running 时非空）
  ci_uds       string   // ci socket UDS 路径
  floatingip   string   // vswitch 浮动 IP
  access_token string   // proxy auth token
  snap_loc     string   // "" | "local" | "remote"
  mmds_secret  string   // 24-hex（mmds.enabled=true 时非空）
  rev          int64    // 单调递增，留存窗口内唯一
}
```

state 变更（running↔paused、任意→dead）均触发 upsert/delete 广播。

### D.6 LaunchSpec / BuildSpec 结构

**LaunchSpec**（task 平面，run-sandbox 一次性消费）：

```yaml
exec: sandbox-ctl
args:
  - run
  - --sandbox-id=<sid>
  - --config=<rundir>/<sid>.yaml
  - --manifest-config=<shared-manifest.yaml>
  - --run-root=/run/sandbox
  - --cgroup-adopt
  - --connect=<host-uds>:<inner-ip>:<port>
  - --stdout-to=journald=sandbox
  - --stderr-to=journald=sandbox
  - --console=journald=console
env:
  MANIFEST_KEY: <64-hex>        # 不落任何文件，仅 env 帧传递
ttl_seconds: <int>
```

**BuildSpec**（task 平面，run-builder 一次性消费）：

```yaml
sandbox_id: <sid>
build_id: <bid>
template_id: <tid>
profile: e2b | bare
from_image: <ref>               # 可空（无 A 阶段）
from_template: <tid>            # 与 from_image 互斥
steps:                          # 可空（无 B 阶段）
  - op: RUN | ENV | ARG | WORKDIR | USER | COPY
    value: <string>
start_cmd: <string>             # 可空（无 C 阶段）
ready_cmd: <string>
resources:
  cpu_count: <int>
  memory_mb: <int>
env:
  MANIFEST_KEY: <64-hex>
  FLATTEN_REGISTRY_USERNAME: <string>
  FLATTEN_REGISTRY_PASSWORD: <string>
  FLATTEN_REGISTRY_TOKEN: <string>
result_path: <abs-path>         # run-builder 写 JSON 结果文件
```

---

*本文档综合自 sandbox-orchestrator、sandbox-runtime、sandbox-vswitch、sandbox-accelerator、kuasar-sandbox 五大子系统共 13 份设计文档，以 node-ctl 为视角提炼特性规格。与各原始文档有出入时，以原始文档为准。*
