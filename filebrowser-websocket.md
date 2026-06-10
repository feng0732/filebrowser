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

## 二、WriteControl 参数语义与代码 Bug 分析（核心）

### 2.1 两个容易混淆的概念：Opcode vs Close Code

WebSocket 协议中存在两个完全不同层面的数字常量，当前代码把它们搞混了：

| 概念 | 含义 | 取值示例 | 属于 |
|------|------|---------|------|
| **Opcode / 消息类型 (Message Type)** | WebSocket 帧的类型，定义帧是做什么用的 | `CloseMessage = 8`, `PingMessage = 9`, `PongMessage = 10`, `TextMessage = 1` | 帧头部（每帧都有） |
| **Close Code / 关闭码** | Close 帧的 payload 中的状态码，定义关闭原因 | `CloseNormalClosure = 1000`, `CloseInternalServerErr = 1011`, `CloseAbnormalClosure = 1006` | Close 帧的 data 部分（只有 Close 帧有） |

### 2.2 gorilla/websocket API 语义

项目使用 [gorilla/websocket v1.5.3](http/go.mod)，其核心 API 为：

```go
// WriteControl 写一个控制帧到连接
// messageType: 必须是 CloseMessage(8), PingMessage(9), 或 PongMessage(10) —— 这是 Opcode
// data: 控制帧的 payload。对于 Close 帧，必须先用 FormatCloseMessage 编码
// deadline: 写入超时
func (c *Conn) WriteControl(messageType int, data []byte, deadline time.Time) error

// FormatCloseMessage 将关闭码和文本编码成 Close 帧的 payload 格式
// 返回: [2字节大端关闭码][UTF-8文本]
func FormatCloseMessage(closeCode int, text string) []byte
```

**正确用法示例**：
```go
// 正确：先编码 payload，再传正确的 Opcode
msg := websocket.FormatCloseMessage(websocket.CloseInternalServerErr, "出错了")
err := conn.WriteControl(websocket.CloseMessage, msg, deadline)
//                    ↑ Opcode=8 (Close)       ↑ payload=[1011,"出错了"]
```

### 2.3 当前代码的 Bug：参数传反了

看 [wsErr 函数 commands.go#L33](http/commands.go#L33)：

```go
func wsErr(ws *websocket.Conn, r *http.Request, status int, err error) {
    txt := http.StatusText(status)
    if err != nil || status >= 400 {
        log.Printf("%s: %v %s %v", r.URL.Path, status, r.RemoteAddr, err)
    }
    // ❌ BUG: 两个参数传反了
    // 第一个参数应该是 Opcode (CloseMessage=8)，却传了关闭码 1011
    // 第二个参数应该是 FormatCloseMessage 编码后的 payload，却传了纯文本
    if err := ws.WriteControl(websocket.CloseInternalServerErr, []byte(txt), time.Now().Add(WSWriteDeadline)); err != nil {
        log.Print(err)
    }
}
```

等价于：
```go
ws.WriteControl(1011, []byte("Internal Server Error"), deadline)
//                ↑ 这是 Close Code，不是 Opcode
//  1011 不是合法的控制帧类型（合法值是 8,9,10）
```

### 2.4 Bug 的影响：wsErr 永远发不出 Close 帧

gorilla/websocket 内部会校验 `WriteControl` 的 `messageType` 参数，必须是 8、9、10 之一。传入 1011 会**立即返回错误**。

所以：
- `wsErr` 函数中 `WriteControl` 调用**永远失败**
- 失败后只是 `log.Print(err)`，没有任何恢复
- 客户端**永远收不到** wsErr 尝试发送的 Close 帧
- 后续 `defer conn.Close()` 只关 TCP，客户端看到的 `onclose.code` 永远是 **1006 (CloseAbnormalClosure)**

之前文档中提到的"code=1011 分支"在实际运行中**不可能发生**，所有走 wsErr 的路径本质上都退化为直接关 TCP。

---

## 三、两种关闭方式的本质区别（对准 gorilla/websocket 行为）

| API | 实际行为 | 客户端 onclose.code |
|-----|----------|---------------------|
| `Conn.Close()` | 只调 `net.Conn.Close()` 关底层 TCP，**不发任何 WebSocket Close 帧** | 1005 / 1006（异常关闭） |
| `WriteControl(CloseMessage, FormatCloseMessage(code, text), deadline)` | 只发 Close 帧到 TCP 缓冲区，**不关 TCP，不等待对端回 Close 帧** | 对应 code（如 1011）——前提是 Close 帧先于 TCP FIN 到达 |
| `WriteControl(CloseMessage, 未编码的文本, deadline)` | payload 格式错误，不是标准的 [code+text] 格式 | 浏览器可能拒绝解析，code 仍可能是 1006 |
| 当前代码 `WriteControl(1011, 纯文本, deadline)` | **参数错误被库拒绝**，返回 error，什么都没发 | 1006 |

### 3.1 RFC 6455 标准关闭握手 vs 当前代码实际做法

**标准流程（RFC 6455 §1.4 / §5.5.1）**：

```
服务器                                    浏览器
  │                                         │
  │  WriteControl(CloseMessage,             │  ← ① 服务器发 Close 帧
  │      FormatCloseMessage(1000, ""))      │     Opcode=8, payload=[1000,""]
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
  │  wsErr 中的 WriteControl(1011, "ISE")   │  ← ① ❌ 参数错误，库直接返回 err
  │  什么帧都没发出去                        │
  │                                         │
  │  return 0, nil → defer conn.Close()     │  ← ② 直接关 TCP
  │─────── TCP FIN ────────────────────────▶│
```

客户端 `onclose.code = 1006 (CloseAbnormalClosure)` —— 所有路径都是这个结果。

### 3.2 正常结束路径完全跳过了协议层关闭

```go
// commands.go L287-L292
if err := cmd.Wait(); err != nil {
    wsErr(conn, r, http.StatusInternalServerError, err)   // 错误走 wsErr（实际也发不出）
}
return 0, nil   // 正常退出：❌ 不调用 wsErr
// defer conn.Close() → 直接关 TCP，不发任何 Close 帧
```

命令正常成功（退出码 0）时，连 wsErr 都不调，直接关 TCP。

---

## 四、代码路径详解

### 4.1 前端部分

#### 4.1.1 WebSocket API 封装 —— [commands.ts](frontend/src/api/commands.ts)

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

#### 4.1.2 UI 组件 —— [Shell.vue](frontend/src/components/Shell.vue)

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

### 4.2 后端路由注册

#### 4.2.1 HTTP 路由 —— [http.go](http/http.go#L85)

```go
api.PathPrefix("/command").Handler(monkey(commandsHandler, "/api/command")).Methods("GET")
```

- 路由路径：`/api/command`（会被 `stripPrefix` 去掉前缀）
- HTTP 方法：`GET`（WebSocket 握手必须使用 GET）
- 处理函数：`commandsHandler`，被 `monkey()` 包装

#### 4.2.2 Handle 包装器 —— [data.go](http/data.go#L66-L101)

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

### 4.3 后端认证中间件

#### 4.3.1 withUser 中间件 —— [auth.go](http/auth.go#L85-L111)

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

### 4.4 WebSocket 核心处理逻辑

#### 4.4.1 Upgrader 配置与错误工具 —— [commands.go](http/commands.go#L18-L39)

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
    // ❌ BUG: Opcode 和 payload 传反了
    // 应该是: WriteControl(CloseMessage, FormatCloseMessage(CloseInternalServerErr, txt), deadline)
    if err := ws.WriteControl(websocket.CloseInternalServerErr, []byte(txt), time.Now().Add(WSWriteDeadline)); err != nil {
        log.Print(err)  // 这里永远会打印错误日志
    }
}
```

**行为总结（修正 Bug 后）**：
1. 条件性日志：当 `err != nil` 或 `status >= 400` 时打印请求路径、状态码、客户端地址和错误
2. **BUG**：`WriteControl` 参数错误，永远返回 err，永远发不出 Close 帧
3. **注意：wsErr 不关闭连接**。调用方 return 后由 `defer conn.Close()` 关 TCP

#### 4.4.2 commandsHandler 完整流程 —— [commands.go](http/commands.go#L41-L120)

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
            wsErr(conn, r, http.StatusInternalServerError, err)  // WriteControl 永远失败
            return 0, nil
        }
        raw = strings.TrimSpace(string(msg))
        if raw != "" {
            break
        }
    }

    if !d.server.EnableExec || !d.user.Perm.Execute {
        if err := conn.WriteMessage(websocket.TextMessage, cmdNotAllowed); err != nil {
            wsErr(conn, r, http.StatusInternalServerError, err)  // WriteControl 永远失败
        }
        return 0, nil
    }

    command, name, err := runner.ParseCommand(d.settings, raw)
    if err != nil {
        if err := conn.WriteMessage(websocket.TextMessage, []byte(err.Error())); err != nil {
            wsErr(conn, r, http.StatusInternalServerError, err)  // WriteControl 永远失败
        }
        return 0, nil
    }

    if !slices.Contains(d.user.Commands, name) {
        if err := conn.WriteMessage(websocket.TextMessage, cmdNotAllowed); err != nil {
            wsErr(conn, r, http.StatusInternalServerError, err)  // WriteControl 永远失败
        }
        return 0, nil
    }

    cmd := exec.Command(command[0], command[1:]...)
    cmd.Dir = d.user.FullPath(r.URL.Path)

    stdout, err := cmd.StdoutPipe()
    if err != nil {
        wsErr(conn, r, http.StatusInternalServerError, err)  // WriteControl 永远失败
        return 0, nil
    }

    stderr, err := cmd.StderrPipe()
    if err != nil {
        wsErr(conn, r, http.StatusInternalServerError, err)  // WriteControl 永远失败
        return 0, nil
    }

    if err := cmd.Start(); err != nil {
        wsErr(conn, r, http.StatusInternalServerError, err)  // WriteControl 永远失败
        return 0, nil
    }

    s := bufio.NewScanner(io.MultiReader(stdout, stderr))
    for s.Scan() {
        if err := conn.WriteMessage(websocket.TextMessage, s.Bytes()); err != nil {
            log.Print(err)   // ← 输出推送失败：只打日志，不中断，不发 Close 帧
        }
    }

    if err := cmd.Wait(); err != nil {
        wsErr(conn, r, http.StatusInternalServerError, err)  // WriteControl 永远失败
    }

    return 0, nil   // ← 正常退出（退出码 0）：❌ 不调 wsErr，直接关 TCP
})
```

---

## 五、stdout 与 stderr 读取顺序精确分析

### 5.1 io.MultiReader 的读取语义

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

### 5.2 这带来了什么行为

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

### 5.3 为什么 stdout 不会无限阻塞

关键在于 `cmd.StdoutPipe()` 和 `cmd.StderrPipe()` 返回的管道在进程退出时会被关闭：

- 当子进程退出或关闭 stdout 文件描述符时，`stdout` Read 会返回 `io.EOF`
- 此时 `MultiReader` 切换到 `stderr`
- 子进程退出后 `stderr` 也会关闭，返回 `io.EOF`
- `MultiReader` 整体返回 `io.EOF`，Scanner 循环结束

**但如果子进程持续向 stdout 写入而不关闭，stderr 中的内容将一直积压，直到 stdout 管道关闭为止。**

### 5.4 对实际使用的影响

大多数命令行工具的行为模式是：
- 正常输出 → stdout
- 错误/警告 → stderr
- 命令结束后 stdout 和 stderr 同时关闭

对于这类命令，`MultiReader` 的串行读取不会造成问题，因为命令结束后 stdout 先 EOF，然后 stderr 也很快 EOF。

但对于**长时间运行的命令**（如 `tail -f`、`ping`），如果有 stderr 输出，该输出会被延迟到 stdout EOF 之后才发送，可能造成错误信息严重滞后。

---

## 六、三类发送操作的失败表现（对准代码事实 + Bug 修正）

整个 WebSocket 处理流程中存在三种不同的发送操作，它们的失败处理逻辑**完全不同**，再加上 wsErr 的 Bug，客户端能否收到消息的实际表现如下：

### 6.1 第一类：普通错误消息（TextMessage）

这类发送出现在权限拒绝、命令解析失败、命令不在白名单三种场景中，代码结构完全一致，以权限拒绝为例 [commands.go#L64-L69](http/commands.go#L64-L69)：

```go
if !d.server.EnableExec || !d.user.Perm.Execute {
    if err := conn.WriteMessage(websocket.TextMessage, cmdNotAllowed); err != nil {
        wsErr(conn, r, http.StatusInternalServerError, err)  // ❌ wsErr 永远发不出 Close 帧
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
- `wsErr` 内部 `WriteControl` 参数错误，**永远返回 err**，Close 帧**永远发不出去**
- 仅服务端打两条日志（一条是应用日志，一条是 WriteControl 失败日志）
- 最终 `return 0, nil` → `defer conn.Close()` 关 TCP
- **客户端视角**：收不到错误文本（WriteMessage 失败了），也收不到任何 Close 帧（wsErr 发不出），最终 `onclose.code = 1006`

**重要结论：WriteMessage 失败时，客户端一定收不到那条错误文本消息。wsErr 也救不了，因为它自己也有 Bug。**

同样的分析适用于：
- 命令解析失败 [commands.go#L73-L78](http/commands.go#L73-L78)
- 命令不在白名单 [commands.go#L80-L86](http/commands.go#L80-L86)

### 6.2 第二类：Close 帧（通过 wsErr 直接调用）—— ❌ 实际永远发不出

这类发送出现在：读取命令失败、管道创建失败、命令启动失败、cmd.Wait 返回错误四种场景。它们不尝试发 TextMessage，直接调用 `wsErr`。以 StdoutPipe 失败为例 [commands.go#L91-L95](http/commands.go#L91-L95)：

```go
stdout, err := cmd.StdoutPipe()
if err != nil {
    wsErr(conn, r, http.StatusInternalServerError, err)  // ❌ WriteControl 永远失败
    return 0, nil
}
```

**由于 wsErr 的 Bug，实际只有一种结果：**

- `wsErr` 内部 `WriteControl` 参数错误，永远返回 err
- 打一条应用日志 + 一条 WriteControl 错误日志
- `defer conn.Close()` 执行 → 直接关 TCP
- **客户端**：`onclose.code = 1006`，`reason = ""`，完全不知道具体错误

**注意：之前文档提到的 code=1011 分支在实际运行中不可能发生。**

同样属于第二类的位置：
- ReadMessage 失败 [commands.go#L52-L55](http/commands.go#L52-L55)
- StderrPipe 创建失败 [commands.go#L97-L101](http/commands.go#L97-L101)
- cmd.Start 失败 [commands.go#L103-L106](http/commands.go#L103-L106)
- cmd.Wait 返回错误 [commands.go#L115-L117](http/commands.go#L115-L117)（注意：此场景之前命令输出的 TextMessage 可能已部分成功发送）

### 6.3 第三类：命令输出逐行推送（Scanner 循环中）

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
  - `cmd.Wait()` 返回 err → 调用 wsErr（同样发不出 Close 帧，直接关 TCP）
  - `cmd.Wait()` 返回 nil → 不发任何 Close 帧，`defer conn.Close()` 直接关 TCP

#### 客户端视角收到的内容

- 失败**之前**已成功发送的行：**能收到**
- 失败的**那一行**以及之后继续失败的所有行：**全部丢失，收不到**
- 循环期间不会收到 Close 帧
- 循环结束后：
  - cmd.Wait 返回 nil → TCP 断开 → `onclose.code = 1006`
  - cmd.Wait 返回 err → wsErr 尝试发 Close 帧（实际失败）→ 关 TCP → `onclose.code = 1006`

#### 为什么不中断循环

即使客户端断开，子进程仍在运行。Go 的 `exec.Cmd` 要求必须调用 `cmd.Wait()` 来回收子进程资源，而 `cmd.Wait()` 要求 stdout/stderr 管道必须被读取完毕（否则子进程可能因管道缓冲区写满而阻塞挂起）。所以 Scanner 必须读完全部输出，才能安全进入 `cmd.Wait()`，避免产生僵尸进程或挂起子进程。

---

## 七、所有路径的客户端可见结果对照表（修正 Bug 后）

以下是 `commandsHandler` 中所有退出位置**对准 gorilla/websocket 行为 + wsErr Bug** 的精确描述：

| 代码行 | 触发条件 | 协议层发送 | 底层操作 | 客户端可见结果（修正 Bug 后） |
|--------|----------|-----------|---------|------------------------------|
| L52-55 | ReadMessage 失败 | wsErr → WriteControl(1011, text) ❌ 参数错误被库拒 | defer Close() 关 TCP | TextMessage: 无<br>onclose.code = **1006**（永远，因为 wsErr 发不出） |
| L64-69 | 权限不足, WriteMessage 成功 | ✗ 不调 wsErr | defer Close() 关 TCP | TextMessage: 收到"Command not allowed."<br>onclose.code = **1006** |
| L65-68 | 权限不足, WriteMessage 失败 | wsErr → WriteControl(1011, text) ❌ 参数错误被库拒 | defer Close() 关 TCP | TextMessage: ❌ 收不到<br>onclose.code = **1006** |
| L73-78 | 解析失败, WriteMessage 成功 | ✗ 不调 wsErr | defer Close() 关 TCP | TextMessage: 收到错误文本<br>onclose.code = **1006** |
| L74-76 | 解析失败, WriteMessage 失败 | wsErr → WriteControl(1011, text) ❌ 参数错误被库拒 | defer Close() 关 TCP | TextMessage: ❌ 收不到<br>onclose.code = **1006** |
| L80-86 | 白名单拒绝, WriteMessage 成功 | ✗ 不调 wsErr | defer Close() 关 TCP | TextMessage: 收到"Command not allowed."<br>onclose.code = **1006** |
| L81-83 | 白名单拒绝, WriteMessage 失败 | wsErr → WriteControl(1011, text) ❌ 参数错误被库拒 | defer Close() 关 TCP | TextMessage: ❌ 收不到<br>onclose.code = **1006** |
| L92-94 | StdoutPipe 创建失败 | wsErr → WriteControl(1011, text) ❌ 参数错误被库拒 | defer Close() 关 TCP | TextMessage: 无<br>onclose.code = **1006** |
| L98-100 | StderrPipe 创建失败 | wsErr → WriteControl(1011, text) ❌ 参数错误被库拒 | defer Close() 关 TCP | TextMessage: 无<br>onclose.code = **1006** |
| L103-105 | cmd.Start 失败 | wsErr → WriteControl(1011, text) ❌ 参数错误被库拒 | defer Close() 关 TCP | TextMessage: 无<br>onclose.code = **1006** |
| L108-113 + L115-117 | 推送后 cmd.Wait 返回 err | 循环中每行尝试 WriteMessage<br>循环结束后 wsErr → WriteControl(1011, text) ❌ 参数错误被库拒 | defer Close() 关 TCP | TextMessage: 成功发送的行收到, 失败的行丢失<br>onclose.code = **1006** |
| L108-113 + L119 | 推送后 cmd.Wait 返回 nil | 循环中每行尝试 WriteMessage<br>❌ 之后不发任何 Close 帧 | defer Close() 关 TCP | TextMessage: 成功发送的行收到, 失败的行丢失<br>onclose.code = **1006** |

**核心规律（修正 Bug 后）**：
- **所有路径**的 `onclose.code` 都是 **1006**（CloseAbnormalClosure）
- 没有任何路径能产生 1011（wsErr 参数错），也没有任何路径能产生 1000（正常退出不发 Close 帧）
- 客户端能收到的 TextMessage 完全取决于各路径中 `WriteMessage` 的成功与否
- 前端 Shell.vue 不检查 code，所以 1006 对用户透明，只有通过浏览器开发者工具或 onclose.code 才能看到

---

## 八、命令结束后的关闭路径详解

### 8.1 正常结束的关闭路径（cmd.Wait 返回 nil）

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

### 8.2 错误结束的关闭路径（cmd.Wait 返回 err / 管道失败 / 启动失败）

```
错误触发点（任意）
    │
    ▼
调用 wsErr(...)
    │
    ├── WriteControl(1011, "Internal Server Error")
    │       │
    │       └── ❌ 参数错误，gorilla/websocket 返回 err
    │           打一条错误日志
    │
    ▼
return 0, nil
    │
    ▼
defer conn.Close() 执行 → 直接关 TCP
    │
    ▼
浏览器 onclose.code = 1006
(CloseAbnormalClosure)
```

### 8.3 关闭路径的重要细节

**1. 正常退出（code=0）一定产生 1006**

这是最反直觉的一点。代码中只有 `cmd.Wait() != nil` 时才调用 `wsErr`，但 wsErr 本身有 Bug 发不出 Close 帧。正常路径连 wsErr 都不调，直接关 TCP。客户端必然看到 `code = 1006`。

**2. 权限拒绝的 WriteMessage 成功路径：先收文本，再收 1006**

用户在终端看到了 "Command not allowed."，然后 onclose 触发，一切看起来正常。但 WebSocket 协议层实际是异常关闭（1006），只是 Shell.vue 不检查 code，用户感知不到差别。

**3. `Conn.Close()` 的幂等性**

无论是否调用过 wsErr、WriteControl 是否成功，`defer conn.Close()` 都会执行。gorilla/websocket 的 `Close()` 内部只是调底层 `net.Conn.Close()`，多次调用安全（Go 的 `net.TCPConn.Close()` 是幂等的，多调返回相同错误）。

**4. gorilla/websocket 默认 CloseHandler 没机会发挥作用**

gorilla/websocket 文档中说"默认 close handler sends a close message to the peer"，但这只有在应用层调用 `ReadMessage` / `NextReader` 收到了对端的 Close 帧时才会触发。commandsHandler 在读取完命令后就进入 Scanner 循环读管道了，不再读 WebSocket，所以这条自动回 Close 帧的逻辑永远不会被触发。

**5. wsErr 的 Bug 导致错误信息完全丢失**

当 WriteMessage 失败时，原代码意图是发一个 1011 Close 帧告诉客户端出错了。但由于参数传反了，客户端永远收不到这个通知，只能看到连接莫名断开（code=1006），完全不知道是什么错误。

---

## 九、连接生命周期完整时序

### 9.1 正常命令执行（全部发送成功，退出码 0）

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

### 9.2 权限不足 + WriteMessage 失败

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
  │                                         │    WriteControl(1011, "ISE")
  │                                         │      → ❌ 参数错误，库返回 err
  │                                         │      log.Print(err)
  │                                         │  return 0, nil
  │                                         │  defer conn.Close() → 关 TCP
  │
  │  客户端视角：
  │  onmessage: 无任何消息                  │
  │  onclose.code = 1006                    │◀── TCP FIN / RST
  │  完全不知道失败原因                      │
```

### 9.3 命令非零退出码

```
浏览器                                    服务器
  │                                         │
  │  ... 命令输出已推送完毕（如全部成功）     │
  │                                         │  Scanner 循环结束
  │                                         │  cmd.Wait() → err (退出码 1)
  │                                         │
  │                                         │  wsErr() 调用:
  │                                         │  WriteControl(1011, "ISE")
  │                                         │    → ❌ 参数错误，库返回 err
  │                                         │    log.Print(err)
  │                                         │
  │                                         │  return 0, nil
  │                                         │  defer conn.Close() → 关 TCP
  │
  │  onmessage: 已收到全部输出行             │
  │  onclose.code = 1006                    │◀── TCP FIN
  │  不知道命令非零退出                      │
```

---

## 十、消息格式

### 10.1 客户端 → 服务器

只有**文本消息（TextMessage）**：命令字符串

示例：
```
"ls -la"
```

### 10.2 服务器 → 客户端

**文本消息（TextMessage）**：命令输出的每一行，或权限/解析错误消息（仅在 WriteMessage 成功时收到）

示例（多条消息）：
```
"total 32"
"drwxr-xr-x  5 user  staff   160 Jun 10 10:00 ."
```

**错误消息（TextMessage）**（仅在 WriteMessage 成功时收到）：
- `"Command not allowed."` — 权限不足或命令不在白名单
- 其他错误文本 — 命令解析失败时的错误信息

**Close 帧**（❌ 代码 Bug 导致实际永远发不出）：
- 代码意图：关闭码 1011（CloseInternalServerErr），消息体 `"Internal Server Error"`
- 实际：参数传反了，WriteControl 永远返回 err，没有 Close 帧发出

**底层 TCP 断开**（所有路径都会发生）：
- 触发浏览器 `onclose`，**所有路径 `code = 1006`**
- `reason = ""`，`wasClean = false`

---

## 十一、安全机制

### 11.1 多层权限校验

| 层级 | 检查点 | 代码位置 |
|------|--------|----------|
| 1 | 全局执行开关 `d.server.EnableExec` | [commands.go#L64](http/commands.go#L64) |
| 2 | 用户执行权限 `d.user.Perm.Execute` | [commands.go#L64](http/commands.go#L64) |
| 3 | 用户允许的命令列表 `d.user.Commands` | [commands.go#L80](http/commands.go#L80) |

### 11.2 工作目录限制

```go
cmd.Dir = d.user.FullPath(r.URL.Path)
```

命令执行的工作目录被限制在用户当前浏览的路径下，结合用户的 Scope 限制。

### 11.3 JWT 认证

WebSocket 握手阶段就完成了用户身份认证。

---

## 十二、命令解析流程

### 12.1 ParseCommand —— [parser.go](runner/parser.go#L10-L25)

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

### 12.2 跨平台命令拆分 —— [commands.go](runner/commands.go#L34-L58)

- **Windows**：自定义解析器，处理反斜杠路径和引号转义
- **Unix/Linux**：使用 `go-shlex` 库解析

---

## 十三、关键文件索引

| 文件 | 作用 |
|------|------|
| [frontend/src/api/commands.ts](frontend/src/api/commands.ts) | 前端 WebSocket API 封装 |
| [frontend/src/components/Shell.vue](frontend/src/components/Shell.vue) | 前端终端 UI 组件 |
| [http/http.go](http/http.go) | 路由注册 |
| [http/commands.go](http/commands.go) | WebSocket 核心处理器（含 wsErr Bug） |
| [http/auth.go](http/auth.go) | JWT 认证中间件 |
| [http/data.go](http/data.go) | 请求上下文与处理包装器 |
| [runner/parser.go](runner/parser.go) | 命令解析 |
| [runner/commands.go](runner/commands.go) | 跨平台命令拆分 |
| [runner/runner.go](runner/runner.go) | Hook 命令执行器（用于事件钩子） |

---

## 十四、设计特点、Bug 与改进空间

### 现有设计特点

1. **简洁**：每个命令一个 WebSocket 连接，简单直接
2. **实时性**：使用 `io.MultiReader` + `bufio.Scanner` 实现逐行推送
3. **安全性**：三层权限校验 + 工作目录限制
4. **资源安全**：即使客户端断开，也保证 `cmd.Wait()` 被调用，避免僵尸进程
5. **前端容错**：Shell.vue 的 onclose 不区分关闭码，天然屏蔽了服务端关闭握手不完整和 wsErr Bug 的问题

### 已确认的 Bug

**Bug 1：wsErr 中 WriteControl 参数传反 —— [commands.go#L33](http/commands.go#L33)**

```go
// ❌ 当前（Bug）：
ws.WriteControl(websocket.CloseInternalServerErr, []byte(txt), ...)
//                ↑ 这是 Close Code (1011)，应该是 Opcode

// ✅ 正确写法：
msg := websocket.FormatCloseMessage(websocket.CloseInternalServerErr, txt)
ws.WriteControl(websocket.CloseMessage, msg, ...)
```

**影响**：wsErr 永远发不出 Close 帧，所有错误场景客户端看到的都是 1006 异常关闭，错误信息完全丢失。

### 潜在改进点

1. **修复 wsErr Bug**：修正 WriteControl 参数，使用 FormatCloseMessage 编码 payload
2. **完成 RFC 6455 关闭握手**：正常退出时也应发送 CloseNormalClosure(1000)，并简短等待对端回 Close 帧后再关 TCP。当前所有路径都是 1006 异常码
3. **wsErr + defer Close 的竞态**：修复 Bug 后，WriteControl 成功但立即关 TCP 仍可能导致 Close 帧丢失。可在两者之间加短暂超时等待或用 SetLinger 控制 TCP 关闭行为
4. **stdout/stderr 串行读取**：`io.MultiReader` 导致 stderr 延迟到 stdout EOF 后才发送。可用两个 goroutine 分别读取 stdout/stderr，通过 channel 合并实现真正的交叉推送
5. **TextMessage 失败时错误消息丢失**：权限/解析场景下 WriteMessage 失败，错误文本就丢了。可考虑把错误信息塞进 Close 帧的 reason 字段，客户端从 `onclose.reason` 中读取
6. **输出推送 WriteMessage 失败不中断**：Scanner 循环期间客户端断开会产生大量重复错误日志。可引入失败计数器，连续失败 N 次后主动 break（仍保证后续走 `cmd.Wait()` 路径）
7. **命令非零退出码的关闭语义**：`cmd.Wait()` 返回错误时用 1011 关闭码语义不精确。可考虑用自定义关闭码（4xxx 范围）携带退出码信息
8. **心跳机制缺失**：没有 ping/pong 心跳，网络异常时可能无法及时感知断连
9. **输入支持**：当前只支持单向输出推送，不支持交互输入（如 `sudo` 密码输入）
10. **ANSI 转义序列处理**：前端只简单过滤颜色码，可考虑完整的终端仿真
