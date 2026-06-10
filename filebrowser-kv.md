# File Browser bbolt KV 存储抽象深度分析

## 一、整体架构分层

File Browser 基于 **bbolt + storm** 组合实现 KV 存储，采用五层架构：

```
┌─────────────────────────────────────────────────────┐
│  领域存储层 (Domain Storage)                        │
│  users.Storage / settings.Storage / auth.Storage    │
│  share.Storage                                      │
│  职责：业务校验、默认值填充、缓存、过期清理          │
├─────────────────────────────────────────────────────┤
│  后端接口层 (StorageBackend)                        │
│  users.StorageBackend / settings.StorageBackend     │
│  share.StorageBackend / auth.StorageBackend         │
│  职责：定义各领域数据访问接口契约                    │
├─────────────────────────────────────────────────────┤
│  Bolt 实现层 (Bolt Backend)                         │
│  storage/bolt/*.go                                  │
│  职责：封装 storm/bbolt 操作，实现具体读写逻辑       │
├─────────────────────────────────────────────────────┤
│  ORM 层 (storm v3 v3.2.1)                          │
│  github.com/asdine/storm/v3                         │
│  职责：结构体→KV 映射、索引管理、查询构建           │
├─────────────────────────────────────────────────────┤
│  存储引擎 (bbolt v1.4.3)                           │
│  go.etcd.io/bbolt                                   │
│  职责：B+Tree KV 存储、ACID 事务、mmap 文件映射     │
└─────────────────────────────────────────────────────┘
```

**核心文件**：

| 文件 | 职责 |
|------|------|
| [storage/bolt/bolt.go](storage/bolt/bolt.go#L13-L31) | 入口装配，组装全部 backend |
| [storage/bolt/utils.go](storage/bolt/utils.go#L11-L22) | 配置类 KV 的 get/save 工具函数 |
| [storage/storage.go](storage/storage.go#L10-L17) | 顶层 Storage 聚合体 |

---

## 二、用户索引机制

### 2.1 User 结构体的 storm 标签

[users/users.go](users/users.go#L21-L39) 中定义了两个 storm 标签，决定索引结构：

```go
type User struct {
    ID       uint   `storm:"id,increment"`  // 主键，自增
    Username string `storm:"unique"`        // 唯一索引
    Password string
    Scope    string
    // ... 其余字段无 storm 标签，不建索引
}
```

**storm 标签含义**：

| 标签 | 含义 | bbolt 映射 |
|------|------|-----------|
| `storm:"id,increment"` | 主键，storm 自动分配递增 ID | 数据键使用 `bucket.NextSequence()` 生成的字节 |
| `storm:"unique"` | 唯一索引，写入时强制唯一约束 | 创建 `__storm.User.Username` 子 bucket |

### 2.2 bbolt 中的 User 索引结构

storm 在 bbolt 中为 User 创建如下结构：

```
Bucket: "User"                              ← 数据 bucket
  ├─ Key: [1] (big-endian uint64)           → Value: codec.Marshal(User{ID:1, ...})
  ├─ Key: [2]                               → Value: codec.Marshal(User{ID:2, ...})
  └─ Key: [3]                               → Value: codec.Marshal(User{ID:3, ...})

Bucket: "__storm.User.ID"                   ← 主键索引 bucket (id 标签自动创建)
  └─ 与数据 bucket 键值相同，用于 One("ID", ...) 查询

Bucket: "__storm.User.Username"             ← 唯一索引 bucket (unique 标签)
  ├─ Key: "admin"                           → Value: [1] (指向数据键)
  └─ Key: "user1"                           → Value: [2] (指向数据键)
```

### 2.3 索引查询的工作流程

**按 ID 查询** — [users.go](storage/bolt/users.go#L19-L42)：

```go
func (st usersBackend) GetBy(i interface{}) (user *users.User, err error) {
    switch i.(type) {
    case uint:   arg = "ID"        // → 查主键索引
    case string: arg = "Username"  // → 查唯一索引
    }
    err = st.db.One(arg, i, user)  // storm 自动走索引 bucket
}
```

`db.One("ID", uint(1), user)` 底层流程：
1. 开启 bbolt 只读事务 `tx.Bucket("User")`
2. 从 `__storm.User.ID` 索引 bucket 查找键 `1`，获取数据键
3. 用数据键从 `User` bucket 读取值
4. 用 codec（默认 protobuf）反序列化为 `users.User`

`db.One("Username", "admin", user)` 底层流程：
1. 从 `__storm.User.Username` 索引 bucket 查找键 `"admin"`，获取数据键 `[1]`
2. 用数据键从 `User` bucket 读取并反序列化

**无索引字段查询**：对 Password、Scope 等字段调用 `db.One()` 会触发全 bucket 扫描，但代码中未使用此模式。

### 2.4 CountAdmins 的直接 bbolt 操作

[users.go](storage/bolt/users.go#L98-L122) 中 `CountAdmins()` 是唯一绕过 storm 查询 API 直接操作 bbolt 的场景：

```go
err := st.db.Bolt.View(func(tx *bolt.Tx) error {
    bucket := tx.Bucket([]byte(reflect.TypeOf(users.User{}).Name()))
    // ↑ 等价于 tx.Bucket([]byte("User"))
    
    c := bucket.Cursor()
    for _, v := c.First(); v != nil; _, v = c.Next() {
        var u users.User
        st.db.Codec().Unmarshal(v, &u)  // 手动反序列化
        if u.Perm.Admin { count++ }
    }
    return nil
})
```

**为什么绕过 storm**：storm 的 `db.Select(q.Eq("Perm.Admin", true))` 会创建临时索引并扫描全表，不如直接遍历原始 bucket 高效。且 `Perm` 是嵌套结构体 `Permissions` 的字段，storm 对嵌套字段的查询支持有限。

---

## 三、认证配置的多态存储

### 3.1 Auther 接口与四种实现

[auth/auth.go](auth/auth.go#L10-L14) 定义接口：

```go
type Auther interface {
    Auth(r *http.Request, usr users.Store, stg *settings.Settings, srv *settings.Server) (*users.User, error)
    LoginPage() bool
}
```

四种实现：

| 类型 | 标识常量 | 结构体 | 持久化字段 |
|------|----------|--------|-----------|
| JSON 认证 | `MethodJSONAuth = "json"` | [JSONAuth](auth/json.go#L28-L30) | `ReCaptcha *ReCaptcha` |
| 代理认证 | `MethodProxyAuth = "proxy"` | [ProxyAuth](auth/proxy.go#L16-L18) | `Header string` |
| Hook 认证 | `MethodHookAuth = "hook"` | [HookAuth](auth/hook.go#L29-L36) | `Command string` |
| 无认证 | `MethodNoAuth = "noauth"` | [NoAuth](auth/none.go#L14) | 无字段 |

### 3.2 多态序列化/反序列化机制

[auth.go](storage/bolt/auth.go#L15-L36) 的 `Get()` 方法实现了类型驱动的反序列化：

```go
func (s authBackend) Get(t settings.AuthMethod) (auth.Auther, error) {
    var auther auth.Auther

    switch t {
    case auth.MethodJSONAuth:  auther = &auth.JSONAuth{}
    case auth.MethodProxyAuth: auther = &auth.ProxyAuth{}
    case auth.MethodHookAuth:  auther = &auth.HookAuth{}
    case auth.MethodNoAuth:    auther = &auth.NoAuth{}
    default:                   return nil, fberrors.ErrInvalidAuthMethod
    }

    return auther, get(s.db, "auther", auther)
    // ↑ 将字节流反序列化到具体类型的指针中
}
```

**关键设计**：

1. **调用方必须先知道认证类型**：`t` 参数来自 `settings.Settings.AuthMethod` 字段
2. **类型选择在前，反序列化在后**：先根据 `AuthMethod` 创建对应类型的零值指针，再让 storm 将字节流解码到该指针
3. **存储路径**：无论哪种类型，都存储在 `config` bucket 的 `auther` 键下，**同一时刻只有一种认证配置**

### 3.3 bbolt 中的认证数据存储

```
Bucket: "config"
  ├─ Key: "auther"    → Value: codec.Marshal(JSONAuth{ReCaptcha:...})
  │                     或 codec.Marshal(ProxyAuth{Header:"X-User"})
  │                     或 codec.Marshal(HookAuth{Command:"/bin/auth.sh"})
  │                     或 codec.Marshal(NoAuth{})
  └─ Key: "settings"  → Value: codec.Marshal(Settings{AuthMethod:"json", ...})
                          ↑ auther 键的类型由这里决定
```

**读取链路**：`Settings.Get()` → 取出 `AuthMethod` → `Auth.Get(AuthMethod)` → 反序列化对应类型

**写入链路**：`Auth.Save(auther)` → `save(db, "auther", auther)` → codec 自动按实际类型序列化

### 3.4 HookAuth 的特殊处理

[HookAuth](auth/hook.go#L29-L36) 中有三个字段标记为 `json:"-"`：

```go
type HookAuth struct {
    Users    users.Store        `json:"-"`  // 运行时注入，不持久化
    Settings *settings.Settings `json:"-"`  // 运行时注入，不持久化
    Server   *settings.Server   `json:"-"`  // 运行时注入，不持久化
    Cred     hookCred           `json:"-"`  // 运行时状态，不持久化
    Fields   hookFields         `json:"-"`  // 运行时状态，不持久化
    Command  string             `json:"command"`  // 唯一持久化字段
}
```

storm 的 codec（基于 protobuf 或 JSON）会忽略 `json:"-"` 标签的字段，因此只有 `Command` 被写入 bbolt。运行时依赖通过 `Auth()` 方法参数注入。

---

## 四、分享数据映射

### 4.1 Link 结构体的 storm 标签

[share/share.go](share/share.go#L10-L20)：

```go
type Link struct {
    Hash         string `storm:"id,index"`  // 主键 + 额外索引
    Path         string `storm:"index"`     // 普通索引
    UserID       uint                        // 无索引
    Expire       int64                        // 无索引
    PasswordHash string                       // 无索引
    Token        string                       // 无索引
}
```

**`storm:"id,index"` 的含义**：
- `id`：指定 Hash 为主键（不同于 User 的自增 ID，Hash 是业务主键）
- `index`：额外创建索引 bucket，用于加速 `db.One("Hash", ...)` 查询

**`storm:"index"` 的含义**：
- 为 Path 创建索引 bucket，支持 `db.Prefix("Path", ...)` 前缀查询

### 4.2 bbolt 中的 Link 索引结构

```
Bucket: "Link"                              ← 数据 bucket
  ├─ Key: "u1-a"                            → Value: codec.Marshal(Link{Hash:"u1-a", Path:"/a", UserID:1, ...})
  ├─ Key: "u1-abc"                          → Value: codec.Marshal(Link{Hash:"u1-abc", Path:"/abc", UserID:1, ...})
  └─ Key: "u2-a"                            → Value: codec.Marshal(Link{Hash:"u2-a", Path:"/a", UserID:2, ...})

Bucket: "__storm.Link.Hash"                 ← Hash 索引 (id,index)
  ├─ Key: "u1-a"                            → Value: "u1-a" (指向数据键)
  └─ Key: "u2-a"                            → Value: "u2-a" (指向数据键)

Bucket: "__storm.Link.Path"                 ← Path 索引 (index)
  ├─ Key: "/a"                              → Value: ["u1-a", "u2-a"] (该路径下所有链接)
  └─ Key: "/abc"                            → Value: ["u1-abc"]
```

### 4.3 查询方式与索引使用

[share.go](storage/bolt/share.go) 中的各查询方法使用了不同的 storm 查询 API：

| 方法 | storm API | 索引使用 | 说明 |
|------|-----------|----------|------|
| `GetByHash(hash)` | `db.One("Hash", hash, &v)` | Hash 索引 | 精确匹配，O(1) |
| `FindByUserID(id)` | `db.Select(q.Eq("UserID", id)).Find(&v)` | **无索引**，全表扫描 | UserID 无 storm 标签 |
| `GetPermanent(path, id)` | `db.Select(q.Eq("Path", path), q.Eq("Expire", 0), q.Eq("UserID", id)).First(&v)` | Path 索引 + 内存过滤 | Expire/UserID 无索引 |
| `Gets(path, id)` | `db.Select(q.Eq("Path", path), q.Eq("UserID", id)).Find(&v)` | Path 索引 + 内存过滤 | UserID 无索引 |
| `All()` | `db.All(&v)` | 无索引，全表遍历 | 简单全量读取 |
| `DeleteWithPathPrefix(prefix, uid)` | `db.Prefix("Path", prefix, &links)` | Path 索引前缀扫描 | 利用 Path 索引的有序性 |

### 4.4 DeleteWithPathPrefix 的两阶段过滤

[share.go](storage/bolt/share.go#L80-L104)：

```go
func (s shareBackend) DeleteWithPathPrefix(pathPrefix string, userID uint) error {
    var links []share.Link
    s.db.Prefix("Path", pathPrefix, &links)    // 第一阶段：Path 索引前缀扫描

    prefix := strings.TrimRight(pathPrefix, "/")
    for _, link := range links {
        if link.UserID != userID { continue }   // 第二阶段：内存过滤 UserID
        if link.Path != prefix && !strings.HasPrefix(link.Path, prefix+"/") { continue }
        s.db.DeleteStruct(&share.Link{Hash: link.Hash})  // 逐条删除
    }
    return err
}
```

**设计细节**：
- `db.Prefix()` 利用 Path 索引的 B+Tree 有序性做前缀扫描，避免全表扫描
- 但前缀匹配可能命中 `/a` 匹配到 `/abc`（字节前缀），需要在内存中二次验证路径层级
- 每条删除是独立写事务，非原子操作

---

## 五、版本键含义

### 5.1 version=2 的写入

[bolt.go](storage/bolt/bolt.go#L20-L23)：

```go
func NewStorage(db *storm.DB) (*storage.Storage, error) {
    // ... 初始化各 backend
    err := save(db, "version", 2)  // 写入 config/version = 2
    // ...
}
```

存储位置：`Bucket("config").Key("version") → Value: 2 (int, codec 编码)`

### 5.2 version=2 的含义推测

代码中没有对 version 的读取逻辑，但从设计意图推断：

- **version 1**：早期版本，可能对应无 `UserHomeBasePath`、无 `Tus` 配置、无 `MinimumPasswordLength` 等字段的旧数据库
- **version 2**：当前版本，引入了 `UserDefaults`、`Branding`、`Tus`、`Commands` 等复杂结构体字段

### 5.3 version 键的实际作用

**现状**：version 值只写不读，是一个**死代码标志**：
- 每次启动都覆盖为 `2`，无法区分数据库原始版本
- 没有 `if version < 2 { migrate() }` 之类的升级逻辑
- 迁移完全依赖读取时默认值填充（隐式迁移）

**潜在风险**：如果未来引入 version 3 的破坏性变更，现有的 version=2 写入逻辑会覆盖旧数据库的版本信息，导致无法判断数据库原始版本。

---

## 六、bbolt KV 读写封装深度分析

### 6.1 两种存储模式的底层映射

#### 模式 A：配置类 KV（`config` bucket 键值对）

封装于 [utils.go](storage/bolt/utils.go#L11-L22)：

```go
func get(db *storm.DB, name string, to interface{}) error {
    err := db.Get("config", name, to)
    // storm.Get 底层：
    //   tx.Bucket("config").Get([]byte(name))  →  bbolt 只读事务
    //   codec.Unmarshal(value, to)
}

func save(db *storm.DB, name string, from interface{}) error {
    return db.Set("config", name, from)
    // storm.Set 底层：
    //   tx.Bucket("config").Put([]byte(name), codec.Marshal(from))  →  bbolt 写事务
}
```

**bbolt 视角的完整操作**：

| 上层调用 | storm API | bbolt 操作 | 事务类型 |
|----------|-----------|-----------|----------|
| `get(db, "settings", &set)` | `db.Get("config", "settings", &set)` | `tx.Bucket("config").Get([]byte("settings"))` + Unmarshal | 只读事务 |
| `save(db, "settings", set)` | `db.Set("config", "settings", set)` | `tx.Bucket("config").Put([]byte("settings"), Marshal(set))` | 写事务 |
| `save(db, "version", 2)` | `db.Set("config", "version", 2)` | `tx.Bucket("config").Put([]byte("version"), Marshal(2))` | 写事务 |

#### 模式 B：结构化实体存储（ORM 映射）

封装于 [users.go](storage/bolt/users.go)、[share.go](storage/bolt/share.go)：

| 上层调用 | storm API | bbolt 操作 | 事务类型 |
|----------|-----------|-----------|----------|
| `st.db.One("ID", 1, user)` | 按主键查 | 索引 bucket 查 → 数据 bucket 读 + Unmarshal | 只读事务 |
| `st.db.One("Username", "admin", user)` | 按唯一索引查 | 唯一索引 bucket 查 → 数据 bucket 读 + Unmarshal | 只读事务 |
| `st.db.All(&allUsers)` | 全量遍历 | `bucket.Cursor()` 遍历所有键值 + 逐条 Unmarshal | 只读事务 |
| `st.db.Save(user)` | 新增保存 | `bucket.NextSequence()` + `bucket.Put(seq, Marshal(user))` + 更新索引 | 写事务 |
| `st.db.UpdateField(user, "Password", val)` | 单字段更新 | 读取旧值 → 合并新字段值 → `bucket.Put` + 更新索引 | 写事务 |
| `st.db.DeleteStruct(user)` | 删除实体 | `bucket.Delete(dataKey)` + 清除索引条目 | 写事务 |
| `s.db.Select(q.Eq(...)).Find(&v)` | 条件查询 | 遍历数据 + 内存过滤（无索引字段）或走索引 | 只读事务 |
| `s.db.Prefix("Path", prefix, &links)` | 前缀查询 | Path 索引 bucket 的 `Cursor().Seek(prefix)` 前向扫描 | 只读事务 |

### 6.2 storm 的 codec 序列化

storm 默认使用 [github.com/asdine/storm/v3/codec/json](https://github.com/asdine/storm) 作为 codec（可配置为 protobuf、msgpack 等）。本项目使用默认 JSON codec。

**这意味着 bbolt 中存储的 Value 是 JSON 字节流**，而非二进制协议。例如：

```json
// Bucket: "User", Key: [1], Value:
{
  "id": 1,
  "username": "admin",
  "password": "$2a$10$...",
  "scope": ".",
  "locale": "en",
  "lockPassword": false,
  "viewMode": "mosaic",
  "singleClick": false,
  "redirectAfterCopyMove": true,
  "perm": {"admin":true, "execute":true, "create":true, ...},
  "commands": [],
  "sorting": {"by":"name", "asc":true},
  "rules": [],
  "hideDotfiles": false,
  "dateFormat": false,
  "aceEditorTheme": ""
}
```

### 6.3 领域层的读写增强

领域 Storage 层在 Bolt Backend 之上增加了业务逻辑：

**读取增强**：

| 领域 | 方法 | 增强逻辑 | 文件 |
|------|------|----------|------|
| Settings | `Get()` | 零值默认填充（UserHomeBasePath、MinimumPasswordLength、Tus、FileMode、DirMode） | [settings/storage.go](settings/storage.go#L28-L62) |
| Users | `Get(baseScope, id)` | `user.Clean(baseScope)` 设置文件系统作用域 | [users/storage.go](users/storage.go#L48-L57) |
| Share | `All()` / `FindByUserID()` / `GetByHash()` | 过期链接惰性删除 | [share/storage.go](share/storage.go#L32-L86) |

**写入增强**：

| 领域 | 方法 | 增强逻辑 | 文件 |
|------|------|----------|------|
| Settings | `Save()` | Key 非空校验、空 slice 规范化（nil → []）、Commands 默认事件填充 | [settings/storage.go](settings/storage.go#L73-L118) |
| Settings | `SaveServer()` | `server.Clean()` 清理 BaseURL 尾部斜杠 | [settings/storage.go](settings/storage.go#L126-L129) |
| Users | `Save(user)` | `user.Clean("")` 校验 Username/Password 非空、规范化空 slice | [users/storage.go](users/storage.go#L94-L100) |
| Users | `Update(user, fields...)` | `user.Clean("", fields...)` + 更新内存时间戳缓存 `s.updated[ID]` | [users/storage.go](users/storage.go#L76-L91) |
| Users | `Delete(id)` | 禁止删除唯一管理员（`IsUniqueAdmin` → `CountAdmins`） | [users/storage.go](users/storage.go#L105-L130) |

---

## 七、事务边界深度分析

### 7.1 storm 隐式事务模型

storm 的每个公共方法内部自动管理 bbolt 事务：

```
storm.Save(entity)
  └─ db.Bolt.Update(func(tx *bolt.Tx) error {   ← 写事务开始
         bucket := tx.Bucket("User")
         bucket.Put(key, value)                  ← 写入数据
         indexBucket.Put(indexKey, dataKey)      ← 写入索引
         return nil                               ← 事务提交
     })
```

```
storm.One(fieldName, value, entity)
  └─ db.Bolt.View(func(tx *bolt.Tx) error {     ← 只读事务开始
         bucket := tx.Bucket("User")
         indexBucket.Get(value)                   ← 查索引
         bucket.Get(dataKey)                      ← 取数据
         codec.Unmarshal(data, entity)            ← 反序列化
         return nil                               ← 事务结束
     })
```

**关键特性**：
- bbolt 同一时刻只允许一个写事务（全局写锁）
- 多个只读事务可以并发执行
- storm 方法返回后事务已提交/回滚，外部无法控制事务边界

### 7.2 显式事务：唯一场景

[users.go](storage/bolt/users.go#L98-L122) 的 `CountAdmins()` 是代码中唯一显式使用 bbolt 原生事务的地方：

```go
st.db.Bolt.View(func(tx *bolt.Tx) error {  // 显式只读事务
    bucket := tx.Bucket([]byte("User"))
    c := bucket.Cursor()
    for _, v := c.First(); v != nil; _, v = c.Next() {
        // 整个遍历在单一事务快照内完成
    }
    return nil
})
```

**为什么必须显式**：如果使用 storm 的 `db.Select(q.Eq("Perm.Admin", true))`，storm 内部也会开启只读事务，但需要先构建临时索引，开销更大。直接使用 bbolt Cursor 遍历保证：
1. 整个遍历过程在同一事务快照内，数据一致性有保证
2. 避免 storm 构建临时索引的额外 I/O
3. 可以手动控制反序列化，跳过不需要的字段

### 7.3 多操作事务风险分析

#### 风险 1：quickSetup 非原子初始化

[cmd/root.go](cmd/root.go#L393-L502)：

```go
func quickSetup(v *viper.Viper, s *storage.Storage) error {
    s.Auth.Save(&auth.JSONAuth{})     // 事务 1：写 config/auther
    s.Settings.Save(set)              // 事务 2：写 config/settings
    s.Settings.SaveServer(ser)        // 事务 3：写 config/server
    s.Users.Save(user)                // 事务 4：写 User bucket
}
```

**风险**：如果事务 2 失败，auther 已写入但 settings 缺失，重启后系统无法正常工作。

**缓解**：quickSetup 仅在数据库文件不存在时执行，且 storm.Open 会创建新文件，失败概率低。

#### 风险 2：DeleteWithPathPrefix 部分失败

[share.go](storage/bolt/share.go#L80-L104)：

```go
for _, link := range links {
    err = errors.Join(err, s.db.DeleteStruct(&share.Link{Hash: link.Hash}))
    // 每条删除是独立写事务
}
```

**风险**：删除 3 条链接时，第 2 条失败 → 第 1 条已删除，第 3 条未删除 → 数据处于中间状态。

**影响**：残留的过期分享链接会在下次读取时被 `share.Storage` 的惰性清理逻辑捕获，最终一致性可接受。

#### 风险 3：users.Update 多字段非原子

[users.go](storage/bolt/users.go#L58-L75)：

```go
func (st usersBackend) Update(user *users.User, fields ...string) error {
    for _, field := range fields {
        if err := st.db.UpdateField(user, field, val); err != nil {
            return err  // 中途失败，部分字段已更新
        }
    }
}
```

**风险**：更新 `["Password", "Scope"]` 时，Password 更新成功但 Scope 失败 → 用户密码已变但作用域未变。

**缓解**：实际调用中大多只更新单字段（HTTP API 的 PATCH 语义），多字段更新场景极少。

### 7.4 事务边界完整总结

| 操作场景 | 事务数 | 原子性 | 失败影响 | 恢复机制 |
|----------|--------|--------|----------|----------|
| 配置读取 `get()` | 1 | ✓ | 无 | — |
| 配置写入 `save()` | 1 | ✓ | 旧值完整保留 | 重试即可 |
| 用户查询 `GetBy()` | 1 | ✓ | 无 | — |
| 用户保存 `Save()` | 1 | ✓ | 不创建新记录 | 重试即可 |
| 用户单字段更新 `UpdateField()` | 1 | ✓ | 旧值保留 | 重试即可 |
| 用户多字段更新 `Update(fields...)` | N | ✗ | 部分字段已更新 | 无自动恢复 |
| 管理员计数 `CountAdmins()` | 1 | ✓ | 无 | — |
| 分享链接批量删除 `DeleteWithPathPrefix()` | N | ✗ | 部分删除 | 下次读取惰性清理 |
| 初始化 `quickSetup()` | 4 | ✗ | 配置不完整 | 删除数据库重试 |
| 认证+设置联动保存 | 2 | ✗ | AuthMethod 与 auther 不一致 | 需手动修复 |

---

## 八、迁移影响分析

### 8.1 当前迁移策略：隐式默认值填充

系统不使用显式迁移脚本，而是在领域 Storage 层读取时补全缺失字段：

| 字段 | 默认值 | 何时引入 | 迁移逻辑位置 |
|------|--------|----------|-------------|
| `UserHomeBasePath` | `"/users"` | 新增字段 | [settings/storage.go](settings/storage.go#L34-L36) |
| `LogoutPage` | `"/login"` | 新增字段 | [settings/storage.go](settings/storage.go#L38-L40) |
| `MinimumPasswordLength` | `12` | 新增字段 | [settings/storage.go](settings/storage.go#L42-L44) |
| `Tus` | `{ChunkSize:Default, RetryCount:Default}` | 新增字段 | [settings/storage.go](settings/storage.go#L46-L51) |
| `FileMode` | `0640` | 新增字段 | [settings/storage.go](settings/storage.go#L53-L55) |
| `DirMode` | `0750` | 新增字段 | [settings/storage.go](settings/storage.go#L57-L59) |

**工作原理**：
1. 旧数据库中不存在这些字段 → JSON 反序列化后为零值（`""` / `0` / 零结构体）
2. `Storage.Get()` 检测零值并填充默认值
3. 默认值仅存在于内存，**不会自动写回数据库**
4. 只有用户通过 API 或 CLI 主动保存时，完整数据才会持久化

### 8.2 隐式迁移的局限性

#### 局限 1：无法处理字段类型变更

如果将 `MinimumPasswordLength` 从 `uint` 改为 `string`：
- 旧数据中的 uint 值反序列化到 string 字段会失败
- storm 会返回 codec 错误，导致 `Settings.Get()` 整体失败
- **无降级路径**，系统无法启动

#### 局限 2：无法处理字段语义变更

如果 `UserHomeBasePath` 的含义从绝对路径改为相对路径：
- 旧数据中的 `/users` 被当作相对路径处理
- 隐式迁移无法感知语义变化
- **静默数据错误**，用户目录可能指向错误位置

#### 局限 3：无法处理数据重组

如果将 `Settings.FileMode` 和 `Settings.DirMode` 合并为 `Settings.Modes{File, Dir}`：
- 旧数据中这两个字段是顶级的，新结构体期望嵌套
- JSON 反序列化时旧字段被忽略，新字段为零值
- **数据丢失**，文件模式回退到默认值

#### 局限 4：version 值被覆盖导致信息丢失

[bolt.go](storage/bolt/bolt.go#L20-L23) 每次启动都执行 `save(db, "version", 2)`：
- 假设用户从 version 1 升级，首次启动后 version 被覆盖为 2
- 未来如果需要执行 version 1→2 的迁移，已无法判断数据库原始版本
- **version 标志在第一次运行后即失效**

### 8.3 迁移可能带来的影响场景

#### 场景 A：新增字段（低风险）

```
旧数据库：{"key":"...", "signup":false}
新代码期望：{"key":"...", "signup":false, "tus":{"chunkSize":..., "retryCount":...}}
```

**结果**：`Tus` 字段反序列化为零结构体，`Storage.Get()` 填充默认值 → **正常工作**

#### 场景 B：删除字段（中风险）

```
旧数据库：{"key":"...", "oldField":"someValue"}
新代码期望：{"key":"..."}  // 无 oldField
```

**结果**：Go 结构体无 `oldField`，JSON 反序列化时忽略 → **正常工作，但旧数据残留**

残留影响：
- 数据库文件体积略大（可忽略）
- 如果未来重新引入同名字段，旧值会被恢复（可能是意外行为）

#### 场景 C：重命名字段（高风险）

```
旧数据库：{"hideLoginButton":true}
新代码期望：{"showLoginButton":false}  // 语义反转+重命名
```

**结果**：
- `hideLoginButton` 被忽略（旧值丢失）
- `showLoginButton` 为零值 `false`，语义恰好正确（但纯属巧合）
- 如果语义不是简单反转，**数据语义错误**

#### 场景 D：AuthMethod 变更（高风险）

```
旧数据库 config/auther：ProxyAuth{Header:"X-User"} (序列化为 JSON)
新代码 AuthMethod 变为 "hook"，但 auther 键仍是 ProxyAuth 的字节流
```

**结果**：
- `Auth.Get("hook")` 创建 `&HookAuth{}` 指针
- JSON 反序列化 ProxyAuth 数据到 HookAuth 结构体
- `Header` 字段被忽略，`Command` 字段为空 → **认证配置静默丢失**

### 8.4 对未来迁移的建议

1. **读取 version 再决定行为**：在 `NewStorage()` 中先 `get(db, "version", &ver)` 判断版本，再执行迁移，最后才写入新版本号
2. **引入显式迁移函数**：`migrateV1toV2(db)` 等函数处理破坏性变更
3. **利用 bbolt 原生事务做原子迁移**：`db.Bolt.Update(func(tx *bolt.Tx) error { ... })` 保证迁移的原子性
4. **增加 CLI 迁移命令**：`filebrowser db migrate` 允许用户手动触发迁移

---

## 九、完整数据映射总览

### 9.1 bbolt 数据库完整结构

```
filebrowser.db
│
├─ Bucket: "config"                          ← 配置类 KV 存储
│   ├─ Key: "settings"                       → JSON(Settings{Key, Signup, AuthMethod, ...})
│   ├─ Key: "server"                         → JSON(Server{Root, Port, Address, ...})
│   ├─ Key: "auther"                         → JSON(JSONAuth|ProxyAuth|HookAuth|NoAuth)
│   └─ Key: "version"                        → JSON(2)
│
├─ Bucket: "User"                            ← 用户实体存储
│   ├─ Key: [1]                              → JSON(User{ID:1, Username:"admin", ...})
│   ├─ Key: [2]                              → JSON(User{ID:2, Username:"user1", ...})
│   └─ ...
│
├─ Bucket: "__storm.User.ID"                 ← User 主键索引
│   └─ Key: [1] → Value: [1]
│
├─ Bucket: "__storm.User.Username"           ← User 唯一索引
│   ├─ Key: "admin"  → Value: [1]
│   └─ Key: "user1"  → Value: [2]
│
├─ Bucket: "Link"                            ← 分享链接实体存储
│   ├─ Key: "abc123"                         → JSON(Link{Hash:"abc123", Path:"/file", UserID:1, ...})
│   └─ ...
│
├─ Bucket: "__storm.Link.Hash"               ← Link 主键+索引
│   └─ Key: "abc123" → Value: "abc123"
│
└─ Bucket: "__storm.Link.Path"               ← Link Path 索引
    ├─ Key: "/file"  → Value: ["abc123"]
    └─ ...
```

### 9.2 存储路径与代码对照

| 数据 | Bucket | Key | 写入 | 读取 |
|------|--------|-----|------|------|
| Settings | `config` | `"settings"` | [config.go](storage/bolt/config.go#L18-L20) | [config.go](storage/bolt/config.go#L13-L16) |
| Server | `config` | `"server"` | [config.go](storage/bolt/config.go#L27-L29) | [config.go](storage/bolt/config.go#L22-L25) |
| Auther | `config` | `"auther"` | [auth.go](storage/bolt/auth.go#L34-L36) | [auth.go](storage/bolt/auth.go#L15-L32) |
| Version | `config` | `"version"` | [bolt.go](storage/bolt/bolt.go#L20-L23) | 无读取 |
| User | `User` | 自增 `[N]` | [users.go](storage/bolt/users.go#L77-L83) | [users.go](storage/bolt/users.go#L19-L42) |
| Link | `Link` | `Hash` 值 | [share.go](storage/bolt/share.go#L68-L70) | [share.go](storage/bolt/share.go#L38-L46) |

### 9.3 数据库初始化与连接

[cmd/utils.go](cmd/utils.go#L157-L194) 中的 `withViperAndStore` 函数是数据库生命周期管理入口：

```go
// 连接数据库
db, err := storm.Open(path, storm.BoltOptions(databasePermissions, nil))
// storm.Open 底层：bbolt.Open(path, 0640, &Options{Timeout: 1s})

// 创建存储层
storage, err := bolt.NewStorage(db)
// NewStorage 内部：写入 version=2，组装四大 backend

// 使用完毕
defer db.Close()
// db.Close 底层：bbolt DB.Close()，等待所有事务完成，同步 mmap，关闭文件
```
