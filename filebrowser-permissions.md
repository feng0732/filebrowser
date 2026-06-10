# FileBrowser 权限模型详解

本文档梳理 FileBrowser 中身份识别、角色权限、资源访问限制之间的完整判断过程。

---

## 1. 核心数据模型

### 1.1 User 用户结构

定义于 [users.go](file:///d:/fz/0601/solo-dogfeeding/code/166-filebrowser/users/users.go#L21-L39)

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

定义于 [permissions.go](file:///d:/fz/0601/solo-dogfeeding/code/166-filebrowser/users/permissions.go#L4-L13)

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

**约束关系**：`Share` 依赖 `Download`——创建分享时同时校验两个权限位（见 [share.go](file:///d:/fz/0601/solo-dogfeeding/code/166-filebrowser/http/share.go#L22-L24)），创建用户时若 `Share=true && Download=false` 则直接拒绝（见 [users.go](file:///d:/fz/0601/solo-dogfeeding/code/166-filebrowser/http/users.go#L159-L161)）。

### 1.3 Rule 访问规则

定义于 [rules.go](file:///d:/fz/0601/solo-dogfeeding/code/166-filebrowser/rules/rules.go#L15-L20)

```
Rule {
    Regex  bool    // 是否为正则匹配
    Allow  bool    // true=允许, false=拒绝
    Path   string  // 路径（精确或前缀匹配）
    Regexp *Regexp // 编译后的正则（Regex=true时使用）
}
```

**匹配逻辑**（[Rule.Matches](file:///d:/fz/0601/solo-dogfeeding/code/166-filebrowser/rules/rules.go#L29-L44)）：
1. 若 `Regex=true`，使用正则匹配
2. 否则使用路径匹配：
   - 路径完全相等 → 匹配
   - 规则路径是目标路径的前缀 → 匹配（自动补 `/` 后缀，防止 `/uploads` 匹配 `/uploads_backup`）
3. 不匹配则跳过该规则

**规则评估顺序**：全局规则先执行，用户规则后执行，**后匹配的覆盖先匹配的**（最后一条匹配的规则决定结果）。详见 [data.Check](file:///d:/fz/0601/solo-dogfeeding/code/166-filebrowser/http/data.go#L37-L64)。

### 1.4 Share 分享链接

定义于 [share.go](file:///d:/fz/0601/solo-dogfeeding/code/166-filebrowser/share/share.go#L10-L20)

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

定义于 [settings.go](file:///d:/fz/0601/solo-dogfeeding/code/166-filebrowser/settings/settings.go#L23-L41)

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

所有认证方式实现 [Auther 接口](file:///d:/fz/0601/solo-dogfeeding/code/166-filebrowser/auth/auth.go#L11-L16)：

```
Auther {
    Auth(r, usr, stg, srv) (*User, error)
    LoginPage() bool
}
```

| 方式 | 标识 | 登录页 | 身份识别逻辑 |
|------|------|--------|-------------|
| **JSON Auth** | `json` | ✅ | 请求体中读取 username+password，比对数据库中用户密码 |
| **Proxy Auth** | `proxy` | ❌ | 从 HTTP Header（可配置）中读取 username，查找或自动创建用户 |
| **Hook Auth** | `hook` | ✅ | 读取 username+password，调用外部命令，根据返回的 `hook.action` 决定：`auth`（创建/更新用户）、`pass`（验证已有用户）、`block`（拒绝） |
| **No Auth** | `noauth` | ❌ | 直接返回 ID=1 的用户 |

### 2.2 自动创建用户的权限限制

**注册用户**（[auth.go#signupHandler](file:///d:/fz/0601/solo-dogfeeding/code/166-filebrowser/http/auth.go#L151-L214)）：
- 强制 `Perm.Admin = false`
- 强制 `Perm.Execute = false`
- 强制 `Commands = []`
- 应用 `Settings.Defaults` 中的其他默认值

**Proxy Auth 自动创建**（[proxy.go#createUser](file:///d:/fz/0601/solo-dogfeeding/code/166-filebrowser/auth/proxy.go#L30-L66)）：
- 强制 `Perm.Admin = false`
- 强制 `Perm.Execute = false`
- 强制 `Commands = []`
- 强制 `LockPassword = true`
- 应用 `Settings.Defaults` 中的其他默认值

**Hook Auth 创建**（[hook.go#SaveUser](file:///d:/fz/0601/solo-dogfeeding/code/166-filebrowser/auth/hook.go#L131-L191)）：
- 权限由外部命令输出决定
- **管理员自动获得所有权限**：若 `user.perm.admin=true`，则所有其他权限位自动设为 true（见 [hook.go#GetUser](file:///d:/fz/0601/solo-dogfeeding/code/166-filebrowser/auth/hook.go#L194-L228)）
- 强制 `LockPassword = true`

### 2.3 JWT 令牌机制

登录成功后签发 JWT（HS256），令牌中包含用户关键信息（[auth.go#printToken](file:///d:/fz/0601/solo-dogfeeding/code/166-filebrowser/http/auth.go#L223-L257)）：

```
authToken {
    User: userInfo { ID, Perm, Commands, ... }
    RegisteredClaims { IssuedAt, ExpiresAt, Issuer="File Browser" }
}
```

令牌提取优先级（[extractor.ExtractToken](file:///d:/fz/0601/solo-dogfeeding/code/166-filebrowser/http/auth.go#L49-L67)）：
1. `X-Auth` 请求头（包含两个点的字符串视为 JWT）
2. `auth` Cookie（仅 GET 请求）

令牌续期条件（[withUser](file:///d:/fz/0601/solo-dogfeeding/code/166-filebrowser/http/auth.go#L85-L111)）：
- 令牌将在 1 小时内过期 → 响应头 `X-Renew-Token: true`
- 用户数据在令牌签发后被更新 → 响应头 `X-Renew-Token: true`
- Proxy Auth 模式下过期令牌可自动续期（如果有配置 LogoutPage）

---

## 3. HTTP 中间件权限链

### 3.1 中间件层级

```
请求 → handle() → withUser() / withAdmin() / withSelfOrAdmin() / withPermShare() / withHashFile() → 业务Handler
```

#### handle()（[data.go#handle](file:///d:/fz/0601/solo-dogfeeding/code/166-filebrowser/http/data.go#L66-L101)）
- 加载全局 Settings
- 构造 `data` 上下文对象
- 处理错误响应

#### withUser()（[auth.go#withUser](file:///d:/fz/0601/solo-dogfeeding/code/166-filebrowser/http/auth.go#L85-L111)）
1. 从请求中提取并验证 JWT
2. 从数据库重新加载用户（确保最新权限）
3. 检查令牌续期需求
4. 将用户对象存入 `d.user`

#### withAdmin()（[auth.go#withAdmin](file:///d:/fz/0601/solo-dogfeeding/code/166-filebrowser/http/auth.go#L113-L121)）
- 包装 `withUser`
- 额外检查 `d.user.Perm.Admin`，非管理员返回 403

#### withSelfOrAdmin()（[users.go#withSelfOrAdmin](file:///d:/fz/0601/solo-dogfeeding/code/166-filebrowser/http/users.go#L57-L71)）
- 包装 `withUser`
- 检查 `d.user.ID == 目标ID || d.user.Perm.Admin`，不满足返回 403

#### withPermShare()（[share.go#withPermShare](file:///d:/fz/0601/solo-dogfeeding/code/166-filebrowser/http/share.go#L20-L28)）
- 包装 `withUser`
- 检查 `d.user.Perm.Share && d.user.Perm.Download`，不满足返回 403

#### withHashFile()（[public.go#withHashFile](file:///d:/fz/0601/solo-dogfeeding/code/166-filebrowser/http/public.go#L17-L98)）
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
| `/api/resources` | GET | withUser | `Download` 控制内容读取 |
| `/api/resources` | DELETE | withUser | `Delete` + 路径≠`/` |
| `/api/resources` | POST | withUser | `Create` + `Check(path)`；覆盖需 `Modify` |
| `/api/resources` | PUT | withUser | `Modify` + `Check(path)`；仅文件 |
| `/api/resources` | PATCH | withUser | `Check(src+dst)`；copy 需 `Create`；rename 需 `Rename`；覆盖需 `Modify` |
| `/api/raw` | GET | withUser | `Download` |
| `/api/preview` | GET | withUser | `Download` |
| `/api/subtitle` | GET | withUser | `Download` |
| `/api/command` | GET(WS) | withUser | `EnableExec` + `Execute` + 命令在 `Commands` 中 |
| `/api/shares` | GET | withPermShare | Admin 查全部，否则查自己的 |
| `/api/share` | GET | withPermShare | Admin 按路径查，否则按用户+路径查 |
| `/api/share` | POST | withPermShare | 目标路径必须存在（Fs.Stat 校验） |
| `/api/share` | DELETE | withPermShare | Admin 或分享创建者本人 |
| `/api/public/share` | GET | withHashFile | 分享链接验证（密码/过期） |
| `/api/public/dl` | GET | withHashFile | 分享链接验证（密码/过期）+ 下载令牌 |
| `/api/search` | GET | withUser | 无额外权限（通过 Fs+Check 限制范围） |
| `/api/usage` | GET | withUser | 无额外权限 |
| `/api/tus` | POST | withUser | `Create` + `Check(path)`；覆盖需 `Modify` |
| `/api/tus` | HEAD/GET | withUser | `Create` + `Check(path)` |
| `/api/tus` | PATCH | withUser | `Create` + `Check(path)` |
| `/api/tus` | DELETE | withUser | `Delete` + 路径≠`/` |

---

## 4. 资源访问限制的三层防线

### 4.1 第一层：文件系统作用域隔离（Scope + ScopedFs）

**构建过程**（[users.go#Clean](file:///d:/fz/0601/solo-dogfeeding/code/166-filebrowser/users/users.go#L92-L98)）：
```
scope = filepath.Join(baseScope, filepath.Join("/", userScope))
user.Fs = files.NewScopedFs(afero.NewOsFs(), scope)
```
- `baseScope` = `server.Root`（服务器根目录）
- `userScope` = 用户配置的 Scope（如 `/users/alice`）
- 最终路径 = `server.Root` + 清理后的用户 Scope

**ScopedFs 的双重保护**（[scoped.go](file:///d:/fz/0601/solo-dogfeeding/code/166-filebrowser/files/scoped.go)）：

1. **词法限制**（afero.BasePathFs）：所有文件操作自动限制在 base 目录下，无法通过 `../` 逃逸
2. **符号链接限制**（ScopedFs.guard + within）：每次操作前检查目标路径解析后的实际位置是否在 scope 内
   - 阻止 scope 内的符号链接指向 scope 外
   - 新文件验证最近已存在的祖先目录
   - 悬空符号链接按其所在目录验证（best-effort）

**所有涉及文件操作的 handler 都通过 `d.user.Fs` 访问文件**，因此 Scope 是硬隔离——即使用户有权限也无法访问 Scope 外的文件。

### 4.2 第二层：路径规则过滤（Rules + Check）

**检查入口**：[data.Check](file:///d:/fz/0601/solo-dogfeeding/code/166-filebrowser/http/data.go#L37-L64)

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

**Check() 在以下场景被调用**：
- 创建文件/上传（POST /resources, POST /tus）
- 修改文件（PUT /resources）
- 移动/重命名（PATCH /resources，源和目标都检查）
- 递归列举（GET /resources/recursive）
- 目录下载/打包（raw handler 中的 getFiles）
- 搜索（通过 Checker 接口传入 search.Search）
- 创建分享时通过 `Fs.Stat` 间接使用 ScopedFs

### 4.3 第三层：操作权限位（Permissions）

即用户的 `Perm` 字段，8 个布尔位精确控制每个操作类型。详见上文 §1.2 和 §3.2。

---

## 5. 分享（Share）场景的权限判断

### 5.1 创建分享

前置条件（[share.go#withPermShare](file:///d:/fz/0601/solo-dogfeeding/code/166-filebrowser/http/share.go#L20-L28)）：
1. 用户已认证（withUser）
2. `Perm.Share = true` 且 `Perm.Download = true`

额外限制（[share.go#sharePostHandler](file:///d:/fz/0601/solo-dogfeeding/code/166-filebrowser/http/share.go#L100-L180)）：
- 目标路径必须存在（`d.user.Fs.Stat`），这同时通过 ScopedFs 阻止了符号链接逃逸 scope 的分享

### 5.2 访问分享（公开端点）

流程（[public.go#withHashFile](file:///d:/fz/0601/solo-dogfeeding/code/166-filebrowser/http/public.go#L17-L98)）：

```
1. 从 URL 提取分享 Hash
2. 查询 Link 记录
3. authenticateShareRequest():
   - 无密码 → 直接通过
   - 有密码 → 验证 X-SHARE-PASSWORD 头（bcrypt比对）或 URL token
   - 过期检查（Expire 字段）
4. 加载创建者用户 → 检查 Perm.Share && Perm.Download
5. 重新绑定 Fs：
   - d.user.Fs = files.NewScopedFs(d.user.Fs, link.Path)
   - 将 Fs 根缩小到分享的文件/目录
6. 设置 checkerPrefix = link.Path（保证规则匹配正确）
7. 执行业务 handler
```

**关键安全措施**：
- Fs 被二次封装：访问者只能看到分享路径下的内容
- 符号链接限制仍然生效（Nested ScopedFs）
- 创建者用户如果被禁用 Share/Download 权限，分享立即失效

### 5.3 下载密码保护的分享

密码保护的分享会生成一个 `Token`。下载时可以通过：
- URL 查询参数 `?token=<token>` 直接下载（避免重复输入密码）
- `X-SHARE-PASSWORD` 请求头输入密码

验证使用 `crypto/subtle.ConstantTimeCompare` 防止时序攻击（[public.go#authenticateShareRequest](file:///d:/fz/0601/solo-dogfeeding/code/166-filebrowser/http/public.go#L136-L161)）。

---

## 6. 命令执行的权限判断

三层检查（[commands.go#commandsHandler](file:///d:/fz/0601/solo-dogfeeding/code/166-filebrowser/http/commands.go#L41-L119)）：

```
1. d.server.EnableExec → 全局开关（Server 配置）
2. d.user.Perm.Execute → 用户权限位
3. 命令名 in d.user.Commands → 命令白名单
```

命令工作目录 = `d.user.FullPath(r.URL.Path)`，即用户 Scope 下的请求路径。

命令定义来自 `Settings.Commands`（map[string][]string），如 `{"git": ["git", "--no-pager"]}`。通过 [runner.ParseCommand](file:///d:/fz/0601/solo-dogfeeding/code/166-filebrowser/runner/parser.go) 解析。

---

## 7. 用户管理的权限判断

### 7.1 非管理员可修改的字段

定义于 [users.go#NonModifiableFieldsForNonAdmin](file:///d:/fz/0601/solo-dogfeeding/code/166-filebrowser/http/users.go#L22-L23)：

```
Username, Scope, LockPassword, Perm, Commands, Rules
```

非管理员尝试修改这些字段时返回 403。

### 7.2 密码修改限制

- `LockPassword = true` 时，非管理员无法修改密码
- JSON Auth 模式下，修改敏感字段（username/password/scope/lockPassword/commands/perm）需要提供当前密码

### 7.3 唯一管理员保护

[storage.go#IsUniqueAdmin](file:///d:/fz/0601/solo-dogfeeding/code/166-filebrowser/users/storage.go#L142-L152)：如果用户是管理员且系统中只剩 1 个管理员，则禁止删除该用户。

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
  │       ├─ 验证分享（密码/过期）
  │       ├─ 加载创建者用户
  │       ├─ 检查创建者 Perm.Share + Perm.Download
  │       ├─ 重新绑定 Fs（二次 Scope 限制）
  │       └─ 执行 handler
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
              │     └─ 不满足 → 403 Forbidden
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
| 用户模型 | [users/users.go](file:///d:/fz/0601/solo-dogfeeding/code/166-filebrowser/users/users.go) | User 结构体、Clean()、FullPath() |
| 权限位 | [users/permissions.go](file:///d:/fz/0601/solo-dogfeeding/code/166-filebrowser/users/permissions.go) | Permissions 结构体（8个布尔位） |
| 规则模型 | [rules/rules.go](file:///d:/fz/0601/solo-dogfeeding/code/166-filebrowser/rules/rules.go) | Rule 结构体、Matches()、MatchHidden() |
| 作用域文件系统 | [files/scoped.go](file:///d:/fz/0601/solo-dogfeeding/code/166-filebrowser/files/scoped.go) | ScopedFs、guard()、within()（防符号链接逃逸） |
| 认证接口 | [auth/auth.go](file:///d:/fz/0601/solo-dogfeeding/code/166-filebrowser/auth/auth.go) | Auther 接口 |
| JSON 认证 | [auth/json.go](file:///d:/fz/0601/solo-dogfeeding/code/166-filebrowser/auth/json.go) | JSONAuth、防时序攻击 |
| 代理认证 | [auth/proxy.go](file:///d:/fz/0601/solo-dogfeeding/code/166-filebrowser/auth/proxy.go) | ProxyAuth、自动创建用户 |
| Hook 认证 | [auth/hook.go](file:///d:/fz/0601/solo-dogfeeding/code/166-filebrowser/auth/hook.go) | HookAuth、外部命令、管理员自动获得全部权限 |
| 无认证 | [auth/none.go](file:///d:/fz/0601/solo-dogfeeding/code/166-filebrowser/auth/none.go) | NoAuth、固定 ID=1 |
| HTTP 认证中间件 | [http/auth.go](file:///d:/fz/0601/solo-dogfeeding/code/166-filebrowser/http/auth.go) | withUser、withAdmin、JWT 处理、注册限制 |
| HTTP 数据上下文 | [http/data.go](file:///d:/fz/0601/solo-dogfeeding/code/166-filebrowser/http/data.go) | data 结构体、Check()（规则评估） |
| 资源操作 | [http/resource.go](file:///d:/fz/0601/solo-dogfeeding/code/166-filebrowser/http/resource.go) | CRUD handler 中的权限检查 |
| 分享操作 | [http/share.go](file:///d:/fz/0601/solo-dogfeeding/code/166-filebrowser/http/share.go) | withPermShare、分享 CRUD |
| 公开访问 | [http/public.go](file:///d:/fz/0601/solo-dogfeeding/code/166-filebrowser/http/public.go) | withHashFile、分享验证、Fs 重新绑定 |
| 命令执行 | [http/commands.go](file:///d:/fz/0601/solo-dogfeeding/code/166-filebrowser/http/commands.go) | 三层命令权限检查 |
| 用户管理 | [http/users.go](file:///d:/fz/0601/solo-dogfeeding/code/166-filebrowser/http/users.go) | withSelfOrAdmin、非管理员字段限制 |
| 全局设置 | [http/settings.go](file:///d:/fz/0601/solo-dogfeeding/code/166-filebrowser/http/settings.go) | withAdmin 保护 |
| TUS 上传 | [http/tus_handlers.go](file:///d:/fz/0601/solo-dogfeeding/code/166-filebrowser/http/tus_handlers.go) | 上传/续传权限检查 |
| 分享模型 | [share/share.go](file:///d:/fz/0601/solo-dogfeeding/code/166-filebrowser/share/share.go) | Link 结构体 |
| 全局配置 | [settings/settings.go](file:///d:/fz/0601/solo-dogfeeding/code/166-filebrowser/settings/settings.go) | Settings、Server 结构体 |
| 默认值 | [settings/defaults.go](file:///d:/fz/0601/solo-dogfeeding/code/166-filebrowser/settings/defaults.go) | UserDefaults.Apply() |
| 用户存储 | [users/storage.go](file:///d:/fz/0601/solo-dogfeeding/code/166-filebrowser/users/storage.go) | 唯一管理员保护、用户 CRUD |
| 错误定义 | [errors/errors.go](file:///d:/fz/0601/solo-dogfeeding/code/166-filebrowser/errors/errors.go) | ErrPermissionDenied、ErrShareRequiresDownload 等 |
