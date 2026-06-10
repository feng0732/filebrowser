# Filebrowser 分享链接与公开访问代码实现分析

## 一、整体架构概览

分享功能的代码分布在以下关键模块：

| 层级 | 模块 | 职责 |
|------|------|------|
| 数据模型 | [share/share.go](share/share.go) | Link 结构体定义 |
| 存储层 | [share/storage.go](share/storage.go) + [storage/bolt/share.go](storage/bolt/share.go) | 存储接口与 BoltDB 实现（含过期清理） |
| HTTP API | [http/share.go](http/share.go) | 分享管理（创建/删除/列表） |
| 公开访问 | [http/public.go](http/public.go) | 公开分享访问（带密码校验 + 沙箱隔离） |
| 路由注册 | [http/http.go](http/http.go#L74-L91) | API 路由挂载 |
| 文件沙箱 | [files/scoped.go](files/scoped.go) | ScopedFs 越界防护（防符号链接逃逸） |
| 权限模型 | [users/permissions.go](users/permissions.go) | Share + Download 权限位 |
| 规则检查 | [http/data.go](http/data.go#L37-L64) | checkerPrefix 规则前缀补偿 |
| 前端 API | [frontend/src/api/share.ts](frontend/src/api/share.ts) + [frontend/src/api/pub.ts](frontend/src/api/pub.ts) | 前端调用封装 |

---

## 二、分享链接生成

### 2.1 数据模型：Link 结构体

定义在 [share/share.go#L10-L20](share/share.go#L10-L20)：

```go
type Link struct {
    Hash         string `json:"hash" storm:"id,index"`      // 6字节随机 → URL-safe Base64（约8字符）
    Path         string `json:"path" storm:"index"`         // 指向的文件/目录绝对路径（用户scope内）
    UserID       uint   `json:"userID"`                     // 创建者用户ID
    Expire       int64  `json:"expire"`                     // Unix 时间戳；0 表示永不过期
    PasswordHash string `json:"password_hash,omitempty"`    // bcrypt 哈希；空表示无密码
    Token        string `json:"token,omitempty"`            // 96字节随机 → URL-safe Base64；仅当设密码时存在
}
```

**关键设计点：**
- **Hash** 是分享链接的唯一标识，直接作为 URL 路径段（如 `/share/MEEuZK-v/`）
- **Token** 是密码保护分享的"免密下载令牌"：知道密码 → 页面验证通过后 → 后端返回 token → 后续下载走 query 参数 `?token=xxx`，避免在 HTTP header 里反复带密码
- **Expire** 使用 `int64` Unix 时间戳，`0` 表示永久分享

### 2.2 创建流程：sharePostHandler

入口在 [http/share.go#L100-L180](http/share.go#L100-L180)，整个流程有严格的边界校验：

```
POST /api/share{path}  body: { password, expires, unit }
        │
        ▼
┌─────────────────────────────────────┐
│  1. withPermShare 权限前置检查       │
│     d.user.Perm.Share && Download   │
│     不满足 → 403 Forbidden          │
└──────────────────┬──────────────────┘
                   ▼
┌─────────────────────────────────────┐
│  2. d.user.Fs.Stat(r.URL.Path)      │
│     ✅ 路径必须当前真实存在           │
│     ✅ ScopedFs 拒绝跟随 scope 外    │
│        的符号链接（返回 permission   │
│        error → 403）                │
│     ❌ 防止为不存在路径创建分享       │
│        （以后有文件就自动暴露）       │
└──────────────────┬──────────────────┘
                   ▼
┌─────────────────────────────────────┐
│  3. 生成 Hash (6 字节随机)           │
│     rand.Read(bytes)                │
│     base64.URLEncoding → 约8字符    │
└──────────────────┬──────────────────┘
                   ▼
┌─────────────────────────────────────┐
│  4. 计算过期时间 Expire              │
│     body.Expires == "" → 0 (永久)   │
│     unit: seconds/minutes/hours(默认│
│          )/days                      │
│     expire = time.Now().Add(add).Unix()
└──────────────────┬──────────────────┘
                   ▼
┌─────────────────────────────────────┐
│  5. 密码处理                         │
│     password != "" →                │
│       bcrypt.GenerateFromPassword   │
│       生成 96 字节 Token             │
│     password == "" →                │
│       PasswordHash = ""             │
│       Token = ""                    │
└──────────────────┬──────────────────┘
                   ▼
┌─────────────────────────────────────┐
│  6. Save 到 BoltDB                   │
│     d.store.Share.Save(s)           │
└─────────────────────────────────────┘
```

### 2.3 前端入口

前端创建分享在 [frontend/src/components/prompts/Share.vue](frontend/src/components/prompts/Share.vue)，调用 [frontend/src/api/share.ts#L18-L41](frontend/src/api/share.ts#L18-L41) 的 `create()`。

注意：有密码的分享，**复制下载链接按钮是 disabled 的**（[Share.vue#L41-L45](frontend/src/components/prompts/Share.vue#L41-L45)），因为直接下载链接无法携带密码。

---

## 三、访问校验（公开访问边界控制）

### 3.1 路由总览

在 [http/http.go#L89-L91](http/http.go#L89-L91)：

```go
public := api.PathPrefix("/public").Subrouter()
public.PathPrefix("/dl").Handler(monkey(publicDlHandler, "/api/public/dl/")).Methods("GET")
public.PathPrefix("/share").Handler(monkey(publicShareHandler, "/api/public/share/")).Methods("GET")
```

两个端点：
- `/api/public/share/{hash}[/{path}]` → 元数据（目录 listing、文件信息）
- `/api/public/dl/{hash}[/{path}]` → 原始下载

两者都走 `withHashFile` 中间件（[http/public.go#L17-L98](http/public.go#L17-L98)）。

### 3.2 withHashFile：核心访问控制链

这是整个公开访问最关键的边界守卫，流程如下：

```
请求到达 withHashFile
        │
        ▼
┌─────────────────────────────────────┐
│  Step 1: 解析 hash + 子路径          │
│  ifPathWithName() 分割 URL           │
│  /api/public/dl/ABC/foo/bar.txt     │
│    → id = "ABC", filePath =         │
│      "/foo/bar.txt"                 │
│  (兼容老浏览器带文件名的格式)         │
└──────────────────┬──────────────────┘
                   ▼
┌─────────────────────────────────────┐
│  Step 2: GetByHash + 过期检查        │
│  d.store.Share.GetByHash(id)        │
│  内部检查 Expire <= Now → 删除 +     │
│  返回 ErrNotExist → 404             │
└──────────────────┬──────────────────┘
                   ▼
┌─────────────────────────────────────┐
│  Step 3: authenticateShareRequest   │
│  密码 / Token 校验                  │
│  详见 3.3                           │
└──────────────────┬──────────────────┘
                   ▼
┌─────────────────────────────────────┐
│  Step 4: 获取分享所有者 User          │
│  d.store.Users.Get(Root, UserID)    │
│  → 检查用户当前是否仍有              │
│    Share + Download 权限            │
│    任一缺失 → 403 Forbidden          │
│  ⚠️  即使链接还没过期，如果用户被     │
│      管理员撤权，分享立即失效         │
└──────────────────┬──────────────────┘
                   ▼
┌─────────────────────────────────────┐
│  Step 5: 初始文件信息检查             │
│  用 link.Path 做一次 NewFileInfo，   │
│  确认分享目标还存在且可访问           │
└──────────────────┬──────────────────┘
                   ▼
┌─────────────────────────────────────┐
│  Step 6: ★ 沙箱隔离 ★                │
│  d.user.Fs = files.NewScopedFs(     │
│      d.user.Fs, basePath)           │
│  将用户文件系统 **重新挂载** 到        │
│  分享的 basePath 根上                │
│  → 后续所有文件操作都被限制在         │
│    分享目录内（含符号链接防护）       │
│  详见 3.4                           │
└──────────────────┬──────────────────┘
                   ▼
┌─────────────────────────────────────┐
│  Step 7: ★ 规则前缀补偿 ★            │
│  d.checkerPrefix = basePath         │
│  因为 Fs 被重挂载，路径变成相对的，   │
│  Check(path) 时会自动拼回前缀，       │
│  确保分享所有者的 deny 规则仍然       │
│  作用于分享子目录                    │
│  详见 3.5                           │
└──────────────────┬──────────────────┘
                   ▼
┌─────────────────────────────────────┐
│  Step 8: 二次文件信息                │
│  在沙箱 Fs + checkerPrefix 下        │
│  重新构建 FileInfo（Expand=true）    │
└──────────────────┬──────────────────┘
                   ▼
         交给 publicShareHandler / publicDlHandler
```

### 3.3 密码校验：authenticateShareRequest

定义在 [http/public.go#L136-L161](http/public.go#L136-L161)：

```go
func authenticateShareRequest(r *http.Request, l *share.Link) (int, error) {
    // 情况 A：无密码 → 直接通过
    if l.PasswordHash == "" {
        return 0, nil
    }

    // 情况 B：带 token query 参数（免密下载令牌）
    // 使用 subtle.ConstantTimeCompare 防时序攻击
    if subtle.ConstantTimeCompare([]byte(r.URL.Query().Get("token")), []byte(l.Token)) == 1 {
        return 0, nil
    }

    // 情况 C：X-SHARE-PASSWORD header 密码
    password := r.Header.Get("X-SHARE-PASSWORD")
    password, _ = url.QueryUnescape(password)
    if password == "" {
        return http.StatusUnauthorized, nil  // 401 → 前端弹密码框
    }
    if err := bcrypt.CompareHashAndPassword(
        []byte(l.PasswordHash), []byte(password)); err != nil {
        if errors.Is(err, bcrypt.ErrMismatchedHashAndPassword) {
            return http.StatusUnauthorized, nil
        }
        return 0, err
    }
    return 0, nil
}
```

**三种认证方式的使用场景：**

| 方式 | 触发条件 | 使用方 |
|------|----------|--------|
| 无密码 | `PasswordHash == ""` | 公开分享的页面和下载 |
| Token query | `?token=xxx` | 前端页面通过密码校验后，后端把 token 写入 Resource，后续下载链接自动带 token |
| X-SHARE-PASSWORD header | 首次访问密码分享 | 前端 [frontend/src/api/pub.ts#L7-L13](frontend/src/api/pub.ts#L7-L13) 在 `fetch()` 时把用户输入的密码 encode 后放 header |

前端密码输入 UI 在 [frontend/src/views/Share.vue#L42-L73](frontend/src/views/Share.vue#L42-L73)：收到 401 后显示密码输入卡片。

### 3.4 ScopedFs：文件系统沙箱（防符号链接逃逸）

最核心的安全边界在 [files/scoped.go](files/scoped.go)。

**两层防护：**
1. **词法层**：内层 `afero.BasePathFs` 把所有路径拼到 base 目录下，防止 `../` 跳出
2. **符号链接层**：`ScopedFs.guard()` 在每个会 dereference symlink 的操作前调用 `within()`，通过 `filepath.EvalSymlinks` 解析磁盘上真实目标路径，确认在 scope 内

关键函数 `within()` [files/scoped.go#L64-L94](files/scoped.go#L64-L94)：

```
请求路径 p
    │
    ▼
EvalSymlinks(scope root)          → root (真实绝对路径)
EvalSymlinks(p 的完整磁盘路径)     → resolved
    │
    ▼
不存在的路径？→ 向上递归找最近存在的祖先再 Eval
    │
    ▼
resolved == root
  || strings.HasPrefix(resolved, root + "/")
    │
    ▼
在 scope 内？是 → OK    否 → os.ErrPermission (403)
```

**防护覆盖的操作**（每个方法开头都调用 `s.guard(name)`）：
- Create / Mkdir / MkdirAll / Open / OpenFile
- Rename（两端都检查）
- Stat / Chmod / Chown / Chtimes / LstatIfPossible

**注意**：Remove / RemoveAll **不做 guard**，因为分享访问是只读的（只经过 publicShareHandler / publicDlHandler，不走写 API）。

### 3.5 checkerPrefix：规则前缀补偿

在 [http/data.go#L37-L64](http/data.go#L37-L64)：

```go
func (d *data) Check(path string) bool {
    // ★ 关键：如果是分享访问，拼回 checkerPrefix
    if d.checkerPrefix != "" {
        path = gopath.Join(d.checkerPrefix, path)
    }
    // ... 然后用完整路径匹配 HideDotfiles、全局 Rules、用户 Rules
}
```

**为什么需要这个？**

假设：
- 用户 Alice 的 scope = `/home/alice`
- Alice 有一条 deny 规则：`/home/alice/projects/private`
- Alice 分享了 `/home/alice/projects`（hash = ABC）

没有 checkerPrefix 时，访问者请求 `ABC/private/secret.txt`：
1. Fs 被重挂载到 `/home/alice/projects`，所以传入 Check 的路径是 `/private/secret.txt`
2. 规则 `/home/alice/projects/private` 匹配不上 → 规则被绕过 ❌

有 checkerPrefix = `/home/alice/projects` 时：
1. Check 里拼回 → `/home/alice/projects/private/secret.txt`
2. deny 规则匹配 → 返回 false → 403 ✅

这个机制在 [http/public_test.go#L147-L255](http/public_test.go#L147-L255) 的 `TestPublicShareHandlerRules` 中有完整的测试覆盖。

### 3.6 所有者权限二次校验

即使分享链接本身有效，系统还会在 withHashFile Step 4 中重新校验所有者当前的权限：

```go
// [http/public.go#L35-L37]
if !user.Perm.Share || !user.Perm.Download {
    return http.StatusForbidden, nil
}
```

这意味着：管理员可以通过**撤销用户的 Share 或 Download 权限**来立即使该用户的所有分享失效，不需要逐个删除链接。

测试用例见 [http/public_test.go#L73-L86](http/public_test.go#L73-L86)。

---

## 四、过期处理机制

### 4.1 过期策略：Lazy（惰性删除）

Filebrowser **不使用后台定时任务**清理过期链接。而是采用 **"访问时检查 + 即时删除"** 的懒加载策略。

过期检查统一封装在 [share/storage.go](share/storage.go) 的 Storage 中间层（注意：不是 bolt 底层实现，而是在 Storage 包装层）：

| 方法 | 过期处理行为 |
|------|-------------|
| `All()` [L39-L46](share/storage.go#L39-L46) | 遍历中遇到过期 → 从 DB Delete → 从返回切片剔除 |
| `FindByUserID()` [L59-L66](share/storage.go#L59-L66) | 同上 |
| `GetByHash()` [L78-L83](share/storage.go#L78-L83) | 单条过期 → Delete + 返回 `ErrNotExist` (404) |
| `Gets()` [L101-L108](share/storage.go#L101-L108) | 遍历清理 |
| `GetsByPath()` [L114-L128](share/storage.go#L114-L128) | 先调 All()（已清理）再过滤 |

判断条件统一为：
```go
if link.Expire != 0 && link.Expire <= time.Now().Unix() { ... }
```
（`Expire == 0` 代表永久分享，永远不清理）

### 4.2 ⚠️ 批量列表场景下的边界问题：range 遍历 + 原地修改切片导致 DB 漏删 + 返回错乱 + panic

#### 问题描述

**`All()`、`FindByUserID()`、`Gets()` 三个批量方法存在「range 遍历 + append 原地修改切片」的严重 bug，包含 DB 漏删、返回切片错乱、runtime panic 三个层面的问题。**

当前实现 [share/storage.go#L39-L46](share/storage.go#L39-L46)：

```go
for i, link := range links {
    if link.Expire != 0 && link.Expire <= time.Now().Unix() {
        if err := s.Delete(link.Hash); err != nil {
            return nil, err
        }
        links = append(links[:i], links[i+1:]...)  // 原地修改切片
    }
}
```

#### 根本原因（Go 切片内存模型精确分析）

##### Go 切片的三字段结构

一个切片在内存中是一个 24 字节结构体（64 位），由 `Data`（指针）、`Len`（长度）、`Cap`（容量）组成：

```
links 变量: [ Data = &arr[0], Len = 4, Cap = 4 ]
                     │
                     ▼
底层数组 arr:     [ A ][ B ][ C ][ D ]
                   0    1    2    3
```

##### `for range slice` 的精确定义（Go 语言规范）

```go
for i, v := range links { ... }
```

等价于：

```go
// 第 0 步：在循环开始前，对 range 表达式求值 ONE TIME
_temp_range := links          // 复制切片头：3 个字段
_len := _temp_range.Len       // 固定迭代次数 = 初始 Len = 4

for i := 0; i < _len; i++ {
    v := *(*Link)(unsafe.Pointer(_temp_range.Data + uintptr(i)*unsafe.Sizeof(Link{})))
    // ↳ 从共享的底层数组偏移 i 处直接读！不是从 temp 副本读元素值！
    
    ... // 循环体，可修改 links 变量本身
}
```

**四个关键结论**：

| # | 机制 | 说明 | 后果 |
|---|------|------|------|
| 1 | **切片头副本** | `_temp_range` 复制了 `(Data, Len, Cap)` 三个字段 | `links.Len` 后续变化不影响迭代次数 |
| 2 | **底层数组共享** | `_temp_range.Data == links.Data`，指针指向同一块内存 | `append` 原地修改时，**后续迭代读到的元素值已被改写** |
| 3 | **`v` 从共享数组读** | `v = arr[i]` 是每次迭代时从当前底层数组**实时读取** | 删除操作会移动数组元素 → 后续 `v` 读到"跑过来"的新值 |
| 4 | **切片操作用新 `Len`** | `links[:i]` 边界检查用当前 `links.Len`（可能已缩短） | `i` 继续递增 → 边界越界 → **panic** |

---

#### 复现场景 A：连续过期（精确到每一行代码执行）

**准备**：`links = [A(过期), B(过期), C(正常), D(过期)]`，底层数组初始 Len=4, Cap=4

```
内存初始状态:
  links: [Data=&arr[0], Len=4, Cap=4]
  arr:   [ A ][ B ][ C ][ D ]
           0    1    2    3
          (过)  (过)  (正)  (过)
```

```go
// ===== 循环开始前 =====
_temp_range = links   // Data=&arr[0], Len=4, Cap=4
_len = 4              // 迭代次数固定为 4 次
```

---

##### ▶️ 第 1 次迭代：`i=0`

```go
// Step 1: 读 link
link = *(_temp_range.Data + 0*Size) = arr[0] = A
// link.Expire → 过期 ✅

// Step 2: DB 删除 (正确)
s.Delete(A.Hash)   // ✅ A 从 DB 删除

// Step 3: 切片操作 —— 关键点！
// links[:0] = [Data=&arr[0], Len=0, Cap=4] → 空切片，头在 arr[0]
// links[1:] = [Data=&arr[1], Len=3, Cap=3] → [B, C, D]
// append → 容量足够，将 [B,C,D] 拷贝到 arr[0..2]
links = append(links[:0], links[1:]...)
```

**内存写入后**：

```
arr:   [ B ][ C ][ D ][ D ]
         0    1    2    3   ← arr[3] 是旧值，未被覆盖
links: [Data=&arr[0], Len=3, Cap=4]   ← Len 从 4 变成 3
```

**本步总结**：DB 正确删 A，切片正确变 [B,C,D]，无异常

---

##### ▶️ 第 2 次迭代：`i=1`

```go
// Step 1: 读 link —— ⚠️ 从共享数组实时读！
link = *(_temp_range.Data + 1*Size) = arr[1] = C
//                       ↑ 不是 B！是 C！
// arr[1] 已经在上一步被 B→C 的拷贝覆盖了
// link.Expire → C 是正常的 ❌，不进入删除分支
```

**什么都没做！** B（原本应该被删）在 arr[0]，但 `i` 已经走到了 1，**B 永远不会被检查到了**。

**内存保持不变**：`arr=[B,C,D,D]`, `links=[B,C,D]`, `Len=3`

**本步总结**：
- ❌ **DB 漏删 B**（B 移到了 arr[0]，但 i=1 跳过了它）
- ❌ **C 被当作正常**（实际读到的是 arr[1]=C，但原 B 已跳过）

---

##### ▶️ 第 3 次迭代：`i=2`

```go
// Step 1: 读 link
link = *(_temp_range.Data + 2*Size) = arr[2] = D
// link.Expire → 过期 ✅

// Step 2: DB 删除
s.Delete(D.Hash)   // ✅ D 从 DB 删除

// Step 3: 切片操作
// 当前 links.Len = 3, i = 2
// links[:2] = [B, C]   (Len=2)
// links[3:] = [Data=&arr[3], Len=0, ...] → 空切片 (Len=3, start=3)
links = append(links[:2], links[3:]...)  // 追加空 → [B, C]
```

**内存写入后**：

```
arr:   [ B ][ C ][ D ][ D ]   (不变)
links: [Data=&arr[0], Len=2, Cap=4]   ← Len 从 3 变成 2
```

**本步总结**：DB 正确删 D，切片变 [B,C]，无异常

---

##### ▶️ 第 4 次迭代：`i=3`

```go
// Step 1: 读 link
link = *(_temp_range.Data + 3*Size) = arr[3] = D
// link.Expire → 过期 ✅ (D 的指针值还在，字段未变)

// Step 2: DB 删除
s.Delete(D.Hash)   // 没问题，Delete 是幂等的，不会报错

// Step 3: 切片操作 —— 💥 BOOM!
// 当前 links.Len = 2, i = 3
links[:3]   // ⚠️ 切片上界 3 > 当前 Len 2
            // → runtime panic: slice bounds out of range [:3] with length 2
```

---

##### 📊 场景 A 最终结果汇总

| 检查项 | 结果 | 说明 |
|--------|------|------|
| DB 中 A | ✅ 已删 | 正确 |
| DB 中 B | ❌ **残留** | 移到 arr[0] 后被跳过，**DB 漏删** |
| DB 中 C | ✅ 保留 | 正确（C 未过期） |
| DB 中 D | ✅ 已删 | 正确 |
| 返回切片 | `[B, C]` | B 在 DB 中不存在 → 后续访问时 404 |
| 运行状态 | 💥 **panic** | `i=3` 时 `links[:3]` 越界 |

**核心问题链**：
1. `i=0` 删 A → `[B,C,D]` 拷入 `arr[0..2]` → **B 移到 arr[0]**
2. `i=1` 读 `arr[1]=C`（正常）→ **跳过 B** → **DB 漏删**
3. 切片 Len 从 4→3→2，但 `i` 继续递增到 3 → **`links[:3]` 越界 panic**

---

#### 复现场景 B：全部连续过期

`links = [A(过期), B(过期), C(过期)]`, Len=3, Cap=3

| i | link = arr[i] | 过期 | DB 删除 | 操作后 arr | 操作后 links.Len | 异常 |
|---|--------------|------|---------|-----------|-----------------|------|
| 0 | A | ✅ | Delete(A) ✅ | `[B,C,C]` | 2 | - |
| 1 | arr[1] = **C** (原本是 B，被覆盖了) | ✅ | Delete(C) ✅ | `[B,C,C]` | 1 | ❌ **DB 漏删 B** |
| 2 | arr[2] = C | ✅ | Delete(C) (幂等) | - | 1 | 💥 `links[:2]` 越界 |

最终：B 在 DB 残留，返回 `[B]`，panic 或 返回错误数据。

---

#### 复现场景 C：非连续过期（中间有正常元素）

`links = [A(过期), B(正常), C(过期)]`, Len=3, Cap=3

| i | link = arr[i] | 过期 | DB 删除 | 操作后 arr | 操作后 links.Len | 异常 |
|---|--------------|------|---------|-----------|-----------------|------|
| 0 | A | ✅ | Delete(A) ✅ | `[B,C,C]` | 2 | - |
| 1 | arr[1] = **C** (原本是 B，被覆盖了！) | ✅ | Delete(C) ✅ | `[B,C,C]` | 1 | ❌ **本应检查 B，实际检查了 C**<br>❌ **C 的过期判断结果被用于了原本 B 的位置** |
| 2 | arr[2] = C | ✅ | Delete(C) | - | 1 | 💥 `links[:2]` 越界 |

**非连续过期同样出问题**：即使中间有正常元素，`i=1` 读到的也是 `arr[1]=C` 而非原本的 B。

---

#### 受影响的 API

| API 端点 | 调用的方法 | 影响程度 |
|----------|-----------|---------|
| `GET /api/shares` | `All()` [share/storage.go#L32-L49](share/storage.go#L32-L49) | 🔴 严重 - DB 漏删 + 返回数据错乱 + 可能 panic |
| `GET /api/share/{path}` | `Gets()` [share/storage.go#L94-L111](share/storage.go#L94-L111) | 🔴 严重 - 同上 |
| `GetsByPath()` [share/storage.go#L113-L128](share/storage.go#L113-L128) | 内部调用 `All()` | 🔴 严重 - 同上 |
| `GET /api/public/share/{hash}` | `GetByHash()` [share/storage.go#L72-L86](share/storage.go#L72-L86) | 🟢 安全 - 单条查询无循环 |
| 删除文件级联 | `DeleteWithPathPrefix()` [storage/bolt/share.go#L80-L104](storage/bolt/share.go#L80-L104) | 🟢 安全 - Bolt 底层实现，不经过 Storage 包装层 |

#### 实际表现总结

| 现象 | 说明 |
|------|------|
| ❌ **DB 漏删** | 连续过期时第 2、4、6… 个元素会被跳过（移动到已检查的索引位置） |
| ❌ **DB 误删** | 非连续时正常元素的位置可能被过期元素覆盖，导致用错对象判断 |
| ❌ **返回切片错乱** | 返回切片中可能包含 DB 中已删除的元素，或漏掉本应保留的元素 |
| 💥 **Runtime Panic** | 只要发生过删除（Len 缩短），后续 `i >= links.Len` 就会触发 `slice bounds out of range` |
| 🔄 **最终一致性** | 如果没 panic，下次调用重新从 DB 读取，可能进一步清理部分残留，但仍可能漏删 |

#### 修复方案

**方案 A：收集而非原地删除（推荐，最清晰，无副作用）**

构建新切片，**永远不修改正在遍历的切片**：

```go
var filtered []*Link
for _, link := range links {
    if link.Expire != 0 && link.Expire <= time.Now().Unix() {
        if err := s.Delete(link.Hash); err != nil {
            return nil, err
        }
        continue
    }
    filtered = append(filtered, link)
}
return filtered, nil
```

**方案 B：反向遍历（最小改动）**

从后往前遍历，删除元素只影响未检查的索引（因为它们在已检查索引的前面）：

```go
for i := len(links) - 1; i >= 0; i-- {
    link := links[i]
    if link.Expire != 0 && link.Expire <= time.Now().Unix() {
        if err := s.Delete(link.Hash); err != nil {
            return nil, err
        }
        links = append(links[:i], links[i+1:]...)
    }
}
```

**方案 C：普通 for 循环 + 手动索引控制**

不使用 range，动态检查 `len(links)`，删除后回退索引确保不跳过：

```go
for i := 0; i < len(links); i++ {
    link := links[i]
    if link.Expire != 0 && link.Expire <= time.Now().Unix() {
        if err := s.Delete(link.Hash); err != nil {
            return nil, err
        }
        links = append(links[:i], links[i+1:]...)
        i--  // 回退，下次循环继续检查当前位置
    }
}
```

### 4.3 另一个维度：删除文件时的级联清理

当用户删除文件或目录时，需要清理指向该路径及其子路径的分享链接。

触发点在 Bolt 存储层的 `DeleteWithPathPrefix()` [storage/bolt/share.go#L80-L104](storage/bolt/share.go#L80-L104)：

```go
func (s shareBackend) DeleteWithPathPrefix(pathPrefix string, userID uint) error {
    // 1. storm Prefix 查询：Path 字段前缀匹配
    var links []share.Link
    s.db.Prefix("Path", pathPrefix, &links)

    prefix := strings.TrimRight(pathPrefix, "/")

    for _, link := range links {
        // 2. 只删本用户的（不碰别人的同名路径分享）
        if link.UserID != userID { continue }

        // 3. 二次精确校验：防 Prefix 字节误匹配
        //    例：删 /a 不应删 /abc
        //    必须 link.Path == "/a"
        //       或 strings.HasPrefix(link.Path, "/a/")
        if link.Path != prefix &&
           !strings.HasPrefix(link.Path, prefix+"/") {
            continue
        }

        s.db.DeleteStruct(&share.Link{Hash: link.Hash})
    }
}
```

测试覆盖在 [storage/bolt/share_test.go#L47-L97](storage/bolt/share_test.go#L47-L97)，验证了：
- `/a` 删除 → `/a` 和 `/a/child.txt` 被删
- `/abc`（字节前缀相似但不是子路径）保留
- 其他用户的分享完全不受影响

### 4.4 关于"使用次数限制"

**当前实现不支持使用次数控制。** Link 结构体中没有 view/download 计数字段，storage 层也没有递减逻辑。需要该功能需要自行扩展：

```go
// 如需扩展，可以在 Link 中加：
type Link struct {
    // ... 现有字段
    MaxDownloads int   `json:"max_downloads"`  // 最大次数，0=不限
    Downloads    int   `json:"downloads"`      // 当前已下载
}
```

并在 `publicDlHandler` 中每次成功下载后原子递增 + 判断是否超限。

---

## 五、边界控制总清单

以下是所有防止分享越权的防护层汇总：

| # | 边界 | 代码位置 | 防止的攻击 |
|---|------|----------|-----------|
| 1 | 创建分享前 Stat 路径存在 | [http/share.go#L108](http/share.go#L108) | 为不存在路径预建分享（TOCTOU） |
| 2 | 创建时 ScopedFs 防 symlink 逃逸 | [files/scoped.go](files/scoped.go) | 分享 scope 外的 symlink 目标 |
| 3 | 创建者需 Share+Download 权限 | [http/share.go#L22-L23](http/share.go#L22-L23) | 无权用户创建分享 |
| 4 | GetByHash 惰性过期清理 | [share/storage.go#L78-L83](share/storage.go#L78-L83) | 过期链接继续可用 |
| 5 | 密码校验（bcrypt + 常量时间比较） | [http/public.go#L136-L161](http/public.go#L136-L161) | 时序攻击破解密码 |
| 6 | 所有者权限二次检查 | [http/public.go#L35-L37](http/public.go#L35-L37) | 用户被撤权后旧链接仍可用 |
| 7 | withHashFile 重新 ScopedFs | [http/public.go#L70](http/public.go#L70) | 通过 `../` 或 symlink 跳出分享目录 |
| 8 | checkerPrefix 规则补偿 | [http/data.go#L42-L43](http/data.go#L42-L43) | 分享子目录绕过所有者 deny 规则 |
| 9 | 删除分享只能本人或 Admin | [http/share.go#L92-L93](http/share.go#L92-L93) | 普通用户删别人的分享 |
| 10 | DeleteWithPathPrefix 精确路径 + UserID 过滤 | [storage/bolt/share.go#L93-L99](storage/bolt/share.go#L93-L99) | 删除级联误删他人链接 |
| 11 | ⚠️ range 遍历 + 原地修改切片 | [share/storage.go#L39-L46](share/storage.go#L39-L46) | DB 漏删过期链接 + 返回切片错乱 + runtime panic（待修复） |

---

## 六、核心 API 端点速查表

| 方法 | 路径 | 处理函数 | 说明 |
|------|------|----------|------|
| GET | `/api/shares` | [shareListHandler](http/share.go#L30-L56) | 列出所有分享（Admin 看全部，普通用户看自己的） |
| GET | `/api/share/{path}` | [shareGetsHandler](http/share.go#L58-L77) | 获取某路径下的分享 |
| POST | `/api/share/{path}` | [sharePostHandler](http/share.go#L100-L180) | 创建分享 |
| DELETE | `/api/share/{hash}` | [shareDeleteHandler](http/share.go#L79-L98) | 删除分享（本人或 Admin） |
| GET | `/api/public/share/{hash}[/{path}]` | [publicShareHandler](http/public.go#L115-L125) | 公开访问：获取文件/目录元数据 |
| GET | `/api/public/dl/{hash}[/{path}]` | [publicDlHandler](http/public.go#L127-L134) | 公开访问：下载文件或打包目录 |
