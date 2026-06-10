# File Browser WebSocket 实时通信代码分析

## 概述

File Browser 项目中的 WebSocket 功能用于**交互式命令执行（Interactive Shell）**，允许用户在 Web 界面底部打开终端窗口，实时执行服务器端命令并查看输出。

---

## 一、整体架构概览

```
前端 (Vue)                          后端 (Go)
┌─────────────────┐               ┌──────────────────────────┐
│ Shell.vue       │─── HTTP GET  ──▶│ 路由注册 /api/command   │
│  (UI 组件)      │   WebSocket    │                          │
└────────┬────────│   Upgrade      │  withUser 中间件 (JWT)   │
         │        │               │                          │
         ▼        │               └──────────┬───────────────┘
┌─────────────────┐                          │
│ commands.ts     │                          ▼
│  (API 封装)     │◀─── WS 连接  ───▶  commandsHandler        │
└─────────────────┘    消息收发        │  (核心处理逻辑)        │
                                 └──────────┬───────────────┘
                                            │
                                            ▼
                                 ┌──────────────────────────┐
                                 │ runner 模块              │
                                 │  - ParseCommand          │
                                 │  - exec.Command          │
                                 │  - stdout/stderr 输出    │
                                 └──────────────────────────┘
```

---

## 二、代码路径详解

### 2.1 前端部分

#### 2.1.1 WebSocket API 封装 —— [commands.ts](frontend/src/api/commands.ts)

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

#### 2.1.2 UI 组件 —— [Shell.vue](frontend/src/components/Shell.vue)

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
  () => {              // onclose: 连接关闭
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

#### 2.2.1 HTTP 路由 —— [http.go](http/http.go#L85)

```go
api.PathPrefix("/command").Handler(monkey(commandsHandler, "/api/command")).Methods("GET")
```

- 路由路径：`/api/command`（会被 `stripPrefix` 去掉前缀）
- HTTP 方法：`GET`（WebSocket 握手必须使用 GET）
- 处理函数：`commandsHandler`，被 `monkey()` 包装

#### 2.2.2 Handle 包装器 —— [data.go](http/data.go#L66-L101)

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
    })

    return stripPrefix(prefix, handler)
}
```

---

### 2.3 后端认证中间件

#### 2.3.1 withUser 中间件 —— [auth.go](http/auth.go#L85-L111)

WebSocket 连接走标准的 JWT 认证流程：

1. 从 `X-Auth` 请求头或 `Cookie("auth")` 提取 token
2. 使用 HS256 算法验证签名
3. 检查 token 是否过期
4. 从 token 中提取 user ID
5. 从数据库加载完整用户信息到 `d.user`
6. 检查是否需要刷新 token

**Token 提取器 extractor —— [auth.go](http/auth.go#L49-L67)：**

```go
func (e extractor) ExtractToken(r *http.Request) (string, error) {
    // 优先级:
    // 1. 先尝试 X-Auth header
    // 2. 若是 GET 请求，再尝试 Cookie("auth")
    // WebSocket 握手是 GET 请求，所以能走 Cookie 认证
}
```

> **重要**：WebSocket 握手请求是 HTTP GET，所以可以通过 Cookie 携带认证信息。

---

### 2.4 WebSocket 核心处理逻辑

#### 2.4.1 Upgrader 配置与错误工具 —— [commands.go](http/commands.go#L18-L39)

```go
const (
    WSWriteDeadline = 10 * time.Second
)

var upgrader = websocket.Upgrader{
    ReadBufferSize:  1024,
    WriteBufferSize: 1024,
}

var cmdNotAllowed = []byte("Command not allowed.")
```

**`wsErr` 函数 —— [commands.go](http/commands.go#L31-L39)：**

这是所有错误场景下统一使用的 WebSocket 关闭函数：

```go
func wsErr(ws *websocket.Conn, r *http.Request, status int, err error) {
    txt := http.StatusText(status)
    if err != nil || status >= 400 {
        log.Printf("%s: %v %s %v", r.URL.Path, status, r.RemoteAddr, err)
    }
    if err := ws.WriteControl(websocket.CloseInternalServerErr, []byte(txt), time.Now().Add(WSWriteDeadline)); err != nil {
        log.Print(err)
    }
}
```

它做的事情：
1. 条件性日志：当 `err != nil` 或 `status >= 400` 时打印请求路径、状态码、客户端地址和错误
2. 发送 **Close 控制帧**：使用 `WriteControl` 发送 `CloseInternalServerErr`（关闭码 1011），消息体为 HTTP 状态文本（如 "Internal Server Error"），写入截止时间 10 秒
3. 如果 Close 帧本身写入失败，仅打印日志，不再做进一步处理

**关键**：`wsErr` 发送的是 Close 帧，不是普通 TextMessage。客户端收到 Close 帧后 `onclose` 会被触发。

#### 2.4.2 commandsHandler 完整流程 —— [commands.go](http/commands.go#L41-L120)

```go
var commandsHandler = withUser(func(w http.ResponseWriter, r *http.Request, d *data) (int, error) {
    conn, err := upgrader.Upgrade(w, r, nil)
    if err != nil {
        return http.StatusInternalServerError, err
    }
    defer conn.Close()

    var raw string
    for {
        _, msg, err := conn.ReadMessage()
        if err != nil {
            wsErr(conn, r, http.StatusInternalServerError, err)
            return 0, nil
        }
        raw = strings.TrimSpace(string(msg))
        if raw != "" {
            break
        }
    }

    if !d.server.EnableExec || !d.user.Perm.Execute {
        if err := conn.WriteMessage(websocket.TextMessage, cmdNotAllowed); err != nil {
            wsErr(conn, r, http.StatusInternalServerError, err)
        }
        return 0, nil
    }

    command, name, err := runner.ParseCommand(d.settings, raw)
    if err != nil {
        if err := conn.WriteMessage(websocket.TextMessage, []byte(err.Error())); err != nil {
            wsErr(conn, r, http.StatusInternalServerError, err)
        }
        return 0, nil
    }

    if !slices.Contains(d.user.Commands, name) {
        if err := conn.WriteMessage(websocket.TextMessage, cmdNotAllowed); err != nil {
            wsErr(conn, r, http.StatusInternalServerError, err)
        }
        return 0, nil
    }

    cmd := exec.Command(command[0], command[1:]...)
    cmd.Dir = d.user.FullPath(r.URL.Path)

    stdout, err := cmd.StdoutPipe()
    if err != nil {
        wsErr(conn, r, http.StatusInternalServerError, err)
        return 0, nil
    }

    stderr, err := cmd.StderrPipe()
    if err != nil {
        wsErr(conn, r, http.StatusInternalServerError, err)
        return 0, nil
    }

    if err := cmd.Start(); err != nil {
        wsErr(conn, r, http.StatusInternalServerError, err)
        return 0, nil
    }

    s := bufio.NewScanner(io.MultiReader(stdout, stderr))
    for s.Scan() {
        if err := conn.WriteMessage(websocket.TextMessage, s.Bytes()); err != nil {
            log.Print(err)
        }
    }

    if err := cmd.Wait(); err != nil {
        wsErr(conn, r, http.StatusInternalServerError, err)
    }

    return 0, nil
})
```

---

## 三、stdout 与 stderr 读取顺序精确分析

### 3.1 io.MultiReader 的读取语义

核心代码位于 [commands.go#L108](http/commands.go#L108)：

```go
s := bufio.NewScanner(io.MultiReader(stdout, stderr))
```

**`io.MultiReader(stdout, stderr)` 的工作方式：**

`io.MultiReader` 将多个 `io.Reader` 串联成一个逻辑 Reader。它的读取规则是**严格的串行顺序**：

1. 先从第一个 Reader（`stdout`）读取
2. 只有当第一个 Reader 返回 `io.EOF` 后，才切换到第二个 Reader（`stderr`）
3. 第二个 Reader 也返回 `io.EOF` 后，`MultiReader` 整体返回 `io.EOF`

**这意味着：stdout 的所有输出被读完后，才开始读 stderr。两者不是交叉合并，而是顺序拼接。**

### 3.2 这带来了什么行为

假设一个命令同时向 stdout 和 stderr 写入：

```
时刻 T1: stdout 输出 "line1\n"
时刻 T2: stderr 输出 "err1\n"
时刻 T3: stdout 输出 "line2\n"
时刻 T4: 命令退出，stdout 关闭
```

客户端实际收到的消息顺序是：

```
"line1"
"line2"
"err1"
```

stderr 的输出 `err1` 即使在 `line2` 之前产生，也要等到 stdout 管道关闭（进程 stdout 写端关闭或进程退出）后才会被读取。

### 3.3 为什么 stdout 不会无限阻塞

关键在于 `cmd.StdoutPipe()` 和 `cmd.StderrPipe()` 返回的管道在进程退出时会被关闭：

- 当子进程退出或关闭 stdout 文件描述符时，`stdout` Read 会返回 `io.EOF`
- 此时 `MultiReader` 切换到 `stderr`
- 子进程退出后 `stderr` 也会关闭，返回 `io.EOF`
- `MultiReader` 整体返回 `io.EOF`，Scanner 循环结束

**但如果子进程持续向 stdout 写入而不关闭，stderr 中的内容将一直积压，直到 stdout 管道关闭为止。**

### 3.4 对实际使用的影响

大多数命令行工具的行为模式是：
- 正常输出 → stdout
- 错误/警告 → stderr
- 命令结束后 stdout 和 stderr 同时关闭

对于这类命令，`MultiReader` 的串行读取不会造成问题，因为命令结束后 stdout 先 EOF，然后 stderr 也很快 EOF。

但对于**长时间运行的命令**（如 `tail -f`、`ping`），如果有 stderr 输出，该输出会被延迟到 stdout EOF 之后才发送，可能造成错误信息严重滞后。

---

## 四、命令结束后的关闭路径详解

### 4.1 正常结束的关闭路径

```
命令执行完成
    │
    ▼
stdout 管道关闭 (EOF)  ──▶  MultiReader 切换到 stderr
    │
    ▼
stderr 管道关闭 (EOF)  ──▶  MultiReader 返回 EOF
    │
    ▼
bufio.Scanner s.Scan() 返回 false，循环退出
    │
    ▼
cmd.Wait() 被调用（第115行）
    │
    ├── Wait() 返回 nil  ──▶  不调用 wsErr
    │                         直接 return 0, nil
    │                            │
    │                            ▼
    │                      defer conn.Close() 执行
    │                      发送 Close 帧，连接关闭
    │
    └── Wait() 返回 err  ──▶  调用 wsErr()
                               发送 Close 帧 (CloseInternalServerErr)
                                  │
                                  ▼
                             return 0, nil
                             defer conn.Close() 执行
                             （此时连接可能已由 wsErr 关闭，
                               Close() 是幂等的，不会报错）
```

### 4.2 各阶段异常的关闭路径

以下是 `commandsHandler` 中所有可能退出函数的位置及其关闭行为：

| 代码行 | 触发条件 | 关闭方式 | 客户端收到的最后一条消息 |
|--------|----------|----------|--------------------------|
| L53 | `ReadMessage` 失败（客户端断连等） | `wsErr` → Close 帧 (1011) | 无（连接已异常） |
| L65-68 | 全局/用户执行权限不足 + WriteMessage 失败 | `wsErr` → Close 帧 (1011) | 之前已收到 "Command not allowed." TextMessage |
| L64-69 | 全局/用户执行权限不足 + WriteMessage 成功 | `defer conn.Close()` → Close 帧 | 收到 "Command not allowed." TextMessage |
| L74-76 | 命令解析失败 + WriteMessage 失败 | `wsErr` → Close 帧 (1011) | 之前已收到错误信息 TextMessage |
| L73-78 | 命令解析失败 + WriteMessage 成功 | `defer conn.Close()` → Close 帧 | 收到错误信息 TextMessage |
| L81-83 | 命令不在白名单 + WriteMessage 失败 | `wsErr` → Close 帧 (1011) | 之前已收到 "Command not allowed." TextMessage |
| L80-86 | 命令不在白名单 + WriteMessage 成功 | `defer conn.Close()` → Close 帧 | 收到 "Command not allowed." TextMessage |
| L92-94 | StdoutPipe 创建失败 | `wsErr` → Close 帧 (1011) | Close 帧体 "Internal Server Error" |
| L98-100 | StderrPipe 创建失败 | `wsErr` → Close 帧 (1011) | Close 帧体 "Internal Server Error" |
| L103-105 | cmd.Start 失败 | `wsErr` → Close 帧 (1011) | Close 帧体 "Internal Server Error" |
| L115-117 | cmd.Wait 返回错误（命令非零退出码） | `wsErr` → Close 帧 (1011) | 之前已收到命令输出；Close 帧体 "Internal Server Error" |
| 正常 | 命令正常结束（退出码 0） | `defer conn.Close()` → Close 帧 | 之前已收到全部命令输出 |

### 4.3 关闭路径的重要细节

**1. 权限拒绝时的"双重消息"**

当权限不足时（第64-69行），代码先发送一条 `TextMessage`（"Command not allowed."），然后 `return 0, nil`。此时 `defer conn.Close()` 会执行，向客户端发送 Close 帧。

客户端的接收顺序：
1. `onmessage` 触发，收到 `"Command not allowed."`
2. `onclose` 触发，连接关闭

**2. 命令非零退出码 ≠ 连接异常**

当 `cmd.Wait()` 返回错误时（命令以非零退出码结束），代码调用 `wsErr` 发送 `CloseInternalServerErr` 关闭帧。这意味着即使命令只是返回了非零退出码（如 `ls` 找不到文件），WebSocket 也会以 1011 错误码关闭，而非正常关闭。

**3. `defer conn.Close()` 与 `wsErr` 的协同**

所有路径最终都会执行 `defer conn.Close()`。如果之前已经通过 `wsErr` 发送了 Close 帧，连接已处于半关闭或完全关闭状态，此时 `conn.Close()` 是幂等操作，不会产生错误。

---

## 五、输出推送时的写入失败处理

### 5.1 Scanner 循环中的写入失败 —— [commands.go#L109-L113](http/commands.go#L109-L113)

```go
s := bufio.NewScanner(io.MultiReader(stdout, stderr))
for s.Scan() {
    if err := conn.WriteMessage(websocket.TextMessage, s.Bytes()); err != nil {
        log.Print(err)
    }
}
```

**当前行为：写入失败后循环并未中断。**

如果 `conn.WriteMessage` 返回错误（例如客户端已断开连接），代码仅 `log.Print(err)` 打印日志，然后继续 `s.Scan()` 读取下一行，再次尝试写入，再次失败，再次打印日志……

这意味着：
- 如果命令产生 1000 行输出且客户端已断开，将产生 1000 条错误日志
- 子进程的输出仍被持续读取（Scanner 继续消费 stdout/stderr），进程不会被提前终止
- 直到 Scanner 读完全部输出（`s.Scan()` 返回 false），循环才结束
- 随后 `cmd.Wait()` 回收子进程资源

**这是一个"读尽后关闭"的策略**，而不是"写入失败立即终止"的策略。子进程不会被 kill，会自然执行到完成。

### 5.2 权限拒绝时的写入失败 —— [commands.go#L64-L69](http/commands.go#L64-L69)

```go
if !d.server.EnableExec || !d.user.Perm.Execute {
    if err := conn.WriteMessage(websocket.TextMessage, cmdNotAllowed); err != nil {
        wsErr(conn, r, http.StatusInternalServerError, err)
    }
    return 0, nil
}
```

这里如果 `WriteMessage` 失败，会调用 `wsErr` 发送 Close 帧。因为这是权限拒绝场景，不存在子进程，所以不需要考虑进程清理。

同样的模式出现在命令解析失败（第74-76行）和命令白名单检查（第81-83行）中。

### 5.3 对比两种写入失败的处理差异

| 场景 | 写入失败后的行为 | 是否中断流程 | 是否 kill 子进程 |
|------|-----------------|-------------|-----------------|
| 权限拒绝 / 解析失败 / 白名单拒绝 | `wsErr` → 发送 Close 帧 → return | 是 | 无子进程 |
| 命令输出推送中 | 仅 `log.Print` | 否 | 否 |

**输出推送阶段不中断的设计原因**：即使客户端断开，子进程仍在运行，Go 的 `exec.Cmd` 要求必须调用 `cmd.Wait()` 来回收子进程资源。如果提前 return 跳过 `cmd.Wait()`，子进程会变成僵尸进程。所以 Scanner 必须读完所有输出，`cmd.Wait()` 才能正确回收。

---

## 六、连接生命周期完整时序

### 6.1 正常命令执行

```
浏览器                                    服务器
  │                                         │
  │  1. HTTP GET /api/command/path          │
  │     Upgrade: websocket                  │
  │     Cookie: auth=<jwt>                  │
  │────────────────────────────────────────▶│
  │                                         │  2. withUser 认证
  │                                         │     解析 JWT，加载用户
  │                                         │
  │  3. 101 Switching Protocols             │
  │     Upgrade: websocket                  │
  │◀────────────────────────────────────────│
  │                                         │
  │  4. onopen 触发                         │
  │     conn.send("ls -la")                 │
  │────────────────────────────────────────▶│
  │                                         │  5. ReadMessage 读取命令
  │                                         │     权限校验通过
  │                                         │     cmd.Start()
  │                                         │
  │  6. onmessage 触发                      │  Scanner 逐行读取 stdout
  │◀── "total 32" ─────────────────────────│  conn.WriteMessage
  │◀── "drwxr-xr-x ..." ──────────────────│
  │◀── "drwxr-xr-x ..." ──────────────────│
  │                                         │
  │                                         │  stdout EOF → 切换到 stderr
  │◀── "permission denied" ────────────────│  Scanner 读取 stderr
  │                                         │
  │                                         │  stderr EOF → Scanner 循环结束
  │                                         │  cmd.Wait() → 退出码 0
  │                                         │  return 0, nil
  │                                         │  defer conn.Close()
  │                                         │
  │  7. onclose 触发                        │  Close 帧
  │◀────────────────────────────────────────│
```

### 6.2 客户端提前断开

```
浏览器                                    服务器
  │                                         │
  │  ... 连接已建立，命令执行中 ...          │
  │                                         │
  │  用户关闭页面/网络断开                   │
  │  ✕ 连接断开                             │
  │                                         │  conn.WriteMessage 返回错误
  │                                         │  log.Print(err)  ← 仅日志
  │                                         │  继续扫描下一行...
  │                                         │  log.Print(err)  ← 重复
  │                                         │  ...
  │                                         │  Scanner 读完全部输出
  │                                         │  cmd.Wait() 回收子进程
  │                                         │  defer conn.Close() ← 幂等
```

### 6.3 命令非零退出码

```
浏览器                                    服务器
  │                                         │
  │  ... 命令输出已全部推送 ...              │
  │                                         │  Scanner 循环结束
  │                                         │  cmd.Wait() 返回 err
  │                                         │    (退出码非零)
  │                                         │  wsErr() 调用
  │                                         │    WriteControl(CloseInternalServerErr)
  │  onclose 触发                           │
  │◀── Close 帧 (1011, "Internal Server    │
  │     Error") ────────────────────────────│
  │                                         │  return 0, nil
  │                                         │  defer conn.Close()
```

---

## 七、消息格式

### 7.1 客户端 → 服务器

只有**文本消息（TextMessage）**：命令字符串

示例：
```
"ls -la"
```

### 7.2 服务器 → 客户端

**文本消息（TextMessage）**：命令输出的每一行

示例（多条消息）：
```
"total 32"
"drwxr-xr-x  5 user  staff   160 Jun 10 10:00 ."
```

**错误消息（TextMessage）**：
- `"Command not allowed."` — 权限不足或命令不在白名单
- 其他错误文本 — 命令解析失败时的错误信息

**Close 帧**：
- 关闭码 1011（CloseInternalServerErr）
- 消息体为 HTTP 状态文本，如 `"Internal Server Error"`
- 由 `wsErr` 函数发送

---

## 八、安全机制

### 8.1 多层权限校验

| 层级 | 检查点 | 代码位置 |
|------|--------|----------|
| 1 | 全局执行开关 `d.server.EnableExec` | [commands.go#L64](http/commands.go#L64) |
| 2 | 用户执行权限 `d.user.Perm.Execute` | [commands.go#L64](http/commands.go#L64) |
| 3 | 用户允许的命令列表 `d.user.Commands` | [commands.go#L80](http/commands.go#L80) |

### 8.2 工作目录限制

```go
cmd.Dir = d.user.FullPath(r.URL.Path)
```

命令执行的工作目录被限制在用户当前浏览的路径下，结合用户的 Scope 限制。

### 8.3 JWT 认证

WebSocket 握手阶段就完成了用户身份认证。

---

## 九、命令解析流程

### 9.1 ParseCommand —— [parser.go](runner/parser.go#L10-L25)

```go
func ParseCommand(s *settings.Settings, raw string) (command []string, name string, err error) {
    name, args, err := SplitCommandAndArgs(raw)

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

- 无 Shell 配置时：直接调用二进制文件 + 参数
- 有 Shell 配置时：`["bash", "-c", "原始命令"]`，由 Shell 解释执行

### 9.2 跨平台命令拆分 —— [commands.go](runner/commands.go#L34-L58)

- **Windows**：自定义解析器，处理反斜杠路径和引号转义
- **Unix/Linux**：使用 `go-shlex` 库解析

---

## 十、关键文件索引

| 文件 | 作用 |
|------|------|
| [frontend/src/api/commands.ts](frontend/src/api/commands.ts) | 前端 WebSocket API 封装 |
| [frontend/src/components/Shell.vue](frontend/src/components/Shell.vue) | 前端终端 UI 组件 |
| [http/http.go](http/http.go) | 路由注册 |
| [http/commands.go](http/commands.go) | WebSocket 核心处理器 |
| [http/auth.go](http/auth.go) | JWT 认证中间件 |
| [http/data.go](http/data.go) | 请求上下文与处理包装器 |
| [runner/parser.go](runner/parser.go) | 命令解析 |
| [runner/commands.go](runner/commands.go) | 跨平台命令拆分 |
| [runner/runner.go](runner/runner.go) | Hook 命令执行器（用于事件钩子） |

---

## 十一、设计特点与改进空间

### 现有设计特点

1. **简洁**：每个命令一个 WebSocket 连接，简单直接
2. **实时性**：使用 `io.MultiReader` + `bufio.Scanner` 实现逐行推送
3. **安全性**：三层权限校验 + 工作目录限制
4. **资源安全**：即使客户端断开，也保证 `cmd.Wait()` 被调用，避免僵尸进程

### 潜在改进点

1. **stdout/stderr 串行读取**：`io.MultiReader` 导致 stderr 延迟到 stdout EOF 后才发送。可用两个 goroutine 分别读取 stdout/stderr，通过 channel 合并实现真正的交叉推送
2. **写入失败后不中断循环**：输出推送中客户端断开时，Scanner 循环仍继续读取，产生重复错误日志。可在写入失败后 `break` 退出循环，仍保证 `cmd.Wait()` 被调用
3. **心跳机制缺失**：没有 ping/pong 心跳，网络异常时可能无法及时感知断连
4. **命令非零退出码的处理**：`cmd.Wait()` 返回错误时以 1011 关闭码关闭 WebSocket，语义不够精确，可考虑用正常关闭码 + 退出码信息
5. **输入支持**：当前只支持单向输出推送，不支持交互输入（如 `sudo` 密码输入）
6. **ANSI 转义序列处理**：前端只简单过滤颜色码，可考虑完整的终端仿真
