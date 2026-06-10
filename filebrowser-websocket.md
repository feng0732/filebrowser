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

## 二、两种关闭方式的本质区别（对准 gorilla/websocket 行为）

这是理解连接收尾语义的核心前提。项目使用的库是 [gorilla/websocket v1.5.3](http/go.mod)，它在 RFC 6455 基础上提供的两个关闭 API 语义完全不同：

### 2.1 `Conn.Close()` —— 只关底层 TCP，不发 WebSocket Close 帧

```go
// gorilla/websocket 官方语义：Close() 关闭底层网络连接。
// 它不会发送 WebSocket 协议层的 Close 控制帧。
defer conn.Close()  // commands.go L51
```

**行为**：
- 调用 `net.Conn.Close()`，直接关闭 TCP 连接
- 浏览器 WebSocket API 感知：TCP FIN/RST → 触发 `onclose`
- 客户端 `onclose.code` = **1006 (CloseAbnormalClosure)** 或 **1005 (CloseNoStatusReceived)** —— 因为没收到任何 Close 帧
- 这是**"暴力关闭"**，不参与 WebSocket 协议层的关闭握手

### 2.2 `WriteControl(CloseMessage, ...)` —— 发 Close 帧，不关 TCP，也不等待握手

```go
// wsErr 内部调用，commands.go L33
ws.WriteControl(websocket.CloseInternalServerErr, []byte(txt), deadline)
```

**行为**：
- 向连接写入一个 WebSocket Close 控制帧（Opcode = 0x8），携带关闭码和原因文本
- **不**关闭底层 TCP 连接
- **不**等待对端回 Close 帧（WebSocket 关闭握手需要双方交换 Close 帧才完整）
- 写入成功后，Close 帧在 TCP 缓冲区里，等待发送出去
- 这只是**"半关闭握手"**的第一步

### 2.3 RFC 6455 标准关闭握手 vs 当前代码的实际做法

**标准流程（RFC 6455 §1.4 / §5.5.1）**：

```
服务器                                    浏览器
  │                                         │
  │  WriteControl(CloseNormalClosure, 1000) │  ← ① 服务器发 Close 帧
  │────────────────────────────────────────▶│
  │                                         │
  │  （库默认 CloseHandler 自动回）           │
  │◀────────────────────────────────────────│  ← ② 浏览器回 Close 帧
  │                                         │
  │  等待收到后再调用                        │
  │  conn.Close()                           │  ← ③ 双方都完成后才关 TCP
  │─────── TCP FIN ────────────────────────▶│
```

客户端 `onclose.code = 1000 (CloseNormalClosure)`，`wasClean = true`。

**当前代码的做法（所有路径）**：

```
服务器                                    浏览器
  │                                         │
  │  WriteControl(CloseInternalServerErr,   │  ← ① wsErr 发 Close 帧 (1011)
  │         1011, "Internal Server Error") │
  │────────────────────────────────────────▶│
  │                                         │
  │  ❌ 不等待对端回 Close 帧                │
  │      立即 return → defer conn.Close()    │  ← ② 立即关 TCP
  │─────── TCP FIN ────────────────────────▶│
  │                                         │
  │  （浏览器可能还没来得及回 Close 帧）        │
```

客户端 `onclose.code` 的结果取决于时序：
- 如果 Close 帧在 TCP FIN 前到达：`code = 1011`（收到了关闭码）
- 如果 TCP FIN 先到达：`code = 1006`（Abnormal Closure，协议没有完成握手）

### 2.4 正常结束路径（cmd.Wait 返回 nil）完全跳过了协议层关闭

```go
// commands.go L287-L292
if err := cmd.Wait(); err != nil {
    wsErr(conn, r, http.StatusInternalServerError, err)   // 只有错误才走 wsErr
}

return 0, nil   // 正常退出：❌ 不调用 wsErr
// defer conn.Close() → 直接关 TCP，不发任何 Close 帧
```

**这意味着：命令正常执行成功（退出码 0）时，客户端看到的反而是异常关闭。**

客户端 `onclose` 结果（几乎必然）：
- `code = 1006` (CloseAbnormalClosure) 或 `code = 1005` (CloseNoStatusReceived)
- `reason = ""`
- `wasClean = false`

这不是 bug，这就是 gorilla/websocket `Conn.Close()` 的定义行为。

---

## 三、代码路径详解

### 3.1 前端部分

#### 3.1.1 WebSocket API 封装 —— [commands.ts](frontend/src/api/commands.ts)

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

**关键点**：

- **协议选择**：根据当前页面协议自动选择 `ws:` 或 `wss:`
- **URL 构建**：`${protocol}//${host}${baseURL}/api/command${当前路径}`
- **连接建立后立即发送命令**：`onopen` 触发时通过 `conn.send(command)` 发送要执行的命令
- **一次性使用**：每次调用 `command()` 都会创建新的 WebSocket 连接
- **onclose 不区分关闭码**：Shell.vue 的 `onclose` 回调没有读取 `event.code`，所以即使 `code = 1006` 异常关闭，前端也不会报错，只是清理 ANSI 码后继续允许输入

#### 3.1.2 UI 组件 —— [Shell.vue](frontend/src/components/Shell.vue)

**调用流程**：

1. 用户在终端输入命令后按回车，触发 `submit()` 方法（第144行）
2. 特殊命令处理：
   - `clear`：清空终端内容
   - `exit`：关闭终端窗口
3. 普通命令：调用 `commands()` API（第174-190行）
4. `onmessage` 回调：实时追加命令输出到终端显示
5. `onclose` 回调：清理 ANSI 颜色码，重新允许输入（**不检查 code，任何关闭都视为正常**）

```javascript
commands(
  this.path,           // 当前文件路径，作为命令执行的工作目录
  cmd,                  // 用户输入的命令
  (event) => {         // onmessage: 接收输出
    results.text += `${event.data}\n`;
    this.scroll();
  },
  () => {              // onclose: 连接关闭（任何 code 都走到这里）
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

### 3.2 后端路由注册

#### 3.2.1 HTTP 路由 —— [http.go](http/http.go#L85)

```go
api.PathPrefix("/command").Handler(monkey(commandsHandler, "/api/command")).Methods("GET")
```

- 路由路径：`/api/command`（会被 `stripPrefix` 去掉前缀）
- HTTP 方法：`GET`（WebSocket 握手必须使用 GET）
- 处理函数：`commandsHandler`，被 `monkey()` 包装

#### 3.2.2 Handle 包装器 —— [data.go](http/data.go#L66-L101)

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

### 3.3 后端认证中间件

#### 3.3.1 withUser 中间件 —— [auth.go](http/auth.go#L85-L111)

WebSocket 连接走标准的 JWT 认证流程：

1. 从 `X-Auth` 请求头或 `Cookie("auth")` 提取 token
2. 使用 HS256 算法验证签名
3. 检查 token 是否过期
4. 从 token 中提取 user ID
5. 从数据库加载完整用户信息到 `d.user`
6. 检查是否需要刷新 token

**Token 提取器 extractor —— [auth.go](http/auth.go#L49-L67)**：

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

### 3.4 WebSocket 核心处理逻辑

#### 3.4.1 Upgrader 配置与错误工具 —— [commands.go](http/commands.go#L18-L39)

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

**`wsErr` 函数 —— [commands.go](http/commands.go#L31-L39)**：

```go
func wsErr(ws *websocket.Conn, r *http.Request, status int, err error) {
    txt := http.StatusText(status)
    if err != nil || status >= 400 {
        log.Printf("%s: %v %s %v", r.URL.Path, status, r.RemoteAddr, err)
    }
    // WriteControl: 只发 Close 帧，不关 TCP，不等待对端回 Close 帧
    if err := ws.WriteControl(websocket.CloseInternalServerErr, []byte(txt), time.Now().Add(WSWriteDeadline)); err != nil {
        log.Print(err)  // Close 帧也发不出去？只打日志，不再重试
    }
}
```

**行为总结**：
1. 条件性日志：当 `err != nil` 或 `status >= 400` 时打印请求路径、状态码、客户端地址和错误
2. 调用 `WriteControl` 尝试发送 Close 帧（关闭码 1011，消息体 HTTP 状态文本），设置 10 秒写超时
3. 如果 WriteControl 失败，仅 `log.Print(err)`，**不做任何恢复**，客户端收不到这个 Close 帧
4. **注意：wsErr 不关闭连接**。调用方 return 后由 `defer conn.Close()` 关 TCP

#### 3.4.2 commandsHandler 完整流程 —— [commands.go](http/commands.go#L41-L120)

```go
var commandsHandler = withUser(func(w http.ResponseWriter, r *http.Request, d *data) (int, error) {
    conn, err := upgrader.Upgrade(w, r, nil)
    if err != nil {
        return http.StatusInternalServerError, err
    }
    defer conn.Close()   // ← 所有路径最终都会执行这个：只关 TCP，不发 Close 帧

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
            log.Print(err)   // ← 输出推送失败：只打日志，不中断，不发 Close 帧
        }
    }

    if err := cmd.Wait(); err != nil {
        wsErr(conn, r, http.StatusInternalServerError, err)  // ← 非零退出码：走 wsErr
    }

    return 0, nil   // ← 正常退出（退出码 0）：❌ 不发任何 Close 帧
})
```

---

## 四、stdout 与 stderr 读取顺序精确分析

### 4.1 io.MultiReader 的读取语义

核心代码位于 [commands.go#L108](http/commands.go#L108)：

```go
s := bufio.NewScanner(io.MultiReader(stdout, stderr))
```

**`io.MultiReader(stdout, stderr)` 的工作方式**：

`io.MultiReader` 将多个 `io.Reader` 串联成一个逻辑 Reader。它的读取规则是**严格的串行顺序**：

1. 先从第一个 Reader（`stdout`）读取
2. 只有当第一个 Reader 返回 `io.EOF` 后，才切换到第二个 Reader（`stderr`）
3. 第二个 Reader 也返回 `io.EOF` 后，`MultiReader` 整体返回 `io.EOF`

**这意味着：stdout 的所有输出被读完后，才开始读 stderr。两者不是交叉合并，而是顺序拼接。**

### 4.2 这带来了什么行为

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

### 4.3 为什么 stdout 不会无限阻塞

关键在于 `cmd.StdoutPipe()` 和 `cmd.StderrPipe()` 返回的管道在进程退出时会被关闭：

- 当子进程退出或关闭 stdout 文件描述符时，`stdout` Read 会返回 `io.EOF`
- 此时 `MultiReader` 切换到 `stderr`
- 子进程退出后 `stderr` 也会关闭，返回 `io.EOF`
- `MultiReader` 整体返回 `io.EOF`，Scanner 循环结束

**但如果子进程持续向 stdout 写入而不关闭，stderr 中的内容将一直积压，直到 stdout 管道关闭为止。**

### 4.4 对实际使用的影响

大多数命令行工具的行为模式是：
- 正常输出 → stdout
- 错误/警告 → stderr
- 命令结束后 stdout 和 stderr 同时关闭

对于这类命令，`MultiReader` 的串行读取不会造成问题，因为命令结束后 stdout 先 EOF，然后 stderr 也很快 EOF。

但对于**长时间运行的命令**（如 `tail -f`、`ping`），如果有 stderr 输出，该输出会被延迟到 stdout EOF 之后才发送，可能造成错误信息严重滞后。

---

## 五、三类发送操作的失败表现（对准代码事实）

整个 WebSocket 处理流程中存在三种不同的发送操作，它们的失败处理逻辑**完全不同**，客户端能否收到消息的表现也截然不同。

### 5.1 第一类：普通错误消息（TextMessage）

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
- 函数返回时 `defer conn.Close()` 执行，**只关 TCP，不发 Close 帧**
- **客户端能收到**：
  1. 一条 TextMessage：内容为 `"Command not allowed."`（或解析错误文本）
  2. TCP 断开 → `onclose.code = 1006`（异常关闭，因为没发 Close 帧）

#### 分支 B：`WriteMessage` 失败（`err != nil`）

- 进入 `if`，调用 `wsErr`
- `wsErr` 内部尝试 `WriteControl(CloseInternalServerErr, ...)`，**这一步同样可能失败**：
  - **子分支 B1：WriteControl 成功** → Close 帧发出 → 随后 `defer conn.Close()` 关 TCP。客户端：收不到错误文本（WriteMessage 已失败），`onclose.code = 1011`（Close 帧到达时）或 `1006`（TCP 先关了）
  - **子分支 B2：WriteControl 也失败** → `wsErr` 仅 `log.Print(err)`。客户端：既收不到错误文本，也收不到 Close 帧。只能靠 `defer conn.Close()` 产生的 TCP 断开或浏览器超时来感知，最终 `onclose.code = 1006`
- 最终 `return 0, nil` → `defer conn.Close()` 执行

**重要结论：WriteMessage 失败时，客户端一定收不到那条错误文本消息。**

同样的分析适用于：
- 命令解析失败 [commands.go#L73-L78](http/commands.go#L73-L78)
- 命令不在白名单 [commands.go#L80-L86](http/commands.go#L80-L86)

### 5.2 第二类：Close 帧（通过 wsErr 直接调用）

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

- Close 帧（1011, "Internal Server Error"）写入 TCP 缓冲区
- 随后 `return 0, nil` → `defer conn.Close()` 关 TCP
- 时序决定结果：
  - Close 帧在 TCP FIN 前到达：`onclose.code = 1011`，`reason = "Internal Server Error"`
  - TCP 先关（罕见但可能）：`onclose.code = 1006`

#### 分支 B：`wsErr` 内部 `WriteControl` 失败

- 仅服务端打一条日志，Close 帧**没发出去**
- `defer conn.Close()` 执行 → 直接关 TCP
- 客户端：`onclose.code = 1006`，`reason = ""`，完全不知道具体错误

**同样属于第二类的位置**：
- ReadMessage 失败 [commands.go#L52-L55](http/commands.go#L52-L55)
- StderrPipe 创建失败 [commands.go#L97-L101](http/commands.go#L97-L101)
- cmd.Start 失败 [commands.go#L103-L106](http/commands.go#L103-L106)
- cmd.Wait 返回错误 [commands.go#L115-L117](http/commands.go#L115-L117)（注意：此场景之前命令输出的 TextMessage 可能已部分成功发送）

### 5.3 第三类：命令输出逐行推送（Scanner 循环中）

代码位于 [commands.go#L108-L113](http/commands.go#L108-L113)：

```go
s := bufio.NewScanner(io.MultiReader(stdout, stderr))
for s.Scan() {
    if err := conn.WriteMessage(websocket.TextMessage, s.Bytes()); err != nil {
        log.Print(err)   // 注意：不 break，不发 Close 帧，不做任何恢复
    }
}
```

这是最特殊的一类，失败处理逻辑**最轻量**：

#### 失败时的表现

- **仅 `log.Print(err)`，不 break 循环，不调用 wsErr，不发送 Close 帧**
- 下一次 `s.Scan()` 继续读取 stdout/stderr，再尝试 `WriteMessage`，大概率再次失败，再次打日志……
- 直到 Scanner 把输出读完（两个管道都 EOF），循环才结束
- 然后才走到 `cmd.Wait()`：
  - `cmd.Wait()` 返回 err → 调用 wsErr（见第二类分析）
  - `cmd.Wait()` 返回 nil → 不发任何 Close 帧，`defer conn.Close()` 直接关 TCP

#### 客户端视角收到的内容

- 失败**之前**已成功发送的行：**能收到**
- 失败的**那一行**以及之后继续失败的所有行：**全部丢失，收不到**
- 循环期间不会收到 Close 帧
- 循环结束后：
  - cmd.Wait 返回 nil → TCP 断开 → `onclose.code = 1006`
  - cmd.Wait 返回 err → wsErr 尝试发 1011 Close 帧（见第二类分析）

#### 为什么不中断循环

即使客户端断开，子进程仍在运行。Go 的 `exec.Cmd` 要求必须调用 `cmd.Wait()` 来回收子进程资源，而 `cmd.Wait()` 要求 stdout/stderr 管道必须被读取完毕（否则子进程可能因管道缓冲区写满而阻塞挂起）。所以 Scanner 必须读完全部输出，才能安全进入 `cmd.Wait()`，避免产生僵尸进程或挂起子进程。

---

## 六、所有路径的客户端可见结果对照表（含 onclose.code）

以下是 `commandsHandler` 中所有退出位置**对准 gorilla/websocket 行为**的精确描述：

| 代码行 | 触发条件 | 协议层发送 | 底层操作 | 客户端可见结果 |
|--------|----------|-----------|---------|---------------|
| L52-55 | ReadMessage 失败 | wsErr → WriteControl(1011) | defer Close() 关 TCP | TextMessage: 无<br>Close 帧: A1=成功 → code=1011(到达时)或1006(TCP先关); A2=失败→code=1006 |
| L64-69 | 权限不足, WriteMessage 成功 | ✗ 不调 wsErr | defer Close() 关 TCP | TextMessage: 收到"Command not allowed."<br>onclose.code=1006(无Close帧) |
| L65-68 | 权限不足, WriteMessage 失败 | wsErr → WriteControl(1011) | defer Close() 关 TCP | TextMessage: ❌ 一定收不到<br>Close 帧: B1=成功→code=1011或1006; B2=失败→code=1006 |
| L73-78 | 解析失败, WriteMessage 成功 | ✗ 不调 wsErr | defer Close() 关 TCP | TextMessage: 收到错误文本<br>onclose.code=1006 |
| L74-76 | 解析失败, WriteMessage 失败 | wsErr → WriteControl(1011) | defer Close() 关 TCP | TextMessage: ❌ 一定收不到<br>Close 帧: 成功→code=1011或1006; 失败→code=1006 |
| L80-86 | 白名单拒绝, WriteMessage 成功 | ✗ 不调 wsErr | defer Close() 关 TCP | TextMessage: 收到"Command not allowed."<br>onclose.code=1006 |
| L81-83 | 白名单拒绝, WriteMessage 失败 | wsErr → WriteControl(1011) | defer Close() 关 TCP | TextMessage: ❌ 一定收不到<br>Close 帧: 成功→code=1011或1006; 失败→code=1006 |
| L92-94 | StdoutPipe 创建失败 | wsErr → WriteControl(1011) | defer Close() 关 TCP | TextMessage: 无<br>Close 帧: 成功→code=1011或1006; 失败→code=1006 |
| L98-100 | StderrPipe 创建失败 | wsErr → WriteControl(1011) | defer Close() 关 TCP | TextMessage: 无<br>Close 帧: 成功→code=1011或1006; 失败→code=1006 |
| L103-105 | cmd.Start 失败 | wsErr → WriteControl(1011) | defer Close() 关 TCP | TextMessage: 无<br>Close 帧: 成功→code=1011或1006; 失败→code=1006 |
| L108-113 + L115-117 | 推送后 cmd.Wait 返回 err (非零退出码) | 循环中每行尝试 WriteMessage<br>循环结束后 wsErr→WriteControl(1011) | defer Close() 关 TCP | TextMessage: 成功发送的行收到, 失败的行丢失<br>Close 帧: 成功→code=1011或1006; 失败→code=1006 |
| L108-113 + L119 | 推送后 cmd.Wait 返回 nil (正常退出) | 循环中每行尝试 WriteMessage<br>❌ 之后不发任何 Close 帧 | defer Close() 关 TCP | TextMessage: 成功发送的行收到, 失败的行丢失<br>**onclose.code=1006 (正常退出反而是异常关闭码)** |

**核心规律**：
- 所有走 `defer conn.Close()` 但没有 Close 帧的路径 → `code = 1006`
- 走 `wsErr` 且 WriteControl 成功的路径 → `code = 1011` 或 `1006`（取决于时序）
- 没有任何路径能产生 `code = 1000`（真正的正常关闭）

---

## 七、命令结束后的关闭路径详解

### 7.1 正常结束的关闭路径（cmd.Wait 返回 nil）

```
命令执行完成 (退出码 0)
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
cmd.Wait() → 返回 nil
    │
    ▼
❌ 不调用 wsErr （不发任何 Close 帧）
    │
    ▼
return 0, nil
    │
    ▼
defer conn.Close() 执行
    │
    └──▶  只关 TCP (FIN/RST)
             ❌ 不参与关闭握手
                    │
                    ▼
              浏览器 onclose.code = 1006
              (CloseAbnormalClosure)
```

### 7.2 错误结束的关闭路径（cmd.Wait 返回 err / 管道失败 / 启动失败）

```
错误触发点（任意）
    │
    ▼
调用 wsErr(...)
    │
    ├── WriteControl(Close1011, "Internal Server Error")
    │       │
    │       ├── 成功 → Close 帧进入 TCP 发送缓冲区
    │       └── 失败 → log.Print，不重试
    │
    ▼
return 0, nil
    │
    ▼
defer conn.Close() 执行 → 立即关 TCP
    │
    └──▶  TCP FIN 发出
        ┌───────────────────────────────────────┐
        │  时序竞态：                            │
        │  · Close 帧在 FIN 前到达 → code=1011  │
        │  · FIN 先于 Close 帧到达 → code=1006  │
        └───────────────────────────────────────┘
```

### 7.3 关闭路径的重要细节

**1. 正常退出（code=0）一定产生 1006**

这是最反直觉的一点。代码中只有 `cmd.Wait() != nil` 时才调用 `wsErr`，所以命令正常成功执行完毕时，完全没有 Close 帧发送，直接关 TCP。客户端必然看到 `code = 1006`。

**2. 权限拒绝的 WriteMessage 成功路径：先收文本，再收 1006**

用户在终端看到了 "Command not allowed."，然后 onclose 触发，一切看起来正常。但 WebSocket 协议层实际是异常关闭（1006），只是 Shell.vue 不检查 code，用户感知不到差别。

**3. wsErr 的 1011 有竞态条件**

`WriteControl` 写入成功只代表数据进了内核 TCP 发送缓冲区。`defer conn.Close()` 立即执行会马上调 `net.Conn.Close()`，如果 Close 帧还在缓冲区没发出去，TCP 可能用 RST 而不是 FIN 关闭（取决于 SO_LINGER 设置和数据量），Close 帧就丢了。这种情况下客户端看到的也是 1006。

**4. `Conn.Close()` 的幂等性**

无论是否调用过 wsErr、WriteControl 是否成功，`defer conn.Close()` 都会执行。gorilla/websocket 的 `Close()` 内部只是调底层 `net.Conn.Close()`，多次调用安全（Go 的 `net.TCPConn.Close()` 是幂等的，多调返回相同错误）。

**5. gorilla/websocket 默认 CloseHandler 没机会发挥作用**

gorilla/websocket 文档中说"默认 close handler sends a close message to the peer"，但这只有在应用层调用 `ReadMessage` / `NextReader` 收到了对端的 Close 帧时才会触发。commandsHandler 在读取完命令后就进入 Scanner 循环读管道了，不再读 WebSocket，所以这条自动回 Close 帧的逻辑永远不会被触发。

---

## 八、连接生命周期完整时序

### 8.1 正常命令执行（全部发送成功，退出码 0）

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
  │                                         │  5. ReadMessage 收到命令
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
  │                                         │  ❌ 不发任何 Close 帧
  │                                         │  return 0, nil
  │                                         │  defer conn.Close()
  │                                         │    └── 只关 TCP
  │                                         │
  │  7. onclose 触发                        │
  │     code = 1006 (Abnormal)              │◀── TCP FIN
  │◀────────────────────────────────────────│
```

### 8.2 权限不足 + WriteMessage 失败 + wsErr 的 WriteControl 也失败

```
浏览器                                    服务器
  │                                         │
  │  ... 连接已建立，发送命令 "rm -rf /"     │
  │────────────────────────────────────────▶│
  │                                         │  ReadMessage 收到命令
  │                                         │  检查权限不足
  │                                         │
  │                                         │  WriteMessage("Command not allowed.")
  │                                         │    → err（连接已异常）
  │                                         │
  │  ✕ 网络异常/客户端已断连                  │  进入 if: 调用 wsErr
  │                                         │    WriteControl(Close 1011)
  │                                         │      → err（写入同样失败）
  │                                         │      log.Print(err)
  │                                         │  return 0, nil
  │                                         │  defer conn.Close() → 关 TCP
  │
  │  客户端视角：
  │  onmessage: 无任何消息                  │
  │  onclose.code = 1006                    │◀── TCP FIN / RST
  │  完全不知道失败原因                      │
```

### 8.3 命令非零退出码（wsErr 的 Close 帧成功到达）

```
浏览器                                    服务器
  │                                         │
  │  ... 命令输出已推送完毕                  │
  │                                         │  Scanner 循环结束
  │                                         │  cmd.Wait() → err (退出码 1)
  │                                         │
  │                                         │  wsErr() 调用:
  │                                         │  WriteControl(Close, [1011, "ISE"])
  │  ◀─── Close 帧 (1011) ─────────────────│  ← 这一步先到达
  │                                         │
  │  (浏览器准备回 Close 帧，但还没发)        │  defer conn.Close()
  │                                         │    └── TCP FIN
  │  ◀─── TCP FIN ─────────────────────────│  ← 可能与 Close 帧同包
  │
  │  onmessage: 已收到全部输出行             │
  │  onclose.code = 1011                    │
  │  reason = "Internal Server Error"       │
```

---

## 九、消息格式

### 9.1 客户端 → 服务器

只有**文本消息（TextMessage）**：命令字符串

示例：
```
"ls -la"
```

### 9.2 服务器 → 客户端

**文本消息（TextMessage）**：命令输出的每一行，或权限/解析错误消息

示例（多条消息）：
```
"total 32"
"drwxr-xr-x  5 user  staff   160 Jun 10 10:00 ."
```

**错误消息（TextMessage）**（仅在 WriteMessage 成功时收到）：
- `"Command not allowed."` — 权限不足或命令不在白名单
- 其他错误文本 — 命令解析失败时的错误信息

**Close 帧**（仅在对应 WriteControl 成功且先于 TCP FIN 到达时收到）：
- 关闭码 1011（CloseInternalServerErr），消息体 `"Internal Server Error"` —— 由 `wsErr` 函数产生
- ❌ 代码中不存在关闭码 1000（正常关闭）的发送路径

**底层 TCP 断开**（所有路径都会发生，可能伴随 Close 帧也可能没有）：
- 触发浏览器 `onclose`，`code = 1005 / 1006`（未收到 Close 帧时）

---

## 十、安全机制

### 10.1 多层权限校验

| 层级 | 检查点 | 代码位置 |
|------|--------|----------|
| 1 | 全局执行开关 `d.server.EnableExec` | [commands.go#L64](http/commands.go#L64) |
| 2 | 用户执行权限 `d.user.Perm.Execute` | [commands.go#L64](http/commands.go#L64) |
| 3 | 用户允许的命令列表 `d.user.Commands` | [commands.go#L80](http/commands.go#L80) |

### 10.2 工作目录限制

```go
cmd.Dir = d.user.FullPath(r.URL.Path)
```

命令执行的工作目录被限制在用户当前浏览的路径下，结合用户的 Scope 限制。

### 10.3 JWT 认证

WebSocket 握手阶段就完成了用户身份认证。

---

## 十一、命令解析流程

### 11.1 ParseCommand —— [parser.go](runner/parser.go#L10-L25)

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

### 11.2 跨平台命令拆分 —— [commands.go](runner/commands.go#L34-L58)

- **Windows**：自定义解析器，处理反斜杠路径和引号转义
- **Unix/Linux**：使用 `go-shlex` 库解析

---

## 十二、关键文件索引

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

## 十三、设计特点与改进空间

### 现有设计特点

1. **简洁**：每个命令一个 WebSocket 连接，简单直接
2. **实时性**：使用 `io.MultiReader` + `bufio.Scanner` 实现逐行推送
3. **安全性**：三层权限校验 + 工作目录限制
4. **资源安全**：即使客户端断开，也保证 `cmd.Wait()` 被调用，避免僵尸进程
5. **前端容错**：Shell.vue 的 onclose 不区分关闭码，天然屏蔽了服务端未做关闭握手的问题

### 潜在改进点

1. **完成 RFC 6455 关闭握手**：正常退出时也应发送 CloseNormalClosure(1000)，并简短等待对端回 Close 帧（可设置较短超时如 100ms）后再关 TCP。当前实现导致所有正常退出都走 1006 异常码
2. **wsErr + defer Close 的竞态**：WriteControl 成功后立即关 TCP 可能导致 Close 帧丢失。可在 wsErr 和 defer 之间加一个短暂的 ReadTimeout + 读循环（读不到就放弃），或者用 `SetLinger` 控制 TCP 关闭行为
3. **stdout/stderr 串行读取**：`io.MultiReader` 导致 stderr 延迟到 stdout EOF 后才发送。可用两个 goroutine 分别读取 stdout/stderr，通过 channel 合并实现真正的交叉推送
4. **TextMessage 失败时错误消息丢失**：权限/解析场景下 WriteMessage 失败，错误文本就丢了。可考虑把错误信息塞进 Close 帧的 reason 字段（即使 WriteMessage 失败，Close 帧可能因为更小而发送成功），客户端从 `onclose.reason` 中读取
5. **输出推送 WriteMessage 失败不中断**：Scanner 循环期间客户端断开会产生大量重复错误日志。可引入失败计数器，连续失败 N 次后主动 break（仍保证后续走 `cmd.Wait()` 路径）
6. **命令非零退出码的关闭语义**：`cmd.Wait()` 返回错误时以 1011 关闭码关闭 WebSocket，语义不精确。可考虑用自定义关闭码（4xxx 范围）携带退出码信息，或先通过一条 TextMessage 传递退出码再用 1000 正常关闭
7. **心跳机制缺失**：没有 ping/pong 心跳，网络异常时可能无法及时感知断连
8. **输入支持**：当前只支持单向输出推送，不支持交互输入（如 `sudo` 密码输入）
9. **ANSI 转义序列处理**：前端只简单过滤颜色码，可考虑完整的终端仿真
