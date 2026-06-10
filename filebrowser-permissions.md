# FileBrowser 权限模型详解

本文档梳理 FileBrowser 中身份识别、角色权限、资源访问限制之间的完整判断过程。

---

## 1. 核心数据模型

### 1.1 User 用户结构

定义于 `users/users.go`

```
User {
    ID                    uint            // 自增主键
    Username              string          // 唯一用户名
    Password              string          // bcrypt 哈希密码
    Scope                 string          // 文件系统作用域（相对路径）
    Locale                string          // 语言
    LockPassword          bool            // 是否锁定密码（禁止修改）
    ViewMode              ViewMode        // list / mosaic
    SingleClick           bool            // 单击打开
    RedirectAfterCopyMove bool            // 复制移动后跳转
    Perm                  Permissions     // 权限位集合（核心）
    Commands              []string        // 允许执行的命令名列表
    Sorting               files.Sorting   // 排序偏好
    Fs                    *ScopedFs       // 作用域文件系统（运行时构建）
    Rules                 []rules.Rule    // 用户级访问规则
    HideDotfiles          bool            // 隐藏点文件
    DateFormat            bool            // 日期格式
    AceEditorTheme        string          // 编辑器主题
}
```

**关键字段关系**：
- `Scope` + `server.Root` → 构建出 `Fs`（ScopedFs），这是文件系统级别的硬隔离
- `Perm` 是权限位集合，控制用户可执行的操作
- `Rules` 是路径级别的细粒度访问控制
- `Commands` 是命令执行白名单

### 1.2 Permissions 权限位

定义于 `users/permissions.go`

| 权限位 | 字段名 | 含义 |
|--------|--------|------|
| `Admin` | `admin` | 管理员：可管理用户、全局设置、查看所有分享 |
| `Execute` | `execute` | 可执行命令（还需在 `Commands` 白名单中） |
| `Create` | `create` | 可创建文件/目录、上传文件、复制文件 |
| `Rename` | `rename` | 可重命名文件/目录 |
| `Modify` | `modify` | 可修改已有文件内容（保存/覆盖上传） |
| `Delete` | `delete` | 可删除文件/目录 |
| `Share` | `share` | 可创建分享链接（必须同时拥有 `Download` 权限） |
| `Download` | `download` | 可下载文件、预览文件、查看字幕 |

**约束关系**：`Share` 依赖 `Download`——创建分享时同时校验两个权限位（见 `http/share.go` withPermShare），创建用户时若 `Share=true && Download=false` 则直接拒绝（见 `http/users.go` userPostHandler / userPutHandler）。

### 1.3 Rule 访问规则

定义于 `rules/rules.go`

```
Rule {
    Regex  bool    // 是否为正则匹配
    Allow  bool    // true=允许, false=拒绝
    Path   string  // 路径（精确或前缀匹配）
    Regexp *Regexp // 编译后的正则（Regex=true时使用）
}
```

**匹配逻辑**（`rules/rules.go` Rule.Matches）：
1. 若 `Regex=true`，使用正则匹配
2. 否则使用路径匹配：
   - 路径完全相等 → 匹配
   - 规则路径是目标路径的前缀 → 匹配（自动补 `/` 后缀，防止 `/uploads` 匹配 `/uploads_backup`）
3. 不匹配则跳过该规则

**规则评估顺序**：全局规则先执行，用户规则后执行，**后匹配的覆盖先匹配的**（最后一条匹配的规则决定结果）。详见 `http/data.go` data.Check。

### 1.4 Share 分享链接

定义于 `share/share.go`

```
Link {
    Hash         string // 分享哈希（URL标识）
    Path         string // 分享的文件/目录路径
    UserID       uint   // 创建者用户ID
    Expire       int64  // 过期时间戳（0=永不过期）
    PasswordHash string // 可选的密码哈希（bcrypt）
    Token        string // 密码保护分享的下载令牌
}
```

### 1.5 Settings 全局配置

定义于 `settings/settings.go`

```
Settings {
    Key                   []byte           // JWT签名密钥
    Signup                bool             // 开放注册
    CreateUserDir         bool             // 自动创建用户目录
    UserHomeBasePath      string           // 用户主目录基路径（默认 /users）
    Defaults              UserDefaults     // 新用户默认值
    AuthMethod            AuthMethod       // 认证方式
    Rules                 []rules.Rule     // 全局访问规则
    Commands              map[string][]string // 命令定义
    Shell                 []string         // 默认Shell
    MinimumPasswordLength uint             // 最小密码长度
    HideDotfiles          bool             // 全局隐藏点文件
    // ...
}

Server {
    Root                  string // 文件系统根目录
    EnableExec            bool   // 全局命令执行开关
    // ...
}
```

---

## 2. 身份识别流程

### 2.1 四种认证方式

所有认证方式实现 `auth/auth.go` 中的 Auther 接口：

```
Auther {
    Auth(r, usr, stg, srv) (*User, error)
    LoginPage() bool
}
```

| 方式 | 标识 | 登录页 | 身份识别逻辑 |
|------|------|--------|-------------|
| **JSON Auth** | `json` | ✅ | 请求体中读取 username+password，比对数据库中已有用户密码；**不自动创建用户** |
| **Proxy Auth** | `proxy` | ❌ | 从 HTTP Header（可配置）中读取 username，查找用户；若不存在则**自动创建** |
| **Hook Auth** | `hook` | ✅ | 读取 username+password，调用外部命令，根据返回的 `hook.action` 决定：`auth`（**创建/更新用户**）、`pass`（验证已有用户）、`block`（拒绝） |
| **No Auth** | `noauth` | ❌ | 直接返回 ID=1 的用户；**不创建任何用户**，依赖初始化时已存在的 ID=1 用户 |

### 2.2 自动创建用户的权限差异对照

四种认证方式中，**JSON Auth 和 NoAuth 不自动创建用户**，只有 Proxy Auth、Hook Auth 以及 Signup 注册端点会创建新用户。以下对照各方式创建用户的权限差异：

| 属性 | Signup 注册 (`http/auth.go` signupHandler) | Proxy Auth 自动创建 (`auth/proxy.go` createUser) | Hook Auth action=auth 创建 (`auth/hook.go` SaveUser) |
|------|---------------------------------------------|--------------------------------------------------|-------------------------------------------------------|
| **Admin** | 强制 `false` | 强制 `false` | 由外部命令输出决定（`user.perm.admin`） |
| **Execute** | 强制 `false` | 强制 `false` | 由外部命令输出决定（`user.perm.execute`）；若 Admin=true 则自动为 true |
| **Create** | 来自 Defaults | 来自 Defaults | 由外部命令输出决定（`user.perm.create`）；若 Admin=true 则自动为 true |
| **Rename** | 来自 Defaults | 来自 Defaults | 由外部命令输出决定（`user.perm.rename`）；若 Admin=true 则自动为 true |
| **Modify** | 来自 Defaults | 来自 Defaults | 由外部命令输出决定（`user.perm.modify`）；若 Admin=true 则自动为 true |
| **Delete** | 来自 Defaults | 来自 Defaults | 由外部命令输出决定（`user.perm.delete`）；若 Admin=true 则自动为 true |
| **Share** | 来自 Defaults | 来自 Defaults | 由外部命令输出决定（`user.perm.share`）；若 Admin=true 则自动为 true |
| **Download** | 来自 Defaults | 来自 Defaults | 由外部命令输出决定（`user.perm.download`）；若 Admin=true 则自动为 true |
| **Commands** | 强制 `[]`（空数组） | 强制 `[]`（空数组） | 由外部命令输出决定（`user.commands`），空格分隔 |
| **LockPassword** | 不设置（默认 false） | 强制 `true` | 强制 `true` |
| **Scope** | 由 MakeUserDir 生成个人目录 | 由 MakeUserDir 生成个人目录 | 由 MakeUserDir 生成个人目录 |
| **Rules** | 不设置（默认 nil → 空） | 不设置（默认 nil → 空） | 不设置（hook 不支持设置 Rules） |
| **其他偏好** | 来自 Defaults.Apply | 来自 Defaults.Apply | 来自 Defaults 初始值 + hook 字段覆盖 |

**关键差异总结**：

1. **Signup 与 Proxy Auth 的权限限制策略一致**——都强制剥夺 Admin 和 Execute，清空 Commands，其余权限继承 Defaults。区别仅在于 LockPassword：Proxy Auth 锁密码（随机生成），Signup 不锁（用户自设）。

2. **Hook Auth 不强制剥夺任何权限**——所有权限位由外部命令输出决定。但存在隐式规则：若 `user.perm.admin=true`，则 `auth/hook.go` GetUser 中所有其他权限位自动置为 true（`isAdmin || GetBoolean(...)`）。因此 Hook Auth 下 Admin 一定拥有全部权限。

3. **NoAuth 不创建用户**——它仅查找 ID=1 的用户，该用户必须在初始化或管理操作中预先存在。其权限完全取决于 ID=1 用户在数据库中的记录。

4. **JSON Auth 不创建用户**——仅在数据库中查找已有用户并验证密码。用户必须由管理员通过 API 或其他认证方式预先创建。

5. **Rules 从不被自动创建流程设置**——所有四种认证方式中，新用户的 Rules 字段均为空（nil），不受 Defaults 或外部命令影响。Rules 只能由管理员后续手动配置。

### 2.3 JWT 令牌机制

登录成功后签发 JWT（HS256），令牌中包含用户关键信息（`http/auth.go` printToken）：

```
authToken {
    User: userInfo { ID, Perm, Commands, ... }
    RegisteredClaims { IssuedAt, ExpiresAt, Issuer="File Browser" }
}
```

令牌提取优先级（`http/auth.go` extractor.ExtractToken）：
1. `X-Auth` 请求头（包含两个点的字符串视为 JWT）
2. `auth` Cookie（仅 GET 请求）

令牌续期条件（`http/auth.go` withUser）：
- 令牌将在 1 小时内过期 → 响应头 `X-Renew-Token: true`
- 用户数据在令牌签发后被更新 → 响应头 `X-Renew-Token: true`
- Proxy Auth 模式下过期令牌可自动续期（如果有配置 LogoutPage）

---

## 3. HTTP 中间件权限链

### 3.1 中间件层级

```
请求 → handle() → withUser() / withAdmin() / withSelfOrAdmin() / withPermShare() / withHashFile() → 业务Handler
```

#### handle()（`http/data.go` handle）
- 加载全局 Settings
- 构造 `data` 上下文对象
- 处理错误响应

#### withUser()（`http/auth.go` withUser）
1. 从请求中提取并验证 JWT
2. 从数据库重新加载用户（确保最新权限）
3. 检查令牌续期需求
4. 将用户对象存入 `d.user`

#### withAdmin()（`http/auth.go` withAdmin）
- 包装 `withUser`
- 额外检查 `d.user.Perm.Admin`，非管理员返回 403

#### withSelfOrAdmin()（`http/users.go` withSelfOrAdmin）
- 包装 `withUser`
- 检查 `d.user.ID == 目标ID || d.user.Perm.Admin`，不满足返回 403

#### withPermShare()（`http/share.go` withPermShare）
- 包装 `withUser`
- 检查 `d.user.Perm.Share && d.user.Perm.Download`，不满足返回 403

#### withHashFile()（`http/public.go` withHashFile）
- 从 URL 提取分享哈希
- 加载分享链接并验证（过期、密码）
- 加载分享创建者用户，检查其 `Share+Download` 权限
- **重新绑定 Fs**：将用户 Fs 根目录缩小到分享路径
- 设置 `checkerPrefix` 以确保规则匹配仍基于用户原始 scope

### 3.2 各 API 端点的权限要求一览

| API 端点 | 方法 | 中间件 | 额外权限检查 |
|----------|------|--------|-------------|
| `/api/login` | POST | 无 | 由认证器决定 |
| `/api/signup` | POST | 无 | `Settings.Signup` 必须为 true |
| `/api/renew` | POST | withUser | 无 |
| `/api/users` | GET | withAdmin | — |
| `/api/users` | POST | withAdmin | JSON Auth 需当前密码；`Share→Download` 约束 |
| `/api/users/{id}` | GET | withSelfOrAdmin | 非管理员看不到 Scope |
| `/api/users/{id}` | PUT | withSelfOrAdmin | 非管理员不可改 Username/Scope/LockPassword/Perm/Commands/Rules；JSON Auth 敏感字段需当前密码；`Share→Download` 约束 |
| `/api/users/{id}` | DELETE | withSelfOrAdmin | JSON Auth 需当前密码；唯一管理员不可删除 |
| `/api/settings` | GET/PUT | withAdmin | — |
| `/api/resources` | GET | withUser | `Download` 控制内容读取；NewFileInfo 内调用 Check(path) |
| `/api/resources` | DELETE | withUser | `Delete` + 路径≠`/` |
| `/api/resources` | POST | withUser | `Create` + `Check(path)`；覆盖需 `Modify` |
| `/api/resources` | PUT | withUser | `Modify` + `Check(path)`；仅文件 |
| `/api/resources` | PATCH | withUser | `Check(src+dst)`；copy 需 `Create`；rename 需 `Rename`；覆盖需 `Modify` |
| `/api/raw` | GET | withUser | `Download`；NewFileInfo 内调用 Check(path)；目录下载递归调用 Check |
| `/api/preview` | GET | withUser | `Download`；NewFileInfo 内调用 Check(path) |
| `/api/subtitle` | GET | withUser | `Download`；NewFileInfo 内调用 Check(path) |
| `/api/command` | GET(WS) | withUser | `EnableExec` + `Execute` + 命令在 `Commands` 中 |
| `/api/shares` | GET | withPermShare | Admin 查全部，否则查自己的 |
| `/api/share` | GET | withPermShare | Admin 按路径查，否则按用户+路径查 |
| `/api/share` | POST | withPermShare | 目标路径必须存在（Fs.Stat 校验，见 §5.1 规则过滤分析） |
| `/api/share` | DELETE | withPermShare | Admin 或分享创建者本人 |
| `/api/public/share` | GET | withHashFile | 分享链接验证（密码/过期）；规则过滤见 §5.2 |
| `/api/public/dl` | GET | withHashFile | 分享链接验证（密码/过期）+ 下载令牌；规则过滤见 §5.2 |
| `/api/search` | GET | withUser | 通过 Checker 接口传入 search.Search，搜索过程中每个路径调用 Check |
| `/api/usage` | GET | withUser | NewFileInfo 内调用 Check(path) |
| `/api/tus` | POST | withUser | `Create` + `Check(path)`；覆盖需 `Modify` |
| `/api/tus` | HEAD/GET | withUser | `Create` + `Check(path)` |
| `/api/tus` | PATCH | withUser | `Create` + `Check(path)` |
| `/api/tus` | DELETE | withUser | `Delete` + 路径≠`/` |

---

## 4. 资源访问限制的三层防线

### 4.1 第一层：文件系统作用域隔离（Scope + ScopedFs）

**构建过程**（`users/users.go` User.Clean）：
```
scope = filepath.Join(baseScope, filepath.Join("/", userScope))
user.Fs = files.NewScopedFs(afero.NewOsFs(), scope)
```
- `baseScope` = `server.Root`（服务器根目录）
- `userScope` = 用户配置的 Scope（如 `/users/alice`）
- 最终路径 = `server.Root` + 清理后的用户 Scope

**ScopedFs 的双重保护**（`files/scoped.go`）：

1. **词法限制**（afero.BasePathFs）：所有文件操作自动限制在 base 目录下，无法通过 `../` 逃逸
2. **符号链接限制**（ScopedFs.guard + within）：每次操作前检查目标路径解析后的实际位置是否在 scope 内
   - 阻止 scope 内的符号链接指向 scope 外
   - 新文件验证最近已存在的祖先目录
   - 悬空符号链接按其所在目录验证（best-effort）

**所有涉及文件操作的 handler 都通过 `d.user.Fs` 访问文件**，因此 Scope 是硬隔离——即使用户有权限也无法访问 Scope 外的文件。

### 4.2 第二层：路径规则过滤（Rules + Check）

**检查入口**：`http/data.go` data.Check

```go
func (d *data) Check(path string) bool {
    // 1. 如果 Fs 被重新绑定（如分享），恢复原始路径前缀
    if d.checkerPrefix != "" {
        path = path.Join(d.checkerPrefix, path)
    }

    // 2. 隐藏点文件
    if d.user.HideDotfiles && rules.MatchHidden(path) {
        return false
    }

    // 3. 全局规则（先执行）
    allow := true
    for _, rule := range d.settings.Rules {
        if rule.Matches(path) {
            allow = rule.Allow
        }
    }

    // 4. 用户规则（后执行，覆盖全局）
    for _, rule := range d.user.Rules {
        if rule.Matches(path) {
            allow = rule.Allow
        }
    }

    return allow
}
```

**重要特性**：
- 规则是**顺序覆盖**的：最后一条匹配的规则生效
- 用户规则**优先于**全局规则（后执行覆盖前者）
- 默认为 `allow=true`（无规则匹配时允许访问）
- `HideDotfiles` 是独立于规则的额外检查，优先级最高
- 分享场景下 `checkerPrefix` 保证规则仍然基于用户原始 scope 路径匹配

**Check() 的两层调用机制**：

Check() 不仅在 handler 中显式调用，还在 `files.NewFileInfo` 内部被隐式调用：

1. **显式调用**——handler 代码中直接调用 `d.Check(path)`，用于在执行操作前判定路径是否允许：
   - 创建文件/上传（POST /resources, POST /tus）
   - 修改文件（PUT /resources）
   - 移动/重命名（PATCH /resources，源和目标都检查）
   - 递归列举（GET /resources/recursive）

2. **隐式调用**——通过 `files.NewFileInfo` 的 `FileOptions.Checker` 参数传入，在以下阶段生效：
   - **入口校验**（`files/file.go` NewFileInfo 第 78 行）：对请求路径本身调用 `opts.Checker.Check(opts.Path)`，不通过则返回 `os.ErrPermission`
   - **目录列举**（`files/file.go` readListing 第 409 行）：遍历目录子项时对每个子路径调用 `checker.Check(fPath)`，不通过则跳过该子项（continue），不会返回错误
   - **符号链接过滤**（`files/file.go` readListing 第 426 行）：若 ScopedFs.Stat 返回 `os.ErrPermission`（符号链接目标逃逸 scope），同样跳过该子项

   这意味着 GET /resources、GET /raw、GET /preview、GET /subtitle、GET /usage 等端点虽然 handler 代码中未显式调用 `d.Check()`，但通过 NewFileInfo 内部的 Checker 调用同样受到规则过滤。

3. **其他间接调用**：
   - 目录下载/打包（`http/raw.go` getFiles）：递归遍历时对每个路径调用 `d.Check(path)`
   - 搜索（`http/search.go`）：通过 Checker 接口传入 `search.Search`，搜索过程中对每个路径调用 Check

### 4.3 第三层：操作权限位（Permissions）

即用户的 `Perm` 字段，8 个布尔位精确控制每个操作类型。详见上文 §1.2 和 §3.2。

---

## 5. 分享（Share）场景的权限判断

### 5.1 创建分享时的规则过滤

创建分享的完整权限检查链如下（`http/share.go` sharePostHandler）：

```
阶段 1: withPermShare 中间件
  ├─ withUser → JWT 验证 + 加载用户
  └─ Perm.Share && Perm.Download → 否则 403

阶段 2: 路径存在性校验（隐含 Scope + 符号链接限制）
  └─ d.user.Fs.Stat(r.URL.Path)
       ├─ ScopedFs 词法限制：路径无法逃逸用户 Scope
       └─ ScopedFs.guard → within：符号链接目标无法逃逸用户 Scope
       └─ 任一不通过 → 返回错误，分享创建失败

阶段 3: 分享数据创建
  └─ 生成 Hash、设置过期、可选密码 → 保存到数据库
```

**关键点**：创建分享时，**不显式调用 `d.Check(r.URL.Path)`**（即不对 Rules 进行检查）。这意味着只要路径在用户的 Scope 内（通过 ScopedFs.Stat 校验），即使该路径被全局或用户规则标记为拒绝（`Allow=false`），用户仍然可以创建指向该路径的分享链接。

然而，这个分享创建后是否能被访问者看到内容，取决于公开访问时的规则过滤（见 §5.2）。创建者自己在正常文件浏览中若访问该路径，会因 NewFileInfo 内的 Checker 校验而看不到该路径的内容。

### 5.2 访问分享时的规则过滤

公开访问分享的完整权限检查链如下（`http/public.go` withHashFile）：

```
阶段 1: 分享链接验证
  ├─ 提取 Hash → 查询 Link 记录
  └─ authenticateShareRequest()
       ├─ 无密码 → 直接通过
       ├─ URL ?token= → ConstantTimeCompare 验证
       └─ X-SHARE-PASSWORD 头 → bcrypt 比对

阶段 2: 创建者权限验证
  └─ 加载创建者用户 (d.store.Users.Get)
       └─ user.Perm.Share && user.Perm.Download → 否则 403

阶段 3: 首次 NewFileInfo（原始 Scope 下）
  └─ files.NewFileInfo{Path: link.Path, Checker: d}
       ├─ 此时 d.user.Fs = 创建者的原始 ScopedFs
       ├─ Check(link.Path)：checkerPrefix 为空，路径为用户 Scope 内的原始路径
       │    ├─ HideDotfiles 检查
       │    ├─ 全局 Rules 匹配
       │    └─ 用户 Rules 匹配
       └─ 若 Check 不通过 → 返回 os.ErrPermission → 403

阶段 4: 重新绑定 Fs + 设置 checkerPrefix
  ├─ d.user.Fs = files.NewScopedFs(d.user.Fs, basePath)
  │    └─ Fs 根缩小到分享路径，符号链接仍受限
  └─ d.checkerPrefix = basePath
       └─ 后续 Check() 调用将路径还原为用户 Scope 内的原始路径

阶段 5: 第二次 NewFileInfo（重新绑定后的 Scope 下）
  └─ files.NewFileInfo{Path: filePath, Checker: d}
       ├─ 对于目录分享：filePath = ifPath（URL 中的子路径）
       ├─ Check(filePath)：
       │    ├─ checkerPrefix 拼接：实际匹配路径 = basePath + filePath
       │    ├─ HideDotfiles 检查（基于还原后的路径）
       │    ├─ 全局 Rules 匹配（基于还原后的路径）
       │    └─ 用户 Rules 匹配（基于还原后的路径）
       └─ 若为目录 + Expand=true → readListing 中对每个子项调用 Check

阶段 6: 执行业务 handler
  ├─ publicShareHandler → 返回文件/目录 JSON
  └─ publicDlHandler → rawFileHandler 或 rawDirHandler
       └─ rawDirHandler.getFiles → 对每个路径调用 d.Check(path)
```

**规则过滤生效的三个阶段**：

| 阶段 | 检查路径 | checkerPrefix | 规则匹配基准 | 不通过后果 |
|------|----------|---------------|-------------|-----------|
| 阶段 3：首次 NewFileInfo | `link.Path` | 空（未设置） | 用户原始 Scope 路径 | 403，分享完全不可访问 |
| 阶段 5：第二次 NewFileInfo | `filePath`（子路径） | `basePath`（= link.Path） | `basePath + filePath`，还原为用户原始 Scope 路径 | 403 或子项被跳过 |
| 阶段 6：业务 handler | 目录子路径 | `basePath` | 同上 | 子项被跳过（下载时省略） |

**checkerPrefix 的作用**：阶段 4 将 Fs 重新绑定到 `basePath` 后，后续传入 Check 的路径是相对于新 Fs 根的路径（如 `/subdir/file.txt`）。但用户的 Rules 是基于原始 Scope 的完整路径编写的（如 `/docs/secret/subdir/file.txt`）。checkerPrefix 将 `basePath` 拼接回去，确保规则匹配使用的是 `/docs/secret/subdir/file.txt` 而非 `/subdir/file.txt`，否则拒绝规则会被绕过。

**关于路径基准的说明**：Rules 中的 `Path` 和 Check() 中匹配的路径，始终是**用户 Scope 内的相对路径**（即 Fs 根之下的路径），而非物理文件系统路径。用户 Scope 通过 `server.Root + user.Scope` 映射到物理路径，但 `server.Root` 不参与规则匹配。例如，若 `server.Root = /srv/fb`，`user.Scope = /users/alice`，则物理路径 `/srv/fb/users/alice/docs/secret` 在规则中对应的是 `/docs/secret`。

**示例**：

前提条件：
- `server.Root = /srv/fb`（仅影响物理路径映射，不参与规则匹配）
- 用户 Scope 为 `/users/alice`，Fs 根映射到物理路径 `/srv/fb/users/alice`
- 用户有一条规则：`{Path: "/docs/secret", Allow: false}`
- 用户分享了 `/docs` 目录（`link.Path = "/docs"`）

访问者请求分享内的子路径 `/secret/file.txt` 时：

```
阶段 5 中的路径拼接过程：

1. 访问者请求: /api/public/share/<hash>/secret/file.txt
2. ifPathWithName 解析: ifPath = /secret/file.txt
3. Fs 重新绑定: d.user.Fs = NewScopedFs(原始Fs, "/docs")
4. checkerPrefix 设置: d.checkerPrefix = "/docs"
5. filePath = /secret/file.txt (相对于重新绑定后的 Fs 根)
6. Check 内路径还原:
   checkerPrefix + filePath = path.Join("/docs", "/secret/file.txt")
                            = "/docs/secret/file.txt"
7. 规则匹配:
   规则 Path="/docs/secret" 匹配 "/docs/secret/file.txt" (前缀匹配) → Allow=false → 拒绝

路径汇总:
  访问子路径 (Fs 相对):  /secret/file.txt
  checkerPrefix:         /docs
  拼接后的检查路径:       /docs/secret/file.txt    ← 规则按此路径匹配
  规则 Path:             /docs/secret             ← 前缀匹配成功
  物理路径 (不参与匹配):  /srv/fb/users/alice/docs/secret/file.txt

若无 checkerPrefix，检查路径仅为 /secret/file.txt，
规则 /docs/secret 无法匹配，拒绝被绕过。
```

### 5.3 下载密码保护的分享

密码保护的分享会生成一个 `Token`。下载时可以通过：
- URL 查询参数 `?token=<token>` 直接下载（避免重复输入密码）
- `X-SHARE-PASSWORD` 请求头输入密码

验证使用 `crypto/subtle.ConstantTimeCompare` 防止时序攻击（`http/public.go` authenticateShareRequest）。

---

## 6. 命令执行的权限判断

三层检查（`http/commands.go` commandsHandler）：

```
1. d.server.EnableExec → 全局开关（Server 配置）
2. d.user.Perm.Execute → 用户权限位
3. 命令名 in d.user.Commands → 命令白名单
```

命令工作目录 = `d.user.FullPath(r.URL.Path)`，即用户 Scope 下的请求路径。

命令定义来自 `Settings.Commands`（map[string][]string），如 `{"git": ["git", "--no-pager"]}`。通过 `runner/parser.go` ParseCommand 解析。

---

## 7. 用户管理的权限判断

### 7.1 非管理员可修改的字段

定义于 `http/users.go` NonModifiableFieldsForNonAdmin：

```
Username, Scope, LockPassword, Perm, Commands, Rules
```

非管理员尝试修改这些字段时返回 403。

### 7.2 密码修改限制

- `LockPassword = true` 时，非管理员无法修改密码
- JSON Auth 模式下，修改敏感字段（username/password/scope/lockPassword/commands/perm）需要提供当前密码

### 7.3 唯一管理员保护

`users/storage.go` IsUniqueAdmin：如果用户是管理员且系统中只剩 1 个管理员，则禁止删除该用户。

### 7.4 用户创建的约束

- 新用户必须满足 `Share → Download` 约束
- 密码必须满足最小长度要求
- 自动创建用户主目录

---

## 8. 完整权限判断流程图

```
HTTP 请求
  │
  ├─ 公开端点 (/api/public/*)
  │   └─ withHashFile
  │       ├─ 提取分享 Hash → 查询 Link
  │       ├─ 验证分享（密码/过期/Token）
  │       ├─ 加载创建者用户
  │       ├─ 检查创建者 Perm.Share + Perm.Download
  │       ├─ 首次 NewFileInfo(link.Path) → Check(path) 规则过滤
  │       ├─ 重新绑定 Fs（二次 Scope 限制）+ 设置 checkerPrefix
  │       ├─ 第二次 NewFileInfo(filePath) → Check(checkerPrefix+path) 规则过滤
  │       └─ 执行 handler → 业务中进一步 Check
  │
  ├─ 认证端点 (/api/login, /api/signup)
  │   └─ 无中间件，由 Auther 认证
  │
  └─ 受保护端点 (/api/*)
      └─ withUser（或其变体）
          ├─ 提取 JWT → 验证签名+过期
          ├─ 从数据库加载最新用户
          └─ 执行 handler
              │
              ├─ 1. 检查 Perm 位（如 Delete, Create, Modify...）
              │     └─ 不满足 → 403 Forbidden
              │
              ├─ 2. 检查 d.Check(path)（Rules 过滤）
              │     ├─ HideDotfiles 检查
              │     ├─ 全局 Rules 匹配
              │     └─ 用户 Rules 匹配（覆盖全局）
              │     └─ 不满足 → 403 Forbidden 或子项被跳过
              │
              └─ 3. 通过 d.user.Fs 执行文件操作
                    ├─ Scope 词法限制（BasePathFs）
                    └─ 符号链接限制（ScopedFs.guard）
                    └─ 不满足 → Permission Error
```

---

## 9. 关键源文件索引

| 领域 | 文件 | 核心内容 |
|------|------|----------|
| 用户模型 | `users/users.go` | User 结构体、Clean()、FullPath() |
| 权限位 | `users/permissions.go` | Permissions 结构体（8个布尔位） |
| 规则模型 | `rules/rules.go` | Rule 结构体、Matches()、MatchHidden() |
| 文件信息 | `files/file.go` | NewFileInfo（入口 Check + readListing 递归 Check） |
| 作用域文件系统 | `files/scoped.go` | ScopedFs、guard()、within()（防符号链接逃逸） |
| 认证接口 | `auth/auth.go` | Auther 接口 |
| JSON 认证 | `auth/json.go` | JSONAuth、防时序攻击 |
| 代理认证 | `auth/proxy.go` | ProxyAuth、自动创建用户（Admin=false, Execute=false, LockPassword=true） |
| Hook 认证 | `auth/hook.go` | HookAuth、外部命令、管理员自动获得全部权限、LockPassword=true |
| 无认证 | `auth/none.go` | NoAuth、固定 ID=1、不创建用户 |
| HTTP 认证中间件 | `http/auth.go` | withUser、withAdmin、JWT 处理、注册限制（Admin=false, Execute=false） |
| HTTP 数据上下文 | `http/data.go` | data 结构体、Check()（规则评估 + checkerPrefix 还原） |
| 资源操作 | `http/resource.go` | CRUD handler 中的权限检查 |
| 分享操作 | `http/share.go` | withPermShare、分享 CRUD（创建时不调 Check） |
| 公开访问 | `http/public.go` | withHashFile、分享验证、Fs 重新绑定、checkerPrefix 设置 |
| 文件下载 | `http/raw.go` | rawHandler、getFiles（递归 Check） |
| 命令执行 | `http/commands.go` | 三层命令权限检查 |
| 用户管理 | `http/users.go` | withSelfOrAdmin、非管理员字段限制 |
| 全局设置 | `http/settings.go` | withAdmin 保护 |
| TUS 上传 | `http/tus_handlers.go` | 上传/续传权限检查 |
| 分享模型 | `share/share.go` | Link 结构体 |
| 全局配置 | `settings/settings.go` | Settings、Server 结构体 |
| 默认值 | `settings/defaults.go` | UserDefaults.Apply() |
| 用户目录 | `settings/dir.go` | MakeUserDir（自动创建用户主目录） |
| 用户存储 | `users/storage.go` | 唯一管理员保护、用户 CRUD |
| 错误定义 | `errors/errors.go` | ErrPermissionDenied、ErrShareRequiresDownload 等 |
