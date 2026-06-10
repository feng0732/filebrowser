# File Browser bbolt KV 存储抽象分析

## 一、整体架构分层

File Browser 使用 **三层架构** 实现 KV 存储抽象，基于 bbolt + storm 组合：

```
┌─────────────────────────────────────────────────────┐
│  领域存储层 (Domain Storage)                        │
│  users.Storage / settings.Storage / auth.Storage    │
│  share.Storage                                      │
│  职责：业务校验、默认值填充、缓存、过期清理          │
├─────────────────────────────────────────────────────┤
│  后端接口层 (StorageBackend)                        │
│  users.StorageBackend / settings.StorageBackend     │
│  职责：定义各领域数据访问接口契约                    │
├─────────────────────────────────────────────────────┤
│  Bolt 实现层 (Bolt Backend)                         │
│  storage/bolt/*.go                                  │
│  职责：封装 storm/bbolt 操作，实现具体读写逻辑       │
├─────────────────────────────────────────────────────┤
│  ORM 层 (storm v3)                                  │
│  github.com/asdine/storm/v3                         │
│  职责：结构化数据到 KV 的映射、索引管理             │
├─────────────────────────────────────────────────────┤
│  存储引擎 (bbolt)                                   │
│  go.etcd.io/bbolt                                   │
│  职责：底层 KV 存储、事务、B+Tree 索引              │
└─────────────────────────────────────────────────────┘
```

**核心文件**：

- 入口装配：[bolt.go](file:///d:/fz/0601/solo-dogfeeding/code/167-filebrowser/storage/bolt/bolt.go#L1-L31)
- 基础工具：[utils.go](file:///d:/fz/0601/solo-dogfeeding/code/167-filebrowser/storage/bolt/utils.go#L1-L22)
- 存储接口：[storage.go](file:///d:/fz/0601/solo-dogfeeding/code/167-filebrowser/storage/storage.go#L1-L17)

---

## 二、读写封装机制

### 2.1 两种数据存储模式

根据数据特性不同，采用两种截然不同的存储模式：

#### 模式 A：配置类 KV 存储（键值对）

**适用数据**：settings、server、auther 等全局单例配置

**封装位置**：[utils.go](file:///d:/fz/0601/solo-dogfeeding/code/167-filebrowser/storage/bolt/utils.go#L11-L22)

```go
// 读：从 "config" bucket 按键读取
func get(db *storm.DB, name string, to interface{}) error {
    err := db.Get("config", name, to)
    if errors.Is(err, storm.ErrNotFound) {
        return fberrors.ErrNotExist
    }
    return err
}

// 写：写入 "config" bucket 指定键
func save(db *storm.DB, name string, from interface{}) error {
    return db.Set("config", name, from)
}
```

**存储结构**（bbolt 视角）：
```
Bucket: "config"
  ├─ Key: "settings"    → Value: 序列化后的 settings.Settings
  ├─ Key: "server"      → Value: 序列化后的 settings.Server
  ├─ Key: "auther"      → Value: 序列化后的 auth.Auther
  └─ Key: "version"     → Value: 2 (int)
```

**典型实现** - [config.go](file:///d:/fz/0601/solo-dogfeeding/code/167-filebrowser/storage/bolt/config.go#L1-L29)：
```go
type settingsBackend struct { db *storm.DB }

func (s settingsBackend) Get() (*settings.Settings, error) {
    set := &settings.Settings{}
    return set, get(s.db, "settings", set)  // 封装调用
}

func (s settingsBackend) Save(set *settings.Settings) error {
    return save(s.db, "settings", set)      // 封装调用
}
```

#### 模式 B：结构化实体存储（ORM 映射）

**适用数据**：users、share links 等多实例实体

**封装方式**：利用 storm 的结构化存储能力，每个结构体自动映射到独立 bucket

**典型实现** - [users.go](file:///d:/fz/0601/solo-dogfeeding/code/167-filebrowser/storage/bolt/users.go#L15-L122)：
```go
type usersBackend struct { db *storm.DB }

// 按唯一字段查询（ID 或 Username）
func (st usersBackend) GetBy(i interface{}) (user *users.User, err error) {
    user = &users.User{}
    var arg string
    switch i.(type) {
    case uint:   arg = "ID"
    case string: arg = "Username"
    }
    err = st.db.One(arg, i, user)  // storm 自动按索引查询
    // ... 错误转换
}

// 查询全部
func (st usersBackend) Gets() ([]*users.User, error) {
    var allUsers []*users.User
    err := st.db.All(&allUsers)    // storm 遍历整个 bucket
    // ...
}

// 保存
func (st usersBackend) Save(user *users.User) error {
    err := st.db.Save(user)        // storm 自动序列化 + 索引
    // ... 错误转换
}

// 部分字段更新
func (st usersBackend) Update(user *users.User, fields ...string) error {
    for _, field := range fields {
        val := reflect.ValueOf(user).Elem().FieldByName(field).Interface()
        if err := st.db.UpdateField(user, field, val); err != nil {
            return err
        }
    }
    return nil
}
```

**存储结构**（bbolt 视角）：
```
Bucket: "User"                     // storm 自动以结构体名创建
  ├─ Key: <ID 字节序列>            → Value: 序列化后的 users.User
  ├─ Bucket: "__storm.User.ID"     → 主键索引
  └─ Bucket: "__storm.User.Username" → 唯一索引

Bucket: "Link"                     // share.Link 实体
  ├─ Key: <ID 字节序列>            → Value: 序列化后的 share.Link
  └─ ... 索引 bucket
```

### 2.2 领域层的额外封装

在 `StorageBackend` 之上，各领域还有一层 `Storage` 封装，负责业务逻辑：

**示例：settings.Storage** - [settings/storage.go](file:///d:/fz/0601/solo-dogfeeding/code/167-filebrowser/settings/storage.go#L28-L118)
```go
func (s *Storage) Get() (*Settings, error) {
    set, err := s.back.Get()       // 调用后端
    // 读取后：默认值填充（隐式迁移）
    if set.UserHomeBasePath == "" {
        set.UserHomeBasePath = DefaultUsersHomeBasePath
    }
    if set.MinimumPasswordLength == 0 {
        set.MinimumPasswordLength = DefaultMinimumPasswordLength
    }
    // ... 更多默认值
    return set, nil
}

func (s *Storage) Save(set *Settings) error {
    // 保存前：数据校验和规范化
    if len(set.Key) == 0 {
        return fberrors.ErrEmptyKey
    }
    // ... 更多校验和默认值填充
    return s.back.Save(set)        // 调用后端
}
```

**示例：share.Storage** - [share/storage.go](file:///d:/fz/0601/solo-dogfeeding/code/167-filebrowser/share/storage.go#L32-L49)
```go
func (s *Storage) All() ([]*Link, error) {
    links, err := s.back.All()     // 调用后端
    // 读取后：自动清理过期链接（业务逻辑）
    for i, link := range links {
        if link.Expire != 0 && link.Expire <= time.Now().Unix() {
            s.Delete(link.Hash)    // 惰性删除
            links = append(links[:i], links[i+1:]...)
        }
    }
    return links, nil
}
```

**示例：users.Storage** - [users/storage.go](file:///d:/fz/0601/solo-dogfeeding/code/167-filebrowser/users/storage.go#L75-L130)
```go
func (s *Storage) Update(user *User, fields ...string) error {
    if err := user.Clean("", fields...); err != nil {  // 数据清理
        return err
    }
    if err := s.back.Update(user, fields...); err != nil {
        return err
    }
    s.mux.Lock()
    s.updated[user.ID] = time.Now().Unix()  // 内存缓存更新时间
    s.mux.Unlock()
    return nil
}

func (s *Storage) Delete(id interface{}) error {
    // 业务规则：禁止删除唯一管理员
    user, err := s.back.GetBy(id)
    if s.IsUniqueAdmin(user) {
        return fberrors.ErrRootUserDeletion
    }
    return s.back.DeleteByID(id)
}
```

---

## 三、事务边界分析

### 3.1 storm 的默认事务行为

storm 的 **每个方法调用都是独立事务**：
- `db.Save()` → 隐式开启写事务
- `db.One()` → 隐式开启读事务
- `db.All()` → 隐式开启读事务
- `db.UpdateField()` → 隐式开启写事务
- `db.DeleteStruct()` → 隐式开启写事务

这意味着：
1. **单操作自动事务**：简单 CRUD 无需手动管理事务
2. **多操作无原子性**：循环中多次调用 storm 方法，每个操作是独立事务，失败无回滚

### 3.2 显式事务使用

代码中只有一处显式使用 bbolt 原生事务：

**CountAdmins** - [users.go](file:///d:/fz/0601/solo-dogfeeding/code/167-filebrowser/storage/bolt/users.go#L98-L122)
```go
func (st usersBackend) CountAdmins() (int, error) {
    count := 0
    err := st.db.Bolt.View(func(tx *bolt.Tx) error {  // 只读事务
        bucket := tx.Bucket([]byte(reflect.TypeOf(users.User{}).Name()))
        if bucket == nil { return nil }
        
        c := bucket.Cursor()
        for _, v := c.First(); v != nil; _, v = c.Next() {
            var u users.User
            if err := st.db.Codec().Unmarshal(v, &u); err != nil {
                return err
            }
            if u.Perm.Admin { count++ }
        }
        return nil
    })
    return count, err
}
```

**设计意图**：
- 需要遍历整个 bucket 做统计，直接使用 bbolt API 避免 storm 的额外开销
- `View()` 开启只读事务，保证统计时的数据一致性快照

### 3.3 事务边界总结表

| 操作 | 事务类型 | 边界范围 | 原子性 |
|------|----------|----------|--------|
| `Get() / GetBy() / One()` | 只读事务 | 单次查询 | ✓ 单操作原子 |
| `Gets() / All() / Find()` | 只读事务 | 单次遍历 | ✓ 快照一致性 |
| `Save()` | 写事务 | 单实体保存 | ✓ 单操作原子 |
| `UpdateField()` | 写事务 | 单字段更新 | ✓ 单操作原子 |
| `DeleteStruct()` | 写事务 | 单实体删除 | ✓ 单操作原子 |
| `DeleteWithPathPrefix()` | 多个写事务 | 循环删除每个链接 | ✗ 部分失败无回滚 |
| `CountAdmins()` | 只读事务 | 整个 bucket 遍历 | ✓ 快照一致性 |
| `quickSetup()` | 多个写事务 | 依次保存 settings/server/auth/user | ✗ 部分失败无回滚 |

**注意**：`quickSetup()` 在 [root.go](file:///d:/fz/0601/solo-dogfeeding/code/167-filebrowser/cmd/root.go#L393-L502) 中依次保存 4 类数据，每个 `Save` 是独立事务，如果中间失败，数据库会处于不一致状态。

---

## 四、迁移关系

### 4.1 数据库版本标识

在 [bolt.go](file:///d:/fz/0601/solo-dogfeeding/code/167-filebrowser/storage/bolt/bolt.go#L20-L23) 中：
```go
func NewStorage(db *storm.DB) (*storage.Storage, error) {
    // ...
    err := save(db, "version", 2)  // 固定写入版本 2
    if err != nil { return nil, err }
    // ...
}
```

**关键点**：
- 版本号硬编码为 `2`，写入 `config` bucket 的 `version` 键
- **没有版本判断和升级逻辑**：每次启动直接覆盖写入
- 代码中没有读取 version 进行分支处理的逻辑

### 4.2 隐式迁移：读取时的默认值填充

实际的"迁移"通过 **领域层读取时补默认值** 实现，这是一种向后兼容的软迁移：

**Settings 读取迁移** - [settings/storage.go](file:///d:/fz/0601/solo-dogfeeding/code/167-filebrowser/settings/storage.go#L28-L62)
```go
func (s *Storage) Get() (*Settings, error) {
    set, err := s.back.Get()
    if set.UserHomeBasePath == "" {
        set.UserHomeBasePath = DefaultUsersHomeBasePath  // 旧版数据没有此字段时补默认
    }
    if set.MinimumPasswordLength == 0 {
        set.MinimumPasswordLength = DefaultMinimumPasswordLength
    }
    if set.Tus == (Tus{}) {
        set.Tus = Tus{
            ChunkSize:  DefaultTusChunkSize,
            RetryCount: DefaultTusRetryCount,
        }
    }
    // ... 更多字段补全
    return set, nil
}
```

**迁移特点**：
1. **无破坏性**：不会修改数据库中的旧数据
2. **惰性执行**：只在读取时生效，不主动扫描升级
3. **仅向前兼容**：新增字段给默认值，删除字段自动忽略
4. **持久化时机**：只有下次主动 `Save` 时，带默认值的完整数据才会写回数据库

### 4.3 命令行参数迁移（非数据库迁移）

在 [root.go](file:///d:/fz/0601/solo-dogfeeding/code/167-filebrowser/cmd/root.go#L36-L70) 中有独立的命令行参数迁移：
```go
var flagNamesMigrations = map[string]string{
    "file-mode":                        "fileMode",
    "dir-mode":                         "dirMode",
    "hide-login-button":                "hideLoginButton",
    "create-user-dir":                  "createUserDir",
    "minimum-password-length":          "minimumPasswordLength",
    // ... 更多映射
}

func migrateFlagNames(_ *pflag.FlagSet, name string) pflag.NormalizedName {
    if newName, ok := flagNamesMigrations[name]; ok {
        log.Printf("DEPRECATION NOTICE: Flag --%s has been deprecated, use --%s instead\n", name, newName)
        name = newName
    }
    return pflag.NormalizedName(name)
}
```

这是 CLI 参数的重命名迁移，与数据库存储无关。

### 4.4 数据库初始化流程

**首次启动**（数据库不存在时）：
```
storm.Open()
  → 创建空数据库文件
bolt.NewStorage(db)
  → 初始化各 backend
  → 写入 version = 2
quickSetup()
  → 写入默认 settings
  → 写入默认 server
  → 写入默认 auth (JSONAuth 或 NoAuth)
  → 写入默认 admin user
```

**后续启动**（数据库已存在时）：
```
storm.Open()
  → 打开已有数据库
bolt.NewStorage(db)
  → 覆盖写入 version = 2
读取数据时
  → Storage.Get() 自动补全缺失字段（隐式迁移）
```

---

## 五、配置与用户数据的存储支撑关系

### 5.1 配置数据存储路径

| 配置类型 | 存储位置 (bucket/key) | 实现文件 |
|----------|----------------------|----------|
| 全局设置 (Settings) | `config/settings` | [config.go](file:///d:/fz/0601/solo-dogfeeding/code/167-filebrowser/storage/bolt/config.go#L13-L20) |
| 服务器设置 (Server) | `config/server` | [config.go](file:///d:/fz/0601/solo-dogfeeding/code/167-filebrowser/storage/bolt/config.go#L22-L29) |
| 认证方式 (Auther) | `config/auther` | [auth.go](file:///d:/fz/0601/solo-dogfeeding/code/167-filebrowser/storage/bolt/auth.go#L15-L36) |
| 版本标识 | `config/version` | [bolt.go](file:///d:/fz/0601/solo-dogfeeding/code/167-filebrowser/storage/bolt/bolt.go#L20-L23) |

### 5.2 用户数据存储路径

| 数据类型 | 存储位置 (bucket) | 实现文件 |
|----------|------------------|----------|
| 用户信息 (User) | `User` bucket | [users.go](file:///d:/fz/0601/solo-dogfeeding/code/167-filebrowser/storage/bolt/users.go#L15-L122) |
| 分享链接 (Link) | `Link` bucket | [share.go](file:///d:/fz/0601/solo-dogfeeding/code/167-filebrowser/storage/bolt/share.go#L14-L104) |

### 5.3 统一入口

所有存储通过 [storage.Storage](file:///d:/fz/0601/solo-dogfeeding/code/167-filebrowser/storage/storage.go#L10-L17) 统一暴露：
```go
type Storage struct {
    Users    users.Store       // 用户数据
    Share    *share.Storage    // 分享链接
    Auth     *auth.Storage     // 认证配置
    Settings *settings.Storage // 系统配置
}
```

在 [bolt.go](file:///d:/fz/0601/solo-dogfeeding/code/167-filebrowser/storage/bolt/bolt.go#L14-L30) 中装配：
```go
func NewStorage(db *storm.DB) (*storage.Storage, error) {
    userStore := users.NewStorage(usersBackend{db: db})
    shareStore := share.NewStorage(shareBackend{db: db})
    settingsStore := settings.NewStorage(settingsBackend{db: db})
    authStore := auth.NewStorage(authBackend{db: db}, userStore)
    // ...
    return &storage.Storage{
        Auth:     authStore,
        Users:    userStore,
        Share:    shareStore,
        Settings: settingsStore,
    }, nil
}
```

---

## 六、设计权衡与注意事项

### 6.1 优点

1. **接口隔离**：各领域定义 `StorageBackend` 接口，更换存储实现只需实现对应接口
2. **分层清晰**：业务逻辑（领域 Storage）与存储实现（Bolt Backend）分离
3. **向后兼容**：读取时补默认值的隐式迁移，避免复杂的数据版本管理
4. **事务简单**：storm 自动管理单操作事务，降低使用门槛

### 6.2 潜在问题

1. **无批量事务**：`DeleteWithPathPrefix` 循环删除是多事务，部分失败会残留数据
2. **无数据迁移脚本**：仅靠默认值填充，无法处理破坏性变更（字段类型变更、数据重组）
3. **版本号无实质作用**：`version=2` 只写不读，无法根据版本执行差异化升级
4. **quickSetup 非原子**：多步独立事务，中间失败会产生不完整的初始化数据

### 6.3 关键入口参考

- 数据库连接：[cmd/utils.go](file:///d:/fz/0601/solo-dogfeeding/code/167-filebrowser/cmd/utils.go#L176-L185)
- 初始化装配：[storage/bolt/bolt.go](file:///d:/fz/0601/solo-dogfeeding/code/167-filebrowser/storage/bolt/bolt.go#L13-L31)
- 快速设置：[cmd/root.go](file:///d:/fz/0601/solo-dogfeeding/code/167-filebrowser/cmd/root.go#L393-L502)
