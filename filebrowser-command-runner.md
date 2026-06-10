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

#### 事件触发点

| 事件名 | 触发代码位置 | 说明 |
|--------|-------------|------|
| `delete` | [resource.go:113](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/resource.go#L113-L115) | 删除文件/目录 |
| `save` | [resource.go:161](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/resource.go#L161-L164)、[resource.go:198](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/resource.go#L198-L201) | 新建或覆写文件 |
| `copy` / `rename` | [resource.go:256](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/resource.go#L256-L258) | action 取自请求参数（PATCH method） |
| `upload` | [tus_handlers.go:234](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/tus_handlers.go#L234) | TUS 分片上传完成 |

**调用形态**：`d.RunHook(func() error { ...实际操作... }, "delete", r.URL.Path, "", d.user)`

#### 执行内部逻辑（exec 函数）

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

| 维度 | 事件钩子模式 | 交互式 Shell 模式 |
|------|-------------|------------------|
| **传输方式** | `os.Stdout` / `os.Stderr`（落服务器日志） | WebSocket `TextMessage`（实时推前端） |
| **输出采集** | 不采集，直接打印 | `StdoutPipe` + `StderrPipe` 逐行读取 |
| **错误影响** | before/阻塞模式错误会**终止主操作** | 仅在前端显示，无副作用 |
| **可用环境变量** | `$FILE` `$SCOPE` `$TRIGGER` `$USERNAME` `$DESTINATION` | 无（仅继承系统环境） |
| **CWD** | 未显式设置（继承 FileBrowser 进程） | `cmd.Dir = 用户当前浏览路径` |
| **异步支持** | 命令末尾加 `&` | 否（WebSocket 生命周期内同步等待） |

---

## 六、关键文件索引

| 模块 | 文件路径 | 核心函数/类型 |
|------|---------|-------------|
| 配置定义 | [settings/settings.go](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/settings/settings.go) | `Settings`, `Server` |
| 用户权限 | [users/users.go](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/users/users.go) | `User` |
| 存储逻辑 | [settings/storage.go](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/settings/storage.go) | `Storage.Save`, `defaultEvents` |
| 命令解析 | [runner/commands.go](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/runner/commands.go) | `SplitCommandAndArgs` |
| 命令解析 | [runner/parser.go](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/runner/parser.go) | `ParseCommand` |
| 钩子执行 | [runner/runner.go](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/runner/runner.go) | `Runner.RunHook`, `Runner.exec` |
| WS 服务端 | [http/commands.go](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/commands.go) | `commandsHandler` |
| 上下文注入 | [http/data.go](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/data.go) | `data.Runner` 初始化 |
| 事件触发 | [http/resource.go](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/resource.go) | `d.RunHook(...)` 多处 |
| 上传事件 | [http/tus_handlers.go](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/http/tus_handlers.go#L234) | upload 钩子 |
| CLI 管理 | [cmd/cmds_add.go](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/cmd/cmds_add.go) 等 | `cmds add/ls/rm` |
| 前端 API | [frontend/src/api/commands.ts](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/frontend/src/api/commands.ts) | `command()` |
| 前端 UI | [frontend/src/components/Shell.vue](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/frontend/src/components/Shell.vue) | Shell 终端组件 |
| 前端配置 | [frontend/src/components/settings/Commands.vue](file:///d:/fz/0601/solo-dogfeeding/code/171-filebrowser/frontend/src/components/settings/Commands.vue) | 用户命令白名单输入 |
