# 竞品分析：AI Sandbox 网络与路由层

> 基于 `sandbox-vswitch`（eBPF/TC 虚拟交换机）和 `node-proxy`（L7 路由转发层）的设计，
> 横向对比主要竞品的网络架构与路由方案。
>
> 分析对象：agent-substrate、AWS Lambda、e2b、AWS AgentCore、Azure AI Foundry Agent、Modal

---

## 一、本系统核心能力摘要

### 1.1 vswitch 层（L2/L3 数据面）

- **纯内核 eBPF/TC 转发**：配置完成后用户态进程退出，TC ingress BPF 程序持续在内核执行；控制进程崩溃/重启数据面不中断（BPF 程序与 maps 经 bpffs pin 持久化）
- **算术槽位寻址**：floating IP = base + slot\_id，GENEVE UDP dst = base + slot\_id，O(1) 查找，无 hash map 跳转
- **结构性沙箱间隔离**：eBPF 源码无 port→port 转发分支，ARP 全代答（广播不出端口），源 MAC 出口强制改写——可静态审计
- **GENEVE 隧道**：每沙箱独享一个 UDP 端口，外层 ECMP/RSS 无需改动；支持 IP-over-GENEVE（省 14 B/包）与 Ether-over-GENEVE 两种封装
- **管理平面提取**：`--mgmt-extract` 把 MMDS/元数据 VIP 段从沙箱 GENEVE 流量中剥离，TC SNAT/DNAT 透明改写，沙箱按 VIP 访问，管理服务无需感知 inner IP；`--mgmt-service` 叠加无状态端口 NAT，无连接跟踪
- **资源规格**：4096 slot/节点（编译期常量），每 slot 112 B，maps ≈ 448 KB；实测 3000+ 沙箱/节点，单沙箱额外内存开销 <60 MiB

### 1.2 node-proxy 层（L7 路由 + 鉴权）

- **routesync 协议**：帧化 JSON over h2c（无 gRPC/protobuf），Worker 主动拨 serve 注册并持挂连接作为路由流，serve 从不主动拨 Worker——控制面字节流不经 serve
- **增量路由分发**：逐条 Upsert + Bookmark（无全量巨帧），断点续传（rev 单调版本号），重同步全程旧路由表仍在服务，无路由空窗
- **park/wake**：对 missing/paused 沙箱的请求先上行 `Wake`（同 sid 去重），server-side 挂起等 `Upsert(running)` 解挂，对 SDK 完全透明
- **确定性 MMDS 密钥**：`HMAC-SHA256(manifest_key, "kuasar-mmds-v1:"+sid)`，N 个对等 Worker 与节点内故障转移天然一致，无主从/共享内存
- **SO_REUSEPORT 水平扩展**：N 个 `node-ctl proxy` Worker 共享数据口，内核分发连接，无锁无主从
- **兜底网关**：数据面请求误达 serve 监听口时，按 sid FNV 哈希选注册 Worker 经其 UDS 反代，CONNECT 经链式 relay 转发

---

## 二、竞品逐项分析

---

### 2.1 e2b

**定位**：开源 AI 代码沙箱平台，亦是本系统的兼容目标（本系统保持 e2b API/SDK 完整兼容）。

**网络架构**：
- 早期为标准 Linux bridge + veth + iptables/nftables；ARP/广播在沙箱间默认穿越
- 网络规则散落在 nftables 链中，密度膨胀后 O(沙箱数) 规则膨胀，难以静态审计
- 无独立 eBPF 数据面，控制进程崩溃会影响网络行为

**L7 路由**：
- `<port>-<sid>.<domain>` Host 模式路由（本系统沿用此约定）
- 早期单节点内嵌 proxy；集群化后引入分发层，但路由分发协议相对简单
- 无 server-side park/wake（沙箱 resume 需客户端重试）

**与本系统对比**：

| 维度 | e2b | 本系统 |
|---|---|---|
| 数据面内核化 | bridge + iptables | eBPF/TC，控制面崩溃不断流 |
| 沙箱间隔离审计性 | 规则散落，难静态审计 | 单一 eBPF 程序，无 port→port 分支 |
| 路由分发 | 简单（早期同进程）| routesync h2c，断点续传，内存有界 |
| 请求挂起 | 客户端重试 | server-side park/wake，透明对 SDK |
| 多 Worker MMDS 一致性 | 无 | 确定性密钥派生，N Worker 天然一致 |
| e2b SDK 兼容 | 原生 | 完整兼容 |

**结论**：本系统是 e2b 在网络基础设施层的生产化升级，在隔离强度、数据面稳定性和路由分发成熟度上有系统性提升。

---

### 2.2 AWS Lambda

**定位**：函数级无服务器计算，调用模型（invocation-based），非会话持久沙箱。

**网络架构**：
- 底层使用 Firecracker microVM（AWS 开源），但网络层由 **Nitro 卡（专用 I/O 硬件）** 提供，在物理层保证沙箱间隔离和带宽隔离
- 每个执行环境获得一个虚拟网络接口，VPC 集成通过 Nitro 硬件卸载完成，无宿主机软件数据面
- 不存在持久的 L7 路由 proxy：Lambda 通过调用 API（`InvokeFunction`）触发，每次调用是独立 HTTP 请求，由 Lambda 前端负载均衡分发，不需要 `(sid, port)` 路由

**关键差异**：
- Lambda 是**无状态调用模型**（函数执行完毕即结束），没有沙箱 paused/running 状态机，不需要 park/wake
- VPC 集成是将客户私有网络打通到执行环境，而非沙箱自身的 L2 网络隔离层
- SnapStart：Java Lambda 支持快照恢复，但仅用于冷启动优化，不是运行时 pause/resume
- 规模：全球数百万并发执行环境，但每个执行环境是短暂的（秒级），不承载持久会话

**网络层能力对比**：

| | Lambda | 本系统 |
|---|---|---|
| 隔离层 | Nitro 硬件卸载 | eBPF/TC 软件（内核） |
| L7 路由 | 无（调用 API 直接触发）| routesync + node-proxy，`(sid, port)` 路由 |
| 沙箱密度 | 数百万（全球托管，不限节点）| 3000+/节点 |
| 持久连接/状态 | 无（无状态调用）| 有（sid 会话模型，pause/resume） |
| pause/resume | 无 | 有（park + MMDS 快照） |
| 用户 VPC 集成 | 完整（ENI 注入）| 无（沙箱网络内部封闭） |
| 快照扇出 | 仅 SnapStart（冷启动）| 运行时 snp 模板 create |

**结论**：Lambda 的网络优势来自 Nitro 专用硬件，规模不可比；但无状态调用模型无法支撑 AI Agent 的**持久会话、实时双向流、pause/resume 快照扇出**等需求，属于不同赛道。

---

### 2.3 AWS AgentCore（Amazon Bedrock AgentCore）

**定位**：AWS 2025 年推出的托管 AI Agent 运行时基础设施，提供 Agent 执行环境、内存、工具调用、知识库等高层能力。

**网络架构**（根据公开信息推断）：
- 底层推测为 **Fargate（Firecracker microVM）或 Lambda** 执行环境
- 网络层复用 AWS VPC/ENI 标准基础设施，无公开的自定义 eBPF 数据面
- L7 路由由 AWS ALB/API Gateway 承载，不暴露 sid 级路由语义
- 安全隔离主要由 IAM + SCP 实现，而非内核数据面定制

**关键差异**：
- AgentCore 是**更高抽象层**（内存库、知识库、工具注册、多 Agent 协作），网络层对用户完全透明，无法定制
- Agent 会话状态由 DynamoDB/S3 持久化，而非 microVM 内存快照，无法做到本系统的**快照扇出**
- 网络延迟受 AWS 区域化部署约束（冷启动通常 >500ms），无本系统 P50 <80ms 的快照恢复性能
- 无 park/wake 语义，Agent 排队在应用层；无 MMDS 模型

**结论**：AgentCore 在 Agent 能力集（工具、记忆、编排）上更丰富，但在**执行基础设施层**（冷启动速度、快照扇出、数据面内核化、持久 TCP 连接路由）上不可比，属于 PaaS vs 基础设施层的不同层次竞争。

---

### 2.4 Azure AI Foundry Agent Service

**定位**：Azure 托管 AI Agent 平台（基于 Azure Container Apps / AKS），提供 Thread 执行模型和工具编排。

**网络架构**：
- 底层为**容器**（非 microVM），使用 **Azure CNI** 提供 Pod 级网络
- 网络隔离基于 Kubernetes NetworkPolicy + Azure NSG，属于 L3/L4 防火墙规则模型
- L7 路由由 Azure API Management / Nginx Ingress 承载
- 无 eBPF 数据面，无 GENEVE 隧道，无 ARP 代答机制；规模扩大时 iptables 规则膨胀与早期 e2b 相同

**关键差异**：
- 容器隔离 vs microVM 隔离：容器共享宿主内核，microVM 有独立内核，安全边界更强
- 无快照/fork 能力（容器不支持 microVM 级内存快照，CRIU 成熟度有限）
- 无 park/wake：Agent 执行是 Thread-based，排队在应用层
- 冷启动：容器拉起通常 1-5s，远慢于本系统 P99 <500ms

**结论**：Azure Foundry 在 Agent 编排（Assistants API 兼容、Thread 管理）上成熟，但网络基础设施层停留在 K8s 标准栈，无法支持本系统的高密度 microVM 场景（3000+ 沙箱/节点，<60 MiB/沙箱）。

---

### 2.5 Modal

**定位**：ML/AI 工作负载云平台，主打 GPU 任务、快速冷启动、函数级弹性扩缩。

**网络架构**：
- 底层使用 **gVisor（runsc）** 容器沙箱，而非 microVM
- gVisor 自带用户态网络栈（netstack），通过宿主机 veth/tun 接入；性能低于 eBPF/TC 内核路径（每个网络包额外经过用户态处理）
- 每个任务获得独立 IP，通过 Modal 的 overlay 网络（推测基于 WireGuard 或 VXLAN）路由

**L7 路由**：
- Modal Web Endpoints：通过 Modal 前端 HTTP 网关路由到特定 Container，无 sid 级别的 `(sid, port)` 路由
- 无 server-side park/wake（任务完成后容器退出，新请求冷启动新容器）
- 无 MMDS / envd 模型

**能力对比**：

| | Modal | 本系统 |
|---|---|---|
| 隔离技术 | gVisor（用户态内核，syscall 过滤）| Firecracker microVM（硬件虚拟化）|
| 网络数据面 | gVisor netstack（用户态）| eBPF/TC（内核，零拷贝）|
| GPU 支持 | 核心能力，多种 GPU 型号 | 无提及 |
| 快照/fork | Container snapshot（成熟度有限）| microVM 内存快照 + snp 扇出 |
| 持久会话 | 有限（Web Endpoint 持久）| 完整（pause/resume + park/wake）|
| 沙箱密度 | 不公开 | 3000+/节点 |

**结论**：Modal 在 GPU 工作负载和 ML 模型推理上有优势；但 gVisor 用户态网络栈在高密度场景下性能劣于 eBPF/TC，且缺乏本系统的快照扇出能力（RL 训练大量并发子沙箱的前提）。

---

### 2.6 agent-substrate（Google，CNCF Sandbox 候选）

**仓库**：https://github.com/agent-substrate/substrate  
**状态**：VERY early development，API 随时变更，591 stars，106 forks  
**社区**：CNCF Slack + ate-dev Google Group，每周四 10-11am PST 例会

**定位**：Kubernetes-native AI Agent 工作负载编排系统，解决 K8s 原生调度对 AI Agent 场景不适配的问题（K8s 设计为数万个长运行 Pod，而 Agent 以空闲为主且需要毫秒级唤醒）。

#### 2.6.1 核心架构

**Actor-Worker 多路复用模型**：把大量 Actor（独立 Agent 会话）复用到少量 Worker（K8s Pod）上，利用 Agent 大多数时间空闲的特性实现超额订阅（Oversubscription）。Demo 展示约 **30x+ 超额订阅**：~250 个有状态 Actor 会话复用在 8 个物理 Pod 上。

**组件**：

| 组件 | 功能 |
|---|---|
| `ate-api-server` | 中心控制面，管理 Actor→Worker 映射关系，绕过 K8s API Server 以降低延迟 |
| `atelet` | 节点级 DaemonSet，管理 Worker Pod 生命周期、协调快照、管理状态迁移 |
| `atecontroller` | K8s controller，协调 WorkerPool 和 ActorTemplate CRD |
| `atenet` | 网络层：DNS + Envoy 路由 + proxy sidecar，流量拦截并触发 Actor 唤醒 |
| `ateom-gvisor` | Pod helper，在 gVisor sandbox 内执行 checkpoint/restore 命令 |
| `podcertcontroller` | Pod 证书签发 |

**高频 Actor 状态**：Actor→Worker 映射存储在 **Redis** 中（高频读写），K8s CRD（WorkerPool、ActorTemplate）仅存配置。

**北极星目标**：
- 激活延迟：P95 < 100ms（wakeup event → ready state）
- 集群规模：10 亿 Actor 总量
- 唤醒吞吐：1000 wakeup events/s

#### 2.6.2 网络层（atenet）

atenet 是 agent-substrate 的网络组件，架构为：**Envoy 代理 + DNS 服务发现 + proxy sidecar**。

**流量路由机制**：
- 每个 Worker Pod 注入 proxy sidecar
- 入站流量经 Envoy 拦截 → 路由判定 → 如目标 Actor 已挂起则触发唤醒（类似本系统的 park/wake）
- DNS 用于 Actor 服务发现（拉取模型），而非本系统的路由推送（push 模型）

**已知技术限制（来自 GitHub Issues）**：
- **Issue #254**：atenet 暂不支持 gRPC（仅 HTTP）
- **Issue #246**：数据面硬编码 IPv4，缺乏 IPv6 支持
- **Issue #265**：Actor 暂不支持监听多端口

**隔离模型**：
- 使用 **gVisor（runsc）**，syscall 过滤隔离；非硬件虚拟化
- 正在探索 Kata Containers 支持（更强隔离边界）
- 快照/检查点：依赖 gVisor 的 CRIU 实现（checkpoint/restore），非 microVM 内存快照

#### 2.6.3 与本系统核心差异

**相同问题，不同抽象层**：两者都解决同一个基本问题——把大量 AI Agent 会话复用到少量物理资源上，并支持 suspend/resume。但实现路径截然不同：

| 维度 | agent-substrate | 本系统 |
|---|---|---|
| **编排层** | Kubernetes-native（CRD，DaemonSet）| 自研编排（sandbox-orchestrator）|
| **隔离技术** | gVisor（用户态 syscall 过滤）| Firecracker microVM（硬件虚拟化）|
| **隔离强度** | 中（共享宿主内核，syscall 过滤）| 高（独立内核，IOMMU 隔离）|
| **网络数据面** | Envoy sidecar + DNS（用户态，L7）| eBPF/TC（内核，L2/L3，零拷贝）|
| **网络架构** | K8s 标准 CNI + Envoy overlay | 自研 eBPF vswitch + GENEVE 隧道 |
| **路由分发** | DNS pull + Redis 状态 | routesync push（帧化 JSON h2c）|
| **快照机制** | gVisor CRIU（RAM + 文件系统）| microVM 内存快照 + snp 扇出 |
| **超额订阅比** | ~30x（demo）| 未单独量化（3000 沙箱/节点）|
| **激活延迟目标** | P95 <100ms（北极星）| P50 <80ms（已实现，快照恢复）|
| **生产就绪度** | VERY early，API 不稳定 | 生产级，e2b SDK 兼容 |
| **单节点密度上限** | 受 K8s Pod 调度约束 | 4096 slot（软件上限，可调）|
| **ARP 隔离** | K8s 网络策略（L3/L4）| eBPF ARP 代答（L2，结构性）|
| **GPU 支持** | 无提及 | 无提及 |
| **状态存储** | Redis + K8s etcd | 内存路由表（bpffs 持久化）|

**网络层差异深析**：

*路由推送 vs 拉取*：本系统的 routesync 是 **push 模型**——serve 主动向所有注册 Worker 广播路由变更（Upsert/Delete），Worker 持本地缓存，单请求无需查询中心节点。agent-substrate 的 atenet 是 **pull 模型**——流量到达时经 DNS 解析 + Redis 查询确定目标 Actor 所在 Worker，存在额外查询 RTT。

*数据面性能*：本系统的 eBPF/TC 数据面在内核完成所有 L2/L3 转发（零拷贝、零上下文切换），实测 TCP 吞吐 ~55 Gbps（GENEVE 路径），单 eBPF 程序执行 ~110 ns/次。agent-substrate 的 Envoy sidecar 属于用户态 L7 代理，每个连接额外引入一次用户态↔内核态切换。

*park/wake 一致性*：本系统的 park/wake 在 Worker 内实现，MMDS 密钥确定性派生保证任意 Worker 可服务任意挂起请求。agent-substrate 的 Actor 唤醒由 atenet Envoy 拦截触发，Actor→Worker 映射依赖 Redis 一致性。

*快照扇出差异*：本系统的 snp 模板 create 可在 60 秒内创建 30000 个子沙箱（RL 训练场景），每个子沙箱有独立 FloatingIP、独立路由条目通过 routesync 广播、独立 MMDS 密钥。agent-substrate 的 CRIU checkpoint/restore 是单 Actor 操作，暂无批量扇出机制。

**结论**：agent-substrate 和本系统是目前最接近的架构竞品——都明确针对 AI Agent 工作负载的 suspend/resume 复用场景，且都在 K8s/编排层之上构建了自己的路由/状态管理层。关键分歧在于：
1. **隔离边界**：gVisor（syscall 过滤）vs microVM（硬件虚拟化），后者安全边界更强，适合多租户代码执行
2. **网络性能**：Envoy sidecar（用户态 L7）vs eBPF/TC（内核 L2/L3），后者在高密度下吞吐和延迟更优
3. **生产就绪度**：agent-substrate 明确标注 VERY early，本系统已具备生产级 SLA
4. **集群扩展策略**：agent-substrate 依托 K8s 生态（etcd、CRD、DaemonSet），扩展路径更标准；本系统有自研 sandbox-orchestrator 集群层（cluster-ctl），单机密度更高但 K8s 集成需额外工作

---

## 三、横向对比矩阵

| 维度 | 本系统 | e2b | Lambda | AgentCore | Azure Foundry | Modal | agent-substrate |
|---|---|---|---|---|---|---|---|
| **隔离技术** | Firecracker microVM | Firecracker microVM | Firecracker microVM | Fargate/Lambda | 容器 | gVisor 容器 | gVisor（+Kata 规划）|
| **网络数据面** | eBPF/TC，纯内核 | bridge+iptables | Nitro 硬件卸载 | AWS VPC/ENI | Azure CNI | gVisor netstack | Envoy sidecar+CNI |
| **L7 路由协议** | routesync h2c（自研）| 简单反代 | 无（调用模型）| AWS ALB | K8s Ingress | Modal 网关 | Envoy+DNS+Redis |
| **路由分发模型** | push（增量帧化 JSON，断点续传）| 无/简单 | N/A | AWS 托管 | K8s 控制面 | N/A | pull（DNS+Redis）|
| **server-side park/wake** | ✓ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓（Envoy 拦截触发）|
| **快照扇出（fork）** | ✓（snp 模板，30K/60s）| 有限 | SnapStart 仅冷启动 | ✗ | ✗ | ✗（CRIU 不成熟）| ✗（无批量扇出）|
| **pause/resume** | ✓ | ✓ | ✗ | ✗ | ✗ | ✗ | ✓（CRIU checkpoint）|
| **超额订阅** | ~50x（3000 slot/60 MiB）| 不公开 | N/A | 托管 | 托管 | 不公开 | ~30x（demo）|
| **冷启动（P99）** | <500ms | ~500ms-1s | ~100-300ms+初始化 | >500ms | 1-5s | ~150ms（声称）| 目标 P95 <100ms（未达）|
| **快照恢复（P50）** | <80ms | 不公开 | SnapStart ~1s | N/A | N/A | N/A | 不公开 |
| **数据面崩溃安全** | ✓（bpffs 持久化）| ✗ | N/A（硬件）| N/A | N/A | N/A | 部分（K8s 自愈）|
| **隔离可审计性** | ✓（静态 eBPF 程序，无 port→port）| ✗（规则散落）| 不可见 | 不可见 | ✗ | 部分（syscall 策略）| 部分（syscall 策略）|
| **GPU 支持** | ✗ | ✗ | 有限（Lambda GPU）| ✓ | ✓ | ✓（核心）| ✗ |
| **e2b SDK 兼容** | ✓（完整）| 原生 | ✗ | ✗ | ✗ | ✗ | ✗ |
| **多 Worker 一致性** | 确定性密钥派生 | N/A | N/A | AWS 托管 | K8s | N/A | Redis + CRIU |
| **ARP 隔离层** | L2（eBPF 代答，结构性）| L3（iptables）| 硬件 | 不可见 | L3（NetworkPolicy）| gVisor netstack | L3（NetworkPolicy）|
| **K8s 集成** | 自研（sandbox-orchestrator）| 自研 | 无 | 无 | 原生 | 无 | 原生（核心）|
| **生产就绪度** | 生产级 | 生产级 | 生产级 | GA | GA | GA | VERY early |

---

## 四、本系统核心差异化总结

从网络/路由层角度，本系统有四个在竞品中普遍缺失的系统性优势：

### 4.1 内核数据面崩溃隔离

eBPF/TC 程序与 BPF maps 通过 bpffs 持久化，控制面进程崩溃不影响数据面转发——所有 BPF filter、maps（slots/config/stats/ifindex_to_slot）、veth/tap 设备均独立于控制进程生命周期。只有 Lambda 的 Nitro 硬件方案有类似效果，但那依赖专用硬件而非软件架构。

### 4.2 快照扇出 + 路由瞬时可用

`snp` 模板 create（microVM 内存快照恢复）+ MMDS 确定性密钥 + routesync 路由广播三者组合，使数百到数万个快照子沙箱在创建后立即可路由、立即鉴权，无需每沙箱单独注册。这是 RL 训练场景（30K sandboxes/60s）的基础设施前提，所有竞品均无此能力组合。agent-substrate 的 CRIU 仅支持单 Actor 恢复，无批量扇出路径。

### 4.3 routesync 有界内存增量同步

逐条 Upsert + Bookmark（无全量巨帧）+ rev 断点续传，使高密度（数千沙箱）下 Worker 重连重同步时内存有界、旧路由表不下线（无路由空窗）。agent-substrate 依赖 Redis 全量查询，高并发重同步时 Redis 成为潜在瓶颈；e2b 集群模式无此机制，重连有路由空窗。

### 4.4 结构性沙箱间 L2 隔离

eBPF 程序中不存在 port→port 转发分支（可静态审计），ARP 全代答（广播帧在 ingress 即被消费），源 MAC 在出口强制改写。这比 K8s NetworkPolicy（L3/L4 ACL，可绕过）、iptables（规则散落，难审计）以及 gVisor netstack（用户态，无法约束 L2 层）的隔离强度均更高且可形式化验证。

---

## 五、潜在补强方向

| 差距 | 现状 | 参照竞品 |
|---|---|---|
| **GPU 直通** | 无提及 | Modal、AgentCore 已是 ML 工作负载标配 |
| **跨节点 L2 SDN** | vswitch 单节点实例，跨宿主依赖 GENEVE 上游网关，无内置跨节点虚拟网络 | OVN/Calico 等提供集群级 L2 overlay |
| **4096 slot 上限** | 编译期常量，裸金属节点可能需要提升 | 需重编 BPF |
| **L4+ 用户策略** | 无 ACL（委托 VMM/TC qdisc），无 L7 过滤 | AgentCore/Azure 有 VPC 级网络策略集成 |
| **用户 VPC 对等** | 无法将沙箱网络直接接入客户 VPC | Lambda ENI 注入、AgentCore VPC Lattice |
| **IPv6** | 内层管理平面为 IPv4，GENEVE 内层支持 IPv6 | agent-substrate Issue #246 同样未解决 |
| **K8s 原生集成** | 自研 sandbox-orchestrator，K8s 用户需额外适配 | agent-substrate 原生 K8s CRD，接入更标准 |

---

## 六、See Also

- `sandbox-vswitch/docs/vswitch.md` — eBPF/TC 虚拟交换机设计文档
- `sandbox-vswitch/docs/tapfd.md` — tap fd 交接协议（provider/consumer 契约）
- `sandbox-orchestrator/docs/node-proxy.md` — L7 路由转发层：routesync、park/wake、MMDS、CONNECT 隧道
- `sandbox-orchestrator/docs/node.md` — 控制面：e2b API、生命周期状态机、密钥模型
- `sandbox-orchestrator/docs/cluster-router.md` — 集群级数据面入口
- https://github.com/agent-substrate/substrate — agent-substrate 源码仓库
