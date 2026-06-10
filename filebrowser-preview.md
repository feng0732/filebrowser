# File Browser 缩略图与预览生成实现梳理

## 一、整体架构概览

File Browser 的缩略图与预览系统采用 **前后端分离** 架构，包含以下核心模块：

| 层级 | 模块 | 职责 |
|------|------|------|
| 前端 | `frontend/src/components/files/ListingItem.vue` | 列表缩略图展示（懒加载） |
| 前端 | `frontend/src/views/files/Preview.vue` | 文件预览页面 |
| 前端 | `frontend/src/api/files.ts` | 构建预览/下载 URL |
| 后端路由 | `http/http.go` | 注册 `/api/preview/{size}/{path}` 路由 |
| 后端处理 | `http/preview.go` | 处理预览请求、缓存读写 |
| 图像处理 | `img/service.go` | 图片解码、缩放、编码 |
| 缓存层 | `diskcache/file_cache.go` | 基于文件系统的磁盘缓存 |
| 缓存失效 | `http/resource.go` | 文件变更时清理缓存 |

---

## 二、后端资源生成

### 2.1 路由注册

在 `http/http.go#L82-L84` 中注册预览 API 路由：

```go
api.PathPrefix("/preview/{size}/{path:.*}").
    Handler(monkey(previewHandler(imgSvc, fileCache, server.EnableThumbnails, server.ResizePreview), "/api/preview")).Methods("GET")
```

**路由参数说明：**
- `{size}`: 预览尺寸，可选值 `thumb`（缩略图）或 `big`（大预览图）
- `{path}`: 文件的虚拟路径

### 2.2 预览请求入口 `previewHandler`

`http/preview.go#L37-L70`

**处理流程：**
1. 权限校验：检查用户 `Download` 权限
2. 解析 `size` 参数为 `PreviewSize` 枚举
3. 创建 `FileInfo` 获取文件元信息（路径、修改时间、扩展名、类型等）
4. 设置 `Content-Disposition` 响应头（基于 `inline` 查询参数）
5. 根据文件类型分发处理（目前仅支持 `image` 类型）

### 2.3 图片预览决策 `handleImagePreview`

`http/preview.go#L72-L111`

**核心决策逻辑：**

| 条件 | 处理方式 |
|------|----------|
| `big` 尺寸且 `resizePreview=false` | 降级：调用 `rawFileHandler` 返回原始文件 |
| `thumb` 尺寸且 `enableThumbnails=false` | 降级：调用 `rawFileHandler` 返回原始文件 |
| 不支持的图片格式 | 降级：调用 `rawFileHandler` 返回原始文件 |
| GIF 格式 | 降级：避免缩放丢失动画，返回原始文件 |
| 缓存命中 | 直接从缓存读取字节并返回 |
| 缓存未命中 | 调用 `createPreview()` 生成新预览 |

**回传核心代码：**
```go
w.Header().Set("Cache-Control", "private")
http.ServeContent(w, r, file.Name, file.ModTime, bytes.NewReader(resizedImage))
```

### 2.4 预览图生成 `createPreview`

`http/preview.go#L113-L153`

**两种预设尺寸配置：**

| 尺寸 | 宽高 | 缩放模式 | 质量 | 输出格式 |
|------|------|----------|------|----------|
| `thumb` | 256x256 | `Fill`（居中裁剪填充） | `Low` | 强制 `JPEG` |
| `big` | 1080x1080 | `Fit`（等比缩放适应） | `Medium` | 保持原格式 |

**异步缓存写入策略：**
生成完成后，先同步返回结果，再通过 goroutine 异步写入缓存，避免阻塞响应：
```go
go func() {
    cacheKey := previewCacheKey(file, previewSize)
    if err := fileCache.Store(context.Background(), cacheKey, buf.Bytes()); err != nil {
        fmt.Printf("failed to cache resized image: %v", err)
    }
}()
```

### 2.5 图片处理服务 `img.Service`

`img/service.go`

#### 2.5.1 并发控制（信号量限流）
使用信号量限制同时处理的图片数量，防止服务器内存过载：
```go
func New(workers int) *Service {
    return &Service{ sem: semaphore.New(workers) }
}
```
默认 worker 数量为 `4`（可通过 `--imageProcessors` 配置）。

#### 2.5.2 格式检测与超大图防护
`detectFormat` 方法：
- 使用 `image.DecodeConfig` 仅读取文件头解析配置信息（不解码全部像素）
- **最大尺寸限制**：宽高均不超过 10000px，防止超大图片导致 OOM
- 支持格式：JPEG、PNG、GIF、TIFF、BMP

#### 2.5.3 EXIF 内嵌缩略图优化
对于 JPEG 格式 + 低质量（`QualityLow`）场景，优先尝试提取 EXIF 内嵌缩略图，跳过完整解码流程：
```go
if config.quality == QualityLow && format == FormatJpeg {
    thm, newWrappedReader, errThm := getEmbeddedThumbnail(wrappedReader)
    if errThm == nil {
        out.Write(thm)
        return nil  // 直接使用内嵌缩略图
    }
}
```

#### 2.5.4 缩放算法与质量等级

| 质量等级 | 重采样滤波器 | 适用场景 |
|----------|-------------|----------|
| `High` | Lanczos | 高质量但速度慢 |
| `Medium` | Box | 平衡性能与质量（big 默认） |
| `Low` | NearestNeighbor | 最快（thumb 默认） |

| 缩放模式 | 行为 |
|----------|------|
| `Fit` | 等比缩放，完整显示图片在目标尺寸内 |
| `Fill` | 居中裁剪后填充目标尺寸 |

#### 2.5.5 EXIF 自动方向校正
解码时启用 `imaging.AutoOrientation(true)`，根据 EXIF 方向信息自动旋转图片。

---

## 三、缓存机制

### 3.1 缓存接口定义

`diskcache/cache.go`

```go
type Interface interface {
    Store(ctx context.Context, key string, value []byte) error
    Load(ctx context.Context, key string) (value []byte, exist bool, err error)
    Delete(ctx context.Context, key string) error
}
```

### 3.2 缓存键生成策略

`http/preview.go#L155-L157`

```go
func previewCacheKey(f *files.FileInfo, previewSize PreviewSize) string {
    return fmt.Sprintf("%x%x%x", f.RealPath(), f.ModTime.Unix(), previewSize)
}
```

**缓存键的三个组成部分：**
1. **文件真实物理路径** (`RealPath()`) - 避免虚拟路径 / 不同用户冲突
2. **文件修改时间戳** (`ModTime.Unix()`) - 文件变更后自动失效
3. **预览尺寸枚举值** (`previewSize`) - thumb 和 big 分别独立缓存

> **设计优势**：无需额外的 TTL 过期机制，文件修改即自动生成新缓存键，老缓存自然过期（不会自动清理，但也不会被命中）。

### 3.3 文件缓存实现 `FileCache`

`diskcache/file_cache.go`

#### 3.3.1 二级哈希目录存储
对缓存键进行 SHA1 哈希，采用两级目录分散存储，避免单目录文件数过多导致的性能问题：
```go
func (f *FileCache) getFileName(key string) string {
    hasher := sha1.New()
    hasher.Write([]byte(key))
    hash := hex.EncodeToString(hasher.Sum(nil))
    return fmt.Sprintf("%s/%s/%s", hash[:1], hash[1:3], hash)
    // 例如: a/bc/abcdef123456...
}
```

#### 3.3.2 细粒度并发锁
每个缓存键对应独立的互斥锁，避免全局锁竞争：
```go
func (f *FileCache) getScopedLocks(key string) sync.Locker {
    // 懒初始化锁映射表（sync.Once）
    // 每个 key 对应独立 Mutex
}
```
Store 和 Delete 操作会获取对应 key 的锁，Load 操作不加锁（读操作可并发）。

#### 3.3.3 权限控制
- 缓存目录权限：`0700`
- 缓存文件权限：`0700`
- 仅当前进程用户可读写，保障数据安全

### 3.4 空缓存实现 `NoOp`

`diskcache/noop_cache.go`

当未配置 `cacheDir` 时使用，所有操作均为 no-op（空操作）。此时每次预览请求都会实时处理图片，没有缓存加速。

### 3.5 缓存初始化流程

`cmd/root.go#L172-L179`

```go
var fileCache diskcache.Interface = diskcache.NewNoOp()  // 默认无缓存
cacheDir := v.GetString("cacheDir")
if cacheDir != "" {
    os.MkdirAll(cacheDir, 0700)
    fileCache = diskcache.New(afero.NewOsFs(), cacheDir)
}
```

**配置方式：**
- 命令行参数：`--cacheDir <path>`
- 环境变量：`FB_CACHE_DIR`
- 默认值：空字符串（禁用磁盘缓存）

### 3.6 缓存失效机制

`http/resource.go#L329-L338`

`delThumbs` 函数遍历所有预览尺寸并删除对应缓存：
```go
func delThumbs(ctx context.Context, fileCache FileCache, file *files.FileInfo) error {
    for _, previewSizeName := range PreviewSizeNames() {
        size, _ := ParsePreviewSize(previewSizeName)
        fileCache.Delete(ctx, previewCacheKey(file, size))
    }
    return nil
}
```

**缓存失效的三个触发场景：**

| 操作 | 触发位置 | 说明 |
|------|----------|------|
| **删除文件** | `resourceDeleteHandler` (`http/resource.go#L108`) | 删除文件前清理所有尺寸的缓存 |
| **覆盖上传** | `resourcePostHandler` (`http/resource.go#L155`) | 覆盖已有文件前清理旧缓存 |
| **重命名/移动** | `patchAction -> rename 分支` (`http/resource.go#L368`) | 重命名前清理源文件缓存 |

> **注意**：通过 PUT / PATCH 直接修改文件内容时，由于 `ModTime` 会变化，缓存键会自然变化，旧缓存不会被命中但也不会被主动删除。

---

## 四、资源回传机制（已核实）

### 4.1 HTTP 响应头总览

经过代码核实，预览接口的响应头如下：

| Header | 值 | 设置位置 | 说明 |
|--------|-----|----------|------|
| `Cache-Control` | `private` | `http/preview.go#L107` | 禁止 CDN / 代理缓存，仅允许客户端缓存 |
| `Content-Disposition` | `inline; ...` | `http/raw.go#L75` | 由 `setContentDisposition` 根据 `inline` 参数设置 |
| `Last-Modified` | 自动 | Go 标准库 `http.ServeContent` | 基于 `file.ModTime` 自动生成 |
| `Content-Type` | 自动 | Go 标准库 `http.ServeContent` | 基于文件扩展名 / 内容嗅探 |
| `Accept-Ranges` | `bytes` | Go 标准库 `http.ServeContent` | 自动宣告支持 Range 请求 |
| **ETag** | **未设置** | - | 代码中未显式设置 ETag |

> **重要核实结论**：`/api/preview` 接口 **没有** 设置 ETag 响应头。ETag 仅在文件上传 / 保存的 API 响应中设置（`http/resource.go#L167-L168`），用于资源更新的并发控制，与预览缓存无关。

### 4.2 Content-Disposition 处理

`http/raw.go#L72-L81`

```go
func setContentDisposition(w http.ResponseWriter, r *http.Request, file *files.FileInfo) {
    if r.URL.Query().Get("inline") == "true" {
        w.Header().Set("Content-Disposition", "inline; filename*=utf-8''"+url.PathEscape(file.Name))
    } else {
        w.Header().Set("Content-Disposition", "attachment; filename*=utf-8''"+url.PathEscape(file.Name))
        w.Header().Set("Content-Type", "application/octet-stream")
    }
}
```

预览请求始终携带 `inline=true` 参数，浏览器会内联展示而非触发下载。

### 4.3 条件请求（Last-Modified / 304）

**已核实**：条件请求能力由 Go 标准库 `http.ServeContent` 自动提供。

**工作流程：**
1. 首次请求时，`ServeContent` 自动设置 `Last-Modified` 响应头（值为 `file.ModTime`）
2. 浏览器缓存后，后续请求携带 `If-Modified-Since` 请求头
3. `ServeContent` 比较时间戳，若未修改则返回 `304 Not Modified`（空 body）
4. 若已修改则返回 `200 OK` + 完整内容 + 新的 `Last-Modified`

**与缓存键的协同关系：**
- 服务端磁盘缓存的键包含 `ModTime`，文件修改后缓存键变化
- 浏览器端的 `If-Modified-Since` 基于 `Last-Modified`，同样在文件修改后失效
- 两层缓存机制互补，进一步减少重复计算和传输

### 4.4 Range 请求（断点续传）

**已核实**：`http.ServeContent` 内置对 `Range` 请求头的支持。

**支持的功能：**
- 单范围 `Range` 请求（`Range: bytes=100-200`）
- `If-Range` 条件（基于 `Last-Modified`，因为没有 ETag）
- `206 Partial Content` 响应
- `Accept-Ranges: bytes` 响应头

**数据来源：**
- 缓存命中时：`bytes.NewReader(resizedImage)` 内存 reader，支持任意 seek
- 降级 raw 时：`fd` 文件句柄，支持任意 seek

### 4.5 rawFileHandler 与 preview 的区别

| 对比项 | `previewHandler` | `rawFileHandler` |
|--------|-----------------|------------------|
| 路径 | `/api/preview/{size}/{path}` | `/api/raw/{path}` |
| Cache-Control | `private` | `private` |
| Content-Security-Policy | 未设置 | `script-src 'none';` |
| X-Content-Type-Options | 未设置 | `nosniff` |
| 内容来源 | 缩放后的图片 / 缓存 / 原始文件 | 原始文件 |
| 适用场景 | 缩略图、图片预览 | 下载、原始文件查看 |

---

## 五、前端实现

### 5.1 预览 URL 构建

`frontend/src/api/files.ts#L219-L226`

```typescript
export function getPreviewURL(file: ResourceItem, size: string) {
  const params = {
    inline: "true",
    key: Date.parse(file.modified),  // 客户端缓存 buster
  };
  return createURL("api/preview/" + size + file.path, params);
}
```

**URL 参数说明：**
- `inline=true`: 触发内联展示模式
- `key=<timestamp>`: 基于文件修改时间戳的缓存 buster
  - 作用：精确控制浏览器缓存
  - 原理：文件修改后 `key` 变化，URL 变化，浏览器视为新资源
  - 与 `Last-Modified` 形成双重保险

### 5.2 列表缩略图

`frontend/src/components/files/ListingItem.vue#L26-L31`

```html
<img
  v-if="!readOnly && type === 'image' && isThumbsEnabled"
  v-lazy="thumbnailUrl"
/>
```

**关键特性：**
- 条件渲染三要素：非只读模式 + 图片类型 + 缩略图功能启用
- 使用 `v-lazy` 指令实现懒加载：滚动到视口才发起请求
- 使用 `thumb` 尺寸（256x256），加载速度快、省流量
- `isThumbsEnabled` 来自 `window.FileBrowser.EnableThumbs` 全局配置

### 5.3 预览页面

`frontend/src/views/files/Preview.vue#L293-L307`

```typescript
const previewUrl = computed(() => {
  if (fileStore.req.type === "image" && !fullSize.value) {
    return api.getPreviewURL(fileStore.req, "big");  // 默认 big 预览
  }
  // 其他类型 或 fullSize 模式，使用原始文件
  return api.getDownloadURL(fileStore.req, true);
});
```

**预览策略：**

| 文件类型 | 默认模式 | 全屏模式 |
|----------|----------|----------|
| 图片 | `big` 尺寸预览（1080p） | 原始文件 |
| 视频 | 原始文件流 | 原始文件流 |
| 音频 | 原始文件流 | 原始文件流 |
| PDF | 原始文件（object 标签） | 原始文件 |
| EPUB | 原始文件（vue-reader 解析） | 原始文件 |
| CSV | 文本内容（前端解析） | 文本内容 |
| 其他 blob | 无预览，显示下载按钮 | - |

**增强体验特性：**
- 前后文件预加载（`<link rel="prefetch">`）
- 键盘左右键切换文件
- 图片预览支持 `fullSize` 切换（原始图 vs 缩放预览）

### 5.4 增强图片查看器

`frontend/src/components/files/ExtendedImage.vue`

用于图片预览的交互式组件，支持：
- 鼠标拖拽平移
- 滚轮缩放（0.25x ~ 4x+）
- 触屏双指缩放
- 双击切换缩放级别（1x → 2x → 4x → 1x）
- TIFF / DNG / CR2 / NEF 等 RAW 格式前端解码（UTIF 库）

### 5.5 配置注入机制

`frontend/src/utils/constants.ts#L16-L17`

```typescript
const enableThumbs: boolean = window.FileBrowser.EnableThumbs;
const resizePreview: boolean = window.FileBrowser.ResizePreview;
```

前端通过服务端注入的全局变量 `window.FileBrowser` 获取配置，无需额外 API 请求。

---

## 六、配置开关

### 6.1 服务端配置

`settings/settings.go#L49-L65`

| 配置项 | 类型 | 命令行参数 | 环境变量 | 默认值 | 说明 |
|--------|------|-----------|----------|--------|------|
| `EnableThumbnails` | bool | `--disableThumbnails` | `FB_DISABLE_THUMBNAILS` | `true` | 是否启用缩略图生成 |
| `ResizePreview` | bool | `--disablePreviewResize` | `FB_DISABLE_PREVIEW_RESIZE` | `true` | 是否启用大图预览缩放 |
| `imageProcessors` | int | `--imageProcessors` | `FB_IMAGE_PROCESSORS` | `4` | 并发图片处理 worker 数 |
| `cacheDir` | string | `--cacheDir` | `FB_CACHE_DIR` | `""` | 缓存目录（空则禁用磁盘缓存） |

配置优先级：命令行参数 > 环境变量 > 配置文件 > 数据库值 > 默认值。

### 6.2 前端配置

通过 `window.FileBrowser` 全局对象注入，对应服务端配置：

| 前端变量 | 对应服务端配置 |
|----------|---------------|
| `enableThumbs` | `server.EnableThumbnails` |
| `resizePreview` | `server.ResizePreview` |

---

## 七、完整调用链路图

```
前端列表 ListingItem.vue
  │
  ├─ 条件判断：type==='image' && enableThumbs
  └─ v-lazy 懒加载触发
      │
      ▼
  <img src="/api/preview/thumb/path/to/img.jpg?inline=true&key=1234567890">
      │
      ▼
================================================ 后端
  http/http.go 路由匹配
      │
      ▼
  http/preview.go previewHandler
      ├─ 权限校验（Perm.Download）
      ├─ 解析 size 参数（thumb / big）
      ├─ NewFileInfo 获取文件元信息
      ├─ setContentDisposition(inline=true)
      └─ handleImagePreview
          │
          ├─ [开关检查] 功能未启用？
          │     └─ 是 → rawFileHandler 返回原始文件
          │
          ├─ [格式检查] 不支持 / GIF？
          │     └─ 是 → rawFileHandler 返回原始文件
          │
          ├─ 生成缓存键：RealPath + ModTime + Size
          │
          ├─ fileCache.Load(cacheKey)
          │     ├─ 命中 → 读取缓存字节
          │     └─ 未命中 → createPreview
          │           │
          │           ├─ imgSvc.Resize
          │           │     ├─ 信号量 Acquire（并发控制）
          │           │     ├─ detectFormat（尺寸 < 10000x10000）
          │           │     ├─ [优化] JPEG+Low → 提取 EXIF 缩略图
          │           │     ├─ imaging.Decode（AutoOrientation）
          │           │     ├─ 缩放：Fit / Fill
          │           │     └─ imaging.Encode
          │           ├─ 同步返回结果字节
          │           └─ [异步 goroutine] fileCache.Store 写入缓存
          │
          ├─ 设置 Cache-Control: private
          └─ http.ServeContent
                ├─ 自动设置 Last-Modified
                ├─ 自动设置 Accept-Ranges: bytes
                ├─ 自动处理 If-Modified-Since → 304
                ├─ 自动处理 Range → 206 Partial Content
                └─ 自动推断 Content-Type
================================================
      │
      ▼
  浏览器缓存（HTTP 缓存 + key 参数控制）
```

---

## 八、关键优化点总结

### 服务端优化

1. **ModTime 天然缓存失效**：缓存键包含文件修改时间，无需额外 TTL 机制
2. **异步缓存写入**：先返回结果，后台异步落盘缓存，降低首响应延迟
3. **EXIF 缩略图提取**：缩略图场景优先利用 JPEG 内嵌数据，跳过完整解码
4. **二级哈希目录**：避免单目录文件数过多导致的文件系统性能问题
5. **细粒度锁**：每个缓存键独立锁，减少并发冲突
6. **信号量限流**：限制同时处理的图片数量，防止内存暴增
7. **超大图防护**：检测超过 10000x10000 的图片直接拒绝处理
8. **降级策略**：功能关闭或格式不支持时，无缝退回原始文件

### 前端优化

9. **懒加载**：列表缩略图使用 `v-lazy`，仅在视口内加载，节省带宽
10. **客户端缓存 buster**：URL 携带 `key` 参数，精确控制浏览器缓存时效
11. **尺寸分级**：列表用 thumb（256px）、预览用 big（1080px）、原图按需切换
12. **预加载**：预览页预加载前后文件，提升切换流畅度

### 协议层优化

13. **Last-Modified / 304**：利用标准 HTTP 缓存机制减少重复传输
14. **Range 请求支持**：大文件可断点续传 / 分段加载
15. **Cache-Control: private**：保障隐私，防止共享缓存泄露
