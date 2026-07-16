# 沙箱资源隔离模型设计

**状态**：草稿  
**日期**：2026-07-16

---

## 目录

1. [背景](#1-背景)
2. [现状分析](#2-现状分析)
   - 2.1 进程与 cgroup 层级
   - 2.2 宿主侧资源控制现状
   - 2.3 Guest 内资源模型现状（多进程场景）
   - 2.4 mem_report 与 balloon 联动机制
   - 2.5 已知缺陷
3. [设计目标](#3-设计目标)
4. [详细设计](#4-详细设计)
   - 4.1 宿主侧：修复 cgroup 共享死锁（P0）
   - 4.2 宿主侧：补全缺失隔离维度（P1）
   - 4.3 宿主侧：节点级兜底 slice caps（P1）
   - 4.4 Guest 侧：多进程 cgroup 重构（P1/P2）
   - 4.5 块 I/O 限流（P2）
   - 4.6 配置模型扩展
5. [升级路径](#5-升级路径)
6. [约束与已知限制](#6-约束与已知限制)

---

## 1. 背景

每个沙箱由两个宿主进程协同运行：

- **sandbox-ctl run**：Go 进程，负责沙箱生命周期管理（launch / pause / resume / teardown）、vsock 通信、内存 balloon 控制、cgroup 资源写入。
- **cloud-hypervisor（CH）**：KVM 虚拟机监控器，托管 guest kernel 及用户工作负载。

guest 内部可能同时存在多类进程：**app**（主业务进程）、**plugins**（LaunchSpec 中声明的伴生进程，独立生命周期）、**exec session**（通过 exec API 创建的临时命令进程）。当前 cgroup 设计仅考虑了 app 单进程场景，在多进程场景下存在若干资源隔离缺陷。

本文分析现状、确立设计目标并给出分阶段实施方案。

---

## 2. 现状分析

### 2.1 进程与 cgroup 层级

**宿主侧进程树：**

```
systemd
└── sandbox-runner@<sid>.service    (systemd unit，Delegate=yes)
    └── sandbox-ctl run             (Go 进程，继承 unit cgroup)
        └── cloud-hypervisor        (fork-exec 子进程，PGID 独立)
```

**宿主侧 cgroup 布局（`--cgroup-adopt` 模式，orchestrator 默认）：**

```
/sys/fs/cgroup/sandbox-runner.slice/
└── sandbox-runner@<sid>.service/          ← unit cgroup（Delegate=yes）
    ├── [sandbox-ctl run]                  ← 本进程，继承至此
    └── [cloud-hypervisor]                 ← fork 子进程，自动继承同一 cgroup
        memory.max  = capacity + 32 MiB
        memory.high = allocatable × 0.875  （Settled 后写）
        memory.swap.max = 0
        cpu.max     = capacity_vcpu × 100000 / 100000µs
        cpu.weight  = clamp(allocatable_cpu × 100, 1, 10000)
```

**Guest 内进程树（多进程场景）：**

```
sandbox-init (PID 1 in guest)
├── app                      ← 主业务进程（CLONE_NEWPID + CLONE_NEWNS）
│   └── <app 子孙进程>
├── plugin-0                 ← 伴生进程（sandbox-init 直接 fork，无新 ns）
├── plugin-1
├── ...
├── exec-join                ← exec session helper（sandbox-init 直接 fork）
│   └── exec-child-joined   ← 实际执行的命令（setns 进入 app 的 mnt+pid ns）
└── exec-join-2              ← 多个并发 exec session
    └── exec-child-joined-2
```

**Guest 内 cgroup 布局（当前实际状态）：**

```
/sys/fs/cgroup/                            ← cgroupv2 根（guest 内挂载）
│   cgroup.procs    ← sandbox-init(PID 1) + exec-join + exec-child-joined
│   cgroup.subtree_control ← "+cpu +memory +io +pids"
│   （根 cgroup 无任何资源限制）
│
└── app/                                   ← 所有业务进程的共享 freeze 域
    ├── cgroup.procs   ← app + 所有 plugins（混合在同一 cgroup）
    ├── cgroup.freeze  ← quiesce 时写 1（同时冻结 app 和全部 plugins）
    └── cgroup.events  ← 轮询 "frozen 1"
        （无任何资源限制：无 memory.max、无 pids.max）
```

### 2.2 宿主侧资源控制现状

| 资源维度 | 实现机制 | 实现位置 | 状态 |
|---------|---------|---------|------|
| 内存硬上限 | `memory.max = capacity + 32 MiB overhead` | `resctl/cgroup.go:86` | ✅ 已实现 |
| 内存软水位 | `memory.high = allocatable × 0.875`（Settled 后写，规避 uffd 启动期死锁） | `resctl/cgroup.go:94` / `resctl/controller.go` | ✅ 已实现 |
| 禁 swap | `memory.swap.max = 0` | `resctl/cgroup.go:97` | ✅ 已实现 |
| CPU 硬配额 | `cpu.max = vcpu × 100000 / 100000µs` | `resctl/cgroup.go:101` | ✅ 已实现 |
| CPU 权重 | `cpu.weight = clamp(allocatable_cpu × 100, 1, 10000)` | `resctl/cgroup.go:103` | ✅ 已实现 |
| 动态内存 | PSI sensor + RequestBudget RPC + balloon vm.resize | `resctl/sensor.go` / `resctl/balloon.go` | ✅ 已实现 |
| OOM 感知 | `memory.events.local` 轮询（1s），urgency=high RequestBudget | `resctl/sensor.go:203-236` | ✅ 已实现 |
| 进程隔离 | CH `Setpgid=true`（独立 PGID） | `sandbox/serveandwait.go:511` | ✅ 已实现 |
| 单元清理 | `KillMode=control-group`（StopUnit → SIGKILL 整个 cgroup） | `orch/units.go:63` | ✅ 已实现 |
| pids.max | — | — | ❌ 缺失 |
| io.max / io.weight | — | — | ❌ 缺失 |
| LimitNOFILE | — | — | ❌ 缺失 |
| sandbox-runner.slice 级上限 | — | — | ❌ 缺失 |

### 2.3 Guest 内资源模型现状（多进程场景）

#### 2.3.1 各进程 cgroup 归属与关键行为

| 进程 | cgroup 归属 | 放置调用 | 放置失败行为 | quiesce 处理 |
|------|-----------|---------|------------|------------|
| sandbox-init（PID 1）| 根 `/`（永久）| — | — | 不冻结（设计保证，保持 vsock 响应）|
| 用户 app | `/app/` | `cgroupPlaceApp(appPid)` `main.go:161` | **fatal**（`die()`，sandbox 启动失败）| 随 `app/cgroup.freeze=1` 冻结 |
| plugin（所有）| `/app/`（与 app 共享）| `cgroupPlaceApp(pid)` `plugin.go:175` | **non-fatal**（仅 log，plugin 留在根 cgroup 继续运行）| 成功放置时随 app 冻结；放置失败时**不被冻结** |
| exec-join helper | 根 `/`（无 cgroupPlaceApp）| — | — | SIGKILL（`killExecChildren()`，在 cgroupFreeze 之前）|
| exec-child-joined | 根 `/`（继承 exec-join）| — | — | SIGKILL（同上）|

#### 2.3.2 多进程场景下 app/ cgroup 的问题

所有 plugins 与 app 进程共享同一个 `/app/` cgroup，当前无任何资源限制（无 `memory.max`、`pids.max`）：

- **内存无隔离**：一个 plugin 内存泄漏耗尽 `app/` 内存预算，会触发 OOM killer 在 `app/` 内任意选择受害进程（可能是 app 本身）
- **pids 无隔离**：任意进程（app 或 plugin）fork bomb 可耗尽 guest PID 空间，影响 sandbox-init 的管控能力
- **无法区分哪个进程消耗了内存**：`memory.current` 只有整体数字，无法定位责任方

#### 2.3.3 exec session 在根 cgroup，完全无限制

exec session 的 exec-join（`exec.go:307-345`）不调用 `cgroupPlaceApp`，进程留在根 cgroup 与 sandbox-init 共处：

- 无 `memory.max`：多个并发 exec session（如 `cat /large/file`）可合计耗尽 guest 可见内存，挤压 sandbox-init
- 无 `pids.max`：exec session 内的命令若 fork bomb，可影响 sandbox-init 的 fork 能力
- 无资源隔离：exec session 的资源占用与 sandbox-init 的资源占用无法分开核算

#### 2.3.4 plugin cgroup 放置 non-fatal 的风险

`plugin.go:175`：

```go
if err := cgroupPlaceApp(pid); err != nil {
    logf("plugin %s: cgroup place pid=%d: %v (continuing unfrozen)", ...)
    // 不 fatal，plugin 在根 cgroup 继续运行
}
```

放置失败的 plugin 在根 cgroup 运行，后果：
1. `cgroupFreeze()` 冻结 `app/` 时该 plugin **不被冻结**，快照包含运行中的 plugin 状态，破坏快照一致性
2. 该 plugin 无任何资源限制，可无限消耗内存，影响 sandbox-init

#### 2.3.5 quiesce 顺序（关键实现细节）

`vsock.go:297-331` 的 quiesce 处理顺序（**并非 exec session 随 app 一起冻结**）：

```
1. beginQuiesce()       ← 阻止新 plugin fork 和新 exec session 接入
2. killExecChildren()   ← SIGKILL 全部 exec session（exec 不在 app/ 中，不会被 freeze）
3. cgroupFreeze()       ← 冻结 app/（app + 成功放置的 plugins 同时 freeze）
4. sync + drop_caches   ← 内存页落盘
5. 关闭 stdio MUX / port-forward
6. → TypeQuiesced       ← 通知 host 可以做快照
```

plugin 在 quiesce 窗口退出时，`onExit()` 设置 `p.pending=true` 而非立即重启，等 `endQuiesce()` 后恢复，确保退出的 plugin 不进入快照。

#### 2.3.6 plugin 与 app 的生命周期关系

| 场景 | plugin 行为 |
|------|-----------|
| app 崩溃 + restart 策略允许重启 | plugins 继续运行（在 app 重启的退避延迟期间也不中断）|
| app 崩溃 + restart 策略 `never` / 正常退出 | `doReboot()` → VM 关机，所有进程（含 plugins）随 VM 终止 |
| sandbox 主动关闭（SIGTERM）| `doReboot()` → VM 关机，plugins 无 graceful shutdown 机会 |

plugin 无法独立于 VM 存活；plugin 的独立生命周期仅在 app 存在且会重启的前提下成立。

### 2.4 mem_report 与 balloon 联动机制

```
[sandbox-init]  每 5s 读 /proc/meminfo (MemAvailable, MemTotal)
     │  vsock CID=2 port:5000  TypeMemReport
     ▼
[sandbox-ctl/serveandwait]  OnMemReport → BalloonController.Hint()
     │
     ▼  balloon 调整策略：
     │  delta = memAvailable - TargetFreeBuffer
     │  TargetFreeBuffer = max(64 MiB, capacity / 32)
     │  Slack = 32 MiB（防抖，|delta| < Slack 时 no-op）
     │  MaxStep = 256 MiB（单步上限）
     │  new_balloon = clamp(current + delta, 0, capacity)
     │
     ▼  每 5s（或 kick，受 5s rate limit）
[CH HTTP API]  PUT /api/v1/vm.resize {"desired_balloon": <bytes>}
```

**为何不用 free_page_reporting（FPR）**：FPR 触发 `madvise(MADV_DONTNEED)` → KVM EPT mmu_notifier shootdown，在高密度场景下 16s 内饿死 guest vsock kthread（`balloon.go:21-26`）。mem_report 替代方案每 tick 只做一次 vm.resize，无此问题。

**balloon deflate_on_oom**：CH 启动参数含 `--balloon size=X,deflate_on_oom=on`（`ch.go:115-119`），guest 内核触发 OOM 时 virtio-balloon 驱动自动 deflate，向 OOM killer 归还页面，防止用户进程因宿主内存超售产生的虚假 OOM。

### 2.5 已知缺陷

#### D-1（高危）：sandbox-ctl 与 CH 共享 cgroup，存在 memory.high 死锁

代码在 `cgroup.go:17-22` 和 `config.go:194-197` 已明确承认：

> sandbox-ctl NEVER joins the sandbox cgroup itself … hitting `memory.high` puts the offender in `TASK_KILLABLE` D-state on return-to-user. The Go scheduler cannot run any goroutine on that thread, so sandbox-ctl can neither send SIGKILL to CH nor reap `cmd.Wait` — **full deadlock observed in density-perf forensics**.

`--cgroup-adopt` 默认模式下这一不变式被打破：sandbox-ctl 继承 unit cgroup，CH fork 后自动在同一 cgroup，AddPID 为 no-op（`cgroup.go:161-164`）。

#### D-2：pids.max 缺失，宿主 fork bomb 无防护

CH 进程及其所有线程（virtio-net/blk worker 等）无进程数上限，可耗尽宿主 PID 空间。

#### D-3：块 I/O 无限流

单个沙箱磁盘读写无配额，可打满宿主存储带宽，影响同节点其他沙箱的 I/O 延迟及 snapshot 写入速度。

#### D-4：sandbox-runner.slice 无节点级兜底

`units.go:32` 生成的 `sandbox-runner.slice` 无 `MemoryMax` / `CPUQuota`，多沙箱合计内存超售没有节点级硬性截断。

#### D-5：fd 数无显式限制

CH 打开大量 fd（memfd、tap、virtio 队列、vsock 连接），依赖系统默认 `RLIMIT_NOFILE`，行为不可预期。

#### D-6：guest app/ cgroup 无资源上限

`app/` cgroup 无 `memory.max` / `pids.max`，用户进程可消耗整个 guest 可见内存，挤压 sandbox-init 的 vsock 管控能力。

#### D-7：exec session 在根 cgroup，无任何资源限制

exec-join 及 exec-child-joined 进程留在根 cgroup（无 `cgroupPlaceApp` 调用，`exec.go:307-345`），既无内存上限，也无 pids 限制。多个并发 exec session 可合计耗尽 guest 可见内存，直接影响 sandbox-init。

#### D-8：plugin cgroup 放置失败是 non-fatal

`plugin.go:175` 放置失败仅 log，plugin 在根 cgroup 中继续运行，不受 `cgroupFreeze()` 约束，导致快照包含运行中的 plugin 状态，破坏快照一致性；同时该 plugin 无资源限制，可影响 sandbox-init。

#### D-9：app 与全部 plugins 共享 app/ cgroup，无内部隔离

任何 plugin 的内存泄漏会触发整个 `app/` cgroup 的 OOM killer，可能误杀 app 进程；无法区分各进程的资源消耗，也无法为不同 plugin 设置差异化限制。

---

## 3. 设计目标

| 编号 | 目标 | 优先级 |
|------|------|------|
| G-1 | 恢复宿主侧 cgroup 不变式：sandbox-ctl 进程树永远不在受资源限制的 cgroup 内 | P0 |
| G-2 | 补全宿主侧 pids.max，防止 fork bomb 耗尽宿主 PID | P1 |
| G-3 | 显式设置 fd 上限（LimitNOFILE） | P1 |
| G-4 | 节点级 slice caps 作为兜底，防止超售失控 | P1 |
| G-5 | guest app/ cgroup 设置聚合资源上限，隔离用户进程与 sandbox-init | P1 |
| G-6 | exec session 进入独立 cgroup，设置 pids.max 和可选 memory.max | P1 |
| G-7 | plugin cgroup 放置失败改为 fatal，消除快照一致性风险 | P1 |
| G-8 | guest 内 per-plugin 独立 cgroup，防止 plugin 互相影响及误杀 app | P2 |
| G-9 | 块 I/O 限流（可选，需设备发现）| P2 |
| G-10 | 向后兼容：不改变 LaunchSpec 外部接口，adopt/non-adopt 两种模式均适配 | — |
| G-11 | 可观测性：资源配置可从 API 查询，超限事件可追踪（OOMReport / PSI event）| — |

---

## 4. 详细设计

### 4.1 宿主侧：修复 cgroup 共享死锁（P0，对应 G-1）

**方案：在 adopt cgroup 下拆分 `ctl/` 与 `ch/` 两级 sub-cgroup**

```
/sys/fs/cgroup/sandbox-runner.slice/
└── sandbox-runner@<sid>.service/      ← unit cgroup（Delegate=yes，本身不设限制）
    ├── ctl/                            ← sandbox-ctl 自身迁入
    │   memory.max:  256 MiB（可配，overhead 预算）
    │   pids.max:    64
    └── ch/                             ← cloud-hypervisor 迁入
        memory.max:  capacity + 32 MiB
        memory.high: allocatable × 0.875（Settled 后写）
        memory.swap.max: 0
        cpu.max:     vcpu × 100000 / 100000µs
        cpu.weight:  clamp(allocatable_cpu × 100, 1, 10000)
        pids.max:    128
```

**操作顺序**（严格，不可颠倒）：

```
1. 读 /proc/self/cgroup → parent（unit cgroup 路径）
2. 写 parent/cgroup.subtree_control "+memory +cpu +pids +io"
   （此时 parent 有 sandbox-ctl 进程，但尚无子 cgroup，符合 v2 规则）
3. mkdir parent/ctl  parent/ch
4. 写 ch/ 资源限制（memory.max / cpu.max / pids.max 等）
5. 写 ctl/ 资源限制（memory.max=256MiB / pids.max=64）
6. fork cloud-hypervisor（子进程自动继承 parent cgroup）
7. AddPID(chPid) → 写 chPid 到 ch/cgroup.procs
8. sandbox-ctl 自身迁入 ctl/ → 写 "0" 到 ctl/cgroup.procs
```

**`resctl/cgroup.go` 变更摘要**：

```go
func joinCgroupAdopt(cfg CgroupConfig) (*Cgroup, error) {
    parent := cfg.Path
    enableSubtreeControl(parent, "memory", "cpu", "pids", "io")

    ctlPath := filepath.Join(parent, "ctl")
    chPath  := filepath.Join(parent, "ch")
    os.Mkdir(ctlPath, 0755)
    os.Mkdir(chPath,  0755)

    writeCgFile(chPath, "memory.max",      formatBytes(cfg.MemoryMaxBytes))
    writeCgFile(chPath, "memory.swap.max", "0")
    writeCgFile(chPath, "cpu.max",         formatCPUMax(cfg.CPUMaxQuotaUs))
    writeCgFile(chPath, "cpu.weight",      formatUint(cfg.CPUWeight))
    writeCgFile(chPath, "pids.max",        "128")
    // memory.high 延迟到 Settled() 后写入（规避 uffd 启动期死锁，同现有逻辑）

    writeCgFile(ctlPath, "memory.max", formatBytes(cfg.CtlMemoryMaxBytes))
    writeCgFile(ctlPath, "pids.max",   "64")

    return &Cgroup{chPath: chPath, ctlPath: ctlPath}, nil
}

func (cg *Cgroup) AddPID(chPid int) error {
    if err := writeCgFile(cg.chPath, "cgroup.procs", strconv.Itoa(chPid)); err != nil {
        return err
    }
    return writeCgFile(cg.ctlPath, "cgroup.procs", "0") // "0" = 当前进程
}
```

**non-adopt 模式**：sandbox-ctl 在外部，CH 独占一个新建 cgroup，已满足不变式，仅需补加 `pids.max=128`。

### 4.2 宿主侧：补全缺失隔离维度（P1，对应 G-2/G-3）

#### 4.2.1 pids.max

4.1 中已作为 `ch/pids.max=128` 和 `ctl/pids.max=64` 的一部分实现。CH 是单进程多线程（实测最大约 24 线程），128 足够并留有余量。non-adopt 模式在 `JoinCgroup` 中同步写入。

#### 4.2.2 LimitNOFILE / NoNewPrivileges（`orch/units.go` 变更）

```ini
[Service]
LimitNOFILE=65536     # CH 需要：memfd + tap + virtio 队列×vCPU + vsock
LimitNPROC=512        # sandbox-ctl + CH 线程数上限（与 pids.max 双重保障）
NoNewPrivileges=yes   # 禁止 setuid/setgid 提权（不影响 CAP_SYS_ADMIN 做 setns）
```

`NoNewPrivileges=yes` 与 `setns(CLONE_NEWNET)` 兼容：`setns` 依赖 capabilities bounding set 中的 `CAP_SYS_ADMIN`，不依赖 setuid 机制。

### 4.3 宿主侧：节点级兜底 slice caps（P1，对应 G-4）

`orch/units.go` 新增 `BuildRunnerSliceUnit()`，按 node 配置生成 `sandbox-runner.slice`：

```ini
[Slice]
MemoryMax=<node_config.runner_slice_memory_max>   # 建议：节点总内存 × 0.85
CPUQuota=<node_config.runner_slice_cpu_quota>%    # 建议：节点 CPU × 0.90 × 100
```

**默认关闭**（值为 0 时不写入），通过 node-ctl config 启用。作用：节点超售失控时 OOM killer 优先在 slice 内选牺牲者，不影响 node-ctl 本身及其他系统服务。

### 4.4 Guest 侧：多进程 cgroup 重构（P1/P2，对应 G-5/G-6/G-7/G-8）

#### 4.4.1 目标 cgroup 布局

**P1 阶段（聚合限制 + exec 隔离）：**

```
/sys/fs/cgroup/                    ← cgroupv2 根，sandbox-init (PID 1) 常驻
│   cgroup.procs    ← sandbox-init
│   cgroup.subtree_control ← "+cpu +memory +io +pids"
│   （根 cgroup 无资源限制）
│
├── exec/                          ← NEW: 全部 exec session 的父容器
│   pids.max: <exec_total_pids>    ← 所有并发 exec 进程总数上限（建议 128）
│   （可选）memory.max: <exec_total_mem>
│   cgroup.subtree_control ← "+memory +pids"
│   ├── <session-id>/              ← 每个 exec session 独立 sub-cgroup
│   │   pids.max:   32             ← 单次 exec 命令 + 子进程上限
│   │   memory.max: <per_exec_max> ← 可选，默认不限
│   └── ...
│
└── app/                           ← 用户 app + plugins 的共享 freeze 域
    memory.max: guest_capacity - init_reserve   ← NEW: 聚合内存上限
    pids.max:   512                             ← NEW: 聚合 pids 上限
    cgroup.freeze ← quiesce 时写 1（递归冻结所有子 cgroup）
    cgroup.procs  ← app + 全部 plugins（P1 阶段，扁平结构不变）
```

**P2 阶段（per-plugin 独立 cgroup）：**

```
└── app/                           ← freeze 域（本身不放进程）
    memory.max: guest_capacity - init_reserve
    pids.max:   512
    cgroup.subtree_control ← "+memory +pids"
    ├── main/                      ← app 进程专属
    │   pids.max: <app_pids>
    └── plugin-<n>/                ← 每个 plugin 独立 sub-cgroup（动态创建/销毁）
        pids.max: 32
        （可选）memory.max: <per_plugin_max>
```

> **cgroup v2 no-internal-process 规则**：P2 阶段 `app/` 有子 cgroup 后不能再直接放进程，app 需迁入 `app/main/`，每个 plugin 进入 `app/plugin-<n>/`。`app/cgroup.freeze=1` 仍然递归冻结所有子 cgroup，freeze 语义不变。

#### 4.4.2 P1：exec session 独立 cgroup（对应 G-6）

**`sandbox-init/cgroup.go` 变更**：

phase1b 阶段在创建 `app/` 的同时创建 `exec/` 根目录，并写入聚合上限：

```go
func cgroupMount() error {
    // ... 现有挂载逻辑 ...
    os.Mkdir("/sys/fs/cgroup/app",  0755)
    os.Mkdir("/sys/fs/cgroup/exec", 0755)  // NEW
    cgroupEnableControllers()
    // 写入 exec/ 聚合上限
    writeCgFile("/sys/fs/cgroup/exec", "pids.max",
        strconv.Itoa(cfg.ExecTotalPidsMax))  // 建议 128
    return nil
}
```

**`sandbox-init/exec.go` 变更**：在 fork exec-join 之前创建 per-session sub-cgroup：

```go
func forkExecChild(spec *proto.ExecSpec, cs childStdio, appPid int) (int, error) {
    sessionID := newSessionID()
    cgPath := fmt.Sprintf("/sys/fs/cgroup/exec/%s", sessionID)
    os.Mkdir(cgPath, 0755)
    writeCgFile(cgPath, "pids.max", "32")
    if cfg.ExecMemoryMaxBytes > 0 {
        writeCgFile(cgPath, "memory.max",
            strconv.FormatUint(cfg.ExecMemoryMaxBytes, 10))
    }

    cmd := exec.Cmd{ /* 现有逻辑 */ }
    if err := cmd.Start(); err != nil {
        os.Remove(cgPath)  // 启动失败，清理 cgroup
        return 0, err
    }
    // 将 exec-join 放入 per-session cgroup
    writeCgFile(cgPath, "cgroup.procs", strconv.Itoa(cmd.Process.Pid))

    // 注册 session 时记录 cgPath，退出后清理
    reg.register(cmd.Process.Pid, sessionID, cgPath)
    return cmd.Process.Pid, nil
}
```

session 结束后由 `onExit()` 调用 `os.Remove(cgPath)` 清理 sub-cgroup（需先确认 cgroup 内无进程）。

#### 4.4.3 P1：plugin cgroup 放置改为 fatal（对应 G-7）

**`sandbox-init/plugin.go:175` 变更**：

```go
// 修改前（non-fatal）：
if err := cgroupPlaceApp(pid); err != nil {
    logf("plugin %s: cgroup place pid=%d: %v (continuing unfrozen)", ...)
}

// 修改后（fatal）：
if err := cgroupPlaceApp(pid); err != nil {
    _ = cmd.Process.Kill()  // 杀死已启动的 plugin 进程
    return 0, fmt.Errorf("plugin %s: cgroup place pid=%d: %w", s.Exec, pid, err)
    // 上层 launch() 捕获错误 → 按 restart 策略退避重启
}
```

放置失败的 plugin 不允许在根 cgroup 中游离。调用方 `launch()` 收到错误后，按 restart 策略（backoff）重试启动。

#### 4.4.4 P1：app/ 聚合资源限制（对应 G-5）

在 `cgroupSetAppLimits()` 中写入聚合上限（在 `phase2ForkApp` 之前，LaunchSpec 解析后调用）：

```go
func cgroupSetAppLimits(cfg AppCgroupConfig) {
    if cfg.MemoryMaxBytes > 0 {
        writeCgFile("/sys/fs/cgroup/app", "memory.max",
            strconv.FormatUint(cfg.MemoryMaxBytes, 10))
    }
    if cfg.PidsMax > 0 {
        writeCgFile("/sys/fs/cgroup/app", "pids.max",
            strconv.Itoa(cfg.PidsMax))
    }
}
```

**`app/memory.max` 计算规则**：

```
app/memory.max = guest_capacity_bytes - init_reserve_bytes
```

| 参数 | 推荐值 | 说明 |
|------|-------|------|
| `init_reserve` | 64 MiB | sandbox-init + Go runtime + vsock 连接缓冲 |
| `app/pids.max` | 512 | app + 全部 plugins 合计进程数（可配置）|

> **与 balloon 的联动**：`app/memory.max` 基于 guest 内核视图（guest_capacity），不随 balloon inflate/deflate 变化。应设置为 capacity 减去 init_reserve，而非当前 allocatable，否则 balloon inflate 后 app 进程仍受旧限制约束。

#### 4.4.5 P2：per-plugin 独立 cgroup（对应 G-8）

每个 plugin 启动时创建 `app/plugin-<n>/` sub-cgroup，plugin 最终退出时删除：

```go
// plugin.go：startPluginProcess 变更（P2）
func startPluginProcess(s proto.PluginSpec, idx int) (int, error) {
    cgPath := fmt.Sprintf("/sys/fs/cgroup/app/plugin-%d", idx)
    os.Mkdir(cgPath, 0755)
    writeCgFile(cgPath, "pids.max", "32")

    cmd := exec.Cmd{ /* 现有逻辑 */ }
    if err := cmd.Start(); err != nil {
        os.Remove(cgPath)
        return 0, err
    }
    if err := writeCgFile(cgPath, "cgroup.procs",
        strconv.Itoa(cmd.Process.Pid)); err != nil {
        _ = cmd.Process.Kill()
        os.Remove(cgPath)
        return 0, fmt.Errorf("plugin %s: cgroup place: %w", s.Exec, err)
    }
    return cmd.Process.Pid, nil
}
```

P2 阶段同时需要将 app 主进程迁入 `app/main/`（受 no-internal-process 规则约束）：

```go
// main.go：phase2ForkApp 后（P2）
os.Mkdir("/sys/fs/cgroup/app/main", 0755)
writeCgFile("/sys/fs/cgroup/app/main", "cgroup.procs", strconv.Itoa(appPid))
// 注意：P2 阶段 cgroupPlaceApp() 写入目标从 app/ 改为 app/main/
```

**sub-cgroup 生命周期**：plugin 最终退出（不再重启）时，`onExit()` 清理对应 sub-cgroup 目录。重启时复用同一 sub-cgroup（向 `cgroup.procs` 写入新 PID 即可）。

### 4.5 块 I/O 限流（P2，对应 G-9）

需要设备 major:minor 发现：

```go
// resctl/iomax.go（新文件）
func DiscoverDevice(path string) (uint32, uint32, error) {
    var stat syscall.Stat_t
    if err := syscall.Stat(path, &stat); err != nil {
        return 0, 0, err
    }
    return unix.Major(stat.Dev), unix.Minor(stat.Dev), nil
}

// resctl/cgroup.go：JoinCgroup 追加
if cfg.IORBytesPerSec > 0 || cfg.IOWBytesPerSec > 0 {
    major, minor, _ := DiscoverDevice(cfg.SandboxRunDir)
    writeCgFile(chPath, "io.max", fmt.Sprintf(
        "%d:%d rbps=%d wbps=%d", major, minor,
        cfg.IORBytesPerSec, cfg.IOWBytesPerSec))
    // 设备发现失败时 best-effort，不阻断启动
}
```

配置项（默认 0 = 不限）：

```yaml
resources:
  control:
    io_rbps: 0
    io_wbps: 0
```

### 4.6 配置模型扩展

**`pkg/config/config.go` 新增字段**：

```go
type ControlConfig struct {
    CgroupPath      string
    Adopt           bool
    CtlMemoryMaxMiB int    // sandbox-ctl 内存预算，默认 256 MiB
    CtlPidsMax      int    // sandbox-ctl pids 上限，默认 64
    IORBytesPerSec  uint64 // 0 = 不限
    IOWBytesPerSec  uint64 // 0 = 不限
    SandboxRunDir   string // 用于 io.max 设备发现
}
```

**LaunchSpec vsock 协议扩展**（新增 guest 内限制字段）：

```go
type LaunchSpec struct {
    // 现有字段...
    AppMemoryMaxBytes  uint64 // app/ memory.max，0 = 不限
    AppPidsMax         int    // app/ pids.max，0 = 不限
    ExecTotalPidsMax   int    // exec/ pids.max（所有 session 合计），默认 128
    ExecMemoryMaxBytes uint64 // 每个 exec session memory.max，0 = 不限
}
```

---

## 5. 升级路径

| 阶段 | 变更内容 | 涉及文件 | 风险 | 测试要点 |
|------|---------|---------|------|---------|
| **P0** | adopt 模式拆分 `ctl/` 与 `ch/` sub-cgroup | `resctl/cgroup.go` | 中：验证 subtree_control 时序 | 密度测试；memory.high 节流下 sandbox-ctl 仍可 SIGKILL CH |
| **P1-a** | `LimitNOFILE=65536` / `LimitNPROC=512` / `NoNewPrivileges=yes` | `orch/units.go` | 低 | CH 启动正常；setns 可用 |
| **P1-b** | `ch/pids.max=128`（P0 含）；non-adopt 同步补 pids.max | `resctl/cgroup.go` | 低 | CH 线程数实测 ≤ 128 |
| **P1-c** | `sandbox-runner.slice` 节点级 caps（默认 off）| `orch/units.go` | 低 | off 时与现状等价 |
| **P1-d** | guest `exec/` cgroup + per-session sub-cgroup | `sandbox-init/cgroup.go` `sandbox-init/exec.go` | 低 | 并发 exec 不超 pids.max；session 结束后 cgroup 清理 |
| **P1-e** | plugin cgroup 放置改为 fatal | `sandbox-init/plugin.go` | 低 | cgroup 放置失败时 plugin 不启动，按策略退避重试 |
| **P1-f** | guest `app/memory.max` 与 `app/pids.max` | `sandbox-init/cgroup.go` `pkg/proto/proto.go` | 低 | OOM 发生在 app/ 内，sandbox-init 存活 |
| **P2-a** | guest per-plugin sub-cgroup（`app/plugin-<n>/`）| `sandbox-init/plugin.go` `sandbox-init/cgroup.go` | 中：no-internal-process 规则；app 需迁入 app/main/ | plugin 重启时 cgroup 复用；最终退出时清理 |
| **P2-b** | `io.max` 块 I/O 限流（默认 off）| `resctl/cgroup.go` `resctl/iomax.go` | 低 | 设备发现失败时不阻断启动 |

---

## 6. 约束与已知限制

### cgroup v2 no-internal-process 规则

非根 cgroup 有子 cgroup 时，不能再直接放入进程。影响：
- P1 阶段：`exec/` 有 per-session sub-cgroup，但 `exec/` 本身不放进程（exec-join 直接放入 sub-cgroup），无冲突
- P2 阶段：`app/` 有 `main/` 和 `plugin-<n>/` sub-cgroup，app 进程必须迁入 `app/main/`，不能再写 `app/cgroup.procs`，需改造 `cgroupPlaceApp()` 调用目标

### AddPID 竞态窗口

`AddPID(chPid)` 必须在 CH 产生任何线程之前执行（Linux 不自动移线程）。`serveandwait.go:525-533` 中 `cmd.Start()` 后立即 AddPID，CH 线程池尚未初始化，此窗口安全，采用新逻辑后不变。

### app/ memory.max 与 balloon 的联动

`app/memory.max` 基于 guest 内核视图（capacity），不随 balloon inflate/deflate 变化。必须设置为 capacity 减去 init_reserve，而非 allocatable。若设置为 allocatable，balloon inflate 后可见内存增加，但 `app/memory.max` 未随之调整，app 进程仍被旧限制约束。

### exec session sub-cgroup 的清理时机

exec session 结束后需等待 sub-cgroup 内进程全部退出才能 `os.Remove()`。可在 `onExit()` 回调中检查 `cgroup.procs` 是否为空后再删除，避免 EBUSY 错误。

### plugin 重启与 sub-cgroup（P2）

plugin 重启时向同一 `app/plugin-<n>/cgroup.procs` 写入新 PID，sub-cgroup 不删除也不重建，limits 保持不变。只有在 plugin 被标记为"永久退出"（restart=never 或 closed）后才删除 sub-cgroup。

### non-adopt 模式兼容

non-adopt 模式（`--cgroup-path <explicit>`）sandbox-ctl 不在目标 cgroup 内，已满足宿主侧隔离不变式，仅需补加 `pids.max=128`，无需拆分 sub-cgroup。

### 磁盘配额

`/run/sandbox/<sid>` 目录的磁盘空间（snapshot、socket、memfd 等）无配额限制，需 tmpfs `size=` 挂载选项或 project quota 实现，超出本文范围。
