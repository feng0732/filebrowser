# File Browser 文件上传与断点续传代码流程详解

## 一、整体架构概述

File Browser 使用 **TUS 协议**（基于 HTTP 的文件断点续传开放协议）实现分片上传和断点续传功能。整体分为以下几层：

```
┌─────────────────────────────────────────────────────────┐
│                     前端 (Vue 3 + Pinia)                 │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────────┐ │
│  │ UploadFiles │  │ upload store │  │  tus.ts (tus-js) │ │
│  │    .vue     │  │    (Pinia)   │  │    客户端库      │ │
│  └─────────────┘  └──────────────┘  └──────────────────┘ │
└────────────────────────────────┬────────────────────────┘
                                 │ HTTP (TUS 协议)
                                 ▼
┌─────────────────────────────────────────────────────────┐
│                     后端 (Go + Gorilla Mux)              │
│  ┌───────────────┐  ┌──────────────────┐  ┌───────────┐ │
│  │ tus_handlers  │  │ UploadCache (缓存)│  │  文件系统  │ │
│  │ (4个HTTP处理器)│  │ ┌────────┐┌──────┐│  │  (afero)  │ │
│  │               │  │ │Memory  ││Redis ││  │           │ │
│  └───────────────┘  │ └────────┘└──────┘│  └───────────┘ │
│                     └──────────────────┘                │
└─────────────────────────────────────────────────────────┘
```

### 关键文件索引

| 文件 | 作用 |
|------|------|
| [tus_handlers.go](file:///d:/fz/0601/solo-dogfeeding/code/177-filebrowser/http/tus_handlers.go) | TUS协议后端处理器（POST/HEAD/PATCH/DELETE） |
| [upload_cache_memory.go](file:///d:/fz/0601/solo-dogfeeding/code/177-filebrowser/http/upload_cache_memory.go) | 内存上传缓存（单实例部署） |
| [upload_cache_redis.go](file:///d:/fz/0601/solo-dogfeeding/code/177-filebrowser/http/upload_cache_redis.go) | Redis上传缓存（多实例部署） |
| [tus.go](file:///d:/fz/0601/solo-dogfeeding/code/177-filebrowser/settings/tus.go) | TUS配置（分片大小、重试次数） |
| [http.go](file:///d:/fz/0601/solo-dogfeeding/code/177-filebrowser/http/http.go) | 路由注册 |
| [upload.ts (store)](file:///d:/fz/0601/solo-dogfeeding/code/177-filebrowser/frontend/src/stores/upload.ts) | 前端上传状态管理（Pinia） |
| [tus.ts (api)](file:///d:/fz/0601/solo-dogfeeding/code/177-filebrowser/frontend/src/api/tus.ts) | 前端TUS客户端封装 |
| [upload.ts (utils)](file:///d:/fz/0601/solo-dogfeeding/code/177-filebrowser/frontend/src/utils/upload.ts) | 上传工具函数（冲突检测、文件扫描） |
| [files.ts (api)](file:///d:/fz/0601/solo-dogfeeding/code/177-filebrowser/frontend/src/api/files.ts) | 文件API（路由选择TUS/传统方式） |
| [UploadFiles.vue](file:///d:/fz/0601/solo-dogfeeding/code/177-filebrowser/frontend/src/components/prompts/UploadFiles.vue) | 上传进度UI组件 |
| [Upload.vue](file:///d:/fz/0601/solo-dogfeeding/code/177-filebrowser/frontend/src/components/prompts/Upload.vue) | 上传入口对话框（冲突处理confirm回调） |
| [ResolveConflict.vue](file:///d:/fz/0601/solo-dogfeeding/code/177-filebrowser/frontend/src/components/prompts/ResolveConflict.vue) | 冲突解决对话框（Resume Transfer 按钮逻辑） |

---

## 二、配置参数

### TUS 默认配置
位于 [tus.go](file:///d:/fz/0601/solo-dogfeeding/code/177-filebrowser/settings/tus.go#L1-L10)：

```go
const DefaultTusChunkSize = 10 * 1024 * 1024  // 分片大小: 10MB
const DefaultTusRetryCount = 5                 // 重试次数: 5次

type Tus struct {
    ChunkSize  uint64 `json:"chunkSize"`   // 每个分片的字节数
    RetryCount uint16 `json:"retryCount"`  // 失败重试次数
}
```

### 缓存 TTL 配置
位于 [upload_cache_memory.go](file:///d:/fz/0601/solo-dogfeeding/code/177-filebrowser/http/upload_cache_memory.go#L12)：

```go
const uploadCacheTTL = 3 * time.Minute  // 上传状态缓存 3 分钟
```

> 设计意图：如果上传中断超过 3 分钟，服务端会自动清理未完成的文件，防止磁盘泄漏。

### 前端重试延迟计算
位于 [tus.ts](file:///d:/fz/0601/solo-dogfeeding/code/177-filebrowser/frontend/src/api/tus.ts#L6-L7)：

```typescript
const RETRY_BASE_DELAY = 1000;    // 基础延迟 1s
const RETRY_MAX_DELAY = 20000;    // 最大延迟 20s
```

重试延迟采用**指数退避**算法：`[0, 1000, 2000, 4000, 8000, ...]`，最多5次重试即 `[0, 1000, 2000, 4000, 8000]`。

---

## 三、上传分片流程详解

### 3.1 路由注册

在 [http.go](file:///d:/fz/0601/solo-dogfeeding/code/177-filebrowser/http/http.go#L67-L70) 中注册 4 个 TUS 端点：

```go
api.PathPrefix("/tus").Handler(monkey(tusPostHandler(uploadCache), "/api/tus")).Methods("POST")     // 创建上传
api.PathPrefix("/tus").Handler(monkey(tusHeadHandler(uploadCache), "/api/tus")).Methods("HEAD", "GET")  // 查询进度
api.PathPrefix("/tus").Handler(monkey(tusPatchHandler(uploadCache), "/api/tus")).Methods("PATCH")   // 上传分片
api.PathPrefix("/tus").Handler(monkey(tusDeleteHandler(uploadCache), "/api/tus")).Methods("DELETE") // 取消上传
```

### 3.2 前端上传入口

#### 步骤 1: 文件扫描与处理
用户选择文件后，调用 [upload.ts](file:///d:/fz/0601/solo-dogfeeding/code/177-filebrowser/frontend/src/utils/upload.ts#L185-L217) 中的 `handleFiles()`：

```typescript
export function handleFiles(files: UploadList, base: string, overwrite = false) {
    // 遍历每个文件/目录
    for (const file of files) {
        // 构造上传路径，处理子目录
        let path = base;
        if (file.fullPath !== undefined) {
            path += url.encodePath(file.fullPath);  // 拖拽/文件夹上传包含完整路径
        } else {
            path += url.encodeRFC5987ValueChars(file.name);  // 单文件上传
        }
        // 检测资源类型: video/audio/image/pdf/text/blob
        const type = file.isDir ? "dir" : detectType((file.file as File).type);
        // 加入上传队列
        uploadStore.upload(path, file.name, file.file ?? null, 
                          file.overwrite || overwrite, type);
    }
}
```

#### 步骤 2: 进入 Pinia 上传队列
[upload store](file:///d:/fz/0601/solo-dogfeeding/code/177-filebrowser/frontend/src/stores/upload.ts#L36-L66) 的 `upload()` 方法：

```typescript
const upload = (path, name, file, overwrite, type) => {
    // 首次上传时添加 beforeunload 监听，防止误关页面
    if (!hasActiveUploads() && !hasPendingUploads()) {
        window.addEventListener("beforeunload", beforeUnload);
        buttons.loading("upload");
    }
    // 创建上传对象
    const upload: Upload = {
        path, name, file, overwrite, type,
        totalBytes: file?.size || 1,
        sentBytes: 0,
        rawProgress: markRaw({ sentBytes: 0 }),  // 不触发响应式的原始进度
    };
    totalBytes.value += upload.totalBytes;
    allUploads.value.push(upload);
    processUploads();  // 开始处理
};
```

#### 步骤 3: 并发控制与调度
[upload store](file:///d:/fz/0601/solo-dogfeeding/code/177-filebrowser/frontend/src/stores/upload.ts#L96-L127) 的 `processUploads()`：

```typescript
const UPLOADS_LIMIT = 5;  // 最多并发 5 个上传

const processUploads = async () => {
    // 全部完成则清理状态
    if (!hasActiveUploads() && !hasPendingUploads()) {
        window.removeEventListener("beforeunload", beforeUnload);
        buttons.success("upload");
        reset();
        fileStore.reload = true;  // 刷新文件列表
        return;
    }
    // 并发限制 + 有待处理
    if (isActiveUploadsOnLimit() && hasPendingUploads()) {
        if (!hasActiveUploads()) {
            // 每秒同步一次进度到响应式状态（减少渲染）
            progressInterval = window.setInterval(syncState, 1000);
        }
        const upload = nextUpload();
        if (upload.type === "dir") {
            // 目录直接 POST 创建
            await api.post(upload.path).catch($showError);
        } else {
            // 文件: 注册进度回调，调用 TUS 上传
            const onUpload = (event: ProgressEvent) => {
                upload.rawProgress.sentBytes = event.loaded;
            };
            await api.post(upload.path, upload.file!, upload.overwrite, onUpload)
                .catch(err => err.message !== "Upload aborted" && $showError(err));
        }
        finishUpload(upload);
    }
};
```

> **性能优化点**：进度值先写入 `rawProgress`（非响应式），每秒通过 `syncState()` 批量同步到响应式状态，避免高频触发 Vue 重渲染。

#### 步骤 4: 选择上传方式
在 [files.ts](file:///d:/fz/0601/solo-dogfeeding/code/177-filebrowser/frontend/src/api/files.ts#L106-L124) 的 `post()` 函数中决定使用 TUS 还是传统上传：

```typescript
export async function post(url, content = "", overwrite = false, onupload) {
    const useResourcesApi =
        url.endsWith("/") ||                             // 创建目录 → 传统 API
        (content instanceof Blob &&                      // 非 http(s) 协议（如 file://）
         !["http:", "https:"].includes(window.location.protocol)) ||
        !(await useTus(content));                        // TUS 不可用 → 传统 API
    return useResourcesApi
        ? postResources(url, content, overwrite, onupload)  // 传统 XMLHttpRequest
        : postTus(url, content, overwrite, onupload);       // TUS 分片上传
}
```

### 3.3 TUS 分片上传核心流程

#### 前端 TUS 客户端初始化
[tus.ts](file:///d:/fz/0601/solo-dogfeeding/code/177-filebrowser/frontend/src/api/tus.ts#L10-L83) 的 `upload()`：

```typescript
export async function upload(filePath, content, overwrite, onupload) {
    const resourcePath = `${tusEndpoint}${filePath}?override=${overwrite}`;
    const authStore = useAuthStore();

    return new Promise<void | string>((resolve, reject) => {
        const upload = new tus.Upload(content, {
            endpoint: `${origin}${baseURL}${resourcePath}`,
            chunkSize: tusSettings.chunkSize,           // 10MB 每片
            retryDelays: computeRetryDelays(tusSettings), // 指数退避重试
            parallelUploads: 1,                         // 单连接串行（保证顺序）
            storeFingerprintForResuming: false,         // 不使用本地指纹存储
            headers: { "X-Auth": authStore.jwt },       // JWT 认证
            onShouldRetry: (err) => {
                const status = err.originalResponse?.getStatus() || 0;
                return status !== 409;  // 文件冲突不重试
            },
            onProgress: (bytesUploaded) => {
                onupload?.({ loaded: bytesUploaded });  // 进度回调
            },
            onSuccess: () => {
                delete CURRENT_UPLOAD_LIST[filePath];
                resolve();
            },
            onError: (error) => {
                delete CURRENT_UPLOAD_LIST[filePath];
                if (error.message === "Upload aborted") return reject(error);
                // 提取详细错误信息
                const message = error instanceof tus.DetailedError
                    ? (error.originalResponse === null
                        ? "000 No connection"
                        : error.originalResponse.getBody())
                    : "Upload failed";
                reject(new Error(message));
            },
        });
        CURRENT_UPLOAD_LIST[filePath] = upload;  // 保存引用以便中止
        upload.start();  // 启动 TUS 上传
    });
}
```

#### TUS 协议标准交互流程

```
前端 tus-js-client            后端 tus_handlers
      |                            |
      |-- POST /api/tus/xxx ------>|  tusPostHandler
      |   Upload-Length: 52428800  |  1. 权限检查
      |                            |  2. 创建空文件
      |                            |  3. 缓存注册 (filePath → size, 3min TTL)
      |<-- 201 Created ------------|  4. 返回 Location 头
      |   Location: /api/tus/xxx   |
      |                            |
      |-- HEAD /api/tus/xxx ------>|  tusHeadHandler  (断点续传时先查进度)
      |                            |  1. 查询缓存中的 Upload-Length
      |                            |  2. 读取磁盘上文件实际大小作为 Upload-Offset
      |<-- 200 OK -----------------|
      |   Upload-Offset: 10485760  |
      |   Upload-Length: 52428800  |
      |                            |
      |-- PATCH /api/tus/xxx ----->|  tusPatchHandler
      |   Upload-Offset: 10485760  |  1. 验证 Content-Type
      |   Content-Type: ...        |  2. 校验 offset == 文件实际大小
      |   [10MB 二进制数据]         |  3. 保持缓存存活 (每2s Touch)
      |                            |  4. Seek + Append 写入
      |                            |  5. Sync 刷盘
      |<-- 204 No Content ---------|  6. 返回新的 Upload-Offset
      |   Upload-Offset: 20971520  |
      |                            |
      |         ...重复 PATCH...    |
      |                            |
      |-- PATCH (最后一片) -------->|
      |                            |  newOffset >= uploadLength
      |                            |  → cache.Complete() 删除缓存
      |                            |  → 触发 upload hook
      |<-- 204 No Content ---------|
      |                            |
```

---

## 四、后端四个处理器详解

### 4.1 POST - 创建上传会话
[tusPostHandler](file:///d:/fz/0601/solo-dogfeeding/code/177-filebrowser/http/tus_handlers.go#L41-L123)

**核心逻辑：**
```go
func tusPostHandler(cache UploadCache) handleFunc {
    return withUser(func(w http.ResponseWriter, r *http.Request, d *data) (int, error) {
        // 1. 权限检查: 需要 Create 权限
        if !d.user.Perm.Create || !d.Check(r.URL.Path) {
            return http.StatusForbidden, nil
        }
        // 2. 检查目标路径是否为目录
        if file != nil && file.IsDir {
            return http.StatusBadRequest, ...
        }
        // 3. 文件已存在且未允许覆盖 → 返回 409 Conflict
        if file != nil && r.URL.Query().Get("override") != "true" {
            return http.StatusConflict, nil
        }
        // 4. 打开/创建文件 (允许覆盖时用 O_TRUNC 清空)
        fileFlags := os.O_CREATE | os.O_WRONLY
        if file != nil && override {
            fileFlags |= os.O_TRUNC
        }
        openFile, err := d.user.Fs.OpenFile(r.URL.Path, fileFlags, ...)
        // 5. 解析期望文件大小 (Upload-Length 头)
        uploadLength, err := getUploadLength(r)  // 从 HTTP Header 读取
        // 6. 关键步骤: 在缓存中注册上传 (TTL 3分钟)
        cache.Register(file.RealPath(), uploadLength)
        // 7. 返回 201 Created, Location 头指向上传 URL
        w.Header().Set("Location", basePath+"/api/tus"+r.URL.EscapedPath())
        return http.StatusCreated, nil
    })
}
```

### 4.2 HEAD - 查询上传进度（续传关键）
[tusHeadHandler](file:///d:/fz/0601/solo-dogfeeding/code/177-filebrowser/http/tus_handlers.go#L125-L154)

这是 **断点续传的核心查询接口**：

```go
func tusHeadHandler(cache UploadCache) handleFunc {
    return withUser(func(w http.ResponseWriter, r *http.Request, d *data) (int, error) {
        w.Header().Set("Cache-Control", "no-store")  // 禁止缓存
        // 从文件系统读取文件实际大小 → 作为已上传字节数
        file, err := files.NewFileInfo(...)
        // 从缓存中读取期望的总大小
        uploadLength, err := cache.GetLength(file.RealPath())
        // 返回两个关键头:
        w.Header().Set("Upload-Offset", strconv.FormatInt(file.Size, 10))  // 断点位置
        w.Header().Set("Upload-Length", strconv.FormatInt(uploadLength, 10))  // 总大小
        return http.StatusOK, nil
    })
}
```

> **续传原理**：tus-js-client 在重传时先发送 HEAD 请求，拿到 `Upload-Offset` 后，从文件的对应位置继续切下一片上传，跳过已上传的部分。

### 4.3 PATCH - 上传分片（数据写入）
[tusPatchHandler](file:///d:/fz/0601/solo-dogfeeding/code/177-filebrowser/http/tus_handlers.go#L156-L239)

这是 **分片数据实际写入** 的处理器：

```go
func tusPatchHandler(cache UploadCache) handleFunc {
    return withUser(func(w http.ResponseWriter, r *http.Request, d *data) (int, error) {
        // 1. 校验 Content-Type 必须为 application/offset+octet-stream
        if r.Header.Get("Content-Type") != "application/offset+octet-stream" {
            return http.StatusUnsupportedMediaType, nil
        }
        // 2. 解析客户端声明的偏移量
        uploadOffset, err := getUploadOffset(r)  // 从 Upload-Offset 头读取
        // 3. 获取当前文件信息
        file, err := files.NewFileInfo(...)
        uploadLength, err := cache.GetLength(file.RealPath())
        // 4. 启动保活定时器 (防止传输期间缓存过期)
        stop := keepUploadActive(cache, file.RealPath())
        defer stop()
        // 5. 关键校验: 文件系统大小必须等于客户端声明的 offset
        //    防止并发/乱序写入导致数据损坏
        if file.Size != uploadOffset {
            return http.StatusConflict, fmt.Errorf(
                "%s file size doesn't match the provided offset: %d",
                file.RealPath(), uploadOffset,
            )
        }
        // 6. 以追加模式打开文件
        openFile, err := d.user.Fs.OpenFile(r.URL.Path, os.O_WRONLY|os.O_APPEND, ...)
        // 7. 定位到目标偏移 (虽然是追加模式，仍显式 Seek 保证正确性)
        _, err = openFile.Seek(uploadOffset, 0)
        // 8. 流式写入数据 (避免大文件内存溢出)
        bytesWritten, err := io.Copy(openFile, r.Body)
        // 9. 刷盘! 防止系统崩溃导致文件损坏
        if err := openFile.Sync(); err != nil { ... }
        // 10. 返回新的偏移量
        newOffset := uploadOffset + bytesWritten
        w.Header().Set("Upload-Offset", strconv.FormatInt(newOffset, 10))
        // 11. 上传完成: 删除缓存 + 触发 hook
        if newOffset >= uploadLength {
            cache.Complete(file.RealPath())
            d.RunHook(func() error { return nil }, "upload", r.URL.Path, "", d.user)
        }
        return http.StatusNoContent, nil
    })
}
```

**保活机制** [keepUploadActive](file:///d:/fz/0601/solo-dogfeeding/code/177-filebrowser/http/tus_handlers.go#L19-L39)：
```go
func keepUploadActive(cache UploadCache, filePath string) func() {
    stop := make(chan bool)
    go func() {
        ticker := time.NewTicker(2 * time.Second)  // 每 2 秒
        defer ticker.Stop()
        for {
            select {
            case <-stop: return
            case <-ticker.C: cache.Touch(filePath)  // 刷新 TTL
            }
        }
    }()
    return func() { close(stop) }
}
```
> 对于大文件单片（如 10MB 在弱网下传输超过 3 分钟），如果不刷新 TTL，缓存会在传输过程中过期被清理，导致后续分片无法识别为续传。

### 4.4 DELETE - 取消上传
[tusDeleteHandler](file:///d:/fz/0601/solo-dogfeeding/code/177-filebrowser/http/tus_handlers.go#L241-L273)

```go
func tusDeleteHandler(cache UploadCache) handleFunc {
    return withUser(func(_ http.ResponseWriter, r *http.Request, d *data) (int, error) {
        // 1. 权限检查: 需要 Delete 权限
        if r.URL.Path == "/" || !d.user.Perm.Delete {
            return http.StatusForbidden, nil
        }
        // 2. 确认该路径有活跃的上传任务
        _, err = cache.GetLength(file.RealPath())
        // 3. 删除磁盘上的未完成文件
        err = d.user.Fs.RemoveAll(r.URL.Path)
        // 4. 从缓存中清除
        cache.Complete(file.RealPath())
        return http.StatusNoContent, nil
    })
}
```

---

## 五、续传状态管理（UploadCache 缓存层）

### 5.1 接口定义
[UploadCache](file:///d:/fz/0601/solo-dogfeeding/code/177-filebrowser/http/upload_cache_memory.go#L14-L32) 是上传状态的抽象：

```go
type UploadCache interface {
    Register(filePath string, fileSize int64)  // 新建上传: 路径 → 期望大小
    Complete(filePath string)                  // 完成/取消: 删除缓存
    GetLength(filePath string) (int64, error)  // 查询: 获取期望总大小
    Touch(filePath string)                     // 保活: 刷新 TTL
    Close()                                    // 清理资源
}
```

### 5.2 内存缓存（单实例部署）
[memoryUploadCache](file:///d:/fz/0601/solo-dogfeeding/code/177-filebrowser/http/upload_cache_memory.go#L34-L74) 使用 `ttlcache` 库：

```go
type memoryUploadCache struct {
    cache *ttlcache.Cache[string, int64]  // key: 文件绝对路径, value: 总字节数
}

func newMemoryUploadCache() *memoryUploadCache {
    cache := ttlcache.New[string, int64]()
    // 过期回调: 自动删除磁盘上的未完成文件
    cache.OnEviction(func(_ context.Context, reason ttlcache.EvictionReason, item *ttlcache.Item[string, int64]) {
        if reason == ttlcache.EvictionReasonExpired {
            fmt.Printf("deleting incomplete upload file: \"%s\"\n", item.Key())
            os.Remove(item.Key())  // 3 分钟无活动 → 清理文件
        }
    })
    go cache.Start()
    return &memoryUploadCache{cache: cache}
}
```

### 5.3 Redis 缓存（多实例部署）
[redisUploadCache](file:///d:/fz/0601/solo-dogfeeding/code/177-filebrowser/http/upload_cache_redis.go#L13-L84) 使用 Redis 存储：

```go
type redisUploadCache struct {
    client *redis.Client
}

func (c *redisUploadCache) filePathKey(filePath string) string {
    return "filebrowser:upload:" + filePath  // Redis key 前缀
}

func (c *redisUploadCache) Register(filePath string, fileSize int64) {
    c.client.Set(ctx, c.filePathKey(filePath), fileSize, uploadCacheTTL)  // 3min TTL
}

func (c *redisUploadCache) GetLength(filePath string) (int64, error) {
    result, err := c.client.Get(ctx, c.filePathKey(filePath)).Result()
    // ... 错误处理 ...
    size, _ := strconv.ParseInt(result, 10, 64)
    c.Touch(filePath)  // 查询时自动续期 TTL
    return size, nil
}
```

### 5.4 缓存创建工厂
[NewUploadCache](file:///d:/fz/0601/solo-dogfeeding/code/177-filebrowser/http/upload_cache_memory.go#L76-L85)：
```go
func NewUploadCache(redisURL string) (UploadCache, error) {
    if redisURL != "" {
        return newRedisUploadCache(redisURL)   // 多实例: Redis
    }
    return newMemoryUploadCache(), nil          // 单实例: 内存
}
```

---

## 六、断点续传恢复过程详解

### 6.1 续传触发条件

**真正的断点续传**（即从断点继续上传，不从头开始）只在非常有限的条件下发生：

| 场景 | 是否刷新页面 | tus-js-client 实例 | 最终行为 |
|------|------------|-------------------|---------|
| 网络短暂中断（15秒重试窗口内） | ❌ 否 | **同一个** | ✅ **真正断点续传**（同一实例 → 先 HEAD 查询断点 → PATCH 继续） |
| 服务端实例滚动重启（15秒窗口内） | ❌ 否 | **同一个** | ✅ **真正断点续传**（跨实例续传，同一前端实例 → HEAD到新实例 → PATCH继续） |
| 用户暂停/恢复（不刷新页面） | ❌ 否 | **同一个** | ✅ **真正断点续传** |
| 页面刷新后重新选择文件（缓存有效，<3分钟） | ✅ 是 | **全新** | ❌ **覆盖重传**（新实例 → 直接 POST → O_TRUNC清空 → 从0开始） |
| 页面刷新后重新选择文件（缓存过期，>3分钟） | ✅ 是 | **全新** | ❌ **从头上传**（新实例 → POST → 从0开始） |

> **关键前提**：本项目 `storeFingerprintForResuming: false`，且未调用 `findPreviousUploads()` + `resumeFromPreviousUpload()`，因此**页面刷新后 tus-js-client 完全不知道之前的上传历史**。即使服务端缓存有效、磁盘有部分文件，也会从头开始。

### 6.2 续传的完整时序

以 "网络中断 10 秒后恢复" 为例：

```
时间轴:
T0  用户点击上传 100MB 文件
T0  POST /api/tus/file.zip  Upload-Length:104857600
T0+ 201 Created  Location:/api/tus/file.zip
T0+ HEAD /api/tus/file.zip → Upload-Offset:0, Upload-Length:104857600
T0+ PATCH offset=0, 10MB数据 → 204, Upload-Offset:10485760  (片1完成)
T1  PATCH offset=10485760, 10MB数据 → 204, Upload-Offset:20971520  (片2完成)
T2  PATCH offset=20971520, 开始传片3...
T3  网络断开! PATCH 请求失败 → 触发重试机制
T3  重试延迟 0ms → PATCH 再次失败
T4  重试延迟 1000ms → PATCH 再次失败
T5  重试延迟 2000ms → PATCH 再次失败
    ...
T13 网络恢复! (距离 T3 过去 10s)
T13 tus-js-client 执行续传流程:
     ├─ HEAD /api/tus/file.zip
     │   服务端: file.Size = 20971520 (20MB)
     │          cache.GetLength = 104857600 (100MB, 缓存仍有效)
     │   返回: Upload-Offset:20971520, Upload-Length:104857600
     │
     ├─ 客户端对比: 已上传 20MB < 总 100MB
     ├─ 从本地文件的 offset=20971520 位置切下 10MB
     └─ PATCH /api/tus/file.zip
         Upload-Offset:20971520
         Content-Type: application/offset+octet-stream
         Body: [file.zip 字节 20971520~31457279]

T13 服务端 tusPatchHandler 校验:
     ├─ file.Size == 20971520 == Upload-Offset ✓
     ├─ 写入数据 10MB → Sync
     └─ 返回 Upload-Offset:31457280 (30MB)

T14 片4 PATCH offset=31457280 → 成功 (40MB)
    ... 继续分片直到完成 ...
T20 最后一片 PATCH (100MB)
     newOffset(104857600) >= uploadLength(104857600)
     → cache.Complete()  删除缓存
     → 触发 upload hook
     → 204 No Content
T20 上传完成!
```

### 6.3 数据一致性保证

续传过程中有三层保障防止数据损坏：

1. **Offset 一致性校验**（[tusPatchHandler](file:///d:/fz/0601/solo-dogfeeding/code/177-filebrowser/http/tus_handlers.go#L195-L204)）
   ```go
   if file.Size != uploadOffset {
       return http.StatusConflict, ...  // 文件大小与客户端声明的偏移不一致，拒绝写入
   }
   ```
   防止客户端误判断点位置、或者多客户端同时写入同一文件。

2. **强制刷盘**（[tusPatchHandler](file:///d:/fz/0601/solo-dogfeeding/code/177-filebrowser/http/tus_handlers.go#L224-L227)）
   ```go
   if err := openFile.Sync(); err != nil { ... }
   ```
   每片写完立即 `fsync`，确保操作系统缓存写入磁盘。即使服务端崩溃，已确认的分片也不会丢失。

3. **缓存过期自动清理**（内存缓存 OnEviction 回调）
   上传中断超过 3 分钟后自动删除未完成的文件和缓存记录，下次重新上传从头开始，避免半损坏文件残留。

---

## 七、边界场景恢复分支详解

### 7.0 关键前提：tus-js-client 行为核准

在分析所有边界场景之前，必须先明确本项目中 tus-js-client 的**真实工作机制**。之前容易误解的地方：

#### tus-js-client 何时会先 HEAD 查询？

tus-js-client 的 `start()` 方法逻辑：

| 条件 | 行为 |
|------|------|
| 设置了 `uploadUrl` 选项 | **先 HEAD** `uploadUrl` 查询进度，失败再 POST 到 `endpoint` |
| `storeFingerprintForResuming: true` 且 localStorage 中有历史记录 | **先 HEAD** 历史 URL 查询进度，失败再 POST |
| 同一个 Upload 实例内的重试（PATCH失败后 retryDelays） | **先 HEAD** 已有的 `upload.url` 查询断点，再继续 PATCH |
| **本项目配置**（无 uploadUrl + storeFingerprintForResuming:false + 新Upload实例） | **直接 POST** 到 endpoint，**不会先 HEAD** |

#### 本项目的关键配置
[tus.ts](file:///d:/fz/0601/solo-dogfeeding/code/177-filebrowser/frontend/src/api/tus.ts#L31-L82)：
```typescript
const upload = new tus.Upload(content, {
    // endpoint 直接包含文件路径！不是标准 TUS 的基础 endpoint
    endpoint: `${origin}${baseURL}/api/tus${filePath}?override=${overwrite}`,
    chunkSize: tusSettings.chunkSize,
    parallelUploads: 1,
    storeFingerprintForResuming: false,  // ❌ 关闭指纹存储
    // ❌ 未设置 uploadUrl 选项
    // ❌ 未调用 findPreviousUploads() / resumeFromPreviousUpload()
    ...
});
upload.start();  // 新实例 → 直接 POST
```

#### 三个核心结论

1. **页面刷新后 = 全新 Upload 实例**：`CURRENT_UPLOAD_LIST` 是内存变量，刷新后丢失，完全不知道之前有过上传
2. **`storeFingerprintForResuming: false`**：tus-js-client 不会在 localStorage 中保存 "文件指纹 → upload URL" 映射，刷新后无从查起
3. **endpoint 本身就是文件 URL**：POST 直接打到 `/api/tus/file.zip?override=xxx`，由后端 tusPostHandler 处理

> **因此，只有"同一个 tus.Upload 实例内的自动重试"才能真正从断点续传。页面刷新后重新选择同名文件，无论缓存是否有效，都会走 POST → (O_TRUNC) → 从头上传的路径。**

---

### 7.1 场景一：同页面内自动重试（真正的断点续传）

这是项目中**唯一真正能断点续传**的场景。

#### 触发条件
- 页面未刷新
- 同一个 tus.Upload 实例（保存在 `CURRENT_UPLOAD_LIST[filePath]`）
- PATCH 请求失败（网络抖动、服务端临时不可用）
- 在 `retryDelays` 次数范围内（默认 5 次：[0, 1000, 2000, 4000, 8000] ms）

#### 恢复时序
```
T0  上传 100MB 文件到 20MB，发送 PATCH offset=20971520
T0+ 网络中断！PATCH 请求失败
T0+ tus-js-client 内部重试机制启动
     ├─ 第1次重试 (0ms 延迟)
     │   ├─ 🔑 关键: 同一实例已保存 this.url = "/api/tus/file.zip"
     │   ├─ tus-js-client 先 HEAD /api/tus/file.zip
     │   │   ├─ tusHeadHandler: file.Size=20971520 (20MB)
     │   │   ├─ tusHeadHandler: cache.GetLength=104857600 (100MB)
     │   │   └─ 返回 Upload-Offset:20971520, Upload-Length:104857600
     │   ├─ tus-js-client: 确认断点在 20MB
     │   ├─ 从本地文件 slice(20971520, 31457280) 取下一片 10MB
     │   └─ 再次发送 PATCH offset=20971520 → 仍然失败 (网络还没恢复)
     │
     ├─ 第2次重试 (1000ms 延迟) → 仍然失败
     ├─ 第3次重试 (2000ms 延迟) → 仍然失败
     ├─ 第4次重试 (4000ms 延迟) → 网络恢复了!
     │   ├─ HEAD /api/tus/file.zip → Upload-Offset:20971520, Upload-Length:104857600
     │   ├─ PATCH offset=20971520, 10MB 数据
     │   │   ├─ tusPatchHandler: file.Size(20MB) == uploadOffset(20MB) ✓
     │   │   ├─ 写入 + Sync
     │   │   └─ 返回 Upload-Offset:31457280
     │   └─ ✅ 成功！从第3片继续
     │
     └─ 后续分片正常上传...
```

#### 代码中的重试延迟计算
[tus.ts computeRetryDelays](file:///d:/fz/0601/solo-dogfeeding/code/177-filebrowser/frontend/src/api/tus.ts#L85-L102)：
```typescript
// 5次重试对应的延迟: [0, 1000, 2000, 4000, 8000]
// 总重试窗口 = 0 + 1000 + 2000 + 4000 + 8000 = 15 秒
// 如果 15 秒内网络不恢复 → 抛错 → 上传终止
for (let i = 0; i < tusSettings.retryCount; i++) {
    retryDelays.push(Math.min(delay, RETRY_MAX_DELAY));
    delay = delay === 0 ? RETRY_BASE_DELAY : Math.min(delay * 2, RETRY_MAX_DELAY);
}
```

> **关键限制**：自动重试窗口只有 15 秒。超过 15 秒网络仍未恢复 → tus-js-client 抛错 → 前端显示错误 → 用户必须手动重新上传。

---

### 7.2 场景二：页面刷新后重新选择同名文件（缓存有效，<3分钟）

#### 场景描述
- 上传到 20MB 断网，超过 15 秒重试窗口 → 上传失败
- 用户刷新了页面
- 1 分钟后网络恢复，用户手动重新选择同一个文件上传
- **此时缓存仍然有效（<3分钟）**，磁盘上也有 20MB 文件

#### 真实行为（⚠️ 不会智能续传，会从头开始！）

```
1. 用户在上传对话框选择文件 → 触发 checkConflict()
   [checkConflict()](file:///d:/fz/0601/solo-dogfeeding/code/177-filebrowser/frontend/src/utils/upload.ts#L43-L89)
   ├─ 拉取服务端目录列表
   ├─ 发现 file.zip 已存在，大小 20MB < 本地 100MB
   └─ isSmallerOnServer = true → 标记为冲突

2. 弹出 ResolveConflict.vue 对话框:
   ├─ 用户点击 "Resume Transfer" 按钮
   │   [resume()](file:///d:/fz/0601/solo-dogfeeding/code/177-filebrowser/frontend/src/components/prompts/ResolveConflict.vue#L207-L216)
   │   └─ item.checked = ["origin"]  (标记为"用源文件覆盖")
   ├─ 或者用户点击 "Override" 按钮
   │   └─ 同样 item.checked = ["origin"]
   └─ confirm → Upload.vue 回调:
      [Upload.vue](file:///d:/fz/0601/solo-dogfeeding/code/177-filebrowser/frontend/src/components/prompts/Upload.vue#L78-L94)
      └─ uploadFiles[item.index].overwrite = true  // 仅仅是设了个标记

3. 进入 upload.handleFiles() → uploadStore.upload() → api.post() → tus.upload()
   创建**全新的 tus.Upload 实例**

4. 🔴 关键: upload.start() 的行为
   ├─ 没有 uploadUrl 选项
   ├─ storeFingerprintForResuming = false
   ├─ 是全新实例，this.url 未设置
   └─ → **直接 POST 到 endpoint** (/api/tus/file.zip?override=true)
      ❌ 不会先 HEAD 查询！

5. [tusPostHandler](file:///d:/fz/0601/solo-dogfeeding/code/177-filebrowser/http/tus_handlers.go#L70-L86) 处理 POST:
   ```go
   if file != nil {
       // 文件已存在 (20MB)
       if r.URL.Query().Get("override") != "true" {
           return http.StatusConflict, nil
       }
       if !d.user.Perm.Modify {
           return http.StatusForbidden, nil
       }
       fileFlags |= os.O_TRUNC  // ✅ 强制清空文件！
   }
   openFile, err := d.user.Fs.OpenFile(r.URL.Path, fileFlags, ...)
   // 此时文件大小被重置为 0
   cache.Register(file.RealPath(), uploadLength)  // 重新注册缓存
   ```

6. 返回 201 Created → tus-js-client 对返回的 Location 发 HEAD:
   ├─ Upload-Offset: 0 (文件被清空了)
   └─ Upload-Length: 104857600

7. 从 offset=0 开始 PATCH，重新上传所有分片
```

#### 为什么"Resume Transfer"按钮名不副实？
按钮叫 "Resume Transfer"（续传），但实际执行的是 `overwrite=true` 的**覆盖重传**。原因是：
- tus-js-client 的 `storeFingerprintForResuming` 被关闭了
- 项目代码也没有调用 `findPreviousUploads()` + `resumeFromPreviousUpload()`
- 所以新实例无法知道之前的上传 URL，只能从 POST 开始

---

### 7.3 场景三：缓存过期（>3分钟）后重新上传

#### 3a. 内存缓存模式（单实例）

内存缓存有 OnEviction 回调，会自动删文件：
```
T0      上传到 20MB 失败（断网超过 15 秒重试窗口）
T0+3min 缓存过期 → OnEviction 回调: os.Remove("/data/file.zip")
        磁盘文件被删除!
T0+4min 用户重新选择 file.zip 上传
        ├─ checkConflict(): 服务端无此文件 → 无冲突
        ├─ tus-js-client 新实例 → 直接 POST /api/tus/file.zip?override=false
        ├─ tusPostHandler: 文件不存在 → 创建新空文件
        ├─ cache.Register()
        └─ 从 offset=0 正常上传
```

内存模式过期代码：[memoryUploadCache OnEviction](file:///d:/fz/0601/solo-dogfeeding/code/177-filebrowser/http/upload_cache_memory.go#L41-L46)
```go
cache.OnEviction(func(_ context.Context, reason ttlcache.EvictionReason, item *ttlcache.Item[string, int64]) {
    if reason == ttlcache.EvictionReasonExpired {
        fmt.Printf("deleting incomplete upload file: \"%s\"\n", item.Key())
        os.Remove(item.Key())
    }
})
```

#### 3b. Redis 缓存模式（多实例）

Redis 没有 OnEviction 回调，磁盘文件残留：
```
T0      上传到 20MB 失败
T0+3min Redis key 过期 → key 自动删除，但 NFS 上 20MB 文件仍在
T0+4min 用户重新选择 file.zip 上传
        ├─ checkConflict(): 发现服务端已有 20MB < 100MB → 冲突
        ├─ 用户点击 Resume Transfer → overwrite=true
        ├─ tus-js-client 新实例 → 直接 POST /api/tus/file.zip?override=true
        ├─ tusPostHandler:
        │   ├─ 文件存在 + override=true → O_TRUNC 清空为 0
        │   ├─ cache.Register() 重新注册
        │   └─ 从 offset=0 开始上传
        └─ 效果同 7.2 场景：覆盖重传
```

> **内存 vs Redis 的行为不一致**：内存模式过期自动删文件，重新上传无冲突；Redis 模式过期留垃圾文件，重新上传会触发冲突对话框 → 选 Resume → 覆盖重传。

#### tusHeadHandler 返回 404 的真正用途
[tusHeadHandler](file:///d:/fz/0601/solo-dogfeeding/code/177-filebrowser/http/tus_handlers.go#L144-L147)：
```go
uploadLength, err := cache.GetLength(file.RealPath())
if err != nil {
    return http.StatusNotFound, err
}
```
这个 404 **只在 "同一个 tus.Upload 实例内的自动重试" 场景下才有意义**（tus-js-client 收到 404 会重新 POST 创建上传）。页面刷新后的场景根本走不到 HEAD，因为直接先 POST 了。

---

### 7.4 场景四：多实例部署下的自动重试

多实例部署（Redis + 共享存储）时，即使服务端实例滚动重启，只要在 tus-js-client 的 15 秒重试窗口内，仍能断点续传。

#### 部署架构
```
负载均衡 → [实例A, 实例B, 实例C] → Redis 缓存 + NFS 共享存储
```

#### 恢复时序
```
T0     POST /api/tus/file.zip → 负载均衡转发到实例A
       └─ 实例A: Redis SET filebrowser:upload:/data/file.zip = 104857600 EX 180
       └─ 实例A: NFS 创建空文件
       └─ 返回 201 Created, tus-js-client 保存 this.url = "/api/tus/file.zip"

T0+5s  PATCH offset=0, 10MB → 实例A → 成功, 返回 Upload-Offset:10485760
T0+10s PATCH offset=10485760, 10MB → 实例A → 成功, 返回 Upload-Offset:20971520
T0+11s 发送 PATCH offset=20971520...
T0+11s 实例A 因滚动升级被终止！连接中断 → PATCH 失败

T0+11s tus-js-client 开始重试:
       ├─ 第1次重试 (0ms)
       │   ├─ 同一实例: this.url = "/api/tus/file.zip"
       │   ├─ 先 HEAD /api/tus/file.zip → 负载均衡转发到实例B
       │   │   ├─ 实例B: Redis GET filebrowser:upload:/data/file.zip → 100MB ✓
       │   │   ├─ 实例B: NFS stat → size=20971520 (20MB) ✓
       │   │   └─ 返回 Upload-Offset:20971520, Upload-Length:104857600
       │   ├─ tus-js-client: 断点确认在 20MB
       │   └─ PATCH offset=20971520, 10MB → 实例B
       │       ├─ 实例B: file.Size(20MB) == uploadOffset(20MB) ✓
       │       ├─ NFS 追加写入 + Sync
       │       └─ 返回 Upload-Offset:31457280
       └─ ✅ 成功！后续 PATCH 可能被转发到任何实例
```

#### 多实例部署强制约束
- **必须共享文件系统**（NFS、分布式存储等）：否则实例B无法读取实例A写入的文件内容和大小，`tusPatchHandler` 的 `file.Size != uploadOffset` 校验会失败
- **所有实例连接同一 Redis 集群**：确保缓存状态共享
- **负载均衡必须保持 HTTP 语义兼容**：POST/HEAD/PATCH/DELETE 都能正确转发

---

### 7.5 场景五：其他边界分支

#### 5a. 用户选择 Skip
检测到冲突后点击 "Skip"，从上传列表中删除：
```typescript
// Upload.vue confirm 回调
if (item.checked.length == 1 && item.checked[0] == "dest") {
    uploadFiles.splice(item.index, 1);  // 从上传列表移除
}
```

#### 5b. 服务端文件已完整 → 自动跳过
上次上传完整完成（100MB），再次选择同一文件：
```typescript
// checkConflict() 中
isSmallerOnServer: file.size > server.size  // 100MB > 100MB = false

// ResolveConflict.vue resume() 按钮
if (item.isSmallerOnServer) {
    item.checked = ["origin"];  // 覆盖
} else {
    item.checked = ["dest"];    // ✅ 服务端文件不小 → 默认选跳过
}
```

> 这是防重复上传逻辑：服务端文件大小 >= 本地文件时，默认认为服务端文件是完整的，自动跳过。

---

### 7.6 所有恢复场景总结对照表

| 场景 | 是否刷新页面 | 缓存状态 | tus-js-client 行为 | 最终结果 |
|------|------------|---------|-------------------|---------|
| **7.1 同页面自动重试** | ❌ 否 | ✅ 有效 | 同一实例 → 先 HEAD → 再 PATCH | ✅ **真正断点续传** |
| **7.2 刷新后重传（缓存有效）** | ✅ 是 | ✅ 有效 (<3min) | 新实例 → 直接 POST → O_TRUNC | ❌ 覆盖重传（从0开始） |
| **7.3a 内存模式过期** | 无所谓 | ❌ 过期 | 文件被删 → 无冲突 → POST 新文件 | ✅ 新上传（从0开始，无冲突提示） |
| **7.3b Redis模式过期** | 无所谓 | ❌ 过期 | 文件残留 → 冲突 → Resume → O_TRUNC | ❌ 覆盖重传（从0开始） |
| **7.4 多实例滚动重启** | ❌ 否 | ✅ 有效 | 同一实例 → HEAD到新实例 → PATCH | ✅ **真正断点续传**（跨实例） |
| **7.5b 服务端文件完整** | - | - | checkConflict 检测大小一致 → 跳过 | ✅ 不上传 |

---

## 八、传统上传方式（非 TUS 回退）

当 TUS 不可用时（如非 HTTP 协议、浏览器不支持），使用 [postResources](file:///d:/fz/0601/solo-dogfeeding/code/177-filebrowser/frontend/src/api/files.ts#L126-L172) 走传统 XMLHttpRequest：

- 前端：使用 XHR `send()` 一次性发送整个 Blob
- 后端：使用 [resourcePostHandler](file:///d:/fz/0601/solo-dogfeeding/code/177-filebrowser/http/resource.go#L125-L178) 的 `writeFile()` 一次性写入

这种方式**不支持断点续传**，适合小文件或非 HTTP 环境。

---

## 九、前端进度显示与交互

[UploadFiles.vue](file:///d:/fz/0601/solo-dogfeeding/code/177-filebrowser/frontend/src/components/prompts/UploadFiles.vue) 组件负责显示上传进度：

### 进度计算优化
```typescript
// 每秒批量同步原始进度到响应式状态（减少渲染次数）
const syncState = () => {
    for (const upload of activeUploads.value) {
        sentBytes.value += upload.rawProgress.sentBytes - upload.sentBytes;
        upload.sentBytes = upload.rawProgress.sentBytes;
    }
};
```

### 速度与 ETA 计算
```typescript
// 滑动窗口平均（取最近5次速度样本，指数平滑）
recentSpeeds.push(currentSpeed);
if (recentSpeeds.length > 5) recentSpeeds.shift();
speed.value = recentSpeedsAverage * 0.2 + speed.value * 0.8;

// ETA = 剩余字节 / 当前速度
eta.value = (uploadStore.totalBytes - uploadStore.sentBytes) / speed.value;
```

### 中止全部上传
[upload.ts (store)](file:///d:/fz/0601/solo-dogfeeding/code/177-filebrowser/frontend/src/stores/upload.ts#L68-L72)：
```typescript
const abort = () => {
    lastUpload.value = Infinity;           // 阻止后续队列处理
    tus.abortAllUploads();                 // 中止所有 TUS 上传
};
```

[tus.ts](file:///d:/fz/0601/solo-dogfeeding/code/177-filebrowser/frontend/src/api/tus.ts#L112-L122)：
```typescript
export function abortAllUploads() {
    for (const filePath in CURRENT_UPLOAD_LIST) {
        CURRENT_UPLOAD_LIST[filePath].abort(true);  // 调用 tus-js-client 的 abort
        CURRENT_UPLOAD_LIST[filePath].options!.onError!(new Error("Upload aborted"));
        delete CURRENT_UPLOAD_LIST[filePath];
    }
}
```

---

## 十、总结：设计亮点与注意事项

### ✅ 设计亮点

1. **协议标准化**：采用 TUS 开放协议而非自实现，客户端使用成熟的 tus-js-client，兼容性有保障
2. **双缓存后端**：UploadCache 接口抽象，支持内存（单实例）和 Redis（多实例）两种部署模式
3. **流式写入**：使用 `io.Copy` 流式拷贝，不把分片内容加载到内存，支持超大文件
4. **多重一致性保障**：Offset 校验 + 每片 Sync + 自动清理过期文件
5. **性能优化**：进度数据非响应式缓冲 + 每秒同步，避免高频重渲染
6. **指数退避重试**：避免网络抖动时雪崩式重试
7. **智能冲突处理**：`resume()` 函数根据 `isSmallerOnServer` 自动判断是续传还是跳过，防止重复上传完整文件

### ⚠️ 注意事项

#### 通用注意事项
1. **只有自动重试才能真正断点续传**：断点续传**仅**发生在同一个 tus.Upload 实例内部的自动重试（15秒重试窗口内）。页面刷新后重新选择文件，无论缓存是否有效，都会从头重传
2. **默认不支持跨页面续传**：`storeFingerprintForResuming: false` 导致 tus-js-client 不在 localStorage 中保存上传 URL，且项目代码未调用 `findPreviousUploads()` + `resumeFromPreviousUpload()`
3. **重试窗口极短**：默认 5 次重试的延迟是 [0, 1000, 2000, 4000, 8000] ms，总窗口只有 15 秒。超过 15 秒网络不恢复 → 上传终止 → 必须手动重新上传
4. **TTL 严格限制**：上传中断超过 3 分钟未恢复，缓存过期
5. **单片串行**：`parallelUploads: 1`，同一文件的多个分片按顺序传输，不能利用多连接加速
6. **10MB 分片大小**：小文件（<10MB）会退化为单请求上传，大文件分片数 = 文件大小 / 10MB

#### 边界场景注意事项
7. **内存 vs Redis 缓存行为不一致**：
   - 内存缓存：过期时通过 `OnEviction` 回调自动删除磁盘文件，重新上传时无冲突
   - Redis 缓存：过期后**不会**自动删文件，会残留半完成文件，重新上传触发冲突对话框 → 选 Resume → 覆盖重传
8. **"Resume Transfer" 按钮名不副实**：
   按钮文字是"续传"，但实际行为是设置 `overwrite=true` 后创建全新的 tus.Upload 实例 → 直接 POST → O_TRUNC 清空已有文件 → **从头重传**。不是真正的断点续传
9. **tusHeadHandler 返回 404 只有一个用途**：
   仅用于同一 tus.Upload 实例的内部自动重试。页面刷新后的重新上传走不到 HEAD 请求，因为 tus-js-client 直接先发 POST
10. **多实例部署强制要求**：
   - 必须使用共享文件系统（NFS、分布式存储等），否则实例间无法读取对方写入的文件
   - 所有实例必须连接同一个 Redis 集群，确保缓存状态共享
11. **防重复上传逻辑**：
   当服务端文件大小 >= 本地文件大小时，`resume()` 会自动选择跳过，认为服务端文件是完整的

#### 实现跨页面续传的改进思路
如果需要支持页面刷新后真正断点续传，需要做以下修改：
- 前端：`storeFingerprintForResuming` 改为 `true`，或手动在上传前调用 `upload.findPreviousUploads()` + `upload.resumeFromPreviousUpload()`
- 后端：确保 POST 收到已有文件且 `override=false` 时，不返回 409 Conflict，而是配合 tus-js-client 继续使用
