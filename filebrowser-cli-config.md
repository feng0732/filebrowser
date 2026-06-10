# File Browser 命令行与配置加载协作分析

## 一、整体架构概览

File Browser 使用 **Cobra + Viper + BoltDB** 三层配置体系：

| 层级 | 技术 | 职责 | 优先级（高→低） |
|------|------|------|----------------|
| 命令行参数 | Cobra (pflag) | 运行时即时覆盖 | 1（最高） |
| 环境变量 | Viper | 容器/部署场景注入 | 2 |
| 配置文件 | Viper (.json/.yaml/.toml) | 持久化静态配置 | 3 |
| 数据库 | BoltDB (Storm) | 业务配置持久化 | 4 |
| 代码默认值 | Flag 默认值 | 兜底 | 5（最低） |

```
用户启动命令
    │
    ▼
Cobra 解析 Flags ───────────────────────────┐
    │                                        │
    ▼                                        │
withViperAndStore 包装器                    │
    │                                        │
    ├─► initViper()                          │
    │    ├─ 读取 --config 指定文件           │
    │    ├─ 自动搜索 ./.filebrowser.*        │
    │    │              ~/.filebrowser.*     │
    │    │              /etc/filebrowser/    │
    │    ├─ 绑定环境变量 FB_*                │
    │    ├─ BindPFlags (合并 Flags)          │
    │    └─ ReadInConfig                     │
    │                                        │
    ├─► 打开/创建 BoltDB 数据库              │
    │    └─ 判断 databaseExisted             │
    │                                        │
    ▼                                        │
rootCmd.RunE 回调 ◄──────────────────────────┘
    │
    ├─► 若 DB 不存在：quickSetup() 写入初始 DB
    │
    ├─► getServerSettings()  ◄── 从 DB 读 + Viper 覆盖
    │
    └─► 构造 HTTP Handler 启动服务
```

---

## 二、入口与命令注册流程

### 2.1 程序入口

[main.go](file:///d:/fz/0601/solo-dogfeeding/code/180-filebrowser/main.go#L9-L12) 极简，直接调用 `cmd.Execute()`：

```go
func main() {
    if err := cmd.Execute(); err != nil {
        os.Exit(1)
    }
}
```

### 2.2 Cobra Root 命令初始化

[cmd/root.go](file:///d:/fz/0601/solo-dogfeeding/code/180-filebrowser/cmd/root.go#L72-L95) 的 `init()` 函数完成三件事：

1. **全局标志名迁移**：兼容旧版 flag 名称（如 `--file-mode` → `--fileMode`），通过 `SetGlobalNormalizationFunc` 实现
2. **注册 Persistent Flags**（所有子命令共享）：
   - `--config / -c`：配置文件路径
   - `--database / -d`：数据库路径（默认 `./filebrowser.db`）
3. **注册 Root Flags**（仅主命令启动服务用）：
   - `--noauth`、`--username`、`--password`、`--socketPerm`、`--cacheDir`、`--redisCacheUrl`、`--imageProcessors`
   - 以及 `addServerFlags()` 添加的服务器相关标志

### 2.3 子命令体系

各子命令通过各自文件的 `init()` 函数调用 `rootCmd.AddCommand()` 注册：

| 命令文件 | 注册内容 |
|---------|---------|
| [cmd/config.go](file:///d:/fz/0601/solo-dogfeeding/code/180-filebrowser/cmd/config.go#L19-L21) | `config` 命令组 |
| [cmd/config_init.go](file:///d:/fz/0601/solo-dogfeeding/code/180-filebrowser/cmd/config_init.go#L11-L14) | `config init` |
| [cmd/config_set.go](file:///d:/fz/0601/solo-dogfeeding/code/180-filebrowser/cmd/config_set.go#L7-L10) | `config set` |
| ... | users/cmds/rules 等子命令 |

---

## 三、Viper 初始化：多源配置合并的核心

### 3.1 initViper() 函数详解

[cmd/utils.go](file:///d:/fz/0601/solo-dogfeeding/code/180-filebrowser/cmd/utils.go#L85-L133) 是配置合并的核心入口：

```go
func initViper(cmd *cobra.Command) (*viper.Viper, error) {
    v := viper.New()

    // Step 1: 确定配置文件路径
    cfgFile, _ := cmd.Flags().GetString("config")
    if cfgFile == "" {
        // 未指定 --config，则自动搜索三个位置
        v.AddConfigPath(".")
        v.AddConfigPath(home)           // $HOME
        v.AddConfigPath("/etc/filebrowser/")
        v.SetConfigName(".filebrowser") // 文件名，自动匹配 .json/.yaml/.toml
    } else {
        v.SetConfigFile(cfgFile)        // 明确指定路径
    }

    // Step 2: 环境变量绑定
    v.SetEnvPrefix("FB")
    v.AutomaticEnv()
    // 关键：camelCase flag → SNAKE_CASE env 映射
    v.SetEnvKeyReplacer(strings.NewReplacer(
        generateEnvKeyReplacements(cmd)...
    ))

    // Step 3: 绑定命令行 Flags（优先级最高的来源）
    v.BindPFlags(cmd.Flags())

    // Step 4: 读取配置文件（失败仅记录日志，不中断）
    v.ReadInConfig()

    return v, nil
}
```

### 3.2 环境变量名映射机制

[generateEnvKeyReplacements()](file:///d:/fz/0601/solo-dogfeeding/code/180-filebrowser/cmd/utils.go#L73-L83) 遍历命令所有 Flags，为每个 flag 生成 `camelCase → SNAKE_CASE` 的替换对：

```
flag: disablePreviewResize
  └─ 替换规则: "DISABLEPREVIEWRESIZE" → "DISABLE_PREVIEW_RESIZE"

最终环境变量: FB_DISABLE_PREVIEW_RESIZE
```

使用 `lo.SnakeCase` 库进行转换，确保嵌套结构如 `branding.disableExternal` 正确映射为 `FB_BRANDING_DISABLE_EXTERNAL`。

### 3.3 Viper 内部优先级机制

Viper 的 `GetXxx()` 调用时按以下顺序查找（先命中即返回）：

1. **显式 Set 的值**（本项目未使用）
2. **命令行 Flag**（通过 `BindPFlags` 绑定）
3. **环境变量**（通过 `AutomaticEnv` + `EnvKeyReplacer`）
4. **配置文件**（通过 `ReadInConfig` 加载）
5. **Key/Value Store**（本项目未使用）
6. **默认值**（pflag 注册时的默认值）

> **注意**：Viper 不会自动将配置写回数据库。配置文件/环境变量/Flags 仅影响当前运行时，不会持久化到 BoltDB。

---

## 四、withViperAndStore：Viper + 数据库的统一包装

### 4.1 包装器模式

[cmd/utils.go](file:///d:/fz/0601/solo-dogfeeding/code/180-filebrowser/cmd/utils.go#L147-L194) 的 `withViperAndStore` 是一个高阶函数，将「Viper 初始化 + 数据库打开」封装成 Cobra 的 `RunE` 函数：

```go
func withViperAndStore(
    fn func(cmd *cobra.Command, args []string, v *viper.Viper, store *store) error,
    options storeOptions,
) cobraFunc {
    return func(cmd *cobra.Command, args []string) error {
        // 1. 初始化 Viper（合并 flags/env/config file）
        v, _ := initViper(cmd)

        // 2. 从 Viper 中获取 database 路径（已合并优先级）
        path, _ := filepath.Abs(v.GetString("database"))

        // 3. 判断数据库是否存在（用于 quickSetup 判断）
        exists, _ := dbExists(path)

        // 4. 根据 storeOptions 做校验
        switch {
        case exists && options.expectsNoDatabase:    // config init 场景
            log.Fatal(path + " already exists")
        case !exists && !allowsNoDatabase:            // 普通子命令
            log.Fatal(path + " does not exist")
        }

        // 5. 打开 BoltDB（通过 Storm ORM）
        db, _ := storm.Open(path, ...)
        defer db.Close()
        storage, _ := bolt.NewStorage(db)

        // 6. 回调实际业务逻辑
        return fn(cmd, args, v, &store{storage, exists})
    }
}
```

### 4.2 storeOptions 语义

| 选项 | 含义 | 使用场景 |
|------|------|---------|
| `expectsNoDatabase: true` | 期望 DB 不存在，存在则报错 | `config init` 初始化新库 |
| `allowsNoDatabase: true` | 允许 DB 不存在，自动 quickSetup | **root 主命令**启动服务 |
| 两者均为 false | DB 必须已存在，否则报错 | `config set`、`users add` 等管理命令 |

### 4.3 withStore 简化版

对于不需要访问 Viper 的子命令（如 `config set`），提供 `withStore` 包装器，内部忽略 `*viper.Viper` 参数。

---

## 五、Root 主命令：配置如何应用到运行时

### 5.1 执行流程

[cmd/root.go](file:///d:/fz/0601/solo-dogfeeding/code/180-filebrowser/cmd/root.go#L157-L279) 的 `RunE` 回调：

```
rootCmd.RunE
    │
    ├─► 数据库不存在？ ──是──► quickSetup(v, Storage)
    │                              │
    │                              ├─ 生成 Settings{Key, 权限默认值...}
    │                              ├─ 从 Viper 读取 baseURL/port/root 等
    │                              ├─ 构造初始 Server 配置
    │                              ├─ 创建 admin 用户
    │                              └─ 全部写入 BoltDB
    │
    ├─► 从 Viper 读取 imageProcessors / cacheDir / redisCacheUrl
    │   （这些仅运行时生效，不存 DB）
    │
    ├─► getServerSettings(v, Storage)  ◄── 核心合并逻辑
    │
    ├─► setupLog(server.Log)
    │
    ├─► 构造 net.Listener（socket / TLS / TCP）
    │
    └─► fbhttp.NewHandler(...) 启动 HTTP 服务
```

### 5.2 getServerSettings()：DB + Viper 的选择性覆盖

[cmd/root.go](file:///d:/fz/0601/solo-dogfeeding/code/180-filebrowser/cmd/root.go#L282-L373) 采用 **「读 DB → 检查 Viper.IsSet → 选择性覆盖」** 的模式：

```go
func getServerSettings(v *viper.Viper, st *storage.Storage) (*settings.Server, error) {
    // 1. 先从数据库读取已保存的 Server 配置
    server, _ := st.Settings.GetServer()

    // 2. 对每个配置项做「Viper 有设置才覆盖」
    if v.IsSet("address")   { server.Address = v.GetString("address") }
    if v.IsSet("port")      { server.Port = v.GetString("port") }
    if v.IsSet("log")       { server.Log = v.GetString("log") }
    if v.IsSet("cert")      { server.TLSCert = v.GetString("cert") }
    if v.IsSet("key")       { server.TLSKey = v.GetString("key") }
    if v.IsSet("root")      { server.Root = v.GetString("root") }
    if v.IsSet("socket")    { server.Socket = v.GetString("socket") }
    if v.IsSet("baseURL")   { server.BaseURL = v.GetString("baseURL") }
    if v.IsSet("tokenExpirationTime")   { server.TokenExpirationTime = v.GetString("tokenExpirationTime") }

    // 3. 布尔标志的取反语义（disableXxx → !enableXxx）
    if v.IsSet("disableThumbnails")     { server.EnableThumbnails = !v.GetBool(...) }
    if v.IsSet("disablePreviewResize")  { server.ResizePreview = !v.GetBool(...) }
    if v.IsSet("disableExec")           { server.EnableExec = !v.GetBool(...) }
    if v.IsSet("disableTypeDetectionByHeader") { server.TypeDetectionByHeader = !v.GetBool(...) }
    if v.IsSet("disableImageResolutionCalc")   { server.ImageResolutionCal = !v.GetBool(...) }

    // 4. 冲突校验：socket 不能与 address/port/cert/key 同时设置
    if isAddrSet && isSocketSet {
        return errors.New("--socket flag cannot be used with ...")
    }

    // 5. 逻辑修正：手动设了 address/port，则清空 DB 中的 socket
    if isAddrSet && server.Socket != "" {
        server.Socket = ""
    }

    return server, nil
}
```

> **关键设计**：使用 `v.IsSet(key)` 而非直接 `v.GetXxx`，确保只有**用户明确提供**的来源（flag/env/config file）才覆盖 DB。这避免了 pflag 默认值（如 `--port=8080`）误覆盖 DB 中的自定义值。

### 5.3 quickSetup()：首次启动的 DB 初始化

[cmd/root.go](file:///d:/fz/0601/solo-dogfeeding/code/180-filebrowser/cmd/root.go#L393-L502) 仅当数据库文件不存在时触发：

```go
func quickSetup(v *viper.Viper, s *storage.Storage) error {
    // 1. 构造 Settings（通用配置，存 DB）
    set := &settings.Settings{
        Key:                   generateKey(),      // 随机 512-bit 密钥
        Signup:                false,
        HideLoginButton:       true,
        CreateUserDir:         false,
        MinimumPasswordLength: settings.DefaultMinimumPasswordLength, // 12
        // ... 其他默认值
    }

    // 2. 选择认证方式并保存
    if v.GetBool("noauth") {
        set.AuthMethod = auth.MethodNoAuth
        s.Auth.Save(&auth.NoAuth{})
    } else {
        set.AuthMethod = auth.MethodJSONAuth
        s.Auth.Save(&auth.JSONAuth{})
    }
    s.Settings.Save(set)

    // 3. 构造 Server（从 Viper 读取当前运行参数作为初始值写入 DB）
    ser := &settings.Server{
        BaseURL:               v.GetString("baseURL"),
        Port:                  v.GetString("port"),
        Log:                   v.GetString("log"),
        // ... 从 Viper 读取当前值
    }
    s.Settings.SaveServer(ser)

    // 4. 创建初始 admin 用户
    username := v.GetString("username")  // 默认 admin
    password := v.GetString("password")  // 空则随机生成
    if password == "" {
        pwd, _ := users.RandomPwd(set.MinimumPasswordLength)
        password, _ = users.ValidateAndHashPwd(pwd, ...)
    }
    user := &users.User{Username: username, Password: password, ...}
    set.Defaults.Apply(user)
    user.Perm.Admin = true
    return s.Users.Save(user)
}
```

---

## 六、config 子命令：配置的持久化读写

### 6.1 config init：创建全新配置

[cmd/config_init.go](file:///d:/fz/0601/solo-dogfeeding/code/180-filebrowser/cmd/config_init.go#L16-L61)：

- 使用 `withStore(..., storeOptions{expectsNoDatabase: true})`
- 要求 DB **不存在**，否则退出
- `getSettings(flags, s, ser, nil, true)` 中 `all=true`：
  - **VisitAll** 遍历所有 flag（包括未设置的，使用默认值）
  - 结果：Settings/Server/Auther 全部字段被完整填充为默认值或 flag 值
- 完整写入数据库

### 6.2 config set：增量更新配置

[cmd/config_set.go](file:///d:/fz/0601/solo-dogfeeding/code/180-filebrowser/cmd/config_set.go#L12-L61)：

- 使用 `withStore(..., storeOptions{})`（DB 必须已存在）
- **先读 DB** 现有配置：
  ```go
  set, _ := st.Settings.Get()
  ser, _ := st.Settings.GetServer()
  auther, _ := st.Auth.Get(set.AuthMethod)
  ```
- `getSettings(flags, set, ser, auther, false)` 中 `all=false`：
  - **Visit** 仅遍历**用户显式设置**的 flag
  - 未设置的字段保留 DB 原值
- 将合并后的对象写回 DB

### 6.3 getSettings()：flag → 结构体的字段映射

[cmd/config.go](file:///d:/fz/0601/solo-dogfeeding/code/180-filebrowser/cmd/config.go#L274-L392) 是 `config init/set` 共用的核心函数：

```go
func getSettings(flags *pflag.FlagSet, set *settings.Settings, ser *settings.Server,
    auther auth.Auther, all bool) (auth.Auther, error) {

    visit := func(flag *pflag.Flag) {
        switch flag.Name {
        // Server 字段
        case "address":   ser.Address, _ = flags.GetString(flag.Name)
        case "port":      ser.Port, _ = flags.GetString(flag.Name)
        case "disableThumbnails":
            ser.EnableThumbnails, _ = flags.GetBool(flag.Name)
            ser.EnableThumbnails = !ser.EnableThumbnails  // 语义取反
        // ... 约 40 个 case 分支

        // Settings 字段
        case "signup":    set.Signup, _ = flags.GetBool(flag.Name)
        case "branding.name": set.Branding.Name, _ = flags.GetString(flag.Name)
        case "tus.chunkSize": set.Tus.ChunkSize, _ = flags.GetUint64(flag.Name)
        // ...
        }
    }

    if all {
        flags.VisitAll(visit)  // 遍历所有 flag（含默认值）
    } else {
        flags.Visit(visit)     // 仅遍历用户设置过的 flag
    }

    // 用户默认配置
    getUserDefaults(flags, &set.Defaults, all)

    // 认证方式处理
    if all {
        set.AuthMethod, auther, _ = getAuthentication(flags)
    } else {
        // 增量模式：传入现有 DB 值作 fallback
        set.AuthMethod, auther, _ = getAuthentication(flags, hasAuth, set, auther)
    }

    return auther, nil
}
```

---

## 七、数据结构：Settings 与 Server 的分离

### 7.1 两层配置模型

[settings/settings.go](file:///d:/fz/0601/solo-dogfeeding/code/180-filebrowser/settings/settings.go#L23-L65) 将配置分为两个独立结构体：

#### settings.Settings — 业务/通用配置
```go
type Settings struct {
    Key                   []byte              // JWT/HMAC 加密密钥
    Signup                bool                // 是否允许注册
    HideLoginButton       bool
    CreateUserDir         bool
    MinimumPasswordLength uint
    Defaults              UserDefaults        // 新用户默认配置
    AuthMethod            AuthMethod          // json/proxy/hook/noauth
    Branding              Branding            // 定制化（名称/主题/颜色）
    Tus                   Tus                 // TUS 断点续传
    Commands              map[string][]string // 事件钩子命令
    Shell                 []string            // Shell 命令前缀
    Rules                 []rules.Rule        // 文件访问规则
    FileMode, DirMode     fs.FileMode         // 新建文件/目录权限
}
```

#### settings.Server — 服务器/网络配置
```go
type Server struct {
    Root, BaseURL          string   // 文件根目录、URL 前缀
    Socket                 string   // Unix Socket 路径
    TLSKey, TLSCert        string   // TLS 证书
    Port, Address          string   // TCP 监听
    Log                    string   // 日志输出：stdout/stderr/文件路径/空
    EnableThumbnails       bool     // 缩略图
    ResizePreview          bool     // 预览图缩放
    EnableExec             bool     // 命令执行（高危，默认关）
    TypeDetectionByHeader  bool     // 通过文件头检测 MIME
    TokenExpirationTime    string   // Session 超时，如 "2h"
}
```

### 7.2 BoltDB 存储层

[storage/bolt/config.go](file:///d:/fz/0601/solo-dogfeeding/code/180-filebrowser/storage/bolt/config.go#L13-L29) 使用 Storm ORM 的 KV 存储，两个独立 bucket：

```go
func (s settingsBackend) Get() (*settings.Settings, error) {
    set := &settings.Settings{}
    return set, get(s.db, "settings", set)   // bucket: "settings"
}

func (s settingsBackend) GetServer() (*settings.Server, error) {
    server := &settings.Server{}
    return server, get(s.db, "server", server)  // bucket: "server"
}
```

---

## 八、三种典型场景的完整数据流

### 场景 1：首次启动，无 DB、无配置文件

```bash
filebrowser --port 9000 --root /data --noauth
```

```
1. Cobra 解析 Flags
   --port=9000, --root=/data, --noauth=true
   --database 使用 pflag 默认值 ./filebrowser.db

2. initViper()
   ├─ 未指定 --config，搜索 ./.filebrowser.* → 无
   ├─ 环境变量 FB_* → 未设置
   └─ BindPFlags 合并 Flags

3. withViperAndStore
   ├─ v.GetString("database") → ./filebrowser.db
   ├─ dbExists() → false
   ├─ allowsNoDatabase=true → 不报错
   └─ 创建 ./filebrowser.db，databaseExisted=false

4. rootCmd.RunE
   ├─ databaseExisted=false → 触发 quickSetup(v, Storage)
   │   ├─ Settings{Key: random, AuthMethod: noauth, ...}
   │   ├─ Server{Port: "9000", Root: "/data", ...}  ← 从 Viper 读取
   │   ├─ 创建 admin 用户
   │   └─ 全部存入 BoltDB
   ├─ imageProcessors=4（pflag 默认值）
   ├─ cacheDir=""（空→禁用）
   ├─ getServerSettings(v, Storage)
   │   ├─ 从 DB 读 Server（刚写入，含 Port=9000）
   │   ├─ v.IsSet("port")=true → 覆盖为 9000（值相同）
   │   └─ v.IsSet("root")=true → 覆盖为 /data
   └─ 启动 HTTP 监听 :9000
```

### 场景 2：已有 DB，使用环境变量覆盖端口

```bash
export FB_PORT=9090
filebrowser
```

```
1. Cobra 解析 Flags
   --port 使用 pflag 默认值 "8080"（IsSet 为 false，因为用户没显式传）

2. initViper()
   ├─ 环境变量 FB_PORT=9090 → 映射为 port=9090
   └─ BindPFlags 合并（但 flag 未显式设置，env 优先级 > pflag 默认值）

3. withViperAndStore
   └─ DB 存在，databaseExisted=true

4. rootCmd.RunE
   ├─ 跳过 quickSetup
   └─ getServerSettings(v, Storage)
       ├─ 从 DB 读 Server{Port: "9000"（上次启动存的值）}
       ├─ v.IsSet("port")=true（env 设置了）
       │   └─ server.Port = v.GetString("port") = "9090"
       └─ 最终使用 9090 启动
```

> **重点**：pflag 的「默认值」不触发 `IsSet=true`，只有**用户显式通过 CLI / env / config file 提供**才会 `IsSet=true`。因此 env 能正确覆盖 DB。

### 场景 3：config set 增量更新

```bash
filebrowser config set --branding.name "My Files" --disableExec
```

```
1. Cobra 解析 Flags
   --branding.name="My Files", --disableExec=true
   (其他 flag 未设置)

2. withStore（无 Viper 参与，config 子命令不用 Viper）

3. configSetCmd.RunE
   ├─ 从 DB 读现有: set, ser, auther
   ├─ getSettings(flags, set, ser, auther, all=false)
   │   └─ flags.Visit(visit)：仅遍历 2 个显式设置的 flag
   │       ├─ "branding.name" → set.Branding.Name = "My Files"
   │       └─ "disableExec" → ser.EnableExec = !true = false
   ├─ 其他字段（如 Signup、Port、AuthMethod）保持 DB 原值
   └─ set/ser/auther 写回 DB
```

---

## 九、设计要点与关键细节

### 9.1 Root 命令 vs Config 子命令的配置源差异

| | Root 主命令 | Config 子命令（init/set） |
|---|------------|--------------------------|
| 使用 Viper | ✅ 是（含 env/config file） | ❌ 否，直接读 `*pflag.FlagSet` |
| 访问 DB | ✅ 读 + 运行时覆盖 | ✅ 读写持久化 |
| Flags 遍历方式 | `viper.IsSet` 判定 | `Visit/VisitAll` 函数 |
| 生效范围 | 当前进程运行时 | 永久写入 BoltDB |

> **为什么 config set 不支持 env/config file？**
> 因为 `config set` 是**管理操作**，语义是「明确告诉我要改什么」，隐式的环境变量/配置文件会导致意外修改。设计上，只有服务启动（root 命令）才需要多源合并，管理命令使用显式 flag 更安全。

### 9.2 布尔标志的「否定语义」模式

项目大量使用 `disableXxx` flag → `!EnableXxx` 字段的模式：

```
CLI Flag:            --disableExec        --disableThumbnails
Viper Key:           disableExec          disableThumbnails
Server 结构体:       EnableExec           EnableThumbnails
转换逻辑:            !v.GetBool(...)      !v.GetBool(...)
```

好处是：**默认值可表达「安全默认」**。如 `--disableExec` 默认 `true` → `EnableExec=false`（命令执行默认关闭，安全优先）。

### 9.3 deprecated 标志名的兼容迁移

[cmd/root.go](file:///d:/fz/0601/solo-dogfeeding/code/180-filebrowser/cmd/root.go#L36-L70) 通过 `SetGlobalNormalizationFunc` 实现旧 flag → 新 flag 自动映射：

```go
var flagNamesMigrations = map[string]string{
    "file-mode":       "fileMode",
    "dir-mode":        "dirMode",
    "baseurl":         "baseURL",
    "cache-dir":       "cacheDir",
    // ... 约 14 条映射
}

func migrateFlagNames(_ *pflag.FlagSet, name string) pflag.NormalizedName {
    if newName, ok := flagNamesMigrations[name]; ok {
        log.Printf("DEPRECATION NOTICE: --%s 已弃用, 使用 --%s\n", name, newName)
        name = newName  // 内部统一替换为新名称
    }
    return pflag.NormalizedName(name)
}
```

用户使用 `--file-mode=0644` → 自动归一化为 `fileMode` → 后续 Viper 绑定、结构体赋值全部使用新名称，实现**无损兼容升级**。

---

## 十、关键代码位置索引

| 功能 | 文件 | 行号 |
|------|------|------|
| 程序入口 | [main.go](file:///d:/fz/0601/solo-dogfeeding/code/180-filebrowser/main.go#L9-L12) | L9-L12 |
| Root 命令定义与 Flags 注册 | [cmd/root.go](file:///d:/fz/0601/solo-dogfeeding/code/180-filebrowser/cmd/root.go#L72-L114) | L72-L114 |
| 标志名迁移（兼容旧版） | [cmd/root.go](file:///d:/fz/0601/solo-dogfeeding/code/180-filebrowser/cmd/root.go#L36-L70) | L36-L70 |
| 服务启动主逻辑（含合并） | [cmd/root.go](file:///d:/fz/0601/solo-dogfeeding/code/180-filebrowser/cmd/root.go#L157-L279) | L157-L279 |
| getServerSettings（DB + Viper 合并） | [cmd/root.go](file:///d:/fz/0601/solo-dogfeeding/code/180-filebrowser/cmd/root.go#L282-L373) | L282-L373 |
| quickSetup（首次启动初始化） | [cmd/root.go](file:///d:/fz/0601/solo-dogfeeding/code/180-filebrowser/cmd/root.go#L393-L502) | L393-L502 |
| initViper（多源合并核心） | [cmd/utils.go](file:///d:/fz/0601/solo-dogfeeding/code/180-filebrowser/cmd/utils.go#L85-L133) | L85-L133 |
| env key 映射（camelCase → SNAKE_CASE） | [cmd/utils.go](file:///d:/fz/0601/solo-dogfeeding/code/180-filebrowser/cmd/utils.go#L73-L83) | L73-L83 |
| withViperAndStore 包装器 | [cmd/utils.go](file:///d:/fz/0601/solo-dogfeeding/code/180-filebrowser/cmd/utils.go#L150-L194) | L150-L194 |
| DB 存在性判断 | [cmd/utils.go](file:///d:/fz/0601/solo-dogfeeding/code/180-filebrowser/cmd/utils.go#L50-L68) | L50-L68 |
| config init（全量初始化） | [cmd/config_init.go](file:///d:/fz/0601/solo-dogfeeding/code/180-filebrowser/cmd/config_init.go#L16-L61) | L16-L61 |
| config set（增量更新） | [cmd/config_set.go](file:///d:/fz/0601/solo-dogfeeding/code/180-filebrowser/cmd/config_set.go#L12-L61) | L12-L61 |
| getSettings（flag → 结构体映射） | [cmd/config.go](file:///d:/fz/0601/solo-dogfeeding/code/180-filebrowser/cmd/config.go#L274-L392) | L274-L392 |
| addServerFlags（服务器类 flag 定义） | [cmd/root.go](file:///d:/fz/0601/solo-dogfeeding/code/180-filebrowser/cmd/root.go#L99-L114) | L99-L114 |
| addConfigFlags（配置类 flag 定义） | [cmd/config.go](file:///d:/fz/0601/solo-dogfeeding/code/180-filebrowser/cmd/config.go#L30-L63) | L30-L63 |
| Settings 结构体定义 | [settings/settings.go](file:///d:/fz/0601/solo-dogfeeding/code/180-filebrowser/settings/settings.go#L23-L41) | L23-L41 |
| Server 结构体定义 | [settings/settings.go](file:///d:/fz/0601/solo-dogfeeding/code/180-filebrowser/settings/settings.go#L49-L65) | L49-L65 |
| BoltDB Settings 读写 | [storage/bolt/config.go](file:///d:/fz/0601/solo-dogfeeding/code/180-filebrowser/storage/bolt/config.go#L13-L29) | L13-L29 |
