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

续传由 **tus-js-client 库自动处理**，在以下场景触发：

| 场景 | 行为 |
|------|------|
| 网络短暂中断（在 retryCount 内） | 自动重试，重试前先 HEAD 查询断点 |
| 用户暂停/恢复 | 重新 `upload.start()` → 自动 HEAD 查询 |
| 页面刷新后重新选择同一文件 | ⚠️ 见下方说明 |
| 超过 TTL（3分钟） | 缓存被清理 → HEAD 返回 404 → 重新创建新上传 |

> **注意**：当前代码设置了 `storeFingerprintForResuming: false`，这意味着 **页面刷新后 tus-js-client 不会自动恢复之前的上传**，需要用户重新选择文件。如果需要跨页面续传，需将此选项改为 `true` 并配合持久化存储。

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

### 7.1 边界场景一：缓存过期（>3分钟）

当上传中断超过 `uploadCacheTTL`（3分钟），缓存会自动过期。此时恢复会走一条独立的代码路径。

#### 过期触发点
内存缓存的过期回调在 [memoryUploadCache](file:///d:/fz/0601/solo-dogfeeding/code/177-filebrowser/http/upload_cache_memory.go#L41-L46)：
```go
cache.OnEviction(func(_ context.Context, reason ttlcache.EvictionReason, item *ttlcache.Item[string, int64]) {
    if reason == ttlcache.EvictionReasonExpired {
        fmt.Printf("deleting incomplete upload file: \"%s\"\n", item.Key())
        os.Remove(item.Key())  // ✅ 过期时自动删除磁盘文件
    }
})
```

> **关键区别**：Redis 缓存**没有**这个 OnEviction 回调，过期后不会自动删文件。参见 7.2 节。

#### 恢复分支时序（内存缓存模式）
```
场景: 上传到50MB后断网，4分钟后网络恢复

T0     用户点击上传 100MB 文件
T0+10s PATCH offset=41943040 完成 → 50MB
T0+11s 网络断开! PATCH失败 → tus-js-client开始重试
T0+11s~T0+15s 重试5次 (0s+1s+2s+4s+8s = 15s) 全部失败
T0+15s tus-js-client抛错 → 上传终止，前端显示错误
T0+3min11s 缓存过期 → OnEviction回调执行 os.Remove → 磁盘文件被删除!
T0+4min  网络恢复，用户手动重新上传同一文件

恢复路径:
  ├─ 用户选择文件 → checkConflict() 检测服务端文件
  │   └─ 服务端文件已被删除 → 无冲突
  ├─ tus-js-client.start()
  │   ├─ 先 HEAD /api/tus/file.zip
  │   │   └─ tusHeadHandler: cache.GetLength() → 缓存已过期
  │   │       └─ return 404 "no active upload found"
  │   └─ tus-js-client收到404 → 认为是新上传
  │       └─ 发起 POST /api/tus/file.zip
  └─ tusPostHandler:
      ├─ 文件不存在 → 创建新空文件
      ├─ cache.Register() 重新注册
      └─ 从 offset=0 重新开始上传
```

#### 关键代码判断点
[tusHeadHandler](file:///d:/fz/0601/solo-dogfeeding/code/177-filebrowser/http/tus_handlers.go#L144-L147) 中缓存查询失败直接返回 404：
```go
uploadLength, err := cache.GetLength(file.RealPath())
if err != nil {
    return http.StatusNotFound, err  // ← 缓存过期时返回404
}
```

---

### 7.2 边界场景二：多实例部署（Redis 缓存）

多实例部署时使用 Redis 作为共享缓存，配合共享文件系统（NFS、分布式存储等）实现跨实例续传。

#### 缓存创建入口
[NewUploadCache](file:///d:/fz/0601/solo-dogfeeding/code/177-filebrowser/http/upload_cache_memory.go#L76-L85)：
```go
func NewUploadCache(redisURL string) (UploadCache, error) {
    if redisURL != "" {
        return newRedisUploadCache(redisURL)   // 多实例: Redis
    }
    return newMemoryUploadCache(), nil          // 单实例: 内存
}
```

#### 恢复分支时序（Redis + 共享存储）
```
部署架构: 负载均衡 → [实例A, 实例B, 实例C] → Redis + NFS共享存储

T0     POST /api/tus/file.zip → 被转发到实例A
       └─ 实例A: cache.Register() → Redis SET key=filebrowser:upload:/data/file.zip value=104857600 EX 180
       └─ 实例A: 在NFS上创建空文件
       └─ 返回 201 Created

T0+5s  PATCH offset=0, 10MB → 实例A
       └─ 实例A: Redis GET → 100MB ✓
       └─ 写入NFS + Sync
       └─ 返回 Upload-Offset:10485760

T0+10s PATCH offset=10485760, 10MB → 实例A
       └─ 返回 Upload-Offset:20971520 (20MB)

T0+11s 网络断开!
T0+12s 实例A因滚动升级被重启
T0+15s 网络恢复 + 实例B就绪
       └─ 负载均衡将重试请求转发到实例B

恢复路径 (实例B处理):
  ├─ tus-js-client自动重试 → HEAD /api/tus/file.zip → 实例B
  │   ├─ 实例B: Redis GET filebrowser:upload:/data/file.zip → 100MB ✓ (缓存共享)
  │   ├─ 实例B: stat NFS上的文件 → size=20971520 ✓ (存储共享)
  │   └─ 返回 Upload-Offset:20971520, Upload-Length:104857600
  ├─ tus-js-client: 已传20MB < 100MB → 从offset=20971520续传
  ├─ PATCH offset=20971520, 10MB → 实例B
  │   ├─ 实例B: file.Size == 20971520 == uploadOffset ✓
  │   ├─ 写入NFS + Sync
  │   └─ 返回 Upload-Offset:31457280
  └─ ...继续后续分片...
```

#### Redis 模式的特有问题
Redis 缓存**没有 OnEviction 回调**，过期后无法自动删除磁盘文件：
```
T0+11s 断网
T0+3min11s Redis键过期 → key被自动删除，但NFS上的20MB文件仍在!
T0+4min 用户重新上传
  ├─ checkConflict() 检测到NFS上已有20MB文件 → 触发冲突对话框
  ├─ 用户选择 Resume → overwrite=true
  ├─ tus-js-client HEAD查询 → Redis已过期 → 404
  ├─ tus-js-client POST /api/tus/file.zip?override=true
  └─ tusPostHandler: 文件存在 + override=true → O_TRUNC清空 → 从0开始传
```

> **潜在改进**：可以通过 Redis Keyspace Notifications 监听过期事件，或者配合定时任务扫描清理半完成文件。

#### 多实例部署强制约束
- **必须共享文件系统**：如果实例A写本地磁盘，实例B无法读取文件大小，`tusHeadHandler` 会返回错误的 Offset，`tusPatchHandler` 的 `file.Size != uploadOffset` 校验会失败
- **Redis 必须同一集群**：所有实例连接同一个 Redis，确保缓存状态一致

---

### 7.3 边界场景三：重新选择同名文件

用户刷新页面或关闭浏览器后，重新选择同一文件上传。这是最复杂的边界场景，有 **4 条独立的恢复分支**。

#### 前置流程：冲突检测
每次上传前都会调用 [checkConflict()](file:///d:/fz/0601/solo-dogfeeding/code/177-filebrowser/frontend/src/utils/upload.ts#L43-L89)：
```typescript
export async function checkConflict(files, basePath, includeDirectories = false) {
    // 1. 拉取目标目录下所有文件
    serverEntries = await api.fetchAll(basePath);
    
    // 2. 对比本地文件列表和服务端文件列表
    files.forEach((file, index) => {
        const server = serverMap.get(conflictKey(file));
        if (server) {
            conflicts.push({
                index,
                origin: { lastModified: file.file?.lastModified, size: file.size },
                dest: { lastModified: server.modified, size: server.size },
                isSmallerOnServer: file.size > server.size,  // ✅ 关键判断
            });
        }
    });
    return conflicts;
}
```

然后弹出 [ResolveConflict.vue](file:///d:/fz/0601/solo-dogfeeding/code/177-filebrowser/frontend/src/components/prompts/ResolveConflict.vue) 对话框，用户有多种选择。

#### 恢复分支决策树
```
用户重新选择同一文件上传
    │
    ├─ checkConflict() 检测到服务端已有同名文件
    │   │
    │   ├─ 缓存仍有效 (<3分钟)
    │   │   ├─ 用户点击 "Resume Transfer" (续传)
    │   │   │   └─ 分支A: 智能续传 → 见7.3.1
    │   │   ├─ 用户点击 "Override" (覆盖)
    │   │   │   └─ 分支A: 同上 (tus-js-client会自动续传，不会真覆盖)
    │   │   └─ 用户点击 "Skip" (跳过)
    │   │       └─ 分支C: 跳过不上传 → 见7.3.3
    │   │
    │   └─ 缓存已过期 (>3分钟)
    │       ├─ 内存模式: 文件已被删除 → 无冲突 → 走新上传
    │       ├─ Redis模式: 文件仍在
    │       │   ├─ 用户点击 "Resume Transfer"
    │       │   │   └─ 分支B: 覆盖重传 → 见7.3.2
    │       │   └─ 用户点击 "Skip"
    │       │       └─ 分支C: 跳过不上传 → 见7.3.3
    │       └─ (服务端文件 == 客户端文件)
    │           └─ 分支D: 完整文件跳过 → 见7.3.4
    │
    └─ 无冲突 → 正常新上传
```

---

### 7.3.1 分支A：缓存有效 → 智能续传（不覆盖）

**场景**：上传到20MB断网，1分钟后用户刷新页面重新上传。

**恢复流程**：
```
1. 冲突检测:
   isSmallerOnServer = (100MB > 20MB) = true

2. 用户点击 "Resume Transfer" 按钮:
   [resume()](file:///d:/fz/0601/solo-dogfeeding/code/177-filebrowser/frontend/src/components/prompts/ResolveConflict.vue#L207-L216)
   ```typescript
   const resume = (event) => {
       conflict.value.forEach((item) => {
           if (item.isSmallerOnServer) {
               item.checked = ["origin"];  // 标记为"保留源文件"=覆盖
           } else {
               item.checked = ["dest"];    // 标记为"保留目标文件"=跳过
           }
       });
       currentPrompt?.confirm(event, conflict.value);
   };
   ```

3. [Upload.vue confirm回调](file:///d:/fz/0601/solo-dogfeeding/code/177-filebrowser/frontend/src/components/prompts/Upload.vue#L78-L94):
   ```typescript
   if (item.checked.length == 1 && item.checked[0] == "origin") {
       uploadFiles[item.index].overwrite = true;  // 设置overwrite=true
   }
   ```

4. tus-js-client.start() 执行:
   ```
   🔴 关键: tus-js-client 会先 HEAD 查询，POST 只在 HEAD 失败时才发
   
   ├─ HEAD /api/tus/file.zip
   │   ├─ cache.GetLength() → 100MB ✓ (缓存仍有效)
   │   ├─ file.Size → 20MB ✓
   │   └─ 返回 Upload-Offset:20971520, Upload-Length:104857600
   └─ tus-js-client: 识别为可续传 → 跳过 POST
   ```

5. 直接 PATCH 从 offset=20971520 继续上传

**重要结论**：虽然 `overwrite=true`，但 `tusPostHandler` 中的 `O_TRUNC` 代码路径**根本不会执行**，因为 tus-js-client 发现可以续传就跳过了 POST 请求。

---

### 7.3.2 分支B：缓存过期 → 覆盖重传

**场景**：上传到20MB断网，4分钟后（Redis模式）用户重新上传。

**恢复流程**：
```
1. 冲突检测: isSmallerOnServer = true (Redis没删文件)
2. 用户点击 "Resume Transfer" → overwrite = true
3. tus-js-client.start():
   ├─ HEAD /api/tus/file.zip
   │   └─ cache.GetLength() → 缓存已过期 → 404
   └─ tus-js-client: 收到404 → 发起 POST

4. [tusPostHandler](file:///d:/fz/0601/solo-dogfeeding/code/177-filebrowser/http/tus_handlers.go#L70-L86) 执行:
   ```go
   if file != nil {
       if r.URL.Query().Get("override") != "true" {
           return http.StatusConflict, nil  // 没override就返回冲突
       }
       if !d.user.Perm.Modify {
           return http.StatusForbidden, nil
       }
       fileFlags |= os.O_TRUNC  // ✅ 清空现有文件
   }
   ```

5. 文件被清空 → 重新注册缓存 → 从 offset=0 开始上传
```

**为什么不清空不行？**
- 服务端文件 20MB，客户端新上传也是同一文件的前20MB，但服务端不知道这一点
- 如果不清空直接 Append，后续 PATCH 的 offset=0 校验会失败（file.Size=20MB != 0）
- 所以只能清空重传

---

### 7.3.3 分支C：用户选择 Skip

**场景**：检测到冲突后，用户点击 "Skip" 按钮。

**代码流程**：
```typescript
// ResolveConflict.vue confirm 回调
if (item.checked.length == 1 && item.checked[0] == "dest") {
    uploadFiles.splice(item.index, 1);  // 从上传列表中删除
}

// 后续
if (uploadFiles.length > 0) {
    upload.handleFiles(uploadFiles, path);  // 只上传剩余文件
}
```

---

### 7.3.4 分支D：服务端文件已完整 → 自动跳过

**场景**：用户上次上传已完整完成（100MB），现在又选了同一个文件上传。

**代码逻辑**：
```typescript
// checkConflict() 中计算
isSmallerOnServer: file.size > server.size  // 100MB > 100MB = false

// resume() 函数中
if (item.isSmallerOnServer) {
    item.checked = ["origin"];  // 覆盖
} else {
    item.checked = ["dest"];    // ✅ 服务端文件不小 → 跳过
}
```

> 这是一个重要的防重复上传逻辑：如果服务端文件大小 >= 本地文件，默认认为服务端的文件是完整的，自动跳过。

---

## 八、传统上传方式（非 TUS 回退）

当 TUS 不可用时（如非 HTTP 协议、浏览器不支持），使用 [postResources](file:///d:/fz/0601/solo-dogfeeding/code/177-filebrowser/frontend/src/api/files.ts#L126-L172) 走传统 XMLHttpRequest：

- 前端：使用 XHR `send()` 一次性发送整个 Blob
- 后端：使用 [resourcePostHandler](file:///d:/fz/0601/solo-dogfeeding/code/177-filebrowser/http/resource.go#L125-L178) 的 `writeFile()` 一次性写入

这种方式**不支持断点续传**，适合小文件或非 HTTP 环境。

---

## 十、前端进度显示与交互

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

## 十一、总结：设计亮点与注意事项

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
1. **默认不支持跨页面续传**：`storeFingerprintForResuming` 设为 `false`，刷新页面后需重新选择文件
2. **TTL 严格限制**：上传中断超过 3 分钟未恢复，缓存过期
3. **单片串行**：`parallelUploads: 1`，同一文件的多个分片按顺序传输，不能利用多连接加速
4. **10MB 分片大小**：小文件（<10MB）会退化为单请求上传，大文件分片数 = 文件大小 / 10MB

#### 边界场景注意事项
5. **内存 vs Redis 缓存行为不一致**：
   - 内存缓存：过期时通过 `OnEviction` 回调自动删除磁盘文件
   - Redis 缓存：过期后**不会**自动删文件，会残留半完成文件，需额外清理机制
6. **Resume 按钮不总是续传**：
   - 缓存有效（<3分钟）→ 智能续传，从断点继续
   - 缓存过期（>3分钟）→ 实际是覆盖重传，从 0 开始
7. **overwrite=true 不代表一定会覆盖**：
   如果 tus-js-client 通过 HEAD 查询发现可以续传，会跳过 POST 请求，`O_TRUNC` 清空代码路径不会执行
8. **多实例部署强制要求**：
   - 必须使用共享文件系统（NFS、分布式存储等）
   - 所有实例必须连接同一个 Redis 集群
9. **防重复上传逻辑**：
   当服务端文件大小 >= 本地文件大小时，`resume()` 会自动选择跳过，认为服务端文件是完整的
