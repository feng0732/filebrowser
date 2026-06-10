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
│  事件钩子：Stdout/Stderr → 服务器日志 + 错误中断流程                  │
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
| **错误影响** | `before_delete` 钩子失败 → 删除操作**不会执行**<br>`after_delete` 钩子失败 → 文件已删，但返回错误给前端 |  |
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
| **失败回滚** | Hook 失败（含 before 钩子或主操作）→ 执行 `d.user.Fs.RemoveAll(r.URL.Path)` 清理半成文件 | [resource.go:172-174](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/resource.go#L172-L174) |
| **环境变量** | `$FILE` = 新建文件路径<br>`$TRIGGER` = `"before_upload"` / `"after_upload"` |  |
| **输出返回** | Stdout/Stderr → 服务器日志<br>成功返回 `201 Created`（由后续逻辑），失败由 `errToStatus` 映射 |  |

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
| **输出返回** | Stdout/Stderr → 服务器日志 |  |

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
| **输出返回** | Stdout/Stderr → 服务器日志 |  |

---

#### 🔹 操作 5：重命名（PATCH /api/resources/{src}?action=rename&destination={dst}）

**路由**：同复制，同 handler，仅 `action` 参数不同

| 阶段 | 详情 | 代码 |
|------|------|------|
| **前置检查** | 1. 解析 `action=rename`、`destination` 参数<br>2. 同复制的路径检查逻辑<br>3. 需 `Perm.Rename` 权限（在 patchAction 内） | [resource.go:213-255](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/resource.go#L213-L255) |
| **RunHook 调用** | `d.RunHook(fn, "rename", src, dst, d.user)` | [resource.go:256-258](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/resource.go#L256-L258) |
| **包装的主操作** | `patchAction(ctx, "rename", src, dst, d, fileCache)`：<br>1. 先删源文件的缩略图缓存<br>2. 调用 `fileutils.MoveFile(...)` | [resource.go:348-373](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/resource.go#L348-L373) |
| **环境变量** | `$FILE` = 源文件路径<br>`$DESTINATION` = 目标路径<br>`$TRIGGER` = `"before_rename"` / `"after_rename"` |  |
| **输出返回** | Stdout/Stderr → 服务器日志 |  |

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
| **特殊说明** | `before_upload` 钩子在**文件已写入完成后**才执行（因为主操作为空）<br>但钩子错误仍然会导致 API 返回错误状态码<br>错误被 `_` 忽略：`_ = d.RunHook(...)`，不影响 HTTP 204 返回 | [tus_handlers.go:234](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/tus_handlers.go#L234) |

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
   - 非阻塞：`cmd.Start()` + 异步 `cmd.Wait()`（失败仅记录日志，不影响主流程）
   - 阻塞：`cmd.Run()`（失败返回错误，可能中断主流程）

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

## 五、输出返回对比

### 5.1 两大模式对比

| 维度 | 事件钩子模式 | 交互式 Shell 模式 |
|------|-------------|------------------|
| **传输方式** | `os.Stdout` / `os.Stderr`（落服务器日志） | WebSocket `TextMessage`（实时推前端） |
| **输出采集** | 不采集，直接打印 | `StdoutPipe` + `StderrPipe` 逐行读取 |
| **错误影响** | before/阻塞模式错误会**终止主操作** | 仅在前端显示，无副作用 |
| **可用环境变量** | `$FILE` `$SCOPE` `$TRIGGER` `$USERNAME` `$DESTINATION` | 无（仅继承系统环境） |
| **CWD** | 未显式设置（继承 FileBrowser 进程） | `cmd.Dir = 用户当前浏览路径` |
| **异步支持** | 命令末尾加 `&` | 否（WebSocket 生命周期内同步等待） |

---

### 5.2 各操作的输出返回链路详解

#### 5.2.1 事件钩子命令输出路径

```
                    Runner.exec()
  ┌───────────────────────────────────────────────┐
  │                                               │
  │  cmd.Stdin  = os.Stdin                        │
  │  cmd.Stdout = os.Stdout  ────►  服务器 stdout │
  │  cmd.Stderr = os.Stderr  ────►  服务器 stderr │
  │                                               │
  │  阻塞模式: cmd.Run()                          │
  │    命令退出码 != 0 → 返回 error → 主操作中止   │
  │                                               │
  │  非阻塞模式 (&): cmd.Start()                  │
  │    go func() { cmd.Wait() }                   │
  │    失败仅打日志，不影响主流程                  │
  └───────────────────────────────────────────────┘
```

**代码位置**：[runner.go:99-117](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/runner/runner.go#L99-L117)

```go
cmd.Stdin = os.Stdin
cmd.Stdout = os.Stdout
cmd.Stderr = os.Stderr

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
    return cmd.Start()
}

log.Printf("[INFO] Blocking Command: \"%s\"", strings.Join(command, " "))
return cmd.Run()
```

---

#### 5.2.2 各操作的错误返回路径

| 操作 | 命令失败时的处理 | 代码位置 |
|------|-----------------|---------|
| **删除** | `before_delete` 失败 → 文件不删，返回错误<br>`after_delete` 失败 → 文件已删，仍返回错误 | [resource.go:117-119](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/resource.go#L117-L119) |
| **普通上传** | 任一阶段失败 → 先 `RemoveAll` 清理半成文件，再返回错误 | [resource.go:172-174](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/resource.go#L172-L174) |
| **保存** | 任一阶段失败 → 直接返回错误（原文件可能已被破坏） | [resource.go:209](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/resource.go#L209) |
| **复制** | `before_copy` 失败 → 不复制<br>`after_copy` 失败 → 已复制，返回错误 | [resource.go:260](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/resource.go#L260) |
| **重命名** | `before_rename` 失败 → 不重命名<br>`after_rename` 失败 → 已重命名，返回错误 | [resource.go:260](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/resource.go#L260) |
| **分片上传** | 钩子错误被 `_` 忽略！<br>`_ = d.RunHook(...)` → 始终返回 HTTP 204 | [tus_handlers.go:234](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/tus_handlers.go#L234) |

> **⚠️ 重要提示**：分片上传完成时的钩子错误被显式忽略了（`_ = d.RunHook(...)`），这意味着即使 `before_upload` 或 `after_upload` 钩子执行失败，API 仍会返回 `204 No Content`，客户端不会感知到错误。这是与普通上传（POST）的关键区别。

---

#### 5.2.3 HTTP 状态码映射

命令错误通过 `errToStatus(err)` 函数映射为 HTTP 状态码，最终返回给前端：

```
命令执行错误（os/exec.ExitError）
    │
    ▼
errToStatus(err) ───► http.StatusInternalServerError (500)
    │
    ▼
http.Error(w, "500 Internal Server Error (exit status 1)", 500)
```

同时服务器日志会记录：
```
202X/XX/XX XX:XX:XX /api/resources/test.txt: 500 127.0.0.1 exit status 1
```

---

#### 5.2.4 交互式 Shell 输出路径

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

**代码位置**：[commands.go:91-117](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/commands.go#L91-L117)

**前端接收**（[Shell.vue:177-189](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/frontend/src/components/Shell.vue#L177-L189)）：
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
    this.canInput = true;
  }
);
```

---

### 5.3 关键差异总结表

| 操作 | before 失败影响 | after 失败影响 | 失败回滚 | 输出去向 |
|------|----------------|---------------|---------|---------|
| **删除** | ❌ 文件保留 | ❌ 文件已删 | 无 | 服务器日志 |
| **普通上传** | ❌ 文件不建 | ✅ 已创建的文件会被删除 | `RemoveAll` 清理 | 服务器日志 |
| **保存** | ❌ 文件保留 | ❌ 文件可能已破坏 | 无 | 服务器日志 |
| **复制** | ❌ 不复制 | ❌ 文件已复制 | 无 | 服务器日志 |
| **重命名** | ❌ 不重命名 | ❌ 文件已重命名 | 无 | 服务器日志 |
| **分片上传** | ⚠️ 文件已写入，错误被忽略 | ⚠️ 文件已写入，错误被忽略 | 无 | 服务器日志 |
| **交互式 Shell** | — | — | — | 前端终端 |

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
