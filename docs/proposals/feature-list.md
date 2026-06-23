# AI 沙箱平台特性列表

**版本** v1.0 · **日期** 2026-06-23  
**数据来源** e2b-dev/infra 源码分析 + kuasar-sandbox node-ctl-spec.md v1.2

---

## 说明

本文档汇总 AI 沙箱平台的完整特性清单，按功能域分组，标注各特性在 e2b 和 kuasar-sandbox 中的支持状态。旨在作为能力对比和路线图规划的基准。

状态标注：
- ✅ 已支持（完整实现）
- 🔶 部分支持（有限制或缺关键细节）
- ❌ 不支持
- 📋 规格已设计但未实现

---

## 一、沙箱生命周期管理

### 1.1 基础生命周期

| 特性 | e2b | kuasar-sandbox | 说明 |
|---|---|---|---|
| 从模板创建沙箱 | ✅ | ✅ | |
| 快照恢复启动（snp 模板）| ✅ | ✅ | |
| 冷启动（img 模板）| ✅ | ✅ | |
| 获取沙箱详情 | ✅ | ✅ | |
| 列出沙箱（分页 + 状态过滤）| ✅ | 🔶 | kuasar-sandbox 仅列本节点 |
| 终止沙箱 | ✅ | ✅ | |
| 优雅终止（grace period）| ✅ | ❌ | e2b 有 TimeoutStopSec；kuasar-sandbox 直接 SIGKILL |

### 1.2 暂停与恢复

| 特性 | e2b | kuasar-sandbox | 说明 |
|---|---|---|---|
| 暂停（全内存快照）| ✅ | ✅ | |
| 暂停（仅文件系统快照）| ✅ | ✅ | kuasar-sandbox 通过 checkpoint.mode 区分 |
| 快照前内存回收（fstrim/sync/drop_caches）| ✅ | ✅ | |
| 快照前文件系统冻结 | ✅ | ✅ | |
| 快照前 heap 收拢 | ✅ | ✅ | |
| Resume（内存快照恢复）| ✅ | ✅ | |
| Resume（冷启动恢复）| ✅ | ✅ | |
| Connect（保活或自动恢复）| ✅ | ✅ | |
| 自动暂停（TTL 到期）| ✅ | ✅ | |
| 自动恢复（数据面触发，用户无感知）| ✅ | ✅ | |
| TTL 刷新（KeepAlive）| ✅ | ✅ | |
| TTL 动态修改 | ✅ | ✅ | |
| 暂停沙箱跨节点迁移 | ✅ | ✅ | kuasar-sandbox export/import token |
| 恢复时超时配置（park_timeout）| ✅ | ✅ | |

### 1.3 沙箱配置

| 特性 | e2b | kuasar-sandbox | 说明 |
|---|---|---|---|
| 注入环境变量 | ✅ | ✅ | |
| 指定用户自定义 metadata | ✅ | ✅ | |
| 创建时指定 CPU 数 | ✅ | ✅ | |
| 创建时指定内存 MB | ✅ | ✅ | |
| 创建时指定磁盘大小 | ✅ | 🔶 | kuasar-sandbox 通过模板固定 |
| 启用 Secure 模式（envd 双重鉴权）| ✅ | ❌ | |
| 指定 AutoPause 策略 | ✅ | ✅ | |
| 指定 AutoResume 策略 | ✅ | ✅ | |
| CA Bundle 注入 | ✅ | ❌ | |
| 运行时在线调整规格 | ❌ | ❌ | 双方均不支持 |

---

## 二、运行时数据面（envd）

### 2.1 进程执行

| 特性 | e2b | kuasar-sandbox | 说明 |
|---|---|---|---|
| 启动进程（任意命令）| ✅ | ✅ | 经 envd |
| 指定执行用户 | ✅ | ✅ | |
| 指定工作目录 | ✅ | ✅ | |
| stdin/stdout/stderr 流式返回 | ✅ | ✅ | |
| PTY 终端支持 | ✅ | ✅ | |
| 列出运行中进程 | ✅ | ✅ | |
| 终止进程 | ✅ | ✅ | |
| 进程 cgroup 管理（freeze/thaw）| ✅ | ✅ | |

### 2.2 文件系统

| 特性 | e2b | kuasar-sandbox | 说明 |
|---|---|---|---|
| 上传文件（multipart）| ✅ | ✅ | |
| 下载文件 | ✅ | ✅ | |
| 文件合并（compose）| ✅ | ✅ | |
| 文件签名访问（URL 鉴权）| ✅ | ✅ | |
| 文件错误细分（NotFound/InvalidUser/DiskFull）| ✅ | ✅ | |

### 2.3 端口与网络访问

| 特性 | e2b | kuasar-sandbox | 说明 |
|---|---|---|---|
| 用户端口自动发现（socat）| ✅ | ✅ | envd port scanner |
| HTTP 透明代理（用户端口）| ✅ | ✅ | httputil.ReverseProxy |
| WebSocket 透明代理 | ✅ | 🔶 | 标准库支持，但未针对 desktop 场景补充路由 |
| gRPC 透明代理 | ✅ | ✅ | h2c 支持 |
| 系统时间同步（与宿主对齐）| ✅ | ✅ | |
| Hyperloop 高性能事件通道 | ✅ | ❌ | e2b 专有特性 |

### 2.4 持久卷

| 特性 | e2b | kuasar-sandbox | 说明 |
|---|---|---|---|
| 创建命名持久卷 | ✅ | ❌ | |
| 卷挂载到沙箱（NFS）| ✅ | ❌ | |
| 卷访问 token 管理 | ✅ | ❌ | |
| 多卷挂载 | ✅ | ❌ | |
| 卷列表与删除 | ✅ | ❌ | |

---

## 三、模板与构建系统

| 特性 | e2b | kuasar-sandbox | 说明 |
|---|---|---|---|
| 注册模板（分配 templateID）| ✅ | ✅ | |
| 从 Docker 镜像构建模板（fromImage）| ✅ | ✅ | |
| 从已有模板构建（fromTemplate）| ✅ | ✅ | |
| RUN/ENV/ARG/WORKDIR/USER 步骤 | ✅ | ✅ | |
| COPY 步骤（文件直传 OBS）| ✅ | ✅ | 依赖 files_storage 配置 |
| startCmd（预热快照）| ✅ | ✅ | |
| readyCmd（就绪探测）| ✅ | ✅ | |
| 构建日志实时流（logsOffset 分页）| ✅ | ✅ | |
| 构建状态查询 | ✅ | ✅ | |
| 快照模板（snp）| ✅ | ✅ | |
| 镜像模板（img）| ✅ | ✅ | |
| 模板列表 | ✅ | ✅ | |
| 模板版本 / 人类可读标签 | ✅ | ❌ | e2b 有 tag/alias；kuasar-sandbox 仅 content key |
| 模板 alias 管理 | ✅ | ❌ | |
| 构建层级增量缓存（cache-from）| ❌ | 🔶 | 双方均无层级缓存；kuasar-sandbox 有 Referrers 幂等（相同镜像跳过）|
| 构建并发数控制 | ✅ | ✅ | |
| 构建磁盘用量限制 | ✅ | ✅ | |
| 构建镜像安全扫描 | ❌ | ❌ | 双方均不支持 |

---

## 四、内存管理与快照加速

| 特性 | e2b | kuasar-sandbox | 说明 |
|---|---|---|---|
| 全内存快照（memfile 序列化）| ✅ | ✅ | |
| 差分快照（脏页追踪）| ✅ | ✅ | |
| 内存页去重（对比父快照）| ✅ | ✅ | |
| UFFD 按需内存加载 | ✅ | ✅ | |
| 异步内存预取（按历史缺页顺序）| ✅ | ✅ | |
| 内存气球（balloon，空闲页上报）| ✅ | ✅ | |
| Balloon hinting（主动提示空闲页）| ✅ | ✅ | |
| Huge pages 支持 | ✅ | ✅ | |
| 快照扇出（同一快照起多沙箱）| ✅ | ✅ | RL 训练核心能力 |
| DAX 共享 runtime 镜像 | ❌ | ✅ | kuasar-sandbox 独有，虚拟 PMEM 跨沙箱共享 page cache |
| 内存统一持有（memfd + 外部 uffd）| ❌ | ✅ | kuasar-sandbox 独有，CH 补丁 |

---

## 五、块存储与缓存

| 特性 | e2b | kuasar-sandbox | 说明 |
|---|---|---|---|
| 内存映射块缓存（mmap）| ✅ | ✅ | |
| 稀疏文件支持（空洞 punch）| ✅ | ✅ | |
| 流式块加载（后台异步拉取）| ✅ | ✅ | |
| 块级去重（content-addressed）| ✅ | ✅ | |
| 块状态追踪（dirty/zero/cached）| ✅ | ✅ | |
| 分层缓存（L1 本地 / L2 集群 / L3 OBS）| ❌ | ✅ | kuasar-sandbox 独有，三级缓存栈 |
| 收敛加密（相同内容 → 相同密文）| ❌ | ✅ | kuasar-sandbox 独有，跨租户去重 |
| Maglev shuffle-sharding 缓存亲和 | ❌ | ✅ | kuasar-sandbox 独有 |
| 确定性展平（EROFS 字节级相同）| ❌ | ✅ | kuasar-sandbox 独有 |

---

## 六、网络控制

### 6.1 基础网络

| 特性 | e2b | kuasar-sandbox | 说明 |
|---|---|---|---|
| 沙箱网络隔离 | ✅ | ✅ | |
| 沙箱间无横向通信 | ✅ | ✅ | |
| 网络 slot 池化管理 | ✅ | ✅ | |
| 互联网访问开关 | ✅ | ❌ | |
| MMDS 沙箱元数据服务 | ✅ | ✅ | |
| 网络接口发送速率限制 | ✅ | ✅ | |
| eBPF/TC 纯内核数据面 | ❌ | ✅ | kuasar-sandbox 独有，控制面崩溃不断流 |
| ARP 代答（L2 结构性隔离）| ❌ | ✅ | kuasar-sandbox 独有 |
| GENEVE 隧道跨宿主传输 | ❌ | ✅ | kuasar-sandbox 独有 |

### 6.2 出站网络策略

| 特性 | e2b | kuasar-sandbox | 说明 |
|---|---|---|---|
| 出站允许规则（域名/CIDR）| ✅ | ❌ | |
| 出站拒绝规则 | ✅ | ❌ | |
| 出站代理（BYOP，自定义 SOCKS5/HTTP）| ✅ | ❌ | |
| 出站规则运行时热更新 | ✅ | ❌ | |
| nftables 防火墙（原子规则更新）| ✅ | ❌ | |
| TCP 出站经 SOCKS5 用户态处理 | ✅ | ❌ | |
| 非 TCP 出站 allow/deny | ✅ | ❌ | |

### 6.3 入站与流量控制

| 特性 | e2b | kuasar-sandbox | 说明 |
|---|---|---|---|
| 入站公网访问开关 | ✅ | ❌ | |
| 流量 access token（用户端口）| ✅ | ✅ | |
| 请求 Host 掩码 | ✅ | ❌ | |
| 域名级请求头变换规则 | ✅ | ❌ | |
| 并发连接数限制（per-sandbox）| ✅ | ✅ | |

---

## 七、代理层

| 特性 | e2b | kuasar-sandbox | 说明 |
|---|---|---|---|
| 双层代理（外部入口 + 节点内）| ✅ | ✅ | |
| Host-based 路由（\<port\>-\<sid\>.\<domain\>）| ✅ | ✅ | |
| HTTP/HTTPS 反向代理 | ✅ | ✅ | |
| WebSocket 透明代理（101 Upgrade）| ✅ | ✅ | 均基于 httputil.ReverseProxy |
| H2C / gRPC 支持 | ✅ | ✅ | |
| 连接池（per-sandbox）| ✅ | ✅ | |
| 连接重试（线性退避）| ✅ | ✅ | |
| 暂停沙箱自动恢复（数据面触发）| ✅ | ✅ | |
| 数据面 worker 水平扩展 | 🔶 | ✅ | kuasar-sandbox SO_REUSEPORT；e2b 有 LB 层 |
| routesync 增量路由同步 | ❌ | ✅ | kuasar-sandbox 独有 |
| 兜底网关（worker 不可用时）| ❌ | ✅ | kuasar-sandbox 独有 |
| 友好错误页（sandbox not found 等）| ✅ | ❌ | |

---

## 八、安全与鉴权

| 特性 | e2b | kuasar-sandbox | 说明 |
|---|---|---|---|
| API Key 认证 | ✅ | ✅ | |
| HMAC 派生 api_key（无查表验证）| ❌ | ✅ | kuasar-sandbox 独有 |
| OAuth2 认证 | ✅ | ❌ | |
| 沙箱 per-sandbox access token | ✅ | ✅ | |
| Secure 模式（envd 双重鉴权）| ✅ | ❌ | |
| CA Bundle 自定义注入 | ✅ | ❌ | |
| 多租户数据隔离（跨租户返回 404）| ✅ | ✅ | |
| AES-256-GCM 密钥加密存储 | ✅ | ✅ | |
| 密钥轮换（keytag 多密钥）| ❌ | ✅ | kuasar-sandbox 独有 |
| 集群密钥 mTLS 分发 | ❌ | ✅ | kuasar-sandbox 独有 |
| API 请求体大小限制 | ✅ | 🔶 | |
| 租户封禁（team blocking）| ✅ | ❌ | |
| Admin API key 管理 | ✅ | ✅ | |
| 构建凭据隔离（不入 guest）| ✅ | ✅ | |
| 运行时数据静态加密（overlay 写层）| ❌ | ❌ | 双方均缺失 |

---

## 九、可观测性

| 特性 | e2b | kuasar-sandbox | 说明 |
|---|---|---|---|
| 沙箱运行时指标（CPU/内存/磁盘）| ✅ | ❌ | |
| 沙箱启动延迟指标 | ✅ | 🔶 | kuasar-sandbox 有内部 startupStats，未暴露 API |
| 节点级资源指标 | ✅ | 🔶 | kuasar-sandbox 有 resource status CLI，无标准 metrics |
| Prometheus 完整指标清单 | ✅ | ❌ | kuasar-sandbox 规格仅 2 个 counter |
| OpenTelemetry 分布式追踪 | ✅ | ❌ | |
| 沙箱日志 API（v1）| ✅ | ❌ | kuasar-sandbox 无日志 API |
| 沙箱日志 API（v2：方向/cursor/level/搜索）| ✅ | ❌ | |
| 构建日志流 | ✅ | ✅ | |
| 审计日志（控制面操作）| ✅ | 🔶 | kuasar-sandbox 有但强制 tmpfs，重启丢失 |
| 审计日志持久化 | ✅ | ❌ | |
| 结构化日志聚合（Loki/ELK 兼容）| ✅ | ❌ | |
| ClickHouse 指标存储 | ✅ | ❌ | |
| SLO/SLA 定义 | ✅ | ❌ | |

---

## 十、资源管控

| 特性 | e2b | kuasar-sandbox | 说明 |
|---|---|---|---|
| 沙箱内存超分（水位仲裁）| ❌ | ✅ | kuasar-sandbox 独有，60× 密度 |
| 动态 burst grant（令牌桶）| ❌ | ✅ | kuasar-sandbox 独有 |
| PSI 反压退避（非 OOM kill）| ❌ | ✅ | kuasar-sandbox 独有 |
| 主动内存回收（settled 阶段）| ❌ | ✅ | kuasar-sandbox 独有 |
| 启动预算池隔离（防惊群）| ❌ | ✅ | kuasar-sandbox 独有 |
| 紧急水位强制回收 | ❌ | ✅ | kuasar-sandbox 独有 |
| per-tenant quota（最大沙箱数）| ✅ | ❌ | |
| per-tenant 并发构建数限制 | ✅ | ❌ | |
| per-tenant API 请求限流 | ✅ | ❌ | |
| cgroup v2 资源隔离 | ✅ | ✅ | |
| 磁盘 I/O 速率限制 | ✅ | ✅ | |
| 网络发送速率限制 | ✅ | ✅ | |
| 熵源设备速率限制 | ✅ | ✅ | |

---

## 十一、运营与计费

| 特性 | e2b | kuasar-sandbox | 说明 |
|---|---|---|---|
| 用量计量（启停时间 × 规格）| ✅ | ❌ | |
| 计费事件推送（外部系统集成）| ✅ | ❌ | |
| Webhook 事件推送（状态变更等）| ✅ | ❌ | |
| 团队级用量聚合统计 | ✅ | ❌ | |
| Admin 强杀团队全部沙箱 | ✅ | ✅ | |
| Admin 取消团队全部构建 | ✅ | ✅ | |
| 节点滚动升级 | ❌ | ❌ | 双方均缺失 |
| 配置中心集成（多节点统一下发）| ❌ | ❌ | 双方均缺失 |

---

## 十二、集群与多节点

| 特性 | e2b | kuasar-sandbox | 说明 |
|---|---|---|---|
| 多节点集群调度 | ✅ | ✅ | |
| 节点亲和性（resume 优先原节点）| ✅ | ✅ | |
| 沙箱跨节点迁移 | ✅ | ✅ | |
| 节点心跳与健康检测 | ✅ | ✅ | |
| 节点排空（drain）| ✅ | ✅ | |
| 集群全局沙箱列表 | ✅ | ❌ | kuasar-sandbox LIST 仅本节点 |
| 放置调度（P2C 负载均衡）| ✅ | ✅ | kuasar-sandbox cluster-ctl scaler |
| shuffle-sharding 调度（缓存亲和）| ❌ | ✅ | kuasar-sandbox 独有 |

---

## 十三、桌面与 Computer Use

| 特性 | e2b | kuasar-sandbox | 说明 |
|---|---|---|---|
| 虚拟桌面环境（Xvfb + WM）| ✅ | ❌ | |
| 鼠标事件注入（xdotool）| ✅ | ❌ | |
| 键盘事件注入 | ✅ | ❌ | |
| 截图 API | ✅ | ❌ | |
| 屏幕流传输（VNC over WebSocket）| ✅ | ❌ | |
| 窗口管理（获取窗口列表/焦点）| ✅ | ❌ | |
| 浏览器预装（Chrome/Firefox）| ✅ | ❌ | |

---

## 十四、Feature Flag 体系

| 特性 | e2b | kuasar-sandbox | 说明 |
|---|---|---|---|
| 运行时 feature flag（LaunchDarkly）| ✅ | ❌ | |
| 单沙箱最大并发连接数 flag | ✅ | ❌ | |
| 内存回收操作超时 flag | ✅ | ❌ | |
| 出站代理开关 flag | ✅ | ❌ | |
| 持久卷开关 flag | ✅ | ❌ | |

---

## 特性统计汇总

| 功能域 | e2b 特性数 | kuasar-sandbox 特性数 | kuasar-sandbox 缺失 |
|---|---|---|---|
| 沙箱生命周期 | 21 | 19 | 2（优雅停机、在线调整规格）|
| 运行时数据面 | 21 | 16 | 5（持久卷全部、Hyperloop）|
| 模板与构建 | 18 | 14 | 4（版本/标签/alias/层缓存）|
| 内存管理 | 10 | 10 | 0（kuasar-sandbox 有额外 2 项）|
| 块存储与缓存 | 7 | 9 | 0（kuasar-sandbox 有额外 4 项）|
| 网络控制 | 18 | 7 | 11（出站策略全部、入站控制）|
| 代理层 | 11 | 12 | 1（友好错误页）|
| 安全与鉴权 | 14 | 11 | 6（OAuth/Secure/CA/封禁/静加密）|
| 可观测性 | 12 | 3 | 9（指标/追踪/日志 API/审计持久化）|
| 资源管控 | 9 | 9 | 3（quota/限流，kuasar-sandbox 有额外 6 项）|
| 运营与计费 | 6 | 2 | 4（计量/Webhook/统计）|
| 集群与多节点 | 7 | 7 | 1（全局列表）|
| 桌面 Computer Use | 7 | 0 | 7（全部）|
| Feature Flag | 5 | 0 | 5（全部）|
| **合计** | **166** | **119** | **~53**（kuasar-sandbox 有 e2b 无的约 12 项）|
