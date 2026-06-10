# File Browser WebSocket 实时通信代码分析

## 概述

File Browser 项目中的 WebSocket 功能主要用于**交互式命令执行（Interactive Shell）**，允许用户在 Web 界面底部打开一个终端窗口，实时执行服务器端命令并查看输出。

---

## 一、整体架构概览

```
前端 (Vue)                          后端 (Go)
┌─────────────────┐               ┌──────────────────────────┐
│ Shell.vue       │─── HTTP GET  ──▶│ 路由注册 /api/command   │
│  (UI 组件)     │   WebSocket    │                        │
└────────┬────────│   Upgrade      │  withUser 中间件 (JWT)   │
         │        │               │                        │
         ▼        │               └──────────┬───────────────┘
┌─────────────────┐                          │
│ commands.ts    │                          ▼
│  (API 封装)    │◀─── WS 连接  ───▶  commandsHandler        │
└─────────────────┘    消息收发        │  (核心处理逻辑)          │
                                 └──────────┬───────────────┘
                                            │
                                            ▼
                                 ┌──────────────────────────┐
                                 │ runner 模块             │
                                 │  - ParseCommand        │
                                 │  - exec.Command       │
                                 │  - stdout/stderr 输出  │
                                 └──────────────────────────┘
```

---

## 二、代码路径详解

### 2.1 前端部分

#### 2.1.1 WebSocket API 封装 —— [commands.ts](file:///d:/fz/0601/solo-dogfeeding/code/174-filebrowser/frontend/src/api/commands.ts)

```typescript
import { baseURL } from "@/utils/constants";
import { removePrefix } from "./utils";

const ssl = window.location.protocol === "https:";
const protocol = ssl ? "wss:" : "ws:";

export default function command(
  url: string,
  command: string,
  onmessage: WebSocket["onmessage"],
  onclose: WebSocket["onclose"]
) {
  url = removePrefix(url);
  url = `${protocol}//${window.location.host}${baseURL}/api/command${url}`;

  const conn = new window.WebSocket(url);
  conn.onopen = () => conn.send(command);
  conn.onmessage = onmessage;
  conn.onclose = onclose;
}
```

**关键点：**

- **协议选择**：根据当前页面协议自动选择 `ws:` 或 `wss:`
- **URL 构建**：`${protocol}//${host}${baseURL}/api/command${当前路径}`
- **连接建立后立即发送命令**：`onopen` 触发时通过 `conn.send(command)` 发送要执行的命令
- **一次性使用**：每次调用 `command()` 都会创建新的 WebSocket 连接，命令执行完毕后关闭

#### 2.1.2 UI 组件 —— [Shell.vue](file:///d:/fz/0601/solo-dogfeeding/code/174-filebrowser/frontend/src/components/Shell.vue)

**调用流程：**

1. 用户在终端输入命令后按回车，触发 `submit()` 方法（第144行）
2. 特殊命令处理：
   - `clear`：清空终端内容
   - `exit`：关闭终端窗口
3. 普通命令：调用 `commands()` API（第174-190行）
4. `onmessage` 回调：实时追加命令输出到终端显示
5. `onclose` 回调：清理 ANSI 颜色码，重新允许输入

```javascript
commands(
  this.path,           // 当前文件路径，作为命令执行的工作目录
  cmd,                  // 用户输入的命令
  (event) => {         // onmessage: 接收输出
    results.text += `${event.data}\n`;
    this.scroll();
  },
  () => {                // onclose: 连接关闭
    results.text = results.text
      .replace(/\u001b\[[0-9;]+m/g, "")  // 过滤 ANSI 颜色码
      .trimEnd();
    this.canInput = true;
    this.$refs.input.focus();
    this.scroll();
  }
);
```

---

### 2.2 后端路由注册

#### 2.2.1 HTTP 路由 —— [http.go](file:///d:/fz/0601/solo-dogfeeding/code/174-filebrowser/http/http.go#L85)

```go
api.PathPrefix("/command").Handler(monkey(commandsHandler, "/api/command")).Methods("GET")
```

**关键点：**

- 路由路径：`/api/command`（会被 `stripPrefix` 去掉前缀）
- HTTP 方法：`GET`（WebSocket 握手必须使用 GET）
- 处理函数：`commandsHandler`，被 `monkey()` 包装

#### 2.2.2 Handle 包装器 —— [data.go](file:///d:/fz/0601/solo-dogfeeding/code/174-filebrowser/http/data.go#L66-L101)

```go
func handle(fn handleFunc, prefix string, store *storage.Storage, server *settings.Server) http.Handler {
    handler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // 1. 设置全局响应头
        for k, v := range globalHeaders {
            w.Header().Set(k, v)
        }

        // 2. 从存储获取全局设置
        settings, err := store.Settings.Get()

        // 3. 构造 data 对象，注入 Runner、store、settings、server
        status, err := fn(w, r, &data{
            Runner:   &runner.Runner{Enabled: server.EnableExec, Settings: settings},
            store:    store,
            settings: settings,
            server:   server,
        })

        // 4. 错误处理与日志
        if status >= 400 || err != nil {
            log.Printf(...)
        }
        if status != 0 {
            http.Error(w, ...)
        }
    })

    return stripPrefix(prefix, handler)
}
```

---

### 2.3 后端认证中间件

#### 2.3.1 withUser 中间件 —— [auth.go](file:///d:/fz/0601/solo-dogfeeding/code/174-filebrowser/http/auth.go#L85-L111)

WebSocket 连接同样走标准的 JWT 认证流程：

```go
func withUser(fn handleFunc) handleFunc {
    return func(w http.ResponseWriter, r *http.Request, d *data) (int, error) {
        // JWT 解析流程:
        // 1. 从 X-Auth 请求头 或 Cookie("auth") 提取 token
        // 2. 使用 HS256 算法验证签名
        // 3. 检查 token 是否过期
        // 4. 从 token 中提取 user ID
        // 5. 从数据库加载完整用户信息到 d.user
        // 6. 检查是否需要刷新 token（过期时间 < 1小时 或 用户信息已更新）

        d.user, err = d.store.Users.Get(d.server.Root, tk.User.ID)
        return fn(w, r, d)
    }
}
```

**Token 提取器 extractor —— [auth.go](file:///d:/fz/0601/solo-dogfeeding/code/174-filebrowser/http/auth.go#L47-L67)：**

```go
func (e extractor) ExtractToken(r *http.Request) (string, error) {
    // 优先级:
    // 1. 先尝试 X-Auth header
    // 2. 若是 GET 请求，再尝试 Cookie("auth")
    // WebSocket 握手是 GET 请求，所以能走 Cookie 认证
}
```

> **重要**：WebSocket 握手请求是 HTTP GET，所以 WebSocket 可以通过 Cookie 携带认证信息。

---

### 2.4 WebSocket 核心处理逻辑

#### 2.4.1 Upgrader 配置 —— [commands.go](file:///d:/fz/0601/solo-dogfeeding/code/174-filebrowser/http/commands.go#L18-L25)

```go
const (
    WSWriteDeadline = 10 * time.Second
)

var upgrader = websocket.Upgrader{
    ReadBufferSize:  1024,
    WriteBufferSize: 1024,
}
```

#### 2.4.2 commandsHandler —— [commands.go](file:///d:/fz/0601/solo-dogfeeding/code/174-filebrowser/http/commands.go#L41-L120)

这是整个 WebSocket 处理的**核心函数**，完整流程：

```go
var commandsHandler = withUser(func(w http.ResponseWriter, r *http.Request, d *data) (int, error) {
    // ──────────────────────────────────────────────
    // 阶段 1: WebSocket 握手升级
    // ──────────────────────────────────────────────
    conn, err := upgrader.Upgrade(w, r, nil)
    if err != nil {
        return http.StatusInternalServerError, err
    }
    defer conn.Close()  // 函数退出时自动关闭连接

    // ──────────────────────────────────────────────
    // 阶段 2: 读取客户端发送的命令
    // ──────────────────────────────────────────────
    var raw string
    for {
        _, msg, err := conn.ReadMessage()   // 阻塞读取第一条消息（即命令）
        if err != nil {
            wsErr(conn, r, http.StatusInternalServerError, err)
            return 0, nil
        }
        raw = strings.TrimSpace(string(msg))
        if raw != "" {
            break
        }
    }

    // ──────────────────────────────────────────────
    // 阶段 3: 权限校验
    // ──────────────────────────────────────────────

    // 3.1 全局执行开关 + 用户执行权限
    if !d.server.EnableExec || !d.user.Perm.Execute {
        conn.WriteMessage(websocket.TextMessage, cmdNotAllowed)
        return 0, nil
    }

    // 3.2 解析命令
    command, name, err := runner.ParseCommand(d.settings, raw)

    // 3.3 检查该命令是否在用户允许列表中
    if !slices.Contains(d.user.Commands, name) {
        conn.WriteMessage(websocket.TextMessage, cmdNotAllowed)
        return 0, nil
    }

    // ──────────────────────────────────────────────
    // 阶段 4: 执行命令并实时推送输出
    // ──────────────────────────────────────────────

    // 4.1 创建命令，设置工作目录为用户当前路径
    cmd := exec.Command(command[0], command[1:]...)
    cmd.Dir = d.user.FullPath(r.URL.Path)

    // 4.2 获取 stdout 和 stderr 管道
    stdout, _ := cmd.StdoutPipe()
    stderr, _ := cmd.StderrPipe()

    // 4.3 启动命令
    cmd.Start()

    // 4.4 使用 Scanner 逐行读取输出并通过 WebSocket 实时推送
    s := bufio.NewScanner(io.MultiReader(stdout, stderr))
    for s.Scan() {
        conn.WriteMessage(websocket.TextMessage, s.Bytes())
    }

    // 4.5 等待命令结束
    cmd.Wait()

    return 0, nil
})
```

---

## 三、连接生命周期详解

### 3.1 连接建立流程

```
浏览器                                服务器
  │                                     │
  │  1. HTTP GET /api/command/path  │
  │     Upgrade: websocket                 │
  │     Cookie: auth=<jwt>                │
  │────────────────────────────────────▶│
  │                                     │
  │  2. withUser 认证                  │
  │     - 解析 JWT，加载用户          │
  │                                     │
  │  3. 101 Switching Protocols     │
  │     Upgrade: websocket                 │
  │◀────────────────────────────────────│
  │                                     │
  │  4. WebSocket 连接已建立             │
  │                                     │
  │  5. 发送命令文本                   │
  │  conn.send(command)                │
  │────────────────────────────────────▶│
  │                                     │
  │  6. 服务器执行命令                  │
  │                                     │
  │  7. 逐行推送 stdout/stderr         │
  │◀────────────────────────────────────│
  │     (多条 TextMessage)                   │
  │                                     │
  │  8. 命令执行完毕，关闭连接            │
  │◀────────────────────────────────────│
  │     Close Frame                       │
```

### 3.2 连接保持机制

**该项目没有心跳/心跳：File Browser 的 WebSocket 实现是"一次性"的，没有实现长连接或心跳机制。**

每次用户执行一条命令都会：

1. 创建一个全新的 WebSocket 连接

2. 发送一条命令

3. 实时接收输出流

4. 命令执行完后关闭连接

**这是一种**"短连接"模式，每个命令对应一个 WebSocket 连接。

---

## 四、消息格式

### 4.1 客户端 → 服务器

只有**文本消息（TextMessage）：命令字符串

示例：
```
"ls -la"
```

### 4.2 服务器 → 客户端

**文本消息（TextMessage）：命令输出的每一行文本

示例（多条消息）：
```
"total 32"
"drwxr-xr-x  5 user  staff   160 Jun 10 10:00 ."
"drwxr-xr-x  3 user  staff    96 Jun 10 09:00 .."
```

**错误消息：**
- `"Command not allowed." — 权限不足
- 其他错误信息

---

## 五、安全机制

### 5.1 多层权限校验

| 层级 | 检查点 | 代码位置 |
|------|--------|-----------|
| 1 | 全局执行开关 | `d.server.EnableExec` | commands.go:64 |
| 2 | 用户执行权限 | `d.user.Perm.Execute` | commands.go:64 |
| 3 | 用户允许的命令列表 | `d.user.Commands` 白名单 | commands.go:80 |

### 5.2 工作目录限制

```go
cmd.Dir = d.user.FullPath(r.URL.Path)
```

命令执行的工作目录被限制在用户当前浏览的路径下，结合用户的 Scope 限制。

### 5.3 JWT 认证

WebSocket 握手阶段就完成了用户身份认证。

---

## 六、命令解析流程

### 6.1 ParseCommand —— [parser.go](file:///d:/fz/0601/solo-dogfeeding/code/174-filebrowser/runner/parser.go#L10-L25)

```go
func ParseCommand(s *settings.Settings, raw string) (command []string, name string, err error) {
    // 1. 按 shell 风格拆分命令和参数
    name, args, err := SplitCommandAndArgs(raw)

    // 2. 如果配置了 Shell（如 ["bash", "-c"]），则通过 shell 执行
    if len(s.Shell) == 0 || s.Shell[0] == "" {
        command = append(command, name)
        command = append(command, args...)
    } else {
        command = append(command, s.Shell...)    // e.g. ["bash", "-c"]
        command = append(command, raw)            // e.g. "ls -la"
    }

    return command, name, nil
}
```

### 6.2 跨平台命令拆分 —— [commands.go](file:///d:/fz/0601/solo-dogfeeding/code/174-filebrowser/runner/commands.go#L34-L58)

- **Windows**：自定义解析器，处理反斜杠路径和引号转义

- **Unix/Linux**：使用 `go-shlex` 库解析

---

## 七、关键文件索引

| 文件 | 作用 |
|------|------|
| [frontend/src/api/commands.ts](file:///d:/fz/0601/solo-dogfeeding/code/174-filebrowser/frontend/src/api/commands.ts) | 前端 WebSocket API 封装 |
| [frontend/src/components/Shell.vue](file:///d:/fz/0601/solo-dogfeeding/code/174-filebrowser/frontend/src/components/Shell.vue) | 前端终端 UI 组件 |
| [http/http.go](file:///d:/fz/0601/solo-dogfeeding/code/174-filebrowser/http/http.go) | 路由注册 |
| [http/commands.go](file:///d:/fz/0601/solo-dogfeeding/code/174-filebrowser/http/commands.go) | WebSocket 核心处理器 |
| [http/auth.go](file:///d:/fz/0601/solo-dogfeeding/code/174-filebrowser/http/auth.go) | JWT 认证中间件 |
| [http/data.go](file:///d:/fz/0601/solo-dogfeeding/code/174-filebrowser/http/data.go) | 请求上下文与处理包装器 |
| [runner/parser.go](file:///d:/fz/0601/solo-dogfeeding/code/174-filebrowser/runner/parser.go) | 命令解析 |
| [runner/commands.go](file:///d:/fz/0601/solo-dogfeeding/code/174-filebrowser/runner/commands.go) | 跨平台命令拆分 |
| [runner/runner.go](file:///d:/fz/0601/solo-dogfeeding/code/174-filebrowser/runner/runner.go) | Hook 命令执行器（用于事件钩子） |

---

## 八、设计特点与改进空间

### 现有设计特点：

1. **简洁**：每个命令一个 WebSocket 连接，简单直接

2. **实时性**：使用 `io.MultiReader` + `bufio.Scanner` 实现 stdout/stderr 合并逐行推送

3. **安全性**：三层权限校验 + 工作目录限制

### 潜在改进点：

1. **心跳机制缺失**：没有 ping/pong 心跳，网络异常时可能无法及时感知断连

2. **复用连接**：每条命令新建连接开销较大，可考虑长连接复用

3. **输入支持**：当前只支持单向输出推送，不支持交互输入（如 `sudo` 密码输入）

4. **ANSI 转义序列处理**：前端只简单过滤颜色码，可考虑完整的终端仿真
