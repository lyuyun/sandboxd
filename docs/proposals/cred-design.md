# proxy 应用凭证管理 — 设计方案

**状态**：草稿 · **版本**：v0.6 · **数据库**：SQLite (modernc.org/sqlite) · **涉及组件**：conductor · proxy · nacre-agent

---

## 目录

1. [背景与目标](#1-背景与目标)
2. [系统角色](#2-系统角色)
3. [整体架构](#3-整体架构)
4. [数据模型](#4-数据模型)
5. [DB Schema — sandbox_credentials](#5-db-schema--sandbox_credentials)
6. [Store 方法](#6-store-方法)
7. [注入路径一：conductor → proxy（routesync 协议扩展）](#7-注入路径一conductor--proxy)
8. [注入路径二：nacre-agent → conductor API](#8-注入路径二nacre-agent--conductor-api)
9. [credshm 存储](#9-credshm-存储)
10. [凭证投递：MMDS 扩展](#10-凭证投递mmds-扩展)
11. [生命周期管理](#11-生命周期管理)
12. [proxy 重启恢复](#12-proxy-重启恢复)
13. [安全设计](#13-安全设计)
14. [错误处理](#14-错误处理)
15. [实现路径](#15-实现路径)

---

## 1 背景与目标

sandbox 内的应用在运行时需要访问外部服务的凭证（数据库密码、API Key、OAuth Token 等）。当前这些凭证在启动前静态注入，无法热更新，扩大了泄露面，且缺乏生命周期管理。

**目标**：在 conductor（持久化 + 分发）和 proxy（投递）中引入应用凭证管理层，使凭证可在应用运行期间动态更新，并通过 MMDS 按需投递给 sandbox 内的应用。

### 设计约束

- conductor 是凭证唯一持久化权威；proxy restart 后凭证可从 conductor 完整恢复
- 写入不阻塞 sandbox 启动关键路径
- worker 凭证查询无 IPC 开销（credshm 共享内存读）
- 凭证全量替换，无版本冲突问题

### 不在本期范围

- credshm at-rest 加密
- 凭证访问审计日志
- 跨节点凭证同步
- 凭证到期自动轮换触发

---

## 2 系统角色

| 角色 | 类型 | 职责 |
|------|------|------|
| `nacre-agent` | 凭证生产者 A | 触发沙箱创建并携带初始凭证；运行期按需全量 PUT 更新凭证；均通过 conductor API 写入 |
| `conductor` | 凭证生产者 B / 持久化权威 | 持久化所有凭证到 SQLite；向 proxy 分发（routesync）；proxy 重启后全量 PUT 恢复 |
| `proxy master` | 凭证仲裁者 | 接收 routesync TypeCredPut 写入 credshm；sandbox 删除时联动清理 |
| `proxy workers` | 凭证查询层 | mmap 只读 credshm；通过 MMDS 端点将全量凭证列表投递给 VM 内应用 |
| `sandbox app` | 凭证消费者 | 向 MMDS `GET /credentials` 获取全量凭证列表，以 session token 认证 |

> **设计决策：conductor 作为唯一持久化权威。** nacre-agent 通过 conductor API（而非直连 proxy）写入凭证。这使 conductor 始终持有完整的凭证快照，proxy 重启后可无损恢复，且与 nacre-agent 已调用 conductor 触发沙箱创建的现有模式一致。

---

## 3 整体架构

### 流程 A：沙箱创建（初始凭证写入）

```
nacre-agent                        conductor                      proxy master
     │                                  │                               │
     │  POST /v1/sandboxes              │                               │
     │  {credentials:[...]}             │                               │
     │ ────────────────────────────────►│                               │
     │                                  │ PutCredentials()              │
     │                                  │ (INSERT OR REPLACE)           │
     │                                  │                               │
     │◄──── 201 {sandboxID} ───────────│                               │
     │                                  │                               │
     │                            [VM 启动中...]                         │
     │                                  │                               │
     │                                  │──── routesync Range ─────────►│
     │                                  │  TypeCredPut(sid, entries)    │ ← 先写凭证
     │                                  │  TypeRouteUpsert(sid, Running)│ ← 再激活路由
     │                                  │                               │ credshm 写入完成
     │                                  │                               │
     │                [app 向 MMDS 查询凭证]                             │
     │                GET /credentials ──────────────────────────────►  │
     │◄─── 200 [{key,value,version,...}]  ────────────────────────────  │
```

> **时序保证**：`TypeCredPut` 与 `TypeRouteUpsert` 共享同一 h2c 有序流，传输层保证先后顺序。worker 拿到 Running 路由时，对应凭证必然已在 credshm 中。

### 流程 B：运行期热更新（全量 PUT）

```
nacre-agent                        conductor                      proxy master
     │                                  │                               │
     │  PUT /v1/sandboxes/{sid}         │                               │
     │      /credentials                │                               │
     │  [{key,value,version}...]        │                               │
     │ ────────────────────────────────►│                               │
     │                                  │ PutCredentials()              │
     │                                  │ (INSERT OR REPLACE)           │
     │                                  │                               │
     │◄──── 200 OK ────────────────────│                               │
     │                                  │──── TypeCredPut(sid,entries) ►│ credshm 全量替换
     │                                  │                               │
```

> **全量替换**：与 infra 的 MMDS PUT 策略一致——凭证字段少，全量替换比维护增量 delta 更简单，无版本冲突问题，也不需要 merge/delete 逻辑。

---

## 4 数据模型

### SandboxCredential（逻辑视图）

```go
// internal/types/types.go（建议新增）
// SandboxCredential 是一条凭证记录的解密后逻辑视图，
// Store 层返回此类型；routesync 和 credshm 均以此为数据源。
type SandboxCredential struct {
    SandboxID   string
    Key         string // 层级标识符，如 "db/primary"、"api/github"
    Value       string // 明文；Store 层负责加解密
    Version     int64  // 生产者单调递增；供 MMDS 轮询感知变更
    Source      string // "conductor" | "nacre-agent"，审计用
    ExpiresUnix int64  // 0 = 不过期；与 manifest_keys.expires_unix 语义一致
    CreatedUnix int64
    UpdatedUnix int64
}
```

### CredentialEntry（协议 / credshm 共用）

```go
// routesync/proto.go
type CredentialEntry struct {
    SandboxID string `json:"sid"`
    Key       string `json:"key"`
    Value     string `json:"value"`
    Version   uint64 `json:"version"`
    Source    string `json:"source"`      // "conductor" | "nacre-agent"
    ExpiresAt int64  `json:"expires_at"` // 0 = 不过期
}

// CredPut 是 conductor 下发给 proxy 的单个 sandbox 全量凭证集（Range 阶段和热更新均使用）。
type CredPut struct {
    SandboxID string            `json:"sid"`
    Entries   []CredentialEntry `json:"entries"`
}
```

### 字段约束

| 字段 | credshm 上限 | 说明 |
|------|-------------|------|
| `SandboxID` | 128 B | 与路由表 SandboxID 一致 |
| `Key` | 128 B | 用 `/` 分层，如 `tls/ingress-cert` |
| `Value` | 2048 B | 原始字节 base64；超限须引用外部存储（本期不实现） |
| `Version` | uint64 | 生产者赋值；供 MMDS 轮询感知变更；推荐用 Unix 毫秒时间戳 |
| `ExpiresAt` | int64 | Unix 秒；worker 在 `ListCredentials` 时惰性过滤已过期条目 |

---

## 5 DB Schema — sandbox_credentials

表追加到 `internal/store/store.go` 的 `const schema` 字符串末尾，与现有表风格完全一致：`_unix` 后缀表示 Unix 秒时间戳，`_enc` 后缀表示经 `secretbox.Box` 加密的密文字段，默认值覆盖所有可选列。

与 Firecracker MMDS 的内存模型同构：一个 sandbox 对应一行，凭证集合序列化为 JSON 后整体加密存储。PUT 是单条 `INSERT OR REPLACE`，无需事务。

```sql
-- internal/store/store.go — 追加到 const schema
CREATE TABLE IF NOT EXISTS sandbox_credentials (
  sandbox_id   TEXT    NOT NULL PRIMARY KEY,
  -- JSON 数组整体用 secretbox.Box 加密；结构见 CredentialEntry
  blob_enc     TEXT    NOT NULL,
  updated_unix INTEGER NOT NULL
);
```

### 设计说明

| 决策点 | 选择 | 理由 |
|--------|------|------|
| 主键 | `sandbox_id` 单列 | 一个 sandbox 一行，与 FC MMDS "一个 VM 一个 JSON 对象"模型同构 |
| 存储格式 | JSON 数组整体加密（`blob_enc`） | 全量替换时序列化一次、加密一次，无需逐 key 加解密；与 FC MMDS `MmdsContentsObject = any` 一致 |
| 写入模式 | `INSERT OR REPLACE` | 单条语句原子替换，无需显式事务、无需 DELETE |
| 无外键 | sandbox_id 不做 FK 约束 | 与现有表约定一致；sandbox 删除由 `DeleteCredentials` 显式清理 |

### 写入模式

```sql
-- PutCredentials 核心 SQL（blob_enc 是 JSON 数组整体加密后的密文）
INSERT OR REPLACE INTO sandbox_credentials (sandbox_id, blob_enc, updated_unix)
VALUES (?, ?, ?);
```

---

## 6 Store 方法

所有方法挂载在现有 `*Store` 上，遵循现有命名约定（动词 + 名词，无前缀）。加解密复用 `s.box`。

```go
// internal/store/store.go — sandbox_credentials CRUD

// PutCredentials 将凭证集序列化为 JSON 整体加密后 INSERT OR REPLACE。
// 用于沙箱创建和热更新；空切片合法，表示清空该 sandbox 的凭证。
func (s *Store) PutCredentials(ctx context.Context, sid string, creds []types.SandboxCredential) error {
    blob, err := json.Marshal(creds)
    if err != nil {
        return fmt.Errorf("store: put credentials %s: marshal: %w", sid, err)
    }
    enc, err := s.box.EncryptString(string(blob))
    if err != nil {
        return fmt.Errorf("store: put credentials %s: encrypt: %w", sid, err)
    }
    _, err = s.db.ExecContext(ctx,
        `INSERT OR REPLACE INTO sandbox_credentials (sandbox_id, blob_enc, updated_unix)
         VALUES (?, ?, ?)`,
        sid, enc, time.Now().Unix())
    return err
}

// GetCredentials 返回 sandbox 的完整凭证集（供 routesync Range 阶段和调试使用）。
func (s *Store) GetCredentials(ctx context.Context, sid string) ([]types.SandboxCredential, error) {
    row := s.db.QueryRowContext(ctx,
        `SELECT blob_enc FROM sandbox_credentials WHERE sandbox_id=?`, sid)
    var enc string
    if err := row.Scan(&enc); err == sql.ErrNoRows {
        return nil, nil
    } else if err != nil {
        return nil, fmt.Errorf("store: get credentials %s: %w", sid, err)
    }
    blob, err := s.box.DecryptString(enc)
    if err != nil {
        return nil, fmt.Errorf("store: get credentials %s: decrypt: %w", sid, err)
    }
    var creds []types.SandboxCredential
    if err := json.Unmarshal([]byte(blob), &creds); err != nil {
        return nil, fmt.Errorf("store: get credentials %s: unmarshal: %w", sid, err)
    }
    return creds, nil
}

// DeleteCredentials 删除 sandbox 的凭证行（sandbox 删除时联动调用）。
func (s *Store) DeleteCredentials(ctx context.Context, sid string) error {
    _, err := s.db.ExecContext(ctx,
        `DELETE FROM sandbox_credentials WHERE sandbox_id=?`, sid)
    return err
}
```

---

## 7 注入路径一：conductor → proxy

### 新增消息类型

```go
// routesync/proto.go — 追加常量与字段
const (
    TypeCredPut = "cred_put" // 一个 sandbox 的全量凭证集（Range 阶段和热更新均使用）
)

// Msg 追加字段（JSON omitempty，旧 subscriber 收到未知 type 时忽略）
type Msg struct {
    // ...原有字段不变...
    CredPut *CredPut `json:"cred_put,omitempty"`
}

// Event 追加凭证事件类型（供 Source.Subscribe channel 广播）
type Event struct {
    Kind    string     // TypeUpsert|TypeDelete|TypeCredPut
    Route   RouteEntry // TypeUpsert
    SID     string     // TypeDelete
    CredPut *CredPut   // TypeCredPut
}
```

### CredSink / CredSource 接口

```go
// routesync/subscribe.go
// CredSink 是 Sink 的可选凭证扩展，通过类型断言接入。
type CredSink interface {
    Sink
    ApplyCredPut(sid string, entries []CredentialEntry)
}

// routesync/serve.go
// CredSource 是 Source 的可选凭证扩展，供 StreamAuthority 在 Range 阶段
// 在每条 TypeUpsert 前交错发送 TypeCredPut。
type CredSource interface {
    GetCredentials(ctx context.Context, sid string) (CredPut, error)
}
```

### StreamAuthority Range 阶段修改

```go
// routesync/serve.go — Range 循环内插入 CredPut
credSrc, hasCredSrc := src.(CredSource)

if err := src.Range(sctx, func(r RouteEntry) error {
    // 先发凭证，再发路由（保证 worker 拿到 Running 路由前凭证已在 credshm）
    if hasCredSrc {
        cp, err := credSrc.GetCredentials(sctx, r.SandboxID)
        if err != nil {
            return err
        }
        if len(cp.Entries) > 0 {
            if err := WriteMsg(w, &Msg{Type: TypeCredPut, CredPut: &cp}); err != nil {
                return err
            }
        }
    }
    return WriteMsg(w, &Msg{Type: TypeUpsert, Route: &r})
}); err != nil {
    return
}
```

### 事件发送

```go
// routesync/serve.go — writeEvent 追加凭证分支
func writeEvent(w io.Writer, ev Event) error {
    m := &Msg{Type: ev.Kind}
    switch ev.Kind {
    case TypeUpsert:
        r := ev.Route; m.Route = &r
    case TypeDelete:
        m.SID = ev.SID
    case TypeCredPut:
        m.CredPut = ev.CredPut
    default:
        return nil
    }
    return WriteMsg(w, m)
}
```

### orch.Orchestrator 实现 CredSource

```go
// orch/orch.go — 实现 CredSource 接口，供 routesync Range 阶段使用
func (o *Orchestrator) GetCredentials(ctx context.Context, sid string) (routesync.CredPut, error) {
    creds, err := o.st.GetCredentials(ctx, sid)
    if err != nil {
        return routesync.CredPut{}, err
    }
    cp := routesync.CredPut{SandboxID: sid}
    for _, c := range creds {
        cp.Entries = append(cp.Entries, routesync.CredentialEntry{
            SandboxID: c.SandboxID, Key: c.Key, Value: c.Value,
            Version: uint64(c.Version), Source: c.Source, ExpiresAt: c.ExpiresUnix,
        })
    }
    return cp, nil
}
```

---

## 8 注入路径二：nacre-agent → conductor API

nacre-agent 已通过 `POST /v1/sandboxes` 调用 conductor。凭证 PATCH 复用同一 API 面，认证机制与现有 `ownsSandbox` 完全一致。

### API 规范

```
// 创建沙箱时携带初始凭证（现有端点扩展请求体）
POST /v1/sandboxes
Authorization: Bearer <api-key>
{
  "template_id": "...",
  "credentials": [
    {"key": "db/primary", "value": "postgres://...", "version": 1, "expires_at": 0},
    {"key": "api/github", "value": "ghp_xxx",        "version": 1, "expires_at": 0}
  ]
}

// 全量替换凭证（运行期热更新，完整替换该 sandbox 的所有凭证）
PUT /v1/sandboxes/{sid}/credentials
Authorization: Bearer <api-key>
[
  {"key": "db/primary", "value": "postgres://new...", "version": 42, "expires_at": 0},
  {"key": "tls/cert",   "value": "-----BEGIN...",     "version": 3,  "expires_at": 0}
]

→ 200 OK
→ 401 / 404

// 列出凭证（调试 / 核对，不返回 value）
GET /v1/sandboxes/{sid}/credentials
Authorization: Bearer <api-key>

→ 200 OK: [{"key":"db/primary","version":42,"expires_at":0},{"key":"tls/cert","version":3,"expires_at":0}]
→ 401 / 404
```

### PUT 处理流程

1. `ownsSandbox(sb, apiKey)` — 验证 api_key 的 HMAC MAC 与 sandbox 的 manifest_key 匹配
2. `store.PutCredentials(ctx, sid, creds)` — marshal + encrypt + `INSERT OR REPLACE`，原子替换
3. 直接用内存中已有的 `creds` 构造 `TypeCredPut` 发入 `routeEvents` channel，proxy 收到后全量替换该 sandbox 的 credshm 条目（无需回读 DB）

> **端到端延迟估算：** nacre-agent 发起 PUT → conductor 写 DB（SQLite WAL，<5ms）→ TypeCredPut 进 channel → routesync 写帧 flush → proxy master credshm 全量替换 → worker 下次查询立即可见。

### handler 骨架

```go
// orch/cred.go

type CredentialRequest struct {
    Key       string `json:"key"`
    Value     string `json:"value"`
    Version   int64  `json:"version"`
    ExpiresAt int64  `json:"expires_at"`
}

// HandlePutCredentials 全量替换 sandbox 的凭证集。
func (o *Orchestrator) HandlePutCredentials(
    ctx context.Context, apiKey, sid string, reqs []CredentialRequest,
) error {
    sb, err := o.st.Get(ctx, sid)
    if err != nil {
        return err
    }
    if !ownsSandbox(sb, apiKey) {
        return ErrNotFound
    }
    creds := make([]types.SandboxCredential, 0, len(reqs))
    for _, r := range reqs {
        creds = append(creds, types.SandboxCredential{
            SandboxID: sid, Key: r.Key, Value: r.Value,
            Version: r.Version, Source: "nacre-agent", ExpiresUnix: r.ExpiresAt,
        })
    }
    if err := o.st.PutCredentials(ctx, sid, creds); err != nil {
        return err
    }
    // 直接使用内存中的 creds，不回读 DB
    cp := routesync.CredPut{SandboxID: sid}
    for _, c := range creds {
        cp.Entries = append(cp.Entries, routesync.CredentialEntry{
            SandboxID: c.SandboxID, Key: c.Key, Value: c.Value,
            Version: uint64(c.Version), Source: c.Source, ExpiresAt: c.ExpiresUnix,
        })
    }
    o.routeEvents <- routesync.Event{Kind: routesync.TypeCredPut, CredPut: &cp}
    return nil
}
```

---

## 9 credshm 存储

新增 `internal/credshm` 包，设计模式与 `internal/proxyshm` 完全一致：文件背景 mmap、master 独占写、workers 只读映射、per-record seqlock 无锁读。

### 内存布局

```go
// credshm/cred.go
const (
    credMagic      uint64 = 0x6B755043524544 // "kuPCRED"
    credSchema     uint32 = 1
    defaultCredCap        = 8192 // ~20 MB；约支持 1000 沙箱 × 8 条凭证
    maxCredSID            = 128
    maxCredKey            = 128
    maxCredValue          = 2048
    maxCredSource         = 16
)

type credHeader struct {
    Magic     uint64
    Schema    uint32
    Capacity  uint32
    SyncGen   uint64 // 与 routesync BeginSync/Bookmark 同步代
    GlobalRev uint64 // 每次写操作递增；worker 用于变更感知
    _         [48]byte
}

// credRecord ~2.4 KB；8192 条 ≈ 20 MB mmap
type credRecord struct {
    Seq       uint64            // seqlock（奇=写入中，偶=稳定）
    Status    uint32            // 0=empty 1=present 2=deleted
    _         uint32
    SyncGen   uint64            // 最后一次写入时的 SyncGen
    Version   uint64            // 凭证版本，供 MMDS 轮询感知变更
    ExpiresAt int64             // Unix 秒；0=不过期
    Rev       uint64            // 写入时的 GlobalRev
    SandboxID [maxCredSID]byte
    Key       [maxCredKey]byte
    Source    [maxCredSource]byte
    Value     [maxCredValue]byte
}
```

### Bookmark（统一清理）

由于所有凭证均通过 routesync 注入，所有记录都参与 SyncGen 管理，Bookmark 无需区分 source：

```go
// credshm/cred.go — Bookmark()
func (t *CredTable) Bookmark() {
    gen := atomic.LoadUint64(&t.header.SyncGen)
    for i := range t.records {
        r := &t.records[i]
        if atomic.LoadUint32(&r.Status) == statusPresent &&
           atomic.LoadUint64(&r.SyncGen) != gen {
            t.deleteRecord(r) // stale：断连期间被删除的 sandbox 或其凭证
        }
    }
    atomic.AddUint64(&t.header.GlobalRev, 1)
}
```

> **与第一版的差异：** 第一版 nacre-agent 直连 proxy，需保护 `source=nacre-agent` 条目不被 Bookmark 清除。现在 nacre-agent 经 conductor，所有条目均受 SyncGen 保护，Bookmark 逻辑大幅简化。

### 查找索引

```go
// credshm/cred.go — 双字段哈希线性探测
func hashCred(sid, key string) uint64 {
    h := fnv.New64a()
    _, _ = h.Write([]byte(sid))
    _, _ = h.Write([]byte{0})
    _, _ = h.Write([]byte(key))
    return h.Sum64()
}
```

### Upsert（version-guarded 写）

```go
// credshm/cred.go — Upsert()
func (t *CredTable) Upsert(e routesync.CredentialEntry) error {
    if t.readonly {
        return errors.New("credshm: read-only")
    }
    idx, ok := t.findSlot(e.SandboxID, e.Key, true)
    if !ok {
        return errors.New("credshm: table full")
    }
    rec := &t.records[idx]
    if atomic.LoadUint32(&rec.Status) == statusPresent {
        if atomic.LoadUint64(&rec.Version) >= e.Version {
            return nil // 版本不更新，静默跳过
        }
    }
    startWrite(rec)
    rec.Status    = statusPresent
    rec.SyncGen   = atomic.LoadUint64(&t.header.SyncGen)
    rec.Version   = e.Version
    rec.ExpiresAt = e.ExpiresAt
    rec.Rev       = atomic.AddUint64(&t.header.GlobalRev, 1)
    _ = putFixed(rec.SandboxID[:], e.SandboxID)
    _ = putFixed(rec.Key[:], e.Key)
    _ = putFixed(rec.Source[:], e.Source)
    _ = putFixed(rec.Value[:], e.Value)
    finishWrite(rec)
    return nil
}
```

### MasterView 实现 CredSink

```go
// proxyshm/view.go — MasterView 追加方法
// ApplyCredPut 全量替换 sandbox 的 credshm 条目：先删旧条目，再写入新条目。
func (v *MasterView) ApplyCredPut(sid string, entries []routesync.CredentialEntry) {
    v.credTable.DeleteBySandbox(sid)
    for _, e := range entries {
        if err := v.credTable.Upsert(e); err != nil {
            v.log.Warn("credshm: apply cred_put", "sid", sid, "key", e.Key, "err", err)
        }
    }
    v.notify.Notify()
}

// ApplyDelete 钩子：sandbox 路由删除时联动清理所有凭证
func (v *MasterView) ApplyDelete(sid string) {
    v.table.Delete(sid)
    v.credTable.DeleteBySandbox(sid)
    v.notify.Notify()
}
```

---

## 10 凭证投递：MMDS 扩展

### 新增端点

```go
// mmds/mmds.go — Handler() 追加路由
mux.HandleFunc("GET /latest/meta-data/credentials", s.listCreds)
```

端点要求 `X-metadata-token`（现有 MMDS session token），复用 `s.verifyToken` 解出 `sid`，无需新的认证逻辑。

### Source 接口扩展

```go
// mmds/mmds.go
type Source interface {
    ByFloatingIP(ip string) (sandboxID string, ok bool)
    SandboxInfo(sandboxID string) (templateID, accessToken string, ok bool)
    MmdsSecret(sandboxID string) (secret []byte, ok bool)
    // 新增
    ListCredentials(sid string) []CredEntry // 含 Value，过滤已过期
}

type CredEntry struct {
    Key       string `json:"key"`
    Value     string `json:"value"`
    Version   uint64 `json:"version"`
    ExpiresAt int64  `json:"expires_at"`
}
```

### 应用轮询模式

```
App (VM)                         MMDS (proxy worker)
  │
  │  1. 首次获取 session token（现有流程，不变）
  │  PUT /latest/api/token  ──────────────────────►
  │◄─── X-metadata-token: <token>  ───────────────
  │
  │  2. 轮询全量凭证（如每 30s）
  │  GET /latest/meta-data/credentials
  │  X-metadata-token: <token>  ──────────────────►
  │◄─── 200 [{"key":"db/primary","value":"...","version":7}, ...]
  │
  │  3. 凭证为空或 sandbox 不存在
  │◄─── 200 []
```

---

## 11 生命周期管理

| 事件 | 触发方 | conductor DB | credshm |
|------|--------|-------------|---------|
| **沙箱创建** | nacre-agent POST /v1/sandboxes（含 credentials 数组） | `PutCredentials` — INSERT OR REPLACE | routesync `TypeCredPut` → `ApplyCredPut` |
| **热更新** | nacre-agent PUT /v1/sandboxes/{sid}/credentials（全量替换） | `PutCredentials` — INSERT OR REPLACE | `TypeCredPut` → `ApplyCredPut`（先 DeleteBySandbox 再逐条 Upsert） |
| **凭证过期** | 惰性过滤（worker 读取时检查 ExpiresAt） | 保留（不主动删除） | 保留；`ListCredentials` 返回时过滤掉已过期条目 |
| **沙箱删除** | conductor 删除 sandbox 路由时联动 | `DeleteCredentials(sid)` — 清理该 sandbox 全部凭证 | `ApplyDelete(sid)` 钩子 → `credTable.DeleteBySandbox(sid)` |

> producer 推荐使用 Unix 毫秒时间戳（`time.Now().UnixMilli()`）保证 version 单调递增，供 app 轮询 MMDS 感知变更。

---

## 12 proxy 重启恢复

conductor 持有所有凭证（含 nacre-agent 写入的历史 PUT），proxy 重启后的恢复与路由表完全共轨，**无需独立机制**。

```
proxy master 重启
     │
     ├─ credshm 文件重建（Create() → 清空所有记录）
     │
     └─ Subscriber.Run() 重连 routesync（指数退避 200ms→5s）
            │
            ├─ session() 调用 sink.BeginSync() → credTable SyncGen++
            │
            ├─ conductor ServeAuthority → Range():
            │    for each running sandbox (ORDER BY id ASC):
            │      ① GetCredentials(sid) → TypeCredPut(sid, entries)  ← 先（单行解密）
            │      ② TypeUpsert(sid, StateRunning)                     ← 后
            │
            └─ Bookmark() → 清理 SyncGen 不匹配的 stale 凭证
                 GlobalRev++，worker 开始正常服务凭证查询
```

> conductor 侧每个 sandbox 只需一次 `SELECT blob_enc` + 解密，比逐行查询更简单。

### 断连期间行为

proxy 重连过程中，credshm 已有旧条目（上次 Bookmark 留下的值），worker 继续用旧凭证服务请求。Bookmark 完成后 stale 条目被清理，`GlobalRev` 递增，worker 下次查询感知到变更。

---

## 13 安全设计

| 威胁 | 缓解措施 |
|------|---------|
| 冒充 nacre-agent 写入伪造凭证 | conductor 校验 `ownsSandbox(sb, apiKey)`：API key 的 HMAC-SHA256 MAC 须由该 sandbox 的 manifest key 验证通过 |
| sandbox A 读取 sandbox B 的凭证 | MMDS session token 绑定具体 sid（`MmdsSecret(sid)` 签名），`verifyToken` 解出 sid 后仅查该 sid 的凭证 |
| 凭证明文落盘 | `blob_enc` 是 JSON 数组整体用 AES-256-GCM（`secretbox.Box`）加密，与 manifest_key_enc 同一密钥；conductor 进程内才有明文 |
| credshm 被其他进程读取 | 文件权限 `0600`，所有者为 proxy 运行用户；与 proxyshm 安全模型一致 |
| routesync 传输中泄露 | conductor ↔ proxy 走 Unix Domain Socket（仅本机可达） |
| 非所有者写入伪造凭证 | conductor `PutCredentials` 要求调用方持有有效 `api_key`（`ownsSandbox`），非所有者无法写入 |
| 过期凭证被使用 | worker `ListCredentials` 过滤 `ExpiresAt`，过期条目不出现在响应中 |

---

## 14 错误处理

| 场景 | 行为 | 恢复 |
|------|------|------|
| credshm 容量满（>8192） | `Upsert` 返回错误；master 记 warn 日志；条目丢弃 | 调大配置项 `cred_table_capacity` 后重启 proxy |
| routesync `TypeCredPut` 帧过大（>1 MiB） | `ReadMsg` 报错，session 结束，subscriber 重连 | 减少单次凭证条数或压缩 value |
| SQLite BUSY（WAL checkpoint） | `busy_timeout=5000ms` 自动等待；超时返回错误 | 调用方重试；凭证写入非关键路径，可延迟 |
| credshm seqlock 自旋超限（64次） | `readRecord` 返回 empty；worker 返回 404 | 应用重试 MMDS GET；正常负载下不应发生 |
| credshm mmap 写失败（tmpfs 满） | proxy master 打印错误并退出；systemd 重启 | 清理 `/run/sandbox/` 下陈旧文件 |

---

## 15 实现路径

| 阶段 | 任务 | 文件 | 优先级 |
|------|------|------|--------|
| **P0 DB 层** | 追加 `sandbox_credentials` 表 DDL 到 `const schema` | `store/store.go` | 必须 |
| | 实现 `PutCredentials / GetCredentials / DeleteCredentials` | `store/store.go` | 必须 |
| **P0 types** | 新增 `SandboxCredential` 结构体 | `types/types.go` | 必须 |
| | 新增 `CredentialEntry / CredPut` 及 `TypeCredPut` 常量 | `routesync/proto.go` | 必须 |
| **P0 credshm** | 实现 `internal/credshm` 包（CredTable、seqlock、Upsert、Delete、Bookmark） | `credshm/cred.go` | 必须 |
| | `MasterView` 持有 CredTable，实现 `ApplyCredPut`；`ApplyDelete` 联动清理 | `proxyshm/view.go` | 必须 |
| | `WorkerView` 持有 CredTable 只读句柄，实现 `ListCredentials` | `proxyshm/view.go` | 必须 |
| **P0 routesync** | `CredSink / CredSource` 接口定义；`subscriber.apply()` 处理 `TypeCredPut` | `routesync/subscribe.go` | 必须 |
| | `Event` 追加 `CredPut` 字段；`writeEvent` 处理 `TypeCredPut` | `routesync/serve.go` | 必须 |
| | `StreamAuthority Range` 阶段插入 CredPut（先凭证后路由） | `routesync/serve.go` | 必须 |
| **P0 conductor** | `Orchestrator` 实现 `CredSource.GetCredentials` | `orch/orch.go` | 必须 |
| | `HandlePutCredentials` / `HandleListCredentials` HTTP handler | `orch/cred.go` | 必须 |
| | sandbox 创建请求体扩展 `credentials` 数组；`lc.Start()` 前调用 `PutCredentials` | `orch/orch.go` | 必须 |
| **P0 MMDS** | `Source` 追加 `ListCredentials`；实现 `listCreds` handler | `mmds/mmds.go` | 必须 |
| | proxy master 初始化时创建 credshm 文件，并传入 `MasterView` | `cmd/node-ctl/proxy.go` | 必须 |
| **P1 测试** | credshm 并发 fuzz；routesync CredPut/Bookmark 集成测试；MMDS handler 单测；store PutCredentials 幂等单测 | `*_test.go` | 推荐 |
| **P2 加固** | 过期条目后台清理 goroutine；credshm 容量监控指标 | `credshm/cred.go` | 可选 |
