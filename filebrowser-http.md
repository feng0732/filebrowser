# File Browser HTTP 路由与中间件分析

## 一、整体架构概览

File Browser 的 HTTP 层使用 **gorilla/mux** 作为路由框架，采用 **函数装饰器模式** 构建中间件链，通过统一的 `handleFunc` 签名串联上下文装配、权限校验和错误处理。

核心入口：[http.go](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/http.go) 中的 `NewHandler` 函数

---

## 二、请求进入流程

### 2.1 总览：请求处理链路

```
HTTP 请求
  → stripPrefix(BaseURL)          [前缀剥离]
  → CSP 中间件                    [安全头注入]
  → mux.Router 路由匹配
    ├─ /health                    [健康检查，无包装]
    ├─ /static/*                  [静态资源，经 handle 包装]
    ├─ /api/*                     [API 接口，经 monkey → handle 包装]
    │   ├─ /login, /signup        [免认证]
    │   ├─ /users/*               [withAdmin 装饰]
    │   ├─ /resources/*           [withUser 装饰]
    │   ├─ /share/*               [withPermShare → withUser 装饰]
    │   └─ /public/*              [withHashFile 装饰，分享鉴权]
    └─ 其他所有路径 → index       [SPA 前端页面，NotFoundHandler]
```

### 2.2 路由注册入口：NewHandler

[NewHandler](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/http.go#L19-L94) 是整个 HTTP 层的构造函数，负责：

1. **清理配置**：`server.Clean()`
2. **创建路由器**：`mux.NewRouter()`
3. **注册 CSP 中间件**：全局 Content-Security-Policy 头
4. **获取静态处理器**：`getStaticHandlers()` 返回 index 和 static 两个 handler
5. **定义 monkey 包装函数**：将 `handleFunc` 包装成标准 `http.Handler`
6. **注册路由**：按层级注册健康检查、静态资源、API 子路由
7. **返回包装后的 Handler**：`stripPrefix(server.BaseURL, r)`

```go
// 核心包装函数 monkey
monkey := func(fn handleFunc, prefix string) http.Handler {
    return handle(fn, prefix, store, server)
}
```

### 2.3 前缀剥离：stripPrefix

[stripPrefix](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/utils.go#L55-L80) 是对标准库 `http.StripPrefix` 的定制版本：

- 如果前缀为空或 "/"，直接返回原 handler
- 如果路径恰好等于前缀（无尾斜杠），重定向到带尾斜杠的版本
- 否则剥离前缀后转发请求

该函数作用于**最外层**，在路由匹配之前统一处理 BaseURL。

---

## 三、上下文装配机制

### 3.1 核心数据结构：data

[data](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/data.go#L20-L34) 是贯穿整个请求生命周期的上下文载体：

```go
type data struct {
    *runner.Runner                  // 命令执行器
    settings *settings.Settings     // 全局设置
    server   *settings.Server       // 服务器配置
    store    *storage.Storage       // 存储层
    user     *users.User            // 当前用户（认证后装配）
    raw      interface{}            // 原始数据（分享场景使用）
    checkerPrefix string            // 规则检查前缀（分享场景使用）
}
```

`data` 同时实现了 `rules.Checker` 接口（[Check 方法](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/data.go#L37-L64)），用于路径权限规则校验。

### 3.2 统一包装器：handle 函数

[handle](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/data.go#L66-L101) 是所有业务 handler 的统一入口包装器，完成**第一层上下文装配**：

```go
func handle(fn handleFunc, prefix string, store *storage.Storage, server *settings.Server) http.Handler {
    handler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // 1. 设置全局响应头
        for k, v := range globalHeaders {
            w.Header().Set(k, v)
        }

        // 2. 从存储层读取设置
        settings, err := store.Settings.Get()
        // ...

        // 3. 构造 data 上下文（第一层：无用户信息）
        status, err := fn(w, r, &data{
            Runner:   &runner.Runner{Enabled: server.EnableExec, Settings: settings},
            store:    store,
            settings: settings,
            server:   server,
        })

        // 4. 统一错误响应（见第四章）
        // ...
    })

    // 5. 剥离路由前缀
    return stripPrefix(prefix, handler)
}
```

**关键特征**：
- 每个请求都会重新从数据库读取 `settings`，保证配置实时生效
- 初始 `data` 中 `user` 为 nil，需后续中间件装配
- 返回值是 `(status int, err error)` 的双值约定，status=0 表示成功

### 3.3 认证装饰器：withUser

[withUser](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/auth.go#L85-L111) 是最核心的用户装配中间件，完成**第二层上下文装配**：

```go
func withUser(fn handleFunc) handleFunc {
    return func(w http.ResponseWriter, r *http.Request, d *data) (int, error) {
        // 1. 从请求中提取并解析 JWT Token
        //    - 优先从 X-Auth Header 提取
        //    - GET 请求回退到 auth Cookie
        //    - 使用 HS256 算法验证签名
        p := jwt.NewParser(...)
        token, err := request.ParseFromRequest(r, &extractor{}, keyFunc, ...)
        if (err != nil || !token.Valid) && !renewableErr(err, d) {
            return http.StatusUnauthorized, nil
        }

        // 2. 检测 Token 是否需要续期
        //    - 过期时间 < 1 小时
        //    - 用户信息在 Token 签发后有更新
        if expiresSoon || updated {
            w.Header().Add("X-Renew-Token", "true")
        }

        // 3. 从存储层加载完整用户信息
        d.user, err = d.store.Users.Get(d.server.Root, tk.User.ID)
        // ...

        // 4. 调用下一个 handler
        return fn(w, r, d)
    }
}
```

### 3.4 权限装饰器链

在 `withUser` 基础上，还有更细粒度的权限装饰器：

| 装饰器 | 位置 | 作用 |
|--------|------|------|
| `withAdmin` | [auth.go#L113-L121](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/auth.go#L113-L121) | 校验用户是否为管理员 |
| `withPermShare` | [share.go#L20-L28](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/share.go#L20-L28) | 校验用户是否有分享和下载权限 |
| `withHashFile` | [public.go#L17-L98](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/public.go#L17-L98) | 分享链接鉴权，重绑定用户和文件系统 |

### 3.5 分享场景的特殊装配：withHashFile

[withHashFile](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/public.go#L17-L98) 是公开分享接口的上下文装配器，流程较为特殊：

1. 从 URL 路径提取分享哈希 ID
2. 根据哈希查找分享记录 `share.Link`
3. 校验分享密码（如有）
4. 根据 `link.UserID` 加载所属用户
5. **重绑定文件系统**：使用 `files.NewScopedFs` 将用户文件系统限定在分享路径内
6. **设置 checkerPrefix**：确保规则校验仍基于用户原始路径
7. 预加载共享文件/目录信息到 `d.raw`

这是一个典型的**上下文重绑定**模式，复用了用户体系但限制了访问范围。

---

## 四、错误响应处理链

### 4.1 错误约定：handleFunc 签名

```go
type handleFunc func(w http.ResponseWriter, r *http.Request, d *data) (int, error)
```

**双返回值约定**：
- `int`：HTTP 状态码，`0` 表示正常响应（已写入）
- `error`：错误信息，可为 nil

### 4.2 统一错误出口：handle 函数

在 [handle 函数](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/data.go#L85-L97) 中进行统一错误处理：

```go
// 1. 错误日志记录（状态码 >= 400 或有错误时）
if status >= 400 || err != nil {
    clientIP := realip.FromRequest(r)
    log.Printf("%s: %v %s %v", r.URL.Path, status, clientIP, err)
}

// 2. 输出 HTTP 错误响应
if status != 0 {
    txt := http.StatusText(status)
    if status == http.StatusBadRequest && err != nil {
        txt += " (" + err.Error() + ")"
    }
    http.Error(w, strconv.Itoa(status)+" "+txt, status)
    return
}
```

**关键行为**：
- 仅当状态码为 `http.StatusBadRequest` 时，才会将错误详情附加到响应体中
- 其他错误只返回标准状态文本，不泄露内部错误信息
- 所有 >= 400 的状态码和错误都会记录日志，包含客户端 IP

### 4.3 错误码转换：errToStatus

[errToStatus](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/utils.go#L30-L51) 将领域错误映射为 HTTP 状态码：

| 错误类型 | HTTP 状态码 |
|----------|-------------|
| `nil` | 200 OK |
| `os.IsPermission` / `ErrPermissionDenied` / `ErrRootUserDeletion` | 403 Forbidden |
| `os.IsNotExist` / `ErrNotExist` | 404 Not Found |
| `os.IsExist` / `ErrExist` | 409 Conflict |
| `ErrInvalidRequestParams` | 400 Bad Request |
| `imgErrors.ErrImageTooLarge` | 413 Request Entity Too Large |
| 其他所有错误 | 500 Internal Server Error |

业务 handler 中常见的调用模式：

```go
file, err := files.NewFileInfo(...)
if err != nil {
    return errToStatus(err), err
}
```

### 4.4 业务层错误定义

领域错误集中定义在 [errors/errors.go](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/errors/errors.go)，包括：

- `ErrExist` / `ErrNotExist` - 资源存在性
- `ErrPermissionDenied` - 权限拒绝
- `ErrInvalidRequestParams` - 请求参数无效
- `ErrSourceIsParent` - 源路径是父目录
- `ErrRootUserDeletion` - 不能删除唯一管理员
- `ErrCurrentPasswordIncorrect` - 当前密码错误
- 自定义错误类型 `ErrShortPassword`（带最小长度信息）

---

## 五、中间件设计模式总结

### 5.1 装饰器模式

所有鉴权/权限中间件都采用**函数装饰器**模式：

```go
// 输入：handleFunc → 输出：handleFunc
func withXxx(fn handleFunc) handleFunc {
    return func(w http.ResponseWriter, r *http.Request, d *data) (int, error) {
        // 前置处理...
        result, err := fn(w, r, d)  // 调用下一层
        // 后置处理...
        return result, err
    }
}
```

这种模式的优势：
- **类型安全**：统一的 `handleFunc` 签名
- **可组合**：多个装饰器可链式嵌套（如 `withAdmin` 内部嵌套 `withUser`）
- **上下文传递**：通过 `*data` 指针传递和累积上下文信息

### 5.2 典型调用链示例

以 `sharePostHandler` 为例：

```
sharePostHandler
  → withPermShare
    → withUser (JWT 认证 + 用户装配)
      → handle (全局头 + settings 装配 + 错误统一处理)
        → stripPrefix
          → mux.Router
            → stripPrefix (最外层 BaseURL)
```

以 `publicShareHandler` 为例：

```
publicShareHandler
  → withHashFile (分享鉴权 + 文件系统重绑定)
    → handle
      → stripPrefix
        → mux.Router
          → stripPrefix (最外层 BaseURL)
```

### 5.3 路径前缀的三层结构

整个系统中有**三层**路径前缀处理：

| 层级 | 位置 | 前缀 | 作用 |
|------|------|------|------|
| 第 1 层 | `NewHandler` 返回值 | `server.BaseURL` | 整个应用的基础 URL |
| 第 2 层 | `handle` 返回值 | 各路由前缀（如 `/api/resources`） | 剥离路由前缀，让 handler 看到相对路径 |
| 第 3 层 | `withHashFile` 内部 | `link.Path`（分享根路径） | 将文件系统限定到分享目录 |

---

## 六、关键文件索引

| 文件 | 核心职责 |
|------|----------|
| [http/http.go](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/http.go) | 路由注册、NewHandler 入口 |
| [http/data.go](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/data.go) | data 上下文、handle 包装器、规则检查 |
| [http/auth.go](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/auth.go) | JWT 认证、withUser/withAdmin 装饰器、登录/注册 |
| [http/utils.go](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/utils.go) | errToStatus 错误码转换、stripPrefix、renderJSON |
| [http/public.go](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/public.go) | 公开分享、withHashFile 装饰器 |
| [http/share.go](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/share.go) | 分享管理 API、withPermShare 装饰器 |
| [http/headers.go](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/headers.go) | 全局响应头（Cache-Control） |
| [http/static.go](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/static.go) | 静态资源和 SPA 页面处理 |
| [errors/errors.go](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/errors/errors.go) | 领域错误定义 |
