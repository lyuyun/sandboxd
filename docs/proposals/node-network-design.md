# Node 级 Inbound/Outbound 网络设计

**版本**: v1.0  
**日期**: 2026-06-24  
**状态**: 草稿

---

# 1. 背景与目标

## 1.1 需求范围

### 设计背景

kuasar-sandbox 已有 sandbox-vswitch（eBPF/TC 虚拟交换机）和 node-proxy（L7 路由转发层）实现高密度 microVM 网络基础设施，但在以下两个维度存在待填补的空白：

1. **出站网络精细化管控缺失**：当前仅有粗粒度 `network.internet=true/false` 开关，缺少竞品（E2B 于 2026-06 新增）已发布的 per-sandbox IP/CIDR 允许/拒绝列表和域名过滤能力
2. **混合节点场景（k8s native pods + agent sandboxes）无设计基线**：生产集群中 agent sandbox 节点可能同时运行 k8s 原生工作负载（监控 DaemonSet、日志采集等），此时 vswitch 与 k8s CNI 的共存方案、inbound 流量分流、outbound 隔离边界均无明确设计

### 竞品调研摘要

#### E2B

- **数据面**：Firecracker microVM + Linux bridge/veth + nftables；规则随沙箱密度线性膨胀，难静态审计
- **入站路由**：`<port>-<sid>.<domain>` L7 反代（本系统兼容此约定）
- **出站管控（2026-06 新增）**：公开 `network` 配置参数，支持 IP 地址和 CIDR 块的 `allow/deny` 列表控制出站流量——验证了此需求的生产优先级
- **关键弱点**：无独立 eBPF 数据面，控制面崩溃影响数据面；无 park/wake；混合节点无专门设计

#### agent-substrate（Google，CNCF Sandbox 候选）

- **隔离技术**：gVisor 容器（syscall 过滤），非 microVM；隔离边界弱于硬件虚拟化
- **网络层（atenet）**：Envoy sidecar + DNS 路由（pull 模型）；每连接引入用户态 RTT
- **出站管控**：依赖 K8s NetworkPolicy（L3/L4），无 per-sandbox 精细策略
- **已知限制**：不支持 gRPC（Issue #254）、IPv6（Issue #246）、多端口（Issue #265）；VERY early development，API 不稳定
- **关键弱点**：Envoy sidecar 用户态 L7 代理 vs kuasar eBPF/TC 内核转发，高密度下延迟和吞吐劣势显著；DNS pull 额外 RTT；无混合节点（agent/pod）隔离设计

#### 竞品横向对比

| 维度 | E2B | agent-substrate | kuasar-sandbox（现状）|
|------|-----|-----------------|----------------------|
| 数据面内核化 | ❌ bridge+iptables | ❌ Envoy 用户态 L7 | ✅ eBPF/TC，控制面崩溃不断流 |
| 沙箱间隔离审计性 | ❌ 规则散落 | ❌ NetworkPolicy | ✅ 单 eBPF 程序，无 port→port 分支 |
| per-sandbox 出站 IP/CIDR | ✅ 2026-06 新增 | ❌ 仅 NetworkPolicy 粒度 | ❌ 仅 on/off（**待设计**）|
| 出站域名过滤 | ❌ | ❌ | ❌（**待设计**）|
| park/wake 透明恢复 | ❌ 需客户端重试 | ❌ | ✅ server-side |
| 混合节点 inbound 分流 | ❌ | △ K8s-native 但无 agent/pod 分离 | ❌（**待设计**）|
| 混合节点 outbound 隔离 | ❌ | △ K8s NetworkPolicy | ❌（**待设计**）|

**kuasar-sandbox 的竞争优势方向**：在 eBPF LPM trie 的策略执行点做 per-sandbox 出站过滤，O(1) 匹配、零额外跳数、可静态审计，相比 iptables O(n) 规则链约有 100x 延迟优势；push-based routesync 无 DNS pull RTT；microVM 硬件隔离边界优于 gVisor。

### 需求列表（RR）

| 编号 | 需求 | 优先级 |
|------|------|--------|
| RR-NET-01 | per-sandbox 出站策略：IP/CIDR allow/deny 列表，支持 allowlist/denylist 两种模式 | P0 |
| RR-NET-02 | per-sandbox 域名出站过滤，支持通配符域名 allow/deny | P1 |
| RR-NET-03 | 混合节点 inbound 分流：agent-vip 专属 IP，与 k8s Service/NodePort 端口空间物理隔离 | P0 |
| RR-NET-04 | 混合节点 outbound 默认隔离：agent sandbox 默认不可访问 k8s ClusterIP/PodCIDR | P0 |
| RR-NET-05 | CNI 共存方案：vswitch transit 设备与 k8s CNI 物理 NIC（或 VLAN）显式分离 | P0 |
| RR-NET-06 | 可选开放 k8s ClusterIP 访问（受控白名单路径）| P2 |
| RR-NET-07 | per-sandbox 出站带宽限制（TC qdisc 集成）| P2 |

### 特性范围（FE）

| 编号 | 特性 | 说明 |
|------|------|------|
| **FE-NET-1** | **出站精细管控** | |
| FE-NET-1.1 | per-sandbox eBPF 出站策略 | slot 级 egress_policy BPF map + LPM trie，支持 internet on/off 和 allow/deny CIDR 列表 |
| FE-NET-1.2 | per-sandbox 域名出站过滤 | 节点 CoreDNS + kuasar-policy 插件，per-sandbox 域名 allow/deny；vswitch 强制 DNS DNAT 防绕过 |
| **FE-NET-2** | **混合节点 Inbound** | |
| FE-NET-2.1 | Agent VIP 隔离 | 节点分配专属 agent-vip（secondary IP 或 VLAN IP），node-proxy 绑定 agent-vip:443 |
| FE-NET-2.2 | 混合节点 inbound 分流 | cluster-router 按节点 agent-vip 注解精确路由，k8s Service 流量走节点主 IP，双方不交叉 |
| **FE-NET-3** | **混合节点 Outbound 隔离** | |
| FE-NET-3.1 | k8s ClusterIP/PodCIDR 默认拒绝 | vswitch eBPF egress policy 默认 SHOT 目标为 cluster CIDR 的出包 |
| FE-NET-3.2 | CNI 共存方案 | 明确两种配置：专用 NIC（推荐）和 VLAN 子接口；文档化 TC hook 冲突分析 |
| FE-NET-3.3 | 可选开放 k8s Service 访问 | per-sandbox `cluster_access=true` 白名单，经 vswitch mgmt-plane DNAT 桥接 |

## 1.2 交付边界

### 边界说明

本设计聚焦**单节点网络层**（inbound 路由分流、outbound 策略执行、混合部署隔离），不涉及：
- 跨节点路由调度（cluster-router.md 覆盖）
- GENEVE 跨宿主隧道的路由协议（vswitch.md 覆盖）
- 沙箱内存、块存储、构建流水线（其他 FE 范畴）

### 客户可感知指标与约束

| 指标 | 目标 |
|------|------|
| 出站策略生效延迟 | < 1 ms（BPF map 原子更新，无规则链重载）|
| 域名策略生效延迟 | < 1 s（CoreDNS gRPC 轮询间隔内）|
| 混合节点 agent inbound 端口冲突 | 0（agent-vip 与 k8s 主 IP 严格分离）|
| 混合节点 agent→k8s pod 默认隔离率 | 100%（eBPF SHOT，可静态审计）|
| LPM 策略匹配延迟（per-packet overhead）| < 200 ns（BPF LPM trie，内核路径）|
| CoreDNS 崩溃影响范围 | 仅新 DNS 查询失败，已建 TCP 连接不中断 |

### 依赖接口约定

| 接口 | 对端 | 协议 |
|------|------|------|
| `vswitch-ctl attach --egress-policy=<json>` | sandbox-vswitch | CLI JSON |
| `vswitch-ctl egress-update --port=N` | sandbox-vswitch | CLI（热更新）|
| `GET /internal/dns-policy?ip=<floatingip>` | node-ctl serve | UDS gRPC |
| `kuasar-sandbox/agent-vip` Node annotation | k8s API Server | K8s 注解 |
| `node-link` agent-vip 字段扩展 | cluster-ctl registry | routesync node-link |

---

# 2. 系统设计

## 2.1 设计原理

### 核心思想

**策略锚定内核数据平面，控制面零热路径介入**

**1. eBPF 原地执行出站策略**

出站策略（allow/deny CIDR）存入 per-slot BPF map，由 TC hook 在内核路径内以 O(prefix_len) 完成 LPM 匹配，不经过任何用户态进程。控制面仅做 `bpf_map_update_elem`，< 1 ms 生效，无连接跟踪状态刷新，无规则链重载。

对比：
- iptables: O(n_rules) 线性遍历，4096 沙箱 × 10 CIDR = 4 万条规则，单包 ~10 µs
- BPF LPM trie: O(prefix_len)，单包 ~100 ns，**约 100x 优势**，且可静态审计（`bpftool map dump`）

**2. 网络命名空间硬隔离取代规则集隔离**

vswitch 在专用 netns（`sw0_vswitch`）运行，agent sandbox floatingip 段不出现在 k8s 集群路由表中。混合节点上 agent sandbox 与 k8s pod 的隔离靠 **netns 边界 + 无路由** 保证，而非 iptables 规则——后者随密度膨胀、可被绕过、难以审计。

**3. 物理 NIC 职责显式分配**

```
eth0 → k8s CNI（Flannel/Calico/Cilium）→ pod 流量
eth1 → vswitch transit → GENEVE → agent sandbox 流量
```

两条 TC hook 链完全独立，不存在 attach 顺序、优先级冲突。若仅有单块物理 NIC，使用 VLAN 子接口（`eth0.200`），Cilium 挂在 `eth0`，vswitch 挂在 `eth0.200`，同样不冲突。

**4. Agent VIP 显式化**

节点 inbound 分配专属 agent-vip（secondary IP 或 VLAN IP）。node-proxy 仅监听 agent-vip:443，k8s NodePort/LoadBalancer 走节点主 IP。双方绑定不同 IP，端口空间在 IP 层隔离，无需 iptables 防火墙规则互相保护。

**5. DNS 过滤与 eBPF IP 过滤两层纵深**

| 层 | 过滤对象 | 执行位置 | 绕过风险 |
|----|---------|---------|---------|
| eBPF LPM egress policy | 目标 IP/CIDR（L3）| TC hook（内核）| 无（TC hook 不可跳过）|
| CoreDNS policy 插件 | 域名（DNS）| 节点本机 UDP/53 | vswitch 强制 DNS DNAT 到 mgmt VIP，防 sandbox 使用外部 DNS |

**6. 默认最小权限出站**

混合节点上，agent sandbox 默认策略：
- `internet = true`（按 `network.internet` 参数）
- `cluster_access = false`（**硬编码 eBPF SHOT**，不受 `internet` 参数影响）
- `mode = open`（IP/CIDR 无限制，除 cluster CIDR）

显式开启 `egress.cluster_access=true` 才走受控白名单路径。

### 关键技术选型

| 出站策略执行方式 | P99 延迟 | 规模上限 | 审计性 | 选型 |
|----------------|---------|---------|--------|------|
| eBPF LPM trie | < 200 ns | 65K 条目/节点 | ✅ bpftool 可读 | ✅ **主路径** |
| iptables/nftables | ~10 µs（40K 规则） | 无上限但性能崩溃 | ❌ 规则散落 | ❌ |
| Envoy/userspace proxy | ~1 ms（含上下文切换）| 受内存限制 | ✅ | ❌（引入额外跳）|

| DNS 过滤方式 | 部署复杂度 | 防绕过 | 选型 |
|------------|---------|--------|------|
| 节点 CoreDNS + 自定义插件 | 中 | ✅ vswitch DNAT | ✅ **选用** |
| Envoy DNS filter | 高（需 sidecar）| △ | ❌ |
| iptables DNS 重定向 | 低 | △ | ❌（与 CNI 规则冲突风险）|

## 2.2 API & DB 设计

### 2.2.1 沙箱创建参数扩展（出站策略）

`POST /sandboxes` 请求体 `network` 字段扩展：

```json
{
  "network": {
    "internet": true,
    "max_connections": 100,
    "egress": {
      "mode": "open",
      "allow_cidrs": ["8.8.8.8/32", "1.1.1.1/32", "203.0.113.0/24"],
      "deny_cidrs":  ["169.254.169.254/32"],
      "allow_domains": ["*.github.com", "pypi.org", "*.npmjs.org"],
      "deny_domains":  ["malicious.example.com"],
      "cluster_access": false
    }
  }
}
```

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `egress.mode` | string | `"open"` | `"open"` 全放通；`"allowlist"` 仅允许 allow_cidrs + allow_domains；`"denylist"` 仅拒绝 deny_cidrs + deny_domains |
| `egress.allow_cidrs` | []string | `[]` | 目标 IP 白名单，CIDR 格式（IPv4/IPv6）|
| `egress.deny_cidrs` | []string | `[]` | 目标 IP 黑名单，CIDR 格式 |
| `egress.allow_domains` | []string | `[]` | DNS 域名白名单，支持 `*.` 前缀通配 |
| `egress.deny_domains` | []string | `[]` | DNS 域名黑名单，同上 |
| `egress.cluster_access` | bool | `false` | true = 允许访问 k8s ClusterIP/PodCIDR；仅混合节点有意义 |

**热更新接口**：

```
PATCH /sandboxes/{id}/network/egress
Body: { "mode": "denylist", "deny_cidrs": ["198.51.100.0/24"] }
→ 更新 BPF map，< 1 ms 生效，无连接中断
```

**沙箱创建响应扩展**（`POST /sandboxes` 200 响应）：

```json
{
  "sandboxId": "abc123",
  "network": {
    "agentVip": "10.100.1.5",
    "egressMode": "open",
    "clusterAccess": false
  }
}
```

### 2.2.2 节点配置（node-ctl.yaml）新增字段

```yaml
network:
  # 混合节点专属：agent inbound 绑定的 VIP
  agent_vip: "10.100.1.5"
  agent_vip_dev: "eth0:1"           # secondary IP alias；或 "eth0.200" VLAN 子接口

  # k8s ClusterIP/PodCIDR，用于 eBPF 默认拒绝策略（混合节点必填）
  cluster_pod_cidrs:
    - "10.244.0.0/16"
  cluster_service_cidrs:
    - "10.96.0.0/12"

  # vswitch 专用 transit 设备（纯 agent 节点可与 k8s CNI 共用 VLAN）
  transit_dev: "eth1"

  # DNS 过滤：节点 CoreDNS 监听地址（mgmt-service DNAT 目标）
  dns_proxy_addr: "127.0.0.1:19253"
  dns_mgmt_vip:   "169.254.169.253"  # vswitch --mgmt-service 映射的 VIP
```

### 2.2.3 DB 扩展（sandboxes 表）

```sql
-- 出站策略持久化（沙箱重建 / migrate 时可恢复）
ALTER TABLE sandboxes ADD COLUMN egress_policy_json TEXT;
-- JSON: {"mode":"open","allow_cidrs":[],"deny_cidrs":[],"allow_domains":[],"deny_domains":[],"cluster_access":false}
```

DNS 策略由 node-ctl 内存维护（随 routesync RouteEntry 同步给 CoreDNS 插件），无独立表。

### 2.2.4 Node Annotation 约定（混合节点）

```yaml
apiVersion: v1
kind: Node
metadata:
  name: node-42
  annotations:
    kuasar-sandbox/agent-vip: "10.100.1.5"      # agent inbound 专属 VIP
    kuasar-sandbox/agent-vip-dev: "eth0:1"       # 绑定设备
    kuasar-sandbox/transit-dev: "eth1"           # vswitch transit NIC
    kuasar-sandbox/node-type: "mixed"            # "agent-only" | "mixed"
```

cluster-router 在节点注册时读取此注解，更新内部路由表：agent 流量 → `agent-vip:443`，而非节点主 IP。

### 2.2.5 routesync RouteEntry 扩展

```go
// 在 RouteEntry 增加 DNS 策略字段（同步给 CoreDNS 插件）
type RouteEntry struct {
    Sid              string
    // ... 现有字段 ...
    DNSPolicy        *DNSPolicy `json:"dns_policy,omitempty"`
}

type DNSPolicy struct {
    Mode         string   `json:"mode"`           // open | allowlist | denylist
    AllowDomains []string `json:"allow_domains"`
    DenyDomains  []string `json:"deny_domains"`
}
```

## 2.3 组件模块与运行态设计

### 2.3.1 整体拓扑

**场景一：纯 Agent 节点**

```
外部流量（SDK / cluster-router）
         │ HTTPS :443
         ▼
┌─────────────────────────────────────────────────────┐
│                   Agent-Only 节点                    │
│                                                     │
│  node-proxy × K（SO_REUSEPORT，监听 :443）           │
│  ├── 按 Host/<port>-<sid> 路由 → floatingip:port    │
│  ├── park/wake（paused 沙箱透明恢复）                │
│  └── routesync 订阅 serve（push 模型）               │
│                                                     │
│  ┌──────────────────────────────────────────────┐  │
│  │           sandbox-vswitch (sw0)               │  │
│  │  eBPF/TC 内核转发，4096 slots                 │  │
│  │                                               │  │
│  │  Inbound（transit → sandbox）                 │  │
│  │    GENEVE 解封 → slot lookup → tap fd         │  │
│  │                                               │  │
│  │  Outbound（sandbox → transit）TC egress hook  │  │
│  │  ┌──────────────────────────────────────┐    │  │
│  │  │ slot_egress_policy[slot_id]          │    │  │
│  │  │  internet: on/off                   │    │  │
│  │  │  cluster_access: false (默认)        │    │  │
│  │  │  mode: open/allowlist/denylist      │    │  │
│  │  │  → LPM allow/deny CIDR 匹配         │    │  │
│  │  └──────────────────────────────────────┘    │  │
│  │                                               │  │
│  │  mgmt-plane（MMDS / DNS VIP）                 │  │
│  │    169.254.169.254:80 → node-proxy MMDS      │  │
│  │    169.254.169.253:53 → CoreDNS :19253       │  │
│  │                                               │  │
│  │  transit dev（eth1）→ GENEVE → 网关/跨宿主    │  │
│  └──────────────────────────────────────────────┘  │
│                                                     │
│  CoreDNS :19253（per-sandbox 域名策略）              │
│  microVM #1…#N（floatingip .96.1…4096）             │
└─────────────────────────────────────────────────────┘
```

**场景二：混合节点（k8s native pods + agent sandboxes）**

```
                    ┌────────────────────────────────────────────────────────────┐
                    │                         混合节点                             │
                    │                                                             │
  k8s Service 流量  │  eth0（主 NIC，k8s CNI：Cilium/Calico/Flannel）             │
  → 节点主 IP ─────►│  ├── pod-1（DaemonSet：监控）veth0 → eth0                  │
  10.0.0.5:30080   │  ├── pod-2（DaemonSet：日志）veth1 → eth0                  │
                    │  └── kube-proxy / Cilium BPF → ClusterIP 10.96.x.x        │
                    │                                                             │
  agent 流量        │  agent-vip: 10.0.0.6（secondary IP，eth0:1）               │
  → agent-vip ─────►│  node-proxy :443（仅监听 agent-vip）                       │
  10.0.0.6:443     │  ├── sid 路由 → floatingip:port（eBPF flowtable）           │
                    │  └── MMDS → 169.254.169.254（vswitch DNAT）                │
                    │                                                             │
                    │  eth1（专用 transit NIC，vswitch netns: sw0_vswitch）       │
                    │  ┌──────────────────────────────────────────────────────┐  │
                    │  │  sandbox-vswitch                                     │  │
                    │  │  slot[i] egress_policy:                              │  │
                    │  │    cluster_access=false → SHOT 10.244.0.0/16        │  │
                    │  │                          SHOT 10.96.0.0/12          │  │
                    │  │    internet=true → 允许公网出站                      │  │
                    │  │                                                      │  │
                    │  │  microVM #1 (floatingip .96.1) ─── tap #1           │  │
                    │  │  microVM #2 (floatingip .96.2) ─── tap #2           │  │
                    │  │  ...                                                 │  │
                    │  └──────────────────────────────────────────────────────┘  │
                    │                                                             │
                    │  CoreDNS :19253（agent sandbox 专用 DNS 代理）               │
                    └────────────────────────────────────────────────────────────┘

隔离边界：
  ┌──────────────────────────────────────────────────────────────────┐
  │  agent sandbox → k8s pod：                                        │
  │    路由无条目（sandbox 默认路由走 GENEVE → eth1 → 外部网关）         │
  │    + eBPF SHOT cluster CIDR（双重保护）                            │
  │  k8s pod → agent floatingip：                                     │
  │    floatingip 段（100.100.96.0/22）不在 k8s 集群路由表              │
  │    pod 无法主动到达 sandbox（除非 cluster-router 显式开放路径）       │
  └──────────────────────────────────────────────────────────────────┘
```

### 2.3.2 出站精细策略：eBPF 数据面设计

**BPF Map 结构**

```c
/* ---- per-slot 策略 ---- */
struct slot_egress_policy {
    __u8  internet;        /* 0=拒绝公网出站, 1=允许 */
    __u8  cluster_access;  /* 0=拒绝 cluster CIDR,  1=允许 */
    __u8  mode;            /* 0=open, 1=allowlist,  2=denylist */
    __u8  _pad;
    __u32 cluster_cidr_idx; /* 指向 cluster CIDR 条目的偏移（节点级共享）*/
};

/* BPF_MAP_TYPE_ARRAY，4096 slots */
struct bpf_map_def SEC("maps") map_egress_policy = {
    .type        = BPF_MAP_TYPE_ARRAY,
    .key_size    = sizeof(__u32),
    .value_size  = sizeof(struct slot_egress_policy),
    .max_entries = 4096,
};

/* ---- per-slot allow/deny CIDR LPM trie ---- */
/* key: prefix_len(32) | slot_id(16) | _pad(16) | IPv4(32) | _pad(64) = 160 bit */
struct lpm_key {
    __u32 prefix_len;
    __u16 slot_id;
    __u16 _pad;
    __be32 ip;
    __u32  _pad2[2];
};

struct bpf_map_def SEC("maps") map_egress_allow_lpm = {
    .type        = BPF_MAP_TYPE_LPM_TRIE,
    .key_size    = sizeof(struct lpm_key),
    .value_size  = sizeof(__u8),
    .max_entries = 65536,           /* 4096 slots × 16 CIDR/slot */
    .map_flags   = BPF_F_NO_PREALLOC,
};

struct bpf_map_def SEC("maps") map_egress_deny_lpm = {
    /* 同 allow，独立 trie 避免 slot_id 前缀碰撞 */
    .type        = BPF_MAP_TYPE_LPM_TRIE,
    .key_size    = sizeof(struct lpm_key),
    .value_size  = sizeof(__u8),
    .max_entries = 65536,
    .map_flags   = BPF_F_NO_PREALLOC,
};

/* ---- 节点级 cluster CIDR（共享，独立 LPM）---- */
struct bpf_map_def SEC("maps") map_cluster_cidr_lpm = {
    .type        = BPF_MAP_TYPE_LPM_TRIE,
    .key_size    = 8,               /* prefix_len(32) + IPv4(32) */
    .value_size  = sizeof(__u8),
    .max_entries = 16,              /* 节点级 cluster CIDR 数量极少 */
    .map_flags   = BPF_F_NO_PREALLOC,
};
```

**TC Egress Hook 执行逻辑**（出站方向：sandbox → transit dev）

```
sandbox 报文到达 transit dev 入口（TC ingress = sandbox 出站视角）
    │
    ├─ derive slot_id（按 src floatingip 算术推导，O(1)）
    ├─ pol = map_egress_policy[slot_id]
    │
    ├─ cluster_access=0 && dst_ip ∈ map_cluster_cidr_lpm → TC_ACT_SHOT
    │
    ├─ internet=0 && is_public_ip(dst_ip) → TC_ACT_SHOT
    │     is_public_ip: 排除 RFC1918 / loopback / link-local / mgmt VIP 段
    │
    ├─ mode=allowlist:
    │     lpm_key = {prefix_len=160, slot_id, dst_ip}
    │     map_egress_allow_lpm[lpm_key] 未命中 → TC_ACT_SHOT
    │
    ├─ mode=denylist:
    │     map_egress_deny_lpm[lpm_key] 命中 → TC_ACT_SHOT
    │
    └─ TC_ACT_OK（放通，继续 GENEVE 封装）
```

**DNS 防绕过 Hook**（附加在同一 TC ingress）

```
src_port=53 或 dst_port=53（UDP/TCP）
    AND dst_ip != dns_mgmt_vip（169.254.169.253）
    → TC_ACT_SHOT   /* 禁止 sandbox 直连外部 DNS，强制走 CoreDNS */
```

**控制面更新路径**

```
POST /sandboxes 或 PATCH .../network/egress
    │
    ▼ node-ctl 解析 egress_policy_json
    │
    ▼ vswitch-ctl attach --port=N --egress-policy=<JSON>
        （或热更新：vswitch-ctl egress-update --port=N）
    │
    ├─ bpf_map_update_elem(&map_egress_policy, &slot_id, &pol, BPF_ANY)
    │
    ├─ for each allow_cidr:
    │      lpm_key = build_lpm_key(slot_id, cidr)
    │      bpf_map_update_elem(&map_egress_allow_lpm, &lpm_key, &one, BPF_ANY)
    │
    └─ for each deny_cidr:
           bpf_map_update_elem(&map_egress_deny_lpm, &lpm_key, &one, BPF_ANY)

全程原子，< 1 ms，TC hook 下一个报文即受新策略约束
```

**沙箱销毁时策略清理**

```
vswitch-ctl detach --port=N
    → 清除 map_egress_policy[slot_id]
    → 遍历 map_egress_allow_lpm，删除 key.slot_id == N 的所有条目
    → 遍历 map_egress_deny_lpm，同上
```

### 2.3.3 DNS 出站域名过滤

**整体数据流**

```
microVM guest
  └── /etc/resolv.conf: nameserver 169.254.169.253
                                ↓ UDP/53
  vswitch mgmt-plane（DNAT: 169.254.169.253:53 → 127.0.0.1:19253）
                                ↓
  CoreDNS :19253（节点本机，systemd.service）
    └── kuasar-policy 插件
          ├─ 按 client src IP（= sandbox floatingip）查 node-ctl UDS gRPC
          │    GetDNSPolicy(floatingip) → {mode, allow_domains, deny_domains}
          │    结果缓存 30 s（sandbox 级，按 rev 失效）
          │
          ├─ mode=allowlist: qname 不匹配任何 allow_domains → NXDOMAIN
          ├─ mode=denylist:  qname 匹配 deny_domains → NXDOMAIN
          └─ 放通 → forward → 上游 DNS（节点 /etc/resolv.conf）
```

**CoreDNS kuasar-policy 插件接口（node-ctl 侧）**

```protobuf
service SandboxDNSPolicy {
    rpc GetPolicy(GetPolicyRequest) returns (PolicyResponse);
    rpc WatchPolicy(WatchRequest) returns (stream PolicyEvent);
}
message GetPolicyRequest { string floating_ip = 1; }
message PolicyResponse {
    string sid       = 1;
    string mode      = 2;   // open | allowlist | denylist
    repeated string allow_domains = 3;
    repeated string deny_domains  = 4;
    int64  rev       = 5;   // routesync rev，用于缓存失效
}
```

**mgmt-service 配置（vswitch-ctl start 参数）**

```bash
vswitch-ctl start sw0 \
  --mgmt-extract=":sw0_m0:169.254.169.254/32,169.254.169.253/32" \
  --mgmt-service="169.254.169.254:80:127.0.0.1:19254"  \  # MMDS
  --mgmt-service="169.254.169.253:53:127.0.0.1:19253"  \  # DNS（TCP）
  --mgmt-service="169.254.169.253:53:127.0.0.1:19253:udp" \  # DNS（UDP）
  ...
```

**域名通配匹配规则**

| allow_domains 条目 | 匹配的域名 |
|-------------------|---------|
| `*.github.com` | `api.github.com`、`raw.githubusercontent.com` 等 |
| `pypi.org` | 精确匹配 `pypi.org` |
| `*.pypi.org` | `files.pythonhosted.org`（子域）|

匹配算法：`strings.HasSuffix(qname+".", "."+pattern+".")` 或 `qname == pattern`（精确）。

### 2.3.4 混合节点 Inbound 分流

**Agent VIP 生命周期**

```
节点启动（node-ctl serve 启动）：
  1. 读取 config.network.agent_vip = "10.0.0.6"
  2. ip addr add 10.0.0.6/32 dev eth0:1
     （或：ip link add eth0.200 type vlan id 200; ip addr add 10.0.0.6/24 dev eth0.200）
  3. node-proxy --data-listen=10.0.0.6:443（仅绑定 agent-vip，非 0.0.0.0）
  4. node-link 上报：{node_ip: "10.0.0.5", agent_vip: "10.0.0.6", transit_dev: "eth1"}

cluster-router 接收注册：
  5. 更新路由表：该节点 agent 流量 → 10.0.0.6:443
     k8s Service 流量走 kube-proxy 原有路径 → 10.0.0.5
```

**分流矩阵（同一物理主机）**

| 流量类型 | 目标 IP | 目标端口 | 处理组件 | 隔离方式 |
|---------|--------|---------|---------|---------|
| Agent SDK 请求 | 10.0.0.6（agent-vip）| 443 | node-proxy | 独立 IP 绑定 |
| k8s NodePort | 10.0.0.5（主 IP）| 30000–32767 | kube-proxy | 独立 IP + 端口段 |
| k8s Ingress Controller | 10.0.0.5（主 IP）| 80/443 | Nginx/Traefik | 独立 IP |
| MMDS（来自 sandbox）| 169.254.169.254（VIP）| 80 | node-proxy MMDS | vswitch DNAT |
| DNS（来自 sandbox）| 169.254.169.253（VIP）| 53 | CoreDNS | vswitch DNAT |

**agent-vip 高可用（可选）**

在纯 agent 节点集群中，若节点使用 BGP/ECMP 通告路由，agent-vip 可作为 anycast VIP 广播，cluster-router 直接由 BGP 路由到最近节点，无需 cluster-router 维护 per-node VIP 映射。

### 2.3.5 CNI 共存方案

**推荐方案 A：专用 NIC（生产首选）**

```
eth0: k8s CNI 主设备
  Cilium/Calico TC hook 附着于 eth0 + pod veth pair
  kube-proxy iptables 规则在主 netns

eth1: vswitch transit 专用
  vswitch TC hook 附着于 eth1（在 sw0_vswitch netns 内）
  两者完全不共享设备，无优先级冲突
```

**方案 B：VLAN 子接口（单 NIC 场景）**

```
eth0: 承载 k8s CNI（Cilium attach eth0 + veth）

eth0.200（VLAN 200）: vswitch transit 子接口
  vswitch start --transit-dev=eth0.200
  Cilium 不 attach VLAN 子接口，TC hook 互不干扰

注意事项：
  - 上游交换机需 trunk 模式，允许 VLAN 200 透传
  - eth0 MTU 需大于 eth0.200 MTU + VLAN header 4B
  - VLAN tag 在内核 802.1Q 层处理，vswitch eBPF 看到的是 untagged 报文
```

**Cilium 与 vswitch TC hook 冲突分析**

| 场景 | Cilium attach 设备 | vswitch attach 设备 | 冲突？ |
|------|-------------------|-------------------|--------|
| 方案 A 专用 NIC | eth0, veth | eth1 | ❌ 无冲突 |
| 方案 B VLAN 子接口 | eth0, veth | eth0.200 | ❌ 无冲突 |
| 共享 eth0（不推荐）| eth0 | eth0 | ⚠️ TC 优先级冲突，需协商 priority |

若必须共享 eth0，通过 `tc filter add dev eth0 ingress priority 100` 让 vswitch BPF 先于 Cilium 执行（Cilium 默认 priority 1），但此方案需在 Cilium 升级时重新验证优先级兼容性，**不推荐生产使用**。

### 2.3.6 可选开放 k8s Service 访问（FE-NET-3.3，P2）

当 `egress.cluster_access=true` 时：

```
步骤 1: node-ctl → vswitch-ctl egress-update --port=N --cluster-access=true
        eBPF map_egress_policy[slot_id].cluster_access = 1

步骤 2: vswitch mgmt-service 新增 DNAT 条目（mgmt-extract 路由 ClusterIP 段）
        sandbox 报文中目标 ClusterIP → mgmt-plane → 宿主机主 IP
        → 宿主机主 netns 内 kube-proxy iptables → 目标 pod

步骤 3: 审计日志记录（cluster_access=true 沙箱的每条 DNS 解析和首包连接）
```

**安全限制**：即使开启 `cluster_access=true`，以下 CIDR 仍在 `deny_cidrs` 默认列表中，需显式 override 才能访问：
- k8s API Server ClusterIP（通常 10.96.0.1）
- etcd 端点（若暴露于 ClusterIP）
- node-ctl serve UDS 宿主路径（不经过网络，无此风险）

### 2.3.7 出站策略与现有网络机制集成

**与 `network.internet=false` 的关系**

`network.internet=false` 是现有参数，设置 `slot_egress_policy.internet=0`，行为不变，向后兼容。新增 `egress.allow_cidrs/deny_cidrs` 叠加在 internet 开关之后生效：

```
internet=false: 公网出站全部 SHOT（不受 allow_cidrs 影响）
internet=true + mode=allowlist: 仅 allow_cidrs 内的公网 IP 可达
internet=true + mode=denylist:  deny_cidrs 内的 IP 不可达，其余公网可达
```

**与 `network.max_connections` 的关系**

`max_connections` 控制 node-proxy 入站并发数（现有功能），与出站策略正交，互不影响。

**与 MMDS Secure 模式的关系**

MMDS（169.254.169.254）属于 mgmt-plane，经 vswitch mgmt-extract 走独立路径，**不经过 egress policy TC hook**，不受 allow/deny CIDR 影响。即使 `internet=false` 的沙箱，MMDS 仍可正常访问。

---

# 3. DFX 设计

## 3.1 性能

### 关键路径延迟目标

| 路径 | P50 | P99 | 瓶颈 |
|------|-----|-----|------|
| eBPF LPM 策略匹配（per-packet）| < 50 ns | < 200 ns | BPF LPM trie 查找深度 |
| eBPF cluster CIDR 拒绝（TC_ACT_SHOT）| < 50 ns | < 100 ns | 单次 LPM 查找 |
| BPF map 出站策略更新（热更新）| < 0.5 ms | < 2 ms | bpf_map_update_elem × N_cidrs |
| CoreDNS 域名策略查询（缓存命中）| < 0.5 ms | < 2 ms | 内存查表 + gRPC UDS |
| CoreDNS 域名策略查询（首次，无缓存）| < 2 ms | < 5 ms | node-ctl UDS gRPC 往返 |
| Agent VIP 分流附加延迟 | 0 | 0 | 仅 IP 绑定差异，无路径开销 |
| 混合节点 k8s→agent 隔离（无路由）| 0（不可达）| — | 无路由，包在路由层丢弃 |

### 资源容量分析

**BPF LPM trie 容量**：

```
BPF_MAP_TYPE_LPM_TRIE 上限（内核默认）: ~1M 条目
本设计使用量:
  4096 slots × 16 CIDR/slot（allow + deny 各）= 65,536 条目
  占比 6.5%，有充足余量

内存占用（LPM trie node = ~128B）:
  65,536 × 128B × 2（allow + deny）= ~16 MiB
```

**CoreDNS 策略缓存**：

```
per-sandbox DNSPolicy 缓存 30 s（rev 变更时主动失效）
单条缓存 ~1 KB（域名列表 × 10）
4096 沙箱 × 1 KB = ~4 MB 内存，可忽略
```

**性能对比（出站策略执行）**：

| 方案 | 4096 沙箱 × 10 CIDR 规模下单包开销 |
|------|-------------------------------|
| iptables（线性规则）| ~10 µs（40K 规则遍历）|
| nftables（set + verdict map）| ~500 ns（hash set 查找）|
| **eBPF LPM trie（本方案）** | **~100–200 ns（O(prefix_len) 查找）** |

## 3.2 可靠性

### 故障场景矩阵

| 故障场景 | agent sandbox 影响 | k8s pod 影响 | 恢复路径 |
|---------|------------------|-------------|---------|
| node-ctl 崩溃重启 | 新策略更新暂停（BPF map bpffs pin，已有策略持续生效）| 无 | systemd 重启 < 5 s；重启后重新注册 CoreDNS gRPC |
| CoreDNS 崩溃 | 新 DNS 查询失败（已建 TCP 连接不中断）| 无（pod 用 kube-dns）| systemd 重启 < 5 s |
| vswitch-ctl 进程退出 | 数据面不中断（BPF pin bpffs）| 无 | 重新 attach 已 pin 的 maps |
| eth1（transit NIC）断线 | agent sandbox outbound 中断 | 无（eth0 独立）| 运维恢复 NIC；vswitch GENEVE 重建 |
| agent-vip 漂移（IP 消失）| inbound 中断（node-proxy 监听失败）| 无 | node-ctl 检测到后重新绑定；cluster-router 降级路由 |
| BPF map 更新失败（map 满）| 新策略无法写入，报 ENOSPC | 无 | 告警 EgressPolicyMapFull；运维清理低活跃 slot |
| Cilium 升级（TC hook 重载）| 无（vswitch 挂 eth1，不共享）| Cilium 自身控制 | — |

### eBPF 策略持久化

BPF map（`map_egress_policy`、`map_egress_allow_lpm`、`map_egress_deny_lpm`、`map_cluster_cidr_lpm`）全部 pin 到 bpffs：

```
/sys/fs/bpf/sw0/
  ├── egress_policy        ← per-slot 策略
  ├── egress_allow_lpm     ← per-slot allow CIDR
  ├── egress_deny_lpm      ← per-slot deny CIDR
  ├── cluster_cidr_lpm     ← 节点级 cluster CIDR（共享）
  └── ... 现有 maps（slots, config, stats, ...）
```

node-ctl/vswitch-ctl 重启后重新 attach 已 pin 的 maps，策略数据不丢失，TC hook 全程持续执行。

### DNS 降级策略

CoreDNS 崩溃时，sandbox 内 DNS 查询超时（2 s 默认），guest 内进程收到 SERVFAIL。

降级选项（可选配置）：
```yaml
dns_proxy_fallback: "8.8.8.8"   # CoreDNS 崩溃时绕过策略的上游（仅 open 模式沙箱生效）
```

allowlist 模式沙箱在 CoreDNS 不可达时应**保守拒绝**（fail-closed），不得降级为放通。

## 3.3 安全

### 出站策略审计性

**eBPF 层**：

```bash
# 查看某 slot 的出站策略
bpftool map lookup pinned /sys/fs/bpf/sw0/egress_policy key 0x00000001

# 列出某 slot 的 allow CIDR（slot_id=1）
bpftool map dump pinned /sys/fs/bpf/sw0/egress_allow_lpm \
  | grep '"slot_id": 1'

# 实时监控出站 DROP 事件
bpftrace -e '
  tracepoint:skb:kfree_skb {
    @drops[args->reason] = count();
  }
  interval:s:5 { print(@drops); clear(@drops); }
'
```

**slot_id 前缀隔离**：LPM key 包含 `slot_id` 字段，slot A 的 allow CIDR 条目的 key 前缀与 slot B 不同，BPF LPM 查找不会跨 slot 误匹配。即使两个 slot 配置了相同 CIDR，策略也严格独立生效。

**DNS 防绕过证明**：vswitch TC hook 对 sandbox 出发的 `dst_port=53 AND dst_ip ≠ dns_mgmt_vip` 报文执行 `TC_ACT_SHOT`。此 hook 在内核执行，sandbox 内无论 root 权限、修改 `/etc/resolv.conf`、还是直接 `sendto(8.8.8.8:53)` 均无法绕过。

### 混合节点 k8s 控制面保护

**防护层次**：

1. **无路由层**：sandbox 内默认路由指向 GENEVE 网关，无 k8s ClusterIP 路由条目，sandbox 内核不知道如何到达 10.96.x.x
2. **eBPF SHOT 层**：即使 sandbox 内注入了路由，报文到达 transit dev 入口时被 `is_cluster_cidr()` 检查拦截
3. **netns 隔离层**：vswitch 在 `sw0_vswitch` netns，sandbox floatingip 段不暴露在主 netns，k8s pod 无法主动到达 sandbox

三层独立防护，单层失效不影响整体安全性。

**k8s API Server 保护（即使开启 cluster_access）**：

默认 deny_cidrs 注入：
```go
// node-ctl 启动时，若检测到混合节点模式，自动向 cluster default egress deny 追加 k8s API Server IP
defaultDenyCIDRs := []string{
    kubeAPIServerClusterIP + "/32",  // 通常 10.96.0.1
}
```

### Agent VIP 与 k8s Service 隔离证明

- node-proxy 绑定 `agent-vip:443`，OS kernel 拒绝其他进程 bind 同一 IP:Port
- kube-proxy 仅操作主 IP（10.0.0.5）的 iptables DNAT 规则，不触及 agent-vip
- 反向：主 IP 上的 NodePort（:30080）不会被 node-proxy 消费，node-proxy 不监听主 IP
- 无需额外 iptables 规则互相保护，IP 层天然分离

### 攻击面分析

| 攻击向量 | 缓解措施 | 防护层 |
|---------|---------|--------|
| sandbox 访问 k8s API Server | eBPF SHOT cluster CIDR + 无路由 | L3+eBPF |
| sandbox 访问同节点 pod | eBPF SHOT PodCIDR + netns 隔离 + 无路由 | netns+eBPF |
| sandbox 使用外部 DNS 绕过域名策略 | vswitch TC SHOT 非 mgmt-VIP UDP/TCP 53 | eBPF |
| sandbox 伪造源 IP | GENEVE 封装固定源，eBPF 出口强制改写源 MAC | eBPF+L2 |
| k8s pod 主动访问 sandbox floatingip | floatingip 不在 k8s 路由表，pod 无法到达 | 无路由 |
| 恶意沙箱写入 BPF map | BPF map 仅 root vswitch-ctl 进程可写；sandbox 无 CAP_SYS_ADMIN | 权限 |
| CoreDNS 投毒（返回错误 IP 绕过 eBPF）| DNS 查询返回的 IP 仍受 eBPF LPM CIDR 过滤，即使解析到允许域名但 IP 在 deny 列表中也拦截 | eBPF |

## 3.4 可运维

### 变更方式

| 变更类型 | 操作 | 影响范围 | 重启是否必要 |
|---------|------|---------|------------|
| 更新沙箱出站策略（CIDR）| `PATCH /sandboxes/{id}/network/egress` | 单沙箱，< 1 ms 生效 | 否 |
| 更新沙箱域名策略 | 同上（`allow_domains/deny_domains`）| 单沙箱，< 1 s 生效（CoreDNS 缓存 TTL）| 否 |
| 更新节点 cluster CIDR（热重载）| `kill -HUP node-ctl` | 全节点，< 100 ms（BPF map 更新）| 否 |
| 切换 agent-vip | 修改 config + drain 节点 + 重启 node-ctl | 节点 inbound 短暂中断 | 是（需 drain）|
| 增加新 CIDR 规则 | `vswitch-ctl egress-update --port=N --add-allow=10.0.0.0/8` | 单 slot，< 1 ms | 否 |
| 切换 DNS 策略模式 | `PATCH .../egress` 更新 `mode` 字段 | 单沙箱，下次 DNS 查询生效 | 否 |
| 添加 CoreDNS 上游 DNS | 修改 CoreDNS Corefile + `systemctl reload coredns` | 全节点 DNS 查询，< 1 s | 否 |
| eBPF 程序升级（vswitch 新版本）| `vswitch-ctl stop sw0 && vswitch-ctl start sw0` | 数据面重建（已建连接中断）| 是（需 drain）|

### 日志与监控预埋点

**新增 Prometheus Metrics**：

```
# 出站策略拦截计数（per-slot，按原因，来自 eBPF perf event 或 per-CPU map）
vswitch_egress_drop_total{slot_id, reason}
# reason: internet_blocked | cluster_blocked | allowlist_miss | denylist_hit | dns_bypass_blocked

# DNS 域名策略命中（per-sandbox，来自 CoreDNS 插件）
coredns_policy_query_total{sandbox_id, result}
# result: allowed | blocked_allowlist | blocked_denylist | error_unreachable

# CoreDNS 策略 gRPC 延迟
coredns_policy_grpc_duration_seconds{quantile}

# agent-vip 分流统计
node_proxy_inbound_by_vip_total{vip_type}
# vip_type: agent | unknown（告警项：表示流量到达非 agent-vip，可能配置错误）

# BPF map 使用率
vswitch_egress_lpm_entries_used{map}  # map: allow | deny
vswitch_egress_lpm_entries_capacity{map}
```

**关键告警**：

| 告警名称 | 触发条件 | 级别 | 处置建议 |
|---------|---------|------|---------|
| AgentVIPConflict | `node_proxy_inbound_by_vip_total{vip_type="unknown"} > 0` | critical | 检查 agent-vip 与 k8s 服务 IP 分配是否重叠 |
| EgressPolicyMapFull | `vswitch_egress_lpm_entries_used / capacity > 0.8` | warning | 排查高 CIDR 数量沙箱，考虑将通配段合并为更大 CIDR |
| CoreDNSDown | CoreDNS health check 失败持续 30 s | critical | 检查 CoreDNS 进程，排查 gRPC 接口可达性 |
| ClusterAccessSuspicious | `vswitch_egress_drop_total{reason="cluster_blocked"}` 某 sandbox 每分钟 > 100 次 | warning | 可能为配置错误或攻击尝试，排查该沙箱 |
| TransitNICDown | vswitch transit dev link DOWN | critical | 检查 eth1 NIC 状态；agent sandbox outbound 全部中断 |
| DNSBypassAttempt | `vswitch_egress_drop_total{reason="dns_bypass_blocked"} > 0` | warning | sandbox 尝试绕过 CoreDNS，排查来源 slot |

**结构化日志示例（egress policy 更新）**：

```json
{
  "ts": "2026-06-24T10:00:00Z",
  "level": "info",
  "component": "node-ctl",
  "subcomponent": "egress-policy",
  "sid": "abc123",
  "slot_id": 42,
  "msg": "egress policy updated",
  "mode": "allowlist",
  "allow_cidrs": ["8.8.8.8/32"],
  "deny_cidrs": [],
  "allow_domains": ["*.github.com"],
  "cluster_access": false,
  "bpf_update_ms": 0.8
}
```

### 线上诊断工具

```bash
# 查看某 sandbox 出站策略（按 slot_id）
vswitch-ctl show egress-policy sw0 --port=<slot>

# 实时出站 DROP 事件流（bpftrace）
bpftrace -e '
  tracepoint:skb:kfree_skb { @[args->reason, args->location] = count(); }
  interval:s:1 { print(@); clear(@); }
'

# CoreDNS 策略命中统计
curl -s http://127.0.0.1:9153/metrics | grep coredns_policy

# 检查 agent-vip 是否正确绑定
ip addr show dev eth0:1 | grep agent

# 检查 vswitch LPM trie 使用量
bpftool map show pinned /sys/fs/bpf/sw0/egress_allow_lpm | grep -E "entries|memlock"

# 验证 DNS 防绕过（应被 SHOT）
# 在宿主机上模拟 sandbox UDP 53 到外部（预期：无响应）
nc -u 8.8.8.8 53  # 在 sandbox floatingip netns 内执行

# 混合节点 NIC 流量分布
ethtool -S eth0 | grep -E "(rx|tx)_packets"
ethtool -S eth1 | grep -E "(rx|tx)_packets"
```

---

# 4. FE 与 US 分工

| FE 编号 | US 编号 | 描述 | 负责人 | 预估工时 |
|--------|--------|------|-------|---------|
| FE-NET-1.1 | US-1 | eBPF egress policy BPF map 数据结构设计（`slot_egress_policy` + LPM key schema）| vswitch team | M（1w）|
| FE-NET-1.1 | US-2 | TC egress hook 策略执行逻辑（internet/cluster/allowlist/denylist 分支）| vswitch team | M（1w）|
| FE-NET-1.1 | US-3 | DNS 防绕过 TC hook（SHOT 非 mgmt-VIP UDP/TCP 53）| vswitch team | S（3d）|
| FE-NET-1.1 | US-4 | `vswitch-ctl attach --egress-policy=<json>` 参数支持 + BPF map 更新 | vswitch team | S（3d）|
| FE-NET-1.1 | US-5 | `vswitch-ctl egress-update` 热更新子命令 | vswitch team | S（3d）|
| FE-NET-1.1 | US-6 | node-ctl `POST /sandboxes` 出站策略参数解析 + vswitch-ctl 调用 | orchestrator team | S（3d）|
| FE-NET-1.1 | US-7 | `PATCH /sandboxes/{id}/network/egress` 热更新 REST API | orchestrator team | S（3d）|
| FE-NET-1.1 | US-8 | sandboxes 表 `egress_policy_json` 字段 + 迁移脚本 | orchestrator team | S（2d）|
| FE-NET-1.2 | US-9 | CoreDNS `kuasar-policy` 插件开发（gRPC 接口 + allowlist/denylist 域名匹配）| infra team | L（2w）|
| FE-NET-1.2 | US-10 | node-ctl DNS 策略 gRPC 接口（`SandboxDNSPolicy` service）| orchestrator team | M（1w）|
| FE-NET-1.2 | US-11 | vswitch `--mgmt-service` DNS VIP 配置 + CoreDNS systemd 部署集成 | infra team | S（3d）|
| FE-NET-1.2 | US-12 | routesync RouteEntry `dns_policy` 字段扩展（下推给 CoreDNS 插件）| orchestrator team | S（2d）|
| FE-NET-2.1 | US-13 | node-ctl `agent_vip` 配置解析 + secondary IP 绑定（startup/shutdown 生命周期）| orchestrator team | S（3d）|
| FE-NET-2.1 | US-14 | node-proxy `--data-listen` 默认从 `0.0.0.0` 改为 `agent-vip:443`（混合节点）| orchestrator team | S（2d）|
| FE-NET-2.2 | US-15 | node-link `agent_vip` 字段上报 + cluster-router 路由表按 agent-vip 更新 | orchestrator team | M（1w）|
| FE-NET-2.2 | US-16 | K8s Node annotation `kuasar-sandbox/agent-vip` 读写 + 节点注册集成 | orchestrator team | S（3d）|
| FE-NET-3.1 | US-17 | 混合节点启动时自动填充 `map_cluster_cidr_lpm`（读 `cluster_pod_cidrs` 配置）| orchestrator team | S（3d）|
| FE-NET-3.1 | US-18 | eBPF hook `is_cluster_cidr()` 函数 + `cluster_access=0` 默认 SHOT 逻辑 | vswitch team | S（3d）|
| FE-NET-3.2 | US-19 | CNI 共存方案文档（专用 NIC + VLAN 两种配置，TC hook 冲突分析）| infra team | S（2d）|
| FE-NET-3.3 | US-20 | `cluster_access=true` 可选开放路径：eBPF 放通 + mgmt-service DNAT 桥接 | vswitch+orchestrator | L（2w）|

**工时说明**：S=3d，M=1w，L=2w；核心路径（US-1~8, US-13~18）为 P0，预计 6 周内可交付 FE-NET-1.1、FE-NET-2.x、FE-NET-3.1；FE-NET-1.2 和 FE-NET-3.3 后续迭代。
