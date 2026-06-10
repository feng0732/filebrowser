# FileBrowser 命令运行器（Command Runner）代码流程梳理

## 概述

FileBrowser 的命令运行机制分为**两大场景**：

1. **事件钩子（Event Hooks）**：在文件操作（如保存、复制、删除、上传、重命名）的 `before_xxx` / `after_xxx` 时机自动执行预配置的命令。
2. **交互式 Shell**：用户通过前端终端界面（WebSocket）手动输入并执行白名单内的命令。

---

## 一、整体架构图

```
┌──────────────────────────────────────────────────────────────────────┐
│                         命令配置层（Configuration）                    │
│                                                                      │
│  CLI (cmds add/ls/rm)  ──►  Settings.Commands (map[string][]string)  │
│  前端设置界面           ──►  Settings.Shell, User.Commands, Perm      │
│  config set            ──►  Server.EnableExec (全局开关)              │
└────────────────────────────────┬─────────────────────────────────────┘
                                 │ 存储（Bolt DB）
                                 ▼
┌──────────────────────────────────────────────────────────────────────┐
│                         命令解析层（Parser）                           │
│                                                                      │
│  SplitCommandAndArgs()  ◄── 按平台（Windows/Unix）拆分命令和参数      │
│  ParseCommand()         ◄── 结合 Shell 配置生成最终执行切片           │
└────────────────────────────────┬─────────────────────────────────────┘
                                 │
                                 ▼
┌──────────────────────────────────────────────────────────────────────┐
│                         命令执行层（Execution）                        │
│                                                                      │
│  ┌──────────────┐        ┌───────────────────────────┐               │
│  │  事件钩子模式 │        │    交互式 Shell 模式       │               │
│  │  RunHook()   │        │  commandsHandler (WS)      │               │
│  │  exec()      │        │  exec.Command + StdoutPipe │               │
│  └──────┬───────┘        └─────────────┬─────────────┘               │
│         │                              │                             │
│         └──────────────┬───────────────┘                             │
│                        ▼                                             │
│              os/exec.Cmd 真正执行系统命令                              │
└────────────────────────────────┬─────────────────────────────────────┘
                                 │
                                 ▼
┌──────────────────────────────────────────────────────────────────────┐
│                         输出返回层（Output）                           │
│                                                                      │
│  事件钩子：Stdout/Stderr → 服务器日志 + 错误中断流程（TUS 除外，错误被忽略） │
│  交互式：Stdout/Stderr → WebSocket TextMessage → 前端 Shell UI       │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 二、命令配置

### 2.1 数据结构定义

#### 全局设置（Settings）

**文件**：[settings.go](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/settings/settings.go#L23-L41)

```go
type Settings struct {
    Commands map[string][]string `json:"commands"`  // 事件钩子命令
    Shell    []string            `json:"shell"`     // Shell 执行器（如 ["bash", "-c"]）
    Defaults UserDefaults        `json:"defaults"`  // 用户默认配置
    // ...
}
```

- **Commands**：键为事件名（如 `before_save`、`after_delete`），值为该事件下执行的命令列表。
- **Shell**：如果配置了 Shell，则所有命令会包裹在 Shell 中执行（如 `bash -c "原始命令"`）。

#### 服务器全局开关（Server）

**文件**：[settings.go](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/settings/settings.go#L49-L65)

```go
type Server struct {
    EnableExec bool `json:"enableExec"`  // 命令执行总开关
    // ...
}
```

#### 用户权限与白名单（User + Permissions）

**文件**：[users.go](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/users/users.go#L21-L39)、[permissions.go](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/users/permissions.go#L4-L13)

```go
type Permissions struct {
    Execute bool `json:"execute"`  // 该用户是否允许执行命令
    // ...
}

type User struct {
    Perm     Permissions `json:"perm"`
    Commands []string    `json:"commands"`  // 该用户允许执行的命令白名单（如 ["git", "ls"]）
    // ...
}
```

#### 用户默认配置（UserDefaults）

**文件**：[defaults.go](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/settings/defaults.go#L10-L22)

```go
type UserDefaults struct {
    Perm     users.Permissions `json:"perm"`
    Commands []string          `json:"commands"`  // 新用户默认白名单
    // ...
}
```

### 2.2 存储层

**文件**：[storage.go](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/settings/storage.go#L64-L110)、[bolt/config.go](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/storage/bolt/config.go#L9-L29)

`Storage.Save()` 在保存配置时会自动初始化 5 个标准事件的 before/after 钩子数组：

```go
var defaultEvents = []string{"save", "copy", "rename", "upload", "delete"}

for _, event := range defaultEvents {
    if _, ok := set.Commands["before_"+event]; !ok {
        set.Commands["before_"+event] = []string{}
    }
    if _, ok := set.Commands["after_"+event]; !ok {
        set.Commands["after_"+event] = []string{}
    }
}
```

持久化通过 Bolt DB（嵌入式 KV 数据库）实现，存储在 bucket `"settings"` 和 `"server"` 中。

### 2.3 CLI 配置工具

**文件**：
- [cmds.go](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/cmd/cmds.go)（根命令）
- [cmds_add.go](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/cmd/cmds_add.go)（添加）
- [cmds_ls.go](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/cmd/cmds_ls.go)（列出）
- [cmds_rm.go](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/cmd/cmds_rm.go)（删除）

| 子命令 | 格式 | 作用 |
|--------|------|------|
| `cmds add` | `filebrowser cmds add <event> <command>` | 向事件钩子追加命令 |
| `cmds ls` | `filebrowser cmds ls [-e event]` | 列出所有/指定事件的钩子命令 |
| `cmds rm` | `filebrowser cmds rm <event> <index> [index_end]` | 按索引删除钩子命令 |

**添加流程**（`cmdsAddCmd`）：
1. 从存储读取 `Settings`
2. `s.Commands[args[0]] = append(s.Commands[args[0]], command)`
3. 调用 `st.Settings.Save(s)` 持久化

### 2.4 前端配置界面

**文件**：[Commands.vue](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/frontend/src/components/settings/Commands.vue)

一个简单的输入框组件，将 `commands` 数组以空格拼接展示，修改时再按空格拆分回数组，通过 `update:commands` 事件交给父组件保存到用户配置。

---

## 三、命令解析

### 3.1 平台相关拆分：SplitCommandAndArgs

**文件**：[commands.go](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/runner/commands.go#L32-L58)

根据运行平台使用不同解析策略：

| 平台 | 解析器 | 说明 |
|------|--------|------|
| **Unix-like** | `shlex.Split()`（第三方库） | 支持单引号、双引号、反斜杠转义 |
| **Windows** | `parseWindowsCommand()`（自实现） | 处理路径反斜杠不做转义的特性 |

输出：`(cmd string, args []string, err error)`

### 3.2 Shell 模式包装：ParseCommand

**文件**：[parser.go](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/runner/parser.go#L7-L25)

```go
func ParseCommand(s *settings.Settings, raw string) (command []string, name string, err error) {
    name, args, err := SplitCommandAndArgs(raw)
    // ...
    if len(s.Shell) == 0 || s.Shell[0] == "" {
        // 无 Shell：直接 [cmd, arg1, arg2, ...]
        command = append(command, name)
        command = append(command, args...)
    } else {
        // 有 Shell：如 ["bash", "-c", "原始命令整串"]
        command = append(command, s.Shell...)
        command = append(command, raw)
    }
    return command, name, nil
}
```

**关键点**：`name` 返回**原始可执行文件名**（不含参数），后续用于交互式 Shell 的白名单校验。

---

## 四、执行触发

### 4.1 模式一：事件钩子（RunHook）

#### 核心入口

**文件**：[runner.go](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/runner/runner.go#L20-L53)

```go
func (r *Runner) RunHook(fn func() error, evt, path, dst string, user *users.User) error {
    // 1. 执行 before_xxx 钩子
    if r.Enabled {
        if val, ok := r.Commands["before_"+evt]; ok {
            for _, command := range val {
                if err := r.exec(command, "before_"+evt, path, dst, user); err != nil {
                    return err  // before 失败会中断主操作
                }
            }
        }
    }

    // 2. 执行主业务函数 fn()
    if err := fn(); err != nil {
        return err
    }

    // 3. 执行 after_xxx 钩子
    if r.Enabled {
        if val, ok := r.Commands["after_"+evt]; ok {
            for _, command := range val {
                if err := r.exec(command, "after_"+evt, path, dst, user); err != nil {
                    return err
                }
            }
        }
    }
    return nil
}
```

#### Runner 注入点

**文件**：[data.go](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/data.go#L66-L99)

HTTP 中间件 `handle()` 构造请求上下文 `data` 时，将 Runner 嵌入：

```go
status, err := fn(w, r, &data{
    Runner:   &runner.Runner{Enabled: server.EnableExec, Settings: settings},
    settings: settings,
    server:   server,
})
```

#### 事件触发点总览

| 事件名 | 触发代码位置 | 说明 |
|--------|-------------|------|
| `delete` | [resource.go:113](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/resource.go#L113-L115) | 删除文件/目录 |
| `save` | [resource.go:198](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/resource.go#L198-L207) | 覆写已有文件（PUT） |
| `upload` | [resource.go:161](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/resource.go#L161-L170) | 新建文件/普通上传（POST） |
| `copy` / `rename` | [resource.go:256](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/resource.go#L256-L258) | action 取自请求参数（PATCH method） |
| `upload` | [tus_handlers.go:234](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/tus_handlers.go#L234) | TUS 分片上传完成 |

**通用调用形态**：`d.RunHook(fn func() error, evt string, path string, dst string, user *users.User)`

---

### 4.1.1 各操作接入命令运行器的完整链路

以下按 **HTTP 方法 → Handler → 前置检查 → RunHook 包装 → 环境变量 → 输出返回** 的顺序逐一剖析。

---

#### 🔹 操作 1：删除（DELETE /api/resources/{path}）

**路由**：[http.go:62](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/http.go#L62)
```go
api.PathPrefix("/resources").Handler(monkey(resourceDeleteHandler(fileCache), "/api/resources")).Methods("DELETE")
```

**处理函数**：[resourceDeleteHandler](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/resource.go#L84-L123)

| 阶段 | 详情 | 代码 |
|------|------|------|
| **前置检查** | 1. 不能删根目录<br>2. 用户需有 `Perm.Delete` 权限<br>3. 删除关联分享（仅记录警告）<br>4. 删除缩略图缓存 | [resource.go:86-111](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/resource.go#L86-L111) |
| **RunHook 调用** | `d.RunHook(fn, "delete", r.URL.Path, "", d.user)` | [resource.go:113-115](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/resource.go#L113-L115) |
| **包装的主操作** | `d.user.Fs.RemoveAll(r.URL.Path)` |  |
| **环境变量** | `$FILE` = 被删文件完整路径<br>`$DESTINATION` = `""`（空）<br>`$TRIGGER` = `"before_delete"` / `"after_delete"` |  |
| **错误影响** | `before_delete` 钩子失败 → 删除操作**不会执行**<br>`after_delete` 钩子失败 → 文件已删，但返回错误给前端<br>成功返回 `204 No Content`，失败 500 |  |
| **输出返回** | Stdout/Stderr → 服务器日志<br>命令失败导致主操作失败时 → HTTP 状态码由 `errToStatus(err)` 映射 |  |

---

#### 🔹 操作 2：普通上传（POST /api/resources/{path}）

**路由**：[http.go:63](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/http.go#L63)
```go
api.PathPrefix("/resources").Handler(monkey(resourcePostHandler(fileCache), "/api/resources")).Methods("POST")
```

**处理函数**：[resourcePostHandler](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/resource.go#L125-L178)

| 阶段 | 详情 | 代码 |
|------|------|------|
| **前置检查** | 1. 用户需有 `Perm.Create` 权限<br>2. 路径规则检查通过<br>3. 若目标已存在且 `override=true`，需 `Perm.Modify` 权限 | [resource.go:127-159](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/resource.go#L127-L159) |
| **RunHook 调用** | `d.RunHook(fn, "upload", r.URL.Path, "", d.user)` | [resource.go:161-170](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/resource.go#L161-L170) |
| **包装的主操作** | `writeFile(d.user.Fs, r.URL.Path, r.Body, ...)` → 创建新文件<br>成功后设置 `ETag` 响应头 |  |
| **失败回滚** | 任何阶段失败（before 钩子 / 主操作 / after 钩子），只要 `err != nil` → 执行 `d.user.Fs.RemoveAll(r.URL.Path)` 清理文件 | [resource.go:172-174](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/resource.go#L172-L174) |
| **环境变量** | `$FILE` = 新建文件路径<br>`$TRIGGER` = `"before_upload"` / `"after_upload"` |  |
| **输出返回** | Stdout/Stderr → 服务器日志<br>成功返回 `200 OK`（`errToStatus(nil)`）+ ETag header，失败由 `errToStatus` 映射为 500 |  |

**注意**：POST 用于**新建**文件，触发事件名为 `"upload"`。

---

#### 🔹 操作 3：保存（PUT /api/resources/{path}）

**路由**：[http.go:64](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/http.go#L64)
```go
api.PathPrefix("/resources").Handler(monkey(resourcePutHandler, "/api/resources")).Methods("PUT")
```

**处理函数**：[resourcePutHandler](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/resource.go#L180-L210)

| 阶段 | 详情 | 代码 |
|------|------|------|
| **前置检查** | 1. 用户需有 `Perm.Modify` 权限<br>2. 路径不能是目录（必须是文件）<br>3. 文件必须已存在（404 否则） | [resource.go:181-196](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/resource.go#L181-L196) |
| **RunHook 调用** | `d.RunHook(fn, "save", r.URL.Path, "", d.user)` | [resource.go:198-207](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/resource.go#L198-L207) |
| **包装的主操作** | `writeFile(d.user.Fs, r.URL.Path, r.Body, ...)` → 覆写文件<br>成功后设置 `ETag` 响应头 |  |
| **环境变量** | `$FILE` = 被覆写文件路径<br>`$TRIGGER` = `"before_save"` / `"after_save"` |  |
| **错误影响** | `before_save` 钩子失败 → 文件不写，返回 500<br>`after_save` 钩子失败 → 文件已写入，仍返回 500（无回滚） |  |
| **输出返回** | Stdout/Stderr → 服务器日志<br>成功返回 `200 OK` + ETag header，失败 500 |  |

**关键区别**：PUT 用于**覆写已有文件**，触发事件名为 `"save"`（而非 `"upload"`）。

---

#### 🔹 操作 4：复制（PATCH /api/resources/{src}?action=copy&destination={dst}）

**路由**：[http.go:65](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/http.go#L65)
```go
api.PathPrefix("/resources").Handler(monkey(resourcePatchHandler(fileCache), "/api/resources")).Methods("PATCH")
```

**处理函数**：[resourcePatchHandler](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/resource.go#L212-L262)

| 阶段 | 详情 | 代码 |
|------|------|------|
| **前置检查** | 1. 解析 `action=copy`、`destination` 参数<br>2. 源/目标路径规则检查<br>3. 不能操作根目录<br>4. 源不能是目标的父目录<br>5. 冲突处理（`override` / `rename`）<br>6. 需 `Perm.Create` 权限（在 patchAction 内） | [resource.go:213-255](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/resource.go#L213-L255) |
| **RunHook 调用** | `d.RunHook(fn, "copy", src, dst, d.user)` | [resource.go:256-258](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/resource.go#L256-L258) |
| **包装的主操作** | `patchAction(ctx, "copy", src, dst, d, fileCache)` → 调用 `fileutils.Copy(...)` | [resource.go:340-347](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/resource.go#L340-L347) |
| **环境变量** | `$FILE` = 源文件路径（**注意是 src 不是 dst**）<br>`$DESTINATION` = 目标路径<br>`$TRIGGER` = `"before_copy"` / `"after_copy"` |  |
| **错误影响** | `before_copy` 钩子失败 → 不复制，返回 500<br>`after_copy` 钩子失败 → 已复制，仍返回 500（无回滚） |  |
| **输出返回** | Stdout/Stderr → 服务器日志<br>成功返回 `200 OK`，失败 500 |  |

---

#### 🔹 操作 5：重命名（PATCH /api/resources/{src}?action=rename&destination={dst}）

**路由**：同复制，同 handler，仅 `action` 参数不同

| 阶段 | 详情 | 代码 |
|------|------|------|
| **前置检查** | 1. 解析 `action=rename`、`destination` 参数<br>2. 同复制的路径检查逻辑<br>3. 需 `Perm.Rename` 权限（在 patchAction 内） | [resource.go:213-255](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/resource.go#L213-L255) |
| **RunHook 调用** | `d.RunHook(fn, "rename", src, dst, d.user)` | [resource.go:256-258](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/resource.go#L256-L258) |
| **包装的主操作** | `patchAction(ctx, "rename", src, dst, d, fileCache)`：<br>1. 先删源文件的缩略图缓存<br>2. 调用 `fileutils.MoveFile(...)` | [resource.go:348-373](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/resource.go#L348-L373) |
| **环境变量** | `$FILE` = 源文件路径<br>`$DESTINATION` = 目标路径<br>`$TRIGGER` = `"before_rename"` / `"after_rename"` |  |
| **错误影响** | `before_rename` 钩子失败 → 不重命名，返回 500<br>`after_rename` 钩子失败 → 已重命名，仍返回 500（无回滚） |  |
| **输出返回** | Stdout/Stderr → 服务器日志<br>成功返回 `200 OK`，失败 500 |  |

---

#### 🔹 操作 6：分片上传（TUS 协议）

TUS 上传分为 **POST 初始化** 和 **PATCH 传数据** 两步，只有后者在上传完成时触发命令钩子。

##### 6.1 TUS POST（初始化分片上传）

**路由**：[http.go:67](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/http.go#L67)
```go
api.PathPrefix("/tus").Handler(monkey(tusPostHandler(uploadCache), "/api/tus")).Methods("POST")
```

**处理函数**：[tusPostHandler](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/tus_handlers.go#L41-L139)

| 阶段 | 详情 |
|------|------|
| **主要操作** | 创建空文件（或截断已有文件）、在 `uploadCache` 注册上传长度、返回 `Location` 头给客户端 |
| **RunHook** | ❌ **无**（此阶段不触发任何命令钩子） |

##### 6.2 TUS PATCH（上传分片数据）

**路由**：[http.go:69](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/http.go#L69)
```go
api.PathPrefix("/tus").Handler(monkey(tusPatchHandler(uploadCache), "/api/tus")).Methods("PATCH")
```

**处理函数**：[tusPatchHandler](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/tus_handlers.go#L142-L239)

| 阶段 | 详情 | 代码 |
|------|------|------|
| **前置检查** | 1. 文件存在且不是目录<br>2. `Upload-Offset` 和 `Upload-Length` 头校验<br>3. 检查缓存中是否有该上传任务 | [tus_handlers.go:149-204](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/tus_handlers.go#L149-L204) |
| **数据写入** | `openFile.Seek(offset)` → `io.Copy(openFile, r.Body)` → `openFile.Sync()` | [tus_handlers.go:206-227](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/tus_handlers.go#L206-L227) |
| **RunHook 触发条件** | 仅当 `newOffset >= uploadLength`（最后一片写完，文件完整） | [tus_handlers.go:232-235](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/tus_handlers.go#L232-L235) |
| **RunHook 调用** | `d.RunHook(func() error { return nil }, "upload", r.URL.Path, "", d.user)` |  |
| **包装的主操作** | `func() error { return nil }` → **空函数**！<br>（因为文件写入已在 Hook 之前完成） |  |
| **环境变量** | `$FILE` = 上传文件路径<br>`$TRIGGER` = `"before_upload"` / `"after_upload"` |  |
| **特殊说明** | `before_upload` 钩子在**文件已写入完成后**才执行（因为主操作为空函数）<br>**钩子错误完全不影响 API 返回**：`_ = d.RunHook(...)` 显式丢弃返回值，始终返回 HTTP 204<br>钩子命令的 stdout/stderr 仍会输出到服务器日志 | [tus_handlers.go:234](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/tus_handlers.go#L234) |

##### 6.3 TUS DELETE（取消分片上传）

**处理函数**：[tusDeleteHandler](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/tus_handlers.go#L241-L273)

| 阶段 | 详情 |
|------|------|
| **操作** | 删除文件 + `cache.Complete()` 清理缓存 |
| **RunHook** | ❌ **无**（不触发 `"delete"` 钩子） |

---

### 4.1.2 事件钩子调用统一时序

```
HTTP 请求到达
    │
    ▼
┌──────────────────────────┐
│ 权限检查 / 参数校验       │ （各 handler 自行实现）
└─────────────┬────────────┘
              │
              ▼
┌──────────────────────────┐
│ d.RunHook(fn, evt, ...)  │
└─────────────┬────────────┘
              │
    ┌─────────┴─────────┐
    │ Enabled = false?  │───跳过──→ fn() 执行
    └─────────┬─────────┘
              │ 是
    ┌─────────┴─────────┐
    │ before_evt 存在?  │
    └─────────┬─────────┘
              │ 是
    ┌─────────▼─────────┐
    │ 循环执行 before 命令│
    │  解析命令 + 环境变量  │
    │  阻塞/异步执行        │
    └─────────┬─────────┘
              │
    ┌─────────▼─────────┐
    │ 有命令失败?       │───是──→ 返回错误，fn() 不执行
    └─────────┬─────────┘
              │ 否
    ┌─────────▼─────────┐
    │ 执行 fn() 主操作   │
    └─────────┬─────────┘
              │
    ┌─────────▼─────────┐
    │ fn() 失败?        │───是──→ 返回错误，after 不执行
    └─────────┬─────────┘
              │ 否
    ┌─────────▼─────────┐
    │ after_evt 存在?   │
    └─────────┬─────────┘
              │ 是
    ┌─────────▼─────────┐
    │ 循环执行 after 命令 │
    └─────────┬─────────┘
              │
              ▼
         返回结果
```

---

### 4.1.3 各操作 RunHook 参数对照表

| 操作 | HTTP 方法 | evt | path 参数 | dst 参数 | 包装的 fn |
|------|----------|-----|----------|---------|----------|
| **删除** | DELETE | `"delete"` | `r.URL.Path` | `""` | `Fs.RemoveAll(path)` |
| **普通上传** | POST | `"upload"` | `r.URL.Path` | `""` | `writeFile(...)` + 设置 ETag |
| **保存** | PUT | `"save"` | `r.URL.Path` | `""` | `writeFile(...)` + 设置 ETag |
| **复制** | PATCH | `"copy"` | `src` | `dst` | `fileutils.Copy(src, dst)` |
| **重命名** | PATCH | `"rename"` | `src` | `dst` | 删缩略图 + `fileutils.MoveFile(src, dst)` |
| **分片上传完成** | PATCH (TUS) | `"upload"` | `r.URL.Path` | `""` | `func() { return nil }`（空） |

---

### 4.1.4 执行内部逻辑（exec 函数）

**文件**：[runner.go](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/runner/runner.go#L55-L118)

关键步骤：

1. **阻塞判断**：命令末尾 `&` → `blocking = false`（异步执行）
2. **命令解析**：调用 `ParseCommand()`
3. **环境变量注入**：参数中 `$FILE`、`$SCOPE`、`$TRIGGER`、`$USERNAME`、`$DESTINATION` 通过 `os.Expand` 替换
4. **额外环境变量**：同样的 5 个变量追加到 `cmd.Env`
5. **IO 定向**：`Stdin/Stdout/Stderr` → 服务器标准流（即日志）
6. **执行**：
   - 非阻塞：`cmd.Start()` 返回的 error 会中断 RunHook；`cmd.Start()` 成功后的 `cmd.Wait()` 失败仅记录日志，不影响主流程
   - 阻塞：`cmd.Run()` 失败返回 error，会中断 RunHook

### 4.2 模式二：交互式 Shell（WebSocket）

#### 路由注册

**文件**：[http.go:85](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/http.go#L85)

```go
api.PathPrefix("/command").Handler(monkey(commandsHandler, "/api/command")).Methods("GET")
```

#### 服务端处理（commandsHandler）

**文件**：[commands.go](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/commands.go#L41-L120)

流程：

1. **升级 WebSocket**：`upgrader.Upgrade(w, r, nil)`
2. **读取第一条消息**：即用户输入的命令字符串
3. **权限三连检查**：
   - `d.server.EnableExec`（全局开关）
   - `d.user.Perm.Execute`（用户执行权限）
   - `slices.Contains(d.user.Commands, name)`（命令名在白名单中，`name` 来自 ParseCommand 返回）
   - 任一失败 → 返回 `"Command not allowed."`
4. **设置工作目录**：`cmd.Dir = d.user.FullPath(r.URL.Path)`（当前浏览目录）
5. **获取 StdoutPipe + StderrPipe**：合并为 `io.MultiReader`
6. **逐行扫描发送**：`bufio.NewScanner` 每行通过 `conn.WriteMessage(TextMessage, ...)` 推给前端
7. **等待结束**：`cmd.Wait()`，WebSocket 关闭

#### 前端 API

**文件**：[commands.ts](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/frontend/src/api/commands.ts#L7-L19)

```ts
export default function command(url, command, onmessage, onclose) {
  const conn = new WebSocket(`${protocol}//${host}${baseURL}/api/command${url}`);
  conn.onopen    = () => conn.send(command);   // 连接建立后立即发送命令
  conn.onmessage = onmessage;                  // 接收输出行
  conn.onclose   = onclose;                    // 执行结束
}
```

#### 前端 UI（Shell 组件）

**文件**：[Shell.vue](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/frontend/src/components/Shell.vue)

- 终端样式的输入输出面板，支持拖拽调整高度
- 内置命令：`clear`（清屏）、`exit`（关闭面板）
- 历史记录（上下箭头切换）
- 提交时：
  1. 将命令与实时输出 push 到 `content` 数组
  2. `onmessage` 回调中追加输出并滚动到底部
  3. `onclose` 时过滤 ANSI 颜色码、解除输入锁定、重新聚焦

---

## 五、输出返回与错误处理链路（代码核准）

### 5.1 响应写入机制与状态码模式

所有 HTTP handler 函数签名为 `func(w, r, d) (int, error)`，返回值由 [handle()](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/data.go#L66-L101) 中间件统一处理。

#### 两种响应模式

| 模式 | handler 返回 | handle() 行为 | 典型场景 |
|------|-------------|--------------|---------|
| **JSON 模式** | `(0, nil)` | 不调用 `http.Error`，handler 自行写响应体 | settings 查询、用户信息获取等 |
| **状态码模式** | `(status, err)` 且 `status != 0` | 调用 `http.Error(w, status_text, status)` 写响应 | 所有资源操作（增删改）、所有错误返回 |

**状态码模式的处理逻辑**（[data.go:85-97](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/data.go#L85-L97)）：

```go
if status >= 400 || err != nil {
    log.Printf("%s: %v %s %v", r.URL.Path, status, clientIP, err)   // 打印日志
}

if status != 0 {
    txt := http.StatusText(status)
    if status == http.StatusBadRequest && err != nil {
        txt += " (" + err.Error() + ")"                              // 仅 400 追加 err 详情
    }
    http.Error(w, strconv.Itoa(status)+" "+txt, status)              // 写响应
    return
}
```

**关键规则**：
- **status 决定一切**：只要 `status != 0` 就走 `http.Error`，与 err 是否为 nil 无关
- **err 仅影响日志和 400 body**：err != nil 且 status >= 400 时打日志；只有 400 状态码会把 err 内容追加到响应 body
- **所有资源操作都是状态码模式**：删除/上传/保存/复制/重命名/分片上传都返回非 0 status

---

### 5.2 各操作成功时的响应与状态码

所有资源操作成功时都走 **状态码模式**（`http.Error`），区别仅在于 status 是 200 还是 204。

#### 成功状态码汇总

| 操作 | HTTP 方法 | 成功 status | handler 中的返回 |
|------|----------|-----------|-----------------|
| **普通上传** | POST | `200 OK` | `errToStatus(nil)` → 200 |
| **保存** | PUT | `200 OK` | `errToStatus(nil)` → 200 |
| **删除** | DELETE | `204 No Content` | `http.StatusNoContent` → 204 |
| **复制** | PATCH | `200 OK` | `errToStatus(nil)` → 200 |
| **重命名** | PATCH | `200 OK` | `errToStatus(nil)` → 200 |
| **分片上传** | PATCH (TUS) | `204 No Content` | `http.StatusNoContent` → 204 |

#### 200 OK 响应的完整链路（以上传/保存为例）

```go
// resource.go:161-176  POST 上传
err = d.RunHook(fn, "upload", ..., d.user)   // err == nil
return errToStatus(err), err                  // → (200, nil)
```

| 环节 | 具体内容 |
|------|---------|
| handler 返回 | `(200, nil)` |
| handle() 判断 | `status != 0` → true → 走 `http.Error` |
| `http.Error` 行为 | 1. 设置 `Content-Type: text/plain; charset=utf-8`<br>2. 设置 `X-Content-Type-Options: nosniff`<br>3. `WriteHeader(200)`<br>4. `Write([]byte("200 OK\n"))` |
| 最终响应 | HTTP 200 + body `"200 OK\n"` + ETag header |
| 响应体语义 | 纯文本状态码描述，**不包含业务数据** |

> **注意**：handler 内部 `fn()` 中通过 `w.Header().Set("ETag", etag)` 设置的 ETag 头会保留，因为 header 设置在 `WriteHeader` 之前均有效。

#### 204 No Content 响应的完整链路（以删除/分片上传为例）

```go
// resource.go:113-119  DELETE 删除
err = d.RunHook(fn, "delete", ..., d.user)   // err == nil
return http.StatusNoContent, nil              // → (204, nil)
```

| 环节 | 具体内容 |
|------|---------|
| handler 返回 | `(204, nil)` |
| handle() 判断 | `status != 0` → true → 走 `http.Error` |
| `http.Error` 行为 | 尝试写入 `"204 No Content\n"` body，但 HTTP 规范规定 204 无 body，实际传输时 body 会被 Go net/http 忽略 |
| 最终响应 | HTTP 204 + 无 body + 无 Content-Length |
| 响应体语义 | 操作成功，无返回内容 |

---

### 5.3 分片上传钩子失败不传播的本质

分片上传（TUS PATCH）是唯一**不将钩子错误传播到响应**的操作。

#### 代码位置

[tus_handlers.go:229-237](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/tus_handlers.go#L229-L237)

```go
w.Header().Set("Upload-Offset", strconv.FormatInt(newOffset, 10))

if newOffset >= uploadLength {
    cache.Complete(file.RealPath())
    _ = d.RunHook(func() error { return nil }, "upload", r.URL.Path, "", d.user)
}

return http.StatusNoContent, nil    // 无条件执行
```

#### 三层因果关系

```
钩子命令执行失败（exit code != 0）
    │
    ▼
RunHook() 返回 exec.ExitError
    │
    ▼
_ = RunHook()  →  错误被丢弃，err 变量不受影响
    │
    ▼
return http.StatusNoContent, nil  →  始终返回 (204, nil)
    │
    ▼
handle() 中 status = 204 → http.Error(204)
    │
    ▼
客户端收到 HTTP 204，完全感知不到钩子失败
```

#### 与普通上传（POST）的对比

| 维度 | 普通上传（POST） | 分片上传（TUS PATCH） |
|------|----------------|---------------------|
| RunHook 返回值处理 | 赋值给 `err` 变量 | 用 `_` 丢弃 |
| 钩子失败 → err | err != nil | err 仍为 nil（不受影响） |
| 钩子失败 → status | errToStatus(err) → 500 | 恒为 204 |
| 钩子失败 → 响应 | 500 Internal Server Error | 204 No Content |
| 钩子失败 → 文件 | after 失败会触发 RemoveAll 删除 | 文件已写入，不受影响 |
| 钩子输出去向 | 服务器日志 | 服务器日志 |

> **统一理解**：钩子命令的 stdout/stderr 永远输出到服务器日志（`os.Stdout`/`os.Stderr`）。区别仅在于——钩子命令的**退出码（exit code）是否会影响 HTTP 响应**。普通上传会影响（通过 err 传播 → errToStatus → 500），分片上传不会（`_` 丢弃了 error）。

---

### 5.4 RunHook 中 before 和 after 命令失败的返回差异

[RunHook](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/runner/runner.go#L21-L53) 的完整逻辑：

```go
func (r *Runner) RunHook(fn func() error, evt, path, dst string, user *users.User) error {
    path = user.FullPath(path)
    dst = user.FullPath(dst)

    // ──── 第一阶段：before 命令 ────
    if r.Enabled {
        if val, ok := r.Commands["before_"+evt]; ok {
            for _, command := range val {
                err := r.exec(command, "before_"+evt, path, dst, user)
                if err != nil {
                    return err    // ← before 失败：fn() 不会执行
                }
            }
        }
    }

    // ──── 第二阶段：主操作 ────
    err := fn()
    if err != nil {
        return err            // ← 主操作失败：after 不会执行
    }

    // ──── 第三阶段：after 命令 ────
    if r.Enabled {
        if val, ok := r.Commands["after_"+evt]; ok {
            for _, command := range val {
                err := r.exec(command, "after_"+evt, path, dst, user)
                if err != nil {
                    return err    // ← after 失败：主操作已完成
                }
            }
        }
    }

    return nil
}
```

#### before 命令失败 vs after 命令失败的关键差异

| 维度 | before 失败 | after 失败 |
|------|-----------|----------|
| **fn() 是否执行** | ❌ 否——fn() 被完全跳过 | ✅ 是——fn() 已成功执行 |
| **文件系统状态** | 原始状态不变（主操作未发生） | 主操作已生效（文件已写入/删除/移动） |
| **RunHook 返回** | `exec()` 的 error | `exec()` 的 error |
| **errToStatus 映射** | `500 Internal Server Error`（默认分支） | `500 Internal Server Error`（默认分支） |
| **对客户端的表象** | 看起来是"操作没发生 + 500" | 看起来是"操作没发生 + 500"，**但实际已生效** |
| **数据一致性** | ✅ 安全——客户端看到 500 后重试没问题 | ⚠️ 不一致——客户端可能因 500 重试，但操作已完成 |

> **⚠️ 重要的数据一致性风险**：after 钩子失败时，主操作已完成但返回 500，客户端可能重试导致重复操作。这在删除场景中无影响（文件已删，重试只会 404），但在复制场景中会导致重复文件。

---

### 5.5 各操作的错误返回完整链路（代码逐行核准）

#### 🔹 删除（DELETE）—— before/after 失败

```go
// resource.go:113-119
err = d.RunHook(func() error {
    return d.user.Fs.RemoveAll(r.URL.Path)       // 主操作
}, "delete", r.URL.Path, "", d.user)

if err != nil {
    return errToStatus(err), err                  // → (500, exec.ExitError)
}
return http.StatusNoContent, nil                  // → (204, nil)
```

| 场景 | RunHook 返回 | handler 返回 | HTTP 响应 | 文件状态 |
|------|-------------|-------------|----------|---------|
| before_delete 阻塞命令失败 | exec error | `(500, error)` | `500 Internal Server Error` | 文件保留 ✅ |
| before_delete 非阻塞命令 `cmd.Start()` 失败 | exec error | `(500, error)` | `500 Internal Server Error` | 文件保留 ✅ |
| 主操作 RemoveAll 失败 | fn() error | `(errToStatus(err), error)` | 取决于错误类型 | 不确定 |
| after_delete 阻塞命令失败 | exec error | `(500, error)` | `500 Internal Server Error` | 文件已删 ⚠️ |
| after_delete 非阻塞命令 `cmd.Start()` 失败 | exec error | `(500, error)` | `500 Internal Server Error` | 文件已删 ⚠️ |
| 全部成功 | nil | `(204, nil)` | `204 No Content` | 文件已删 |

**非阻塞命令的特殊情况**：命令末尾加 `&` 时，`cmd.Start()` 失败（如可执行文件不存在）仍会返回 error，但 `cmd.Start()` 成功后的 `cmd.Wait()` 失败不会返回 error（仅在 goroutine 中打印日志）。

---

#### 🔹 普通上传（POST）—— before/after 失败

```go
// resource.go:161-176
err = d.RunHook(func() error {
    info, writeErr := writeFile(d.user.Fs, r.URL.Path, r.Body, ...)
    if writeErr != nil { return writeErr }
    w.Header().Set("ETag", etag)
    return nil
}, "upload", r.URL.Path, "", d.user)

if err != nil {
    _ = d.user.Fs.RemoveAll(r.URL.Path)           // 失败回滚
}

return errToStatus(err), err
```

| 场景 | RunHook 返回 | handler 返回 | HTTP 响应 | 文件状态 |
|------|-------------|-------------|----------|---------|
| before_upload 命令失败 | exec error | `(500, error)` | `500 Internal Server Error` | 文件不存在（fn 未执行），回滚 RemoveAll 无害 |
| 主操作 writeFile 失败 | fn() error | `(errToStatus(err), error)` | 取决于错误类型 | RemoveAll 尝试清理可能残留的半成文件 |
| after_upload 命令失败 | exec error | `(500, error)` | `500 Internal Server Error` | 文件已写入 + **RemoveAll 回滚删掉它** ⚠️ |
| 全部成功 | nil | `(200, nil)` | `200 OK` + ETag | 文件已创建 |

> **⚠️ 上传场景的隐蔽行为**：after_upload 钩子失败时，`err != nil` → 执行 `RemoveAll` 删除文件。这意味着一个 after 钩子的失败会导致**已经成功写入的文件被删除**。这可能不是预期行为——文件写入成功，但因 after 通知命令失败而回滚。

---

#### 🔹 保存（PUT）—— before/after 失败

```go
// resource.go:198-209
err = d.RunHook(func() error {
    info, writeErr := writeFile(d.user.Fs, r.URL.Path, r.Body, ...)
    if writeErr != nil { return writeErr }
    w.Header().Set("ETag", etag)
    return nil
}, "save", r.URL.Path, "", d.user)

return errToStatus(err), err                        // 无回滚
```

| 场景 | RunHook 返回 | handler 返回 | HTTP 响应 | 文件状态 |
|------|-------------|-------------|----------|---------|
| before_save 命令失败 | exec error | `(500, error)` | `500 Internal Server Error` | 原文件保留 ✅ |
| 主操作 writeFile 失败 | fn() error | `(errToStatus(err), error)` | 取决于错误类型 | 原文件可能已被截断破坏 ⚠️ |
| after_save 命令失败 | exec error | `(500, error)` | `500 Internal Server Error` | 新内容已写入，无回滚 ⚠️ |
| 全部成功 | nil | `(200, nil)` | `200 OK` + ETag | 文件已更新 |

> **注意**：PUT 保存**没有回滚逻辑**（不像 POST 上传有 RemoveAll）。`writeFile` 使用 `O_RDWR|O_O_CREATE|O_TRUNC`（[resource.go:303](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/resource.go#L303)），会先截断文件再写入。如果写入中途失败，原文件内容已丢失。

---

#### 🔹 复制/重命名（PATCH）—— before/after 失败

```go
// resource.go:256-261
err = d.RunHook(func() error {
    return patchAction(r.Context(), action, src, dst, d, fileCache)
}, action, src, dst, d.user)

return errToStatus(err), err                        // 无回滚
```

| 场景 | RunHook 返回 | handler 返回 | HTTP 响应 | 文件状态 |
|------|-------------|-------------|----------|---------|
| before_copy/rename 失败 | exec error | `(500, error)` | `500 Internal Server Error` | 原文件不变 ✅ |
| 主操作 patchAction 失败 | fn() error | `(errToStatus(err), error)` | 取决于错误类型 | 不确定 |
| after_copy 失败 | exec error | `(500, error)` | `500 Internal Server Error` | 目标文件已创建，无回滚 ⚠️ |
| after_rename 失败 | exec error | `(500, error)` | `500 Internal Server Error` | 文件已移动，无回滚 ⚠️ |
| 全部成功 | nil | `(200, nil)` | `200 OK` | 操作完成 |

---

#### 🔹 分片上传（TUS PATCH）—— 钩子失败的实际影响

```go
// tus_handlers.go:229-237
newOffset := uploadOffset + bytesWritten
w.Header().Set("Upload-Offset", strconv.FormatInt(newOffset, 10))

if newOffset >= uploadLength {
    cache.Complete(file.RealPath())
    _ = d.RunHook(func() error { return nil }, "upload", r.URL.Path, "", d.user)
}

return http.StatusNoContent, nil                    // ← 无论钩子成败都走这里
```

**逐行分析**：

1. 文件数据已在 [tus_handlers.go:218](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/tus_handlers.go#L218) 处通过 `io.Copy(openFile, r.Body)` 写入并 Sync
2. `cache.Complete()` 已标记上传完成
3. `_ = d.RunHook(...)` 用 `_` 显式丢弃返回值
4. `return http.StatusNoContent, nil` **无条件执行**

| 场景 | RunHook 返回 | handler 返回 | HTTP 响应 | 文件状态 |
|------|-------------|-------------|----------|---------|
| before_upload 阻塞命令失败 | exec error（被 `_` 丢弃） | `(204, nil)` | `204 No Content` | 文件已写入，缓存已标记完成 |
| before_upload 非阻塞命令失败 | exec error（被 `_` 丢弃） | `(204, nil)` | `204 No Content` | 同上 |
| after_upload 阻塞命令失败 | exec error（被 `_` 丢弃） | `(204, nil)` | `204 No Content` | 同上 |
| 钩子全部成功 | nil（被 `_` 丢弃） | `(204, nil)` | `204 No Content` | 文件已写入 |
| 中间分片（未触发钩子） | — | `(204, nil)` | `204 No Content` | 数据追加写入 |

**分片上传钩子失败的本质**：
- **文件不受影响**——数据早已写入并同步到磁盘
- **客户端不受影响**——始终收到 204
- **钩子命令的 stdout/stderr 仍会输出到服务器日志**——因为 `exec()` 中 `cmd.Stdout = os.Stdout`
- **但钩子命令的失败无法传播到客户端或阻止操作完成**
- 这与普通上传（POST）形成鲜明对比：POST 中 after_upload 失败会触发 RemoveAll 删除文件

---

### 5.6 非阻塞命令（`&` 后缀）的失败行为

[exec()](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/runner/runner.go#L55-L118) 中非阻塞模式的关键逻辑：

```go
if strings.HasSuffix(raw, "&") {
    blocking = false
    raw = strings.TrimSpace(strings.TrimSuffix(raw, "&"))
}

// ...解析命令...

if !blocking {
    log.Printf("[INFO] Nonblocking Command: \"%s\"", strings.Join(command, " "))
    defer func() {
        go func() {
            err := cmd.Wait()
            if err != nil {
                log.Printf("[INFO] Nonblocking Command \"%s\" failed: %s", ...)
            }
        }()
    }()
    return cmd.Start()    // ← 只返回 Start 的错误
}
```

| 失败点 | 阻塞模式 | 非阻塞模式（`&`） |
|--------|---------|------------------|
| `cmd.Start()` 失败（如可执行文件不存在） | 返回 error → RunHook 中断 | 返回 error → RunHook 中断 |
| `cmd.Run()` / `cmd.Wait()` 失败（命令执行出错） | 返回 error → RunHook 中断 | **仅打日志**，RunHook 继续 ✅ |
| 对主操作的影响 | before 失败会阻止 fn() 执行 | Start 失败阻止，Wait 失败**不阻止** |

**这意味着**：非阻塞命令只要能成功启动（`cmd.Start()` 返回 nil），即使命令实际执行失败（exit code != 0），也不会影响后续流程。

---

### 5.7 errToStatus 映射表

[errToStatus](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/utils.go#L30-L51) 函数将不同错误类型映射为 HTTP 状态码：

```go
func errToStatus(err error) int {
    switch {
    case err == nil:                          → 200 OK
    case os.IsPermission(err):                → 403 Forbidden
    case os.IsNotExist(err):                  → 404 Not Found
    case os.IsExist(err):                     → 409 Conflict
    case errors.Is(err, ErrPermissionDenied): → 403 Forbidden
    case errors.Is(err, ErrInvalidRequestParams): → 400 Bad Request
    case errors.Is(err, ErrRootUserDeletion): → 403 Forbidden
    case errors.Is(err, ErrImageTooLarge):    → 413 Request Entity Too Large
    default:                                  → 500 Internal Server Error
    }
}
```

**钩子命令执行错误（`*exec.ExitError`）**不属于以上任何特定类型，**始终走 default 分支 → 500**。

这也意味着钩子失败时，`handle()` 中间件会打印日志：
```
/api/resources/test.txt: 500 127.0.0.1 exit status 1
```

但客户端收到的 body 只有 `500 Internal Server Error`，**不包含具体的钩子失败原因**（只有 400 状态码才会追加 err 信息到 body）。

---

### 5.8 交互式 Shell 的输出返回

```
       commandsHandler (WebSocket)
┌──────────────────────────────────────────────┐
│                                              │
│  cmd.StdoutPipe()  ──┐                       │
│                       ├──► io.MultiReader    │
│  cmd.StderrPipe()  ──┘       │              │
│                              ▼              │
│                    bufio.NewScanner          │
│                              │              │
│                for s.Scan() {                │
│                  conn.WriteMessage(          │
│                    TextMessage, s.Bytes()    │
│                  )  ─────►  前端 Shell UI    │
│                }                             │
│                                              │
│  cmd.Wait() 结束后 WebSocket 关闭            │
└──────────────────────────────────────────────┘
```

**代码位置**：[commands.go:88-117](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/commands.go#L88-L117)

**与事件钩子的关键区别**：
- 交互式 Shell 用 `StdoutPipe` / `StderrPipe` 采集输出推给前端
- 事件钩子用 `os.Stdout` / `os.Stderr` 直接打印到服务器日志
- 交互式 Shell 的命令失败仅在前端显示，不影响任何文件操作
- 交互式 Shell 的命令工作目录为 `cmd.Dir = d.user.FullPath(r.URL.Path)`（当前浏览路径）

**前端接收**（[Shell.vue:174-189](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/frontend/src/components/Shell.vue#L174-L189)）：
```js
commands(
  this.path,
  cmd,
  (event) => {
    results.text += `${event.data}\n`;  // 实时追加每行输出
    this.scroll();
  },
  () => {
    results.text = results.text
      .replace(/\u001b\[[0-9;]+m/g, "")  // 过滤 ANSI 颜色码
      .trimEnd();
    this.canInput = true;                 // 解锁输入
  }
);
```

---

### 5.9 关键差异总结表

| 操作 | before 失败 → 文件状态 | after 失败 → 文件状态 | 失败回滚 | 成功状态码 | 失败状态码 |
|------|----------------------|---------------------|---------|----------|----------|
| **删除** | 保留 ✅ | 已删 ⚠️ | 无 | 204 | 500 |
| **普通上传** | 不存在 ✅ | **已写入但被 RemoveAll 删除** ⚠️ | RemoveAll | 200 | 500 |
| **保存** | 保留 ✅ | 已覆写（无回滚）⚠️ | 无 | 200 | 500 |
| **复制** | 未复制 ✅ | 已复制（无回滚）⚠️ | 无 | 200 | 500 |
| **重命名** | 未重命名 ✅ | 已重命名（无回滚）⚠️ | 无 | 200 | 500 |
| **分片上传** | 已写入（错误被 `_` 忽略）⚠️ | 已写入（错误被 `_` 忽略）⚠️ | 无 | 204 | **始终 204** |
| **交互式 Shell** | — | — | — | WebSocket 关闭 | 前端显示 |

---

## 六、关键文件索引

| 模块 | 文件路径 | 核心函数/类型 |
|------|---------|-------------|
| 配置定义 | [settings/settings.go](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/settings/settings.go) | `Settings`, `Server`, `Server.EnableExec` |
| 用户权限 | [users/users.go](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/users/users.go) | `User.Commands`, `User.Perm` |
| 权限结构 | [users/permissions.go](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/users/permissions.go) | `Permissions.Execute` |
| 存储逻辑 | [settings/storage.go](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/settings/storage.go) | `Storage.Save`, `defaultEvents` |
| Bolt 存储 | [storage/bolt/config.go](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/storage/bolt/config.go) | `settingsBackend` |
| 命令解析 | [runner/commands.go](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/runner/commands.go) | `SplitCommandAndArgs`, `parseWindowsCommand` |
| 命令解析 | [runner/parser.go](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/runner/parser.go) | `ParseCommand` |
| 钩子执行 | [runner/runner.go](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/runner/runner.go) | `Runner.RunHook`, `Runner.exec` |
| 路由注册 | [http/http.go](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/http.go#L60-L70) | 所有资源 API 路由 |
| WS 服务端 | [http/commands.go](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/commands.go) | `commandsHandler` |
| 上下文注入 | [http/data.go](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/data.go) | `data.Runner` 初始化 |
| 删除/上传/保存 | [http/resource.go](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/resource.go) | `resourceDeleteHandler`, `resourcePostHandler`, `resourcePutHandler`, `resourcePatchHandler`, `patchAction` |
| 分片上传 | [http/tus_handlers.go](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/tus_handlers.go) | `tusPostHandler`, `tusPatchHandler`, `tusDeleteHandler` |
| CLI 管理 | [cmd/cmds.go](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/cmd/cmds.go) | `cmdsCmd`, `printEvents` |
| CLI 添加 | [cmd/cmds_add.go](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/cmd/cmds_add.go) | `cmdsAddCmd` |
| CLI 列出 | [cmd/cmds_ls.go](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/cmd/cmds_ls.go) | `cmdsLsCmd` |
| CLI 删除 | [cmd/cmds_rm.go](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/cmd/cmds_rm.go) | `cmdsRmCmd` |
| 前端 API | [frontend/src/api/commands.ts](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/frontend/src/api/commands.ts) | `command()` |
| 前端 UI | [frontend/src/components/Shell.vue](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/frontend/src/components/Shell.vue) | Shell 终端组件 |
| 前端配置 | [frontend/src/components/settings/Commands.vue](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/frontend/src/components/settings/Commands.vue) | 用户命令白名单输入 |
