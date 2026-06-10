# Filebrowser 分享链接与公开访问代码实现分析

## 一、整体架构概览

分享功能的代码分布在以下关键模块：

| 层级 | 模块 | 职责 |
|------|------|------|
| 数据模型 | [share/share.go](file:///d:/fz/0601/solo-dogfeeding/code/170-filebrowser/share/share.go) | Link 结构体定义 |
| 存储层 | [share/storage.go](file:///d:/fz/0601/solo-dogfeeding/code/170-filebrowser/share/storage.go) + [storage/bolt/share.go](file:///d:/fz/0601/solo-dogfeeding/code/170-filebrowser/storage/bolt/share.go) | 存储接口与 BoltDB 实现（含过期清理） |
| HTTP API | [http/share.go](file:///d:/fz/0601/solo-dogfeeding/code/170-filebrowser/http/share.go) | 分享管理（创建/删除/列表） |
| 公开访问 | [http/public.go](file:///d:/fz/0601/solo-dogfeeding/code/170-filebrowser/http/public.go) | 公开分享访问（带密码校验 + 沙箱隔离） |
| 路由注册 | [http/http.go](file:///d:/fz/0601/solo-dogfeeding/code/170-filebrowser/http/http.go#L74-L91) | API 路由挂载 |
| 文件沙箱 | [files/scoped.go](file:///d:/fz/0601/solo-dogfeeding/code/170-filebrowser/files/scoped.go) | ScopedFs 越界防护（防符号链接逃逸） |
| 权限模型 | [users/permissions.go](file:///d:/fz/0601/solo-dogfeeding/code/170-filebrowser/users/permissions.go) | Share + Download 权限位 |
| 规则检查 | [http/data.go](file:///d:/fz/0601/solo-dogfeeding/code/170-filebrowser/http/data.go#L37-L64) | checkerPrefix 规则前缀补偿 |
| 前端 API | [frontend/src/api/share.ts](file:///d:/fz/0601/solo-dogfeeding/code/170-filebrowser/frontend/src/api/share.ts) + [frontend/src/api/pub.ts](file:///d:/fz/0601/solo-dogfeeding/code/170-filebrowser/frontend/src/api/pub.ts) | 前端调用封装 |

---

## 二、分享链接生成

### 2.1 数据模型：Link 结构体

定义在 [share/share.go#L10-L20](file:///d:/fz/0601/solo-dogfeeding/code/170-filebrowser/share/share.go#L10-L20)：

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

入口在 [http/share.go#L100-L180](file:///d:/fz/0601/solo-dogfeeding/code/170-filebrowser/http/share.go#L100-L180)，整个流程有严格的边界校验：

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

前端创建分享在 [frontend/src/components/prompts/Share.vue](file:///d:/fz/0601/solo-dogfeeding/code/170-filebrowser/frontend/src/components/prompts/Share.vue)，调用 [frontend/src/api/share.ts#L18-L41](file:///d:/fz/0601/solo-dogfeeding/code/170-filebrowser/frontend/src/api/share.ts#L18-L41) 的 `create()`。

注意：有密码的分享，**复制下载链接按钮是 disabled 的**（[Share.vue#L41-L45](file:///d:/fz/0601/solo-dogfeeding/code/170-filebrowser/frontend/src/components/prompts/Share.vue#L41-L45)），因为直接下载链接无法携带密码。

---

## 三、访问校验（公开访问边界控制）

### 3.1 路由总览

在 [http/http.go#L89-L91](file:///d:/fz/0601/solo-dogfeeding/code/170-filebrowser/http/http.go#L89-L91)：

```go
public := api.PathPrefix("/public").Subrouter()
public.PathPrefix("/dl").Handler(monkey(publicDlHandler, "/api/public/dl/")).Methods("GET")
public.PathPrefix("/share").Handler(monkey(publicShareHandler, "/api/public/share/")).Methods("GET")
```

两个端点：
- `/api/public/share/{hash}[/{path}]` → 元数据（目录 listing、文件信息）
- `/api/public/dl/{hash}[/{path}]` → 原始下载

两者都走 `withHashFile` 中间件（[http/public.go#L17-L98](file:///d:/fz/0601/solo-dogfeeding/code/170-filebrowser/http/public.go#L17-L98)）。

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

定义在 [http/public.go#L136-L161](file:///d:/fz/0601/solo-dogfeeding/code/170-filebrowser/http/public.go#L136-L161)：

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
| X-SHARE-PASSWORD header | 首次访问密码分享 | 前端 [pub.ts#L7-L13](file:///d:/fz/0601/solo-dogfeeding/code/170-filebrowser/frontend/src/api/pub.ts#L7-L13) 在 `fetch()` 时把用户输入的密码 encode 后放 header |

前端密码输入 UI 在 [frontend/src/views/Share.vue#L42-L73](file:///d:/fz/0601/solo-dogfeeding/code/170-filebrowser/frontend/src/views/Share.vue#L42-L73)：收到 401 后显示密码输入卡片。

### 3.4 ScopedFs：文件系统沙箱（防符号链接逃逸）

最核心的安全边界在 [files/scoped.go](file:///d:/fz/0601/solo-dogfeeding/code/170-filebrowser/files/scoped.go)。

**两层防护：**
1. **词法层**：内层 `afero.BasePathFs` 把所有路径拼到 base 目录下，防止 `../` 跳出
2. **符号链接层**：`ScopedFs.guard()` 在每个会 dereference symlink 的操作前调用 `within()`，通过 `filepath.EvalSymlinks` 解析磁盘上真实目标路径，确认在 scope 内

关键函数 `within()` [scoped.go#L64-L94](file:///d:/fz/0601/solo-dogfeeding/code/170-filebrowser/files/scoped.go#L64-L94)：

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

在 [http/data.go#L37-L64](file:///d:/fz/0601/solo-dogfeeding/code/170-filebrowser/http/data.go#L37-L64)：

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

这个机制在 [public_test.go#L147-L255](file:///d:/fz/0601/solo-dogfeeding/code/170-filebrowser/http/public_test.go#L147-L255) 的 `TestPublicShareHandlerRules` 中有完整的测试覆盖。

### 3.6 所有者权限二次校验

即使分享链接本身有效，系统还会在 withHashFile Step 4 中重新校验所有者当前的权限：

```go
// [http/public.go#L35-L37]
if !user.Perm.Share || !user.Perm.Download {
    return http.StatusForbidden, nil
}
```

这意味着：管理员可以通过**撤销用户的 Share 或 Download 权限**来立即使该用户的所有分享失效，不需要逐个删除链接。

测试用例见 [public_test.go#L73-L86](file:///d:/fz/0601/solo-dogfeeding/code/170-filebrowser/http/public_test.go#L73-L86)。

---

## 四、过期处理机制

### 4.1 过期策略：Lazy（惰性删除）

Filebrowser **不使用后台定时任务**清理过期链接。而是采用 **"访问时检查 + 即时删除"** 的懒加载策略。

过期检查统一封装在 [share/storage.go](file:///d:/fz/0601/solo-dogfeeding/code/170-filebrowser/share/storage.go) 的 Storage 中间层（注意：不是 bolt 底层实现，而是在 Storage 包装层）：

| 方法 | 过期处理行为 |
|------|-------------|
| `All()` [L39-L46](file:///d:/fz/0601/solo-dogfeeding/code/170-filebrowser/share/storage.go#L39-L46) | 遍历中遇到过期 → 从 DB Delete → 从返回切片剔除 |
| `FindByUserID()` [L59-L66](file:///d:/fz/0601/solo-dogfeeding/code/170-filebrowser/share/storage.go#L59-L66) | 同上 |
| `GetByHash()` [L78-L83](file:///d:/fz/0601/solo-dogfeeding/code/170-filebrowser/share/storage.go#L78-L83) | 单条过期 → Delete + 返回 `ErrNotExist` (404) |
| `Gets()` [L101-L108](file:///d:/fz/0601/solo-dogfeeding/code/170-filebrowser/share/storage.go#L101-L108) | 遍历清理 |
| `GetsByPath()` [L114-L128](file:///d:/fz/0601/solo-dogfeeding/code/170-filebrowser/share/storage.go#L114-L128) | 先调 All()（已清理）再过滤 |

判断条件统一为：
```go
if link.Expire != 0 && link.Expire <= time.Now().Unix() { ... }
```
（`Expire == 0` 代表永久分享，永远不清理）

### 4.2 另一个维度：删除文件时的级联清理

当用户删除文件或目录时，需要清理指向该路径及其子路径的分享链接。

触发点在 Bolt 存储层的 `DeleteWithPathPrefix()` [storage/bolt/share.go#L80-L104](file:///d:/fz/0601/solo-dogfeeding/code/170-filebrowser/storage/bolt/share.go#L80-L104)：

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

测试覆盖在 [storage/bolt/share_test.go#L47-L97](file:///d:/fz/0601/solo-dogfeeding/code/170-filebrowser/storage/bolt/share_test.go#L47-L97)，验证了：
- `/a` 删除 → `/a` 和 `/a/child.txt` 被删
- `/abc`（字节前缀相似但不是子路径）保留
- 其他用户的分享完全不受影响

### 4.3 关于"使用次数限制"

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
| 1 | 创建分享前 Stat 路径存在 | [share.go#L108](file:///d:/fz/0601/solo-dogfeeding/code/170-filebrowser/http/share.go#L108) | 为不存在路径预建分享（TOCTOU） |
| 2 | 创建时 ScopedFs 防 symlink 逃逸 | [scoped.go](file:///d:/fz/0601/solo-dogfeeding/code/170-filebrowser/files/scoped.go) | 分享 scope 外的 symlink 目标 |
| 3 | 创建者需 Share+Download 权限 | [share.go#L22-L23](file:///d:/fz/0601/solo-dogfeeding/code/170-filebrowser/http/share.go#L22-L23) | 无权用户创建分享 |
| 4 | GetByHash 惰性过期清理 | [storage.go#L78-L83](file:///d:/fz/0601/solo-dogfeeding/code/170-filebrowser/share/storage.go#L78-L83) | 过期链接继续可用 |
| 5 | 密码校验（bcrypt + 常量时间比较） | [public.go#L136-L161](file:///d:/fz/0601/solo-dogfeeding/code/170-filebrowser/http/public.go#L136-L161) | 时序攻击破解密码 |
| 6 | 所有者权限二次检查 | [public.go#L35-L37](file:///d:/fz/0601/solo-dogfeeding/code/170-filebrowser/http/public.go#L35-L37) | 用户被撤权后旧链接仍可用 |
| 7 | withHashFile 重新 ScopedFs | [public.go#L70](file:///d:/fz/0601/solo-dogfeeding/code/170-filebrowser/http/public.go#L70) | 通过 `../` 或 symlink 跳出分享目录 |
| 8 | checkerPrefix 规则补偿 | [data.go#L42-L43](file:///d:/fz/0601/solo-dogfeeding/code/170-filebrowser/http/data.go#L42-L43) | 分享子目录绕过所有者 deny 规则 |
| 9 | 删除分享只能本人或 Admin | [share.go#L92-L93](file:///d:/fz/0601/solo-dogfeeding/code/170-filebrowser/http/share.go#L92-L93) | 普通用户删别人的分享 |
| 10 | DeleteWithPathPrefix 精确路径 + UserID 过滤 | [bolt/share.go#L93-L99](file:///d:/fz/0601/solo-dogfeeding/code/170-filebrowser/storage/bolt/share.go#L93-L99) | 删除级联误删他人链接 |

---

## 六、核心 API 端点速查表

| 方法 | 路径 | 处理函数 | 说明 |
|------|------|----------|------|
| GET | `/api/shares` | [shareListHandler](file:///d:/fz/0601/solo-dogfeeding/code/170-filebrowser/http/share.go#L30-L56) | 列出所有分享（Admin 看全部，普通用户看自己的） |
| GET | `/api/share/{path}` | [shareGetsHandler](file:///d:/fz/0601/solo-dogfeeding/code/170-filebrowser/http/share.go#L58-L77) | 获取某路径下的分享 |
| POST | `/api/share/{path}` | [sharePostHandler](file:///d:/fz/0601/solo-dogfeeding/code/170-filebrowser/http/share.go#L100-L180) | 创建分享 |
| DELETE | `/api/share/{hash}` | [shareDeleteHandler](file:///d:/fz/0601/solo-dogfeeding/code/170-filebrowser/http/share.go#L79-L98) | 删除分享（本人或 Admin） |
| GET | `/api/public/share/{hash}[/{path}]` | [publicShareHandler](file:///d:/fz/0601/solo-dogfeeding/code/170-filebrowser/http/public.go#L115-L125) | 公开访问：获取文件/目录元数据 |
| GET | `/api/public/dl/{hash}[/{path}]` | [publicDlHandler](file:///d:/fz/0601/solo-dogfeeding/code/170-filebrowser/http/public.go#L127-L134) | 公开访问：下载文件或打包目录 |
