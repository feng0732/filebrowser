# File Browser 缩略图与预览生成实现梳理

## 一、整体架构概览

File Browser 的缩略图与预览系统采用 **前后端分离** 架构，包含以下核心模块：

| 层级 | 模块 | 职责 |
|------|------|------|
| 前端 | ListingItem.vue / Preview.vue | 发起预览请求、展示缩略图和预览 |
| 前端 | api/files.ts | 构建预览 URL |
| 后端路由 | http/http.go | 注册 `/api/preview/{size}/{path}` 路由 |
| 后端处理 | http/preview.go | 处理预览请求、缓存读写 |
| 图像处理 | img/service.go | 图片解码、缩放、编码 |
| 缓存层 | diskcache/file_cache.go | 基于文件系统的磁盘缓存 |
| 缓存失效 | http/resource.go | 文件变更时清理缓存 |

---

## 二、后端实现

### 2.1 路由注册

在 [http.go](file:///d:/fz/0601/solo-dogfeeding/code/179-filebrowser/http/http.go#L82-L84) 中注册预览 API 路由：

```go
api.PathPrefix("/preview/{size}/{path:.*}").
    Handler(monkey(previewHandler(imgSvc, fileCache, server.EnableThumbnails, server.ResizePreview), "/api/preview")).Methods("GET")
```

**路由参数说明：**
- `{size}`: 预览尺寸，可选值 `thumb`（缩略图）或 `big`（大预览图）
- `{path}`: 文件的虚拟路径

### 2.2 预览请求入口 `previewHandler`

[preview.go](file:///d:/fz/0601/solo-dogfeeding/code/179-filebrowser/http/preview.go#L37-L70)

**处理流程：**
1. 权限校验：检查用户 `Download` 权限
2. 解析 `size` 参数为 `PreviewSize` 枚举
3. 创建 `FileInfo` 获取文件元信息
4. 设置 `Content-Disposition` 响应头
5. 根据文件类型分发处理（仅支持 `image` 类型）

### 2.3 图片预览处理 `handleImagePreview`

[preview.go](file:///d:/fz/0601/solo-dogfeeding/code/179-filebrowser/http/preview.go#L72-L111)

**核心决策逻辑：**

| 条件 | 处理方式 |
|------|----------|
| `big` 尺寸且 `resizePreview=false` | 直接返回原始文件 |
| `thumb` 尺寸且 `enableThumbnails=false` | 直接返回原始文件 |
| 不支持的格式 或 GIF 格式 | 直接返回原始文件 |
| 缓存命中 | 从缓存读取并返回 |
| 缓存未命中 | 调用 `createPreview()` 生成新预览 |

**回传响应头：**
```go
w.Header().Set("Cache-Control", "private")
http.ServeContent(w, r, file.Name, file.ModTime, bytes.NewReader(resizedImage))
```
使用 `http.ServeContent` 自动处理 `Range` 请求和 `Last-Modified` / `ETag`。

### 2.4 预览图创建 `createPreview`

[preview.go](file:///d:/fz/0601/solo-dogfeeding/code/179-filebrowser/http/preview.go#L113-L153)

**两种预设尺寸配置：**

| 尺寸 | 宽高 | 缩放模式 | 质量 | 输出格式 |
|------|------|----------|------|----------|
| `thumb` | 256x256 | `Fill`（居中裁剪填充） | `Low` | `JPEG` |
| `big` | 1080x1080 | `Fit`（等比缩放适应） | `Medium` | 保持原格式 |

**异步缓存写入：**
生成完成后，通过 goroutine 异步写入缓存，避免阻塞响应：
```go
go func() {
    cacheKey := previewCacheKey(file, previewSize)
    if err := fileCache.Store(context.Background(), cacheKey, buf.Bytes()); err != nil {
        fmt.Printf("failed to cache resized image: %v", err)
    }
}()
```

### 2.5 图片处理服务 `img.Service`

[service.go](file:///d:/fz/0601/solo-dogfeeding/code/179-filebrowser/img/service.go)

#### 2.5.1 并发控制
使用信号量限制同时处理的图片数量，防止服务器过载：
```go
func New(workers int) *Service {
    return &Service{ sem: semaphore.New(workers) }
}
```
默认 worker 数量为 `4`（可通过 `--imageProcessors` 配置）。

#### 2.5.2 格式检测与尺寸限制
`detectFormat` 方法：
- 先读取文件头解析配置信息（不解码全部像素）
- **最大尺寸限制**：宽高均不超过 10000px，防止超大图片导致 OOM
- 支持格式：JPEG、PNG、GIF、TIFF、BMP

#### 2.5.3 EXIF 内嵌缩略图优化
对于 JPEG 格式 + 低质量（`QualityLow`）场景，优先尝试提取 EXIF 内嵌缩略图，跳过完整解码流程：
```go
if config.quality == QualityLow && format == FormatJpeg {
    thm, newWrappedReader, errThm := getEmbeddedThumbnail(wrappedReader)
    if errThm == nil {
        _, err = out.Write(thm)
        return nil  // 直接使用内嵌缩略图
    }
}
```

#### 2.5.4 缩放算法与质量等级

| 质量等级 | 重采样滤波器 | 适用场景 |
|----------|-------------|----------|
| `High` | Lanczos | 高质量但慢 |
| `Medium` | Box | 平衡性能与质量（big 默认） |
| `Low` | NearestNeighbor | 最快（thumb 默认） |

| 缩放模式 | 行为 |
|----------|------|
| `Fit` | 等比缩放，完整显示图片在目标尺寸内 |
| `Fill` | 居中裁剪后填充目标尺寸 |

---

## 三、缓存机制

### 3.1 缓存接口定义

[cache.go](file:///d:/fz/0601/solo-dogfeeding/code/179-filebrowser/diskcache/cache.go)

```go
type Interface interface {
    Store(ctx context.Context, key string, value []byte) error
    Load(ctx context.Context, key string) (value []byte, exist bool, err error)
    Delete(ctx context.Context, key string) error
}
```

### 3.2 缓存键生成策略

[preview.go](file:///d:/fz/0601/solo-dogfeeding/code/179-filebrowser/http/preview.go#L155-L157)

```go
func previewCacheKey(f *files.FileInfo, previewSize PreviewSize) string {
    return fmt.Sprintf("%x%x%x", f.RealPath(), f.ModTime.Unix(), previewSize)
}
```

**键的组成部分：**
1. 文件真实物理路径（避免虚拟路径冲突）
2. 文件修改时间戳（文件变更后自动失效）
3. 预览尺寸枚举值（thumb/big 分别缓存）

> **设计优势**：无需额外的缓存过期策略，文件修改即自动生成新缓存键。

### 3.3 文件缓存实现 `FileCache`

[file_cache.go](file:///d:/fz/0601/solo-dogfeeding/code/179-filebrowser/diskcache/file_cache.go)

#### 3.3.1 目录分层存储
对缓存键进行 SHA1 哈希，采用二级目录分散存储，避免单目录文件过多：
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
    // 懒初始化锁映射表，每个 key 独立 Mutex
}
```

#### 3.3.3 权限控制
缓存目录权限为 `0700`，文件权限为 `0700`，仅当前用户可读写。

### 3.4 空缓存实现 `NoOp`

[noop_cache.go](file:///d:/fz/0601/solo-dogfeeding/code/179-filebrowser/diskcache/noop_cache.go)

当未配置 `cacheDir` 时使用，所有操作均为 no-op，此时每次预览请求都会重新处理图片。

### 3.5 缓存初始化

[root.go](file:///d:/fz/0601/solo-dogfeeding/code/179-filebrowser/cmd/root.go#L172-L179)

```go
var fileCache diskcache.Interface = diskcache.NewNoOp()  // 默认无缓存
cacheDir := v.GetString("cacheDir")
if cacheDir != "" {
    os.MkdirAll(cacheDir, 0700)
    fileCache = diskcache.New(afero.NewOsFs(), cacheDir)
}
```
通过 `--cacheDir` 命令行参数或 `FB_CACHE_DIR` 环境变量启用磁盘缓存。

### 3.6 缓存失效机制

[resource.go](file:///d:/fz/0601/solo-dogfeeding/code/179-filebrowser/http/resource.go#L329-L338)

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

**缓存失效触发场景：**

| 操作 | 触发位置 | 说明 |
|------|----------|------|
| 删除文件 | `resourceDeleteHandler` | 删除文件前清理缓存 |
| 覆盖上传 | `resourcePostHandler` | 覆盖已有文件前清理缓存 |
| 重命名/移动 | `patchAction -> rename 分支` | 重命名前清理源文件缓存 |

> **注意**：文件内容通过 `PUT` / `PATCH` 直接修改时，由于 ModTime 变化，缓存键会自然变化，无需主动清理。

---

## 四、资源回传机制

### 4.1 HTTP 响应头设置

[raw.go](file:///d:/fz/0601/solo-dogfeeding/code/179-filebrowser/http/raw.go#L72-L81)

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

预览请求始终带 `inline=true` 参数，以 `inline` 方式内联展示。

### 4.2 预览响应特有头

| Header | 值 | 作用 |
|--------|-----|------|
| `Cache-Control` | `private` | 禁止 CDN/代理缓存，仅客户端缓存 |
| `Content-Security-Policy` | `script-src 'none';` | 禁止资源中执行脚本（仅 raw handler） |
| `X-Content-Type-Options` | `nosniff` | 禁止 MIME 嗅探（仅 raw handler） |

### 4.3 Range 请求支持

通过 `http.ServeContent` 自动实现：
- 支持 `Range` / `If-Range` 头
- 返回 `206 Partial Content`
- 支持断点续传（对大尺寸预览图友好）

---

## 五、前端实现

### 5.1 预览 URL 构建

[files.ts](file:///d:/fz/0601/solo-dogfeeding/code/179-filebrowser/frontend/src/api/files.ts#L219-L226)

```typescript
export function getPreviewURL(file: ResourceItem, size: string) {
  const params = {
    inline: "true",
    key: Date.parse(file.modified),  // 客户端缓存 buster
  };
  return createURL("api/preview/" + size + file.path, params);
}
```

**URL 参数：**
- `inline=true`: 以 inline 模式响应
- `key=<timestamp>`: 基于文件修改时间的缓存 buster，确保文件更新后浏览器重新请求

### 5.2 列表缩略图

[ListingItem.vue](file:///d:/fz/0601/solo-dogfeeding/code/179-filebrowser/frontend/src/components/files/ListingItem.vue#L26-L31)

```html
<img
  v-if="!readOnly && type === 'image' && isThumbsEnabled"
  v-lazy="thumbnailUrl"
/>
```

**关键点：**
- 条件渲染：仅图片类型 + 非只读模式 + 缩略图启用时显示
- 使用 `v-lazy` 指令实现懒加载（滚动到视口才加载）
- 使用 `thumb` 尺寸

### 5.3 预览页面

[Preview.vue](file:///d:/fz/0601/solo-dogfeeding/code/179-filebrowser/frontend/src/views/files/Preview.vue#L293-L307)

```typescript
const previewUrl = computed(() => {
  if (fileStore.req.type === "image" && !fullSize.value) {
    return api.getPreviewURL(fileStore.req, "big");  // 默认使用 big 预览
  }
  // 其他类型或 fullSize 模式直接使用原始文件
  return api.getDownloadURL(fileStore.req, true);
});
```

**功能特性：**
- 默认使用 `big` 尺寸（1080p）预览
- 支持 `fullSize` 切换，直接加载原始文件
- 图片、音频、视频、PDF、EPUB、CSV 等类型的差异化渲染
- 前后文件预加载（`link rel="prefetch"`）

### 5.4 增强图片查看器

[ExtendedImage.vue](file:///d:/fz/0601/solo-dogfeeding/code/179-filebrowser/frontend/src/components/files/ExtendedImage.vue)

用于图片预览的交互组件，支持：
- 鼠标拖拽平移
- 滚轮缩放
- 触屏双指缩放
- 双击切换缩放级别
- TIFF/DNG/CR2/NEF 等 RAW 格式前端解码（UTIF 库）

---

## 六、配置开关

### 6.1 服务端配置

[settings.go](file:///d:/fz/0601/solo-dogfeeding/code/179-filebrowser/settings/settings.go#L49-L65)

| 配置项 | 类型 | 命令行参数 | 默认值 | 说明 |
|--------|------|-----------|--------|------|
| `EnableThumbnails` | bool | `--disableThumbnails` | `true` | 是否启用缩略图生成 |
| `ResizePreview` | bool | `--disablePreviewResize` | `true` | 是否启用大图预览缩放 |
| `imageProcessors` | int | `--imageProcessors` | `4` | 并发图片处理 worker 数 |
| `cacheDir` | string | `--cacheDir` | `""` | 缓存目录（空则禁用缓存） |

### 6.2 前端配置注入

[constants.ts](file:///d:/fz/0601/solo-dogfeeding/code/179-filebrowser/frontend/src/utils/constants.ts#L16-L17)

```typescript
const enableThumbs: boolean = window.FileBrowser.EnableThumbs;
const resizePreview: boolean = window.FileBrowser.ResizePreview;
```

前端通过全局变量 `window.FileBrowser` 获取服务端注入的配置。

---

## 七、完整调用链路图

```
前端列表 ListingItem.vue
  └─> 懒加载 <img src="api/preview/thumb/path/to/img.jpg?inline=true&key=xxx">
        └─> http/preview.go previewHandler
              ├─> 权限校验
              ├─> 解析尺寸参数
              ├─> handleImagePreview
              │     ├─> [开关检查] 未启用? → rawFileHandler 返回原图
              │     ├─> [格式检查] 不支持/GIF? → rawFileHandler 返回原图
              │     ├─> fileCache.Load(cacheKey)
              │     │     ├─> 命中 → 直接返回缓存字节
              │     │     └─> 未命中 → createPreview
              │     │           ├─> imgSvc.Resize
              │     │           │     ├─> 信号量获取
              │     │           │     ├─> detectFormat (尺寸<10000x10000)
              │     │           │     ├─> [优化] JPEG+Low? 尝试提取 EXIF 缩略图
              │     │           │     ├─> imaging.Decode (AutoOrientation)
              │     │           │     ├─> imaging.Fit / Fill 缩放
              │     │           │     └─> imaging.Encode 编码
              │     │           ├─> 同步返回结果
              │     │           └─> [异步 goroutine] fileCache.Store 写入缓存
              │     └─> http.ServeContent (支持 Range)
              └─> HTTP 响应 (Cache-Control: private)
```

---

## 八、关键优化点总结

1. **ModTime 天然缓存失效**：缓存键包含 ModTime，无需额外 TTL 机制
2. **异步缓存写入**：先返回结果，后台异步落盘缓存，降低首响应延迟
3. **EXIF 缩略图提取**：缩略图场景优先利用 JPEG 内嵌数据，跳过完整解码
4. **二级哈希目录**：避免单目录文件数过多导致的性能问题
5. **细粒度锁**：每个缓存键独立锁，减少并发冲突
6. **信号量限流**：限制同时处理的图片数量，防止内存暴增
7. **超大图防护**：检测超过 10000x10000 的图片直接拒绝处理
8. **前端懒加载**：列表缩略图仅在视口内加载，节省带宽
9. **客户端缓存 buster**：URL 携带 key 参数，精确控制浏览器缓存
