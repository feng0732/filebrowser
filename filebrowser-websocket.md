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
2. **尝试发送 Close 控制帧**：使用 `WriteControl` 发送 `CloseInternalServerErr`（关闭码 1011），消息体为 HTTP 状态文本，写入截止时间 10 秒
3. 如果 Close 帧本身写入失败，仅打印日志，不再做进一步处理

**关键事实**：`wsErr` 内部的 `WriteControl` 也可能失败。失败时只是 `log.Print(err)`，没有任何恢复机制，客户端不会收到这个 Close 帧。

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
"line1"   ← 来自 stdout
"line2"   ← 来自 stdout
"err1"    ← 来自 stderr（被延迟到 stdout EOF 之后）
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

## 四、三类发送操作的失败表现（对准代码事实）

整个 WebSocket 处理流程中存在三种不同的发送操作，它们的失败处理逻辑**完全不同**，客户端能否收到消息的表现也截然不同。

### 4.1 第一类：普通错误消息（TextMessage）

这类发送出现在权限拒绝、命令解析失败、命令不在白名单三种场景中，代码结构完全一致，以权限拒绝为例 [commands.go#L64-L69](http/commands.go#L64-L69)：

```go
if !d.server.EnableExec || !d.user.Perm.Execute {
    if err := conn.WriteMessage(websocket.TextMessage, cmdNotAllowed); err != nil {
        wsErr(conn, r, http.StatusInternalServerError, err)
    }
    return 0, nil
}
```

**必须分两个分支分析，不能把失败场景当成已收到：**

#### 分支 A：`WriteMessage` 成功（`err == nil`）

- `if` 不进入，直接 `return 0, nil`
- 函数返回时 `defer conn.Close()` 执行，发送 Close 帧
- **客户端能收到**：
  1. 一条 TextMessage：内容为 `"Command not allowed."`（或解析错误文本）
  2. 一条 Close 帧（正常关闭）

#### 分支 B：`WriteMessage` 失败（`err != nil`）

- 进入 `if`，调用 `wsErr`
- `wsErr` 内部尝试 `WriteControl(CloseInternalServerErr, ...)`，**这一步同样可能失败**：
  - **子分支 B1：WriteControl 成功** → 客户端收到 Close 帧（1011, "Internal Server Error"），**但收不到那条 TextMessage 错误消息**（因为 WriteMessage 已经失败了）
  - **子分支 B2：WriteControl 也失败** → `wsErr` 仅 `log.Print(err)`，不再重试。客户端此时什么控制消息都收不到，只能等待底层 TCP 连接断开或浏览器的 WebSocket 超时机制
- 最终 `return 0, nil` → `defer conn.Close()` 执行（如果 TCP 连接还可用，会再尝试底层关闭；如果连接已不可用，什么都不会发送）

**重要结论：WriteMessage 失败时，客户端一定收不到那条错误文本消息。之前把此场景描述成"之前已收到"是错误的。**

同样的分析适用于：
- 命令解析失败 [commands.go#L73-L78](http/commands.go#L73-L78)
- 命令不在白名单 [commands.go#L80-L86](http/commands.go#L80-L86)

### 4.2 第二类：Close 帧（通过 wsErr 直接调用）

这类发送出现在：读取命令失败、管道创建失败、命令启动失败、cmd.Wait 返回错误四种场景。它们不尝试发 TextMessage，直接调用 `wsErr`。以 StdoutPipe 失败为例 [commands.go#L91-L95](http/commands.go#L91-L95)：

```go
stdout, err := cmd.StdoutPipe()
if err != nil {
    wsErr(conn, r, http.StatusInternalServerError, err)
    return 0, nil
}
```

同样分两个分支：

#### 分支 A：`wsErr` 内部 `WriteControl` 成功

- 客户端收到 Close 帧（关闭码 1011，消息体 "Internal Server Error"）
- `defer conn.Close()` 随后执行（幂等，不会重复发送）

#### 分支 B：`wsErr` 内部 `WriteControl` 失败

- 仅服务端打一条日志，客户端**收不到任何 Close 帧**
- 后续 `defer conn.Close()` 执行时，如果 TCP 已不可用则什么都不会发生
- 客户端只能靠 WebSocket 协议层的超时或底层 TCP 的断开（FIN/RST）来感知连接出了问题

**同样属于第二类的位置：**
- ReadMessage 失败 [commands.go#L52-L55](http/commands.go#L52-L55)
- StderrPipe 创建失败 [commands.go#L97-L101](http/commands.go#L97-L101)
- cmd.Start 失败 [commands.go#L103-L106](http/commands.go#L103-L106)
- cmd.Wait 返回错误 [commands.go#L115-L117](http/commands.go#L115-L117)（注意：此场景之前命令输出的 TextMessage 可能已部分成功发送）

### 4.3 第三类：命令输出逐行推送（Scanner 循环中）

代码位于 [commands.go#L108-L113](http/commands.go#L108-L113)：

```go
s := bufio.NewScanner(io.MultiReader(stdout, stderr))
for s.Scan() {
    if err := conn.WriteMessage(websocket.TextMessage, s.Bytes()); err != nil {
        log.Print(err)
    }
}
```

这是最特殊的一类，失败处理逻辑**最轻量**：

#### 失败时的表现

- **仅 `log.Print(err)`，不 break 循环，不调用 wsErr，不发送 Close 帧**
- 下一次 `s.Scan()` 继续读取 stdout/stderr，再尝试 `WriteMessage`，大概率再次失败，再次打日志……
- 直到 Scanner 把输出读完（两个管道都 EOF），循环才结束
- 然后才走到 `cmd.Wait()`，如果退出码非零再调用 `wsErr`

#### 客户端视角收到的内容

- 失败**之前**已成功发送的行：**能收到**
- 失败的**那一行**以及之后继续失败的所有行：**全部丢失，收不到**
- 循环期间不会收到 Close 帧
- 循环结束后：
  - 若 cmd.Wait 返回 nil → defer conn.Close() → 客户端可能收到正常 Close 帧（如果连接此时已恢复可用的话）
  - 若 cmd.Wait 返回 err → wsErr 尝试发 1011 Close 帧（这一步同样可能再次失败）

#### 为什么不中断循环

即使客户端断开，子进程仍在运行。Go 的 `exec.Cmd` 要求必须调用 `cmd.Wait()` 来回收子进程资源，而 `cmd.Wait()` 要求 stdout/stderr 管道必须被读取完毕（否则子进程可能因管道缓冲区写满而阻塞挂起）。所以 Scanner 必须读完全部输出，才能安全进入 `cmd.Wait()`，避免产生僵尸进程或挂起子进程。

---

## 五、各阶段退出路径对照表（含成功/失败分支）

以下是 `commandsHandler` 中所有退出位置及其**对代码事实的精确描述**，不再把"尝试发送"误写成"已接收"：

| 代码行 | 触发条件 | 发送操作顺序 | 客户端最终能收到什么 |
|--------|----------|-------------|----------------------|
| L52-55 | ReadMessage 失败 | 直接调用 wsErr → return → defer Close | A: wsErr 的 Close 帧成功 → 收到 1011 Close 帧<br>B: wsErr 的 Close 帧也失败 → 仅靠 TCP 断开感知 |
| L64-69 | 权限不足 | ① 尝试 WriteMessage("Command not allowed") | A1: ①成功 → 收到错误文本 + 正常 Close 帧 |
| | | ② 若①失败则调用 wsErr → return → defer Close | A2: ①失败 + wsErr 的 Close 帧成功 → 收不到错误文本，收到 1011 Close 帧<br>A3: ①失败 + wsErr 的 Close 帧也失败 → 仅靠 TCP 断开感知 |
| L73-78 | 命令解析失败 | ① 尝试 WriteMessage(错误文本) | B1: ①成功 → 收到错误文本 + 正常 Close 帧 |
| | | ② 若①失败则调用 wsErr → return → defer Close | B2: ①失败 + wsErr 的 Close 帧成功 → 收不到错误文本，收到 1011 Close 帧<br>B3: ①失败 + wsErr 的 Close 帧也失败 → 仅靠 TCP 断开感知 |
| L80-86 | 命令不在白名单 | ① 尝试 WriteMessage("Command not allowed") | C1: ①成功 → 收到错误文本 + 正常 Close 帧 |
| | | ② 若①失败则调用 wsErr → return → defer Close | C2: ①失败 + wsErr 的 Close 帧成功 → 收不到错误文本，收到 1011 Close 帧<br>C3: ①失败 + wsErr 的 Close 帧也失败 → 仅靠 TCP 断开感知 |
| L92-94 | StdoutPipe 创建失败 | 直接调用 wsErr → return → defer Close | D1: wsErr 的 Close 帧成功 → 收到 1011 Close 帧<br>D2: wsErr 的 Close 帧也失败 → 仅靠 TCP 断开感知 |
| L98-100 | StderrPipe 创建失败 | 直接调用 wsErr → return → defer Close | E1: wsErr 的 Close 帧成功 → 收到 1011 Close 帧<br>E2: wsErr 的 Close 帧也失败 → 仅靠 TCP 断开感知 |
| L103-105 | cmd.Start 失败 | 直接调用 wsErr → return → defer Close | F1: wsErr 的 Close 帧成功 → 收到 1011 Close 帧<br>F2: wsErr 的 Close 帧也失败 → 仅靠 TCP 断开感知 |
| L108-113 + L115-117 | Scanner 循环结束后 cmd.Wait 返回 err | 循环中每行 WriteMessage 失败仅打日志<br>循环结束后调用 wsErr → return → defer Close | 循环中成功发送的行：能收到；失败的行：全部丢失<br>G1: wsErr 的 Close 帧成功 → 额外收到 1011 Close 帧<br>G2: wsErr 的 Close 帧也失败 → 仅靠 TCP 断开感知 |
| L108-113 + L119 | Scanner 循环结束后 cmd.Wait 返回 nil | 循环中每行 WriteMessage 失败仅打日志<br>return → defer Close | 循环中成功发送的行：能收到；失败的行：全部丢失<br>→ 收到正常 Close 帧（TCP 仍可用时） |

---

## 六、命令结束后的关闭路径详解

### 6.1 正常结束的关闭路径

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
cmd.Wait() 被调用（L115）
    │
    ├── Wait() 返回 nil  ──▶  不调用 wsErr
    │                         直接 return 0, nil
    │                            │
    │                            ▼
    │                      defer conn.Close() 执行
    │                      发送正常 Close 帧
    │
    └── Wait() 返回 err  ──▶  调用 wsErr()
                               尝试发送 Close 帧 (CloseInternalServerErr)
                                  │
                                  ├── 成功 → 客户端收到 1011 Close 帧
                                  │
                                  └── 失败 → 仅打日志，客户端无感知
                                          │
                                          ▼
                                     return 0, nil
                                     defer conn.Close() 执行
                                     （此时连接可能已由 wsErr 关闭，
                                       Close() 是幂等的，不会报错）
```

### 6.2 关闭路径的重要细节

**1. 权限拒绝时"双重消息"只在 WriteMessage 成功时才成立**

当 WriteMessage 成功时，客户端先收到 TextMessage，再收到 defer 产生的 Close 帧，共两条消息。但如果 WriteMessage 本身就失败了，客户端至多只能收到 Close 帧，错误文本是丢失的。

**2. 命令非零退出码 ≠ 连接异常**

当 `cmd.Wait()` 返回错误时（命令以非零退出码结束），代码调用 `wsErr` 发送 `CloseInternalServerErr`（关闭码 1011）。这意味着即使命令只是业务语义上的失败（如 `ls` 找不到文件返回退出码 1），WebSocket 也被标为"服务器内部错误"关闭，语义不精确。

**3. `defer conn.Close()` 是所有路径的最后一道防线**

无论中间是否调用过 `wsErr`，所有路径最终都会执行 `defer conn.Close()`。gorilla/websocket 的 `Close()` 方法是幂等的，多次调用安全。

**4. wsErr 的 WriteControl 失败是"静默失败"**

客户端此时只能依赖 WebSocket 内部的超时检测或 TCP 层的连接断开（FIN/RST）来感知。不会有任何额外的通知。

---

## 七、连接生命周期完整时序

### 7.1 正常命令执行（全部发送成功）

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
  │  6. onmessage 触发 (N 次)               │  Scanner 逐行读取
  │◀── "total 32" ─────────────────────────│  WriteMessage(全部成功)
  │◀── "drwxr-xr-x ..." ──────────────────│
  │◀── ... ────────────────────────────────│
  │                                         │  stdout EOF → stderr
  │◀── "permission denied" ────────────────│  stderr EOF
  │                                         │
  │                                         │  Scanner 循环结束
  │                                         │  cmd.Wait() → 退出码 0
  │                                         │  return 0, nil
  │                                         │  defer conn.Close()
  │                                         │
  │  7. onclose 触发 (正常)                 │  正常 Close 帧
  │◀────────────────────────────────────────│
```

### 7.2 权限不足 + WriteMessage 失败

```
浏览器                                    服务器
  │                                         │
  │  ... 连接已建立，发送命令 "rm -rf /"     │
  │────────────────────────────────────────▶│
  │                                         │  ReadMessage 收到命令
  │                                         │  检查权限不足
  │                                         │
  │                                         │  WriteMessage("Command not allowed.")
  │                                         │    → 返回 err（连接刚断）
  │                                         │
  │  ✕ 用户网络已断开                        │  进入 if: 调用 wsErr
  │                                         │    WriteControl(Close 1011)
  │                                         │      → 再次失败（连接仍断）
  │                                         │      log.Print(err)
  │                                         │  return 0, nil
  │                                         │  defer conn.Close()
  │
  │  客户端视角：                            │
  │  无任何 onmessage 收到                  │
  │  无任何 onclose 收到（短时间内）         │
  │  最终靠浏览器 WS 超时 / TCP RST 感知    │
```

### 7.3 命令输出推送中 WriteMessage 失败（客户端提前断开）

```
浏览器                                    服务器
  │                                         │
  │  ... 输出推送中，已收到前 10 行          │
  │  ✕ 用户关闭页面                          │
  │                                         │
  │                                         │  L110: WriteMessage(第11行) → err
  │                                         │         log.Print(err) ← 仅日志
  │                                         │  s.Scan() → 继续读第12行
  │                                         │  L110: WriteMessage(第12行) → err
  │                                         │         log.Print(err) ← 重复
  │                                         │  ... 循环继续，每行都失败都打日志 ...
  │                                         │
  │                                         │  Scanner 读完全部输出，循环结束
  │                                         │  cmd.Wait() → 回收子进程
  │                                         │  return 0, nil
  │                                         │  defer conn.Close()
  │
  │  客户端视角：                            │
  │  onmessage: 仅收到前 10 行              │
  │  第 11 行及之后：全部丢失                │
  │  onclose: 页面已销毁，回调不会执行       │
```

---

## 八、消息格式

### 8.1 客户端 → 服务器

只有**文本消息（TextMessage）**：命令字符串

示例：
```
"ls -la"
```

### 8.2 服务器 → 客户端

**文本消息（TextMessage）**：命令输出的每一行，或权限/解析错误消息

示例（多条消息）：
```
"total 32"
"drwxr-xr-x  5 user  staff   160 Jun 10 10:00 ."
```

**错误消息（TextMessage）**（仅在 WriteMessage 成功时收到）：
- `"Command not allowed."` — 权限不足或命令不在白名单
- 其他错误文本 — 命令解析失败时的错误信息

**Close 帧**（仅在对应 WriteControl 成功时收到）：
- 正常关闭：由 `defer conn.Close()` 产生
- 错误关闭：关闭码 1011（CloseInternalServerErr），消息体为 HTTP 状态文本如 `"Internal Server Error"`，由 `wsErr` 函数产生

---

## 九、安全机制

### 9.1 多层权限校验

| 层级 | 检查点 | 代码位置 |
|------|--------|----------|
| 1 | 全局执行开关 `d.server.EnableExec` | [commands.go#L64](http/commands.go#L64) |
| 2 | 用户执行权限 `d.user.Perm.Execute` | [commands.go#L64](http/commands.go#L64) |
| 3 | 用户允许的命令列表 `d.user.Commands` | [commands.go#L80](http/commands.go#L80) |

### 9.2 工作目录限制

```go
cmd.Dir = d.user.FullPath(r.URL.Path)
```

命令执行的工作目录被限制在用户当前浏览的路径下，结合用户的 Scope 限制。

### 9.3 JWT 认证

WebSocket 握手阶段就完成了用户身份认证。

---

## 十、命令解析流程

### 10.1 ParseCommand —— [parser.go](runner/parser.go#L10-L25)

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

### 10.2 跨平台命令拆分 —— [commands.go](runner/commands.go#L34-L58)

- **Windows**：自定义解析器，处理反斜杠路径和引号转义
- **Unix/Linux**：使用 `go-shlex` 库解析

---

## 十一、关键文件索引

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

## 十二、设计特点与改进空间

### 现有设计特点

1. **简洁**：每个命令一个 WebSocket 连接，简单直接
2. **实时性**：使用 `io.MultiReader` + `bufio.Scanner` 实现逐行推送
3. **安全性**：三层权限校验 + 工作目录限制
4. **资源安全**：即使客户端断开，也保证 `cmd.Wait()` 被调用，避免僵尸进程

### 潜在改进点

1. **stdout/stderr 串行读取**：`io.MultiReader` 导致 stderr 延迟到 stdout EOF 后才发送。可用两个 goroutine 分别读取 stdout/stderr，通过 channel 合并实现真正的交叉推送
2. **TextMessage 失败时错误消息丢失**：权限/解析场景下 WriteMessage 失败，错误文本就丢了。可考虑把错误信息塞进 Close 帧的 reason 字段，客户端从 onclose 中读取
3. **wsErr 的 WriteControl 静默失败**：Close 帧发送失败时客户端无感知。可考虑设置 SetPongHandler / 心跳，结合合理的超时时间
4. **输出推送 WriteMessage 失败不中断**：Scanner 循环期间客户端断开会产生大量重复错误日志。可引入失败计数器，连续失败 N 次后主动 break，但仍保证后续走 `cmd.Wait()` 路径
5. **命令非零退出码的关闭语义**：`cmd.Wait()` 返回错误时以 1011 关闭码关闭 WebSocket，语义不够精确。可考虑用自定义关闭码（如 4xxx 范围）携带退出码信息，或通过一条 TextMessage 先传递退出码再正常关闭
6. **心跳机制缺失**：没有 ping/pong 心跳，网络异常时可能无法及时感知断连
7. **输入支持**：当前只支持单向输出推送，不支持交互输入（如 `sudo` 密码输入）
8. **ANSI 转义序列处理**：前端只简单过滤颜色码，可考虑完整的终端仿真
