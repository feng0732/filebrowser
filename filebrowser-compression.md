# FileBrowser 压缩解压处理流程

## 概述

FileBrowser 的压缩解压功能主要围绕**文件归档下载**展开，使用第三方库 `github.com/mholt/archives` 实现多种格式的归档打包。目前项目中**仅实现了压缩（归档下载）功能**，尚未实现上传压缩包后自动解压的功能。

存在两条独立的下载路径，最终都汇聚到同一套后端归档逻辑：
- **路径 A**：普通文件列表下载（需 JWT 鉴权）
- **路径 B**：公共分享下载（需分享 hash + 可选 token 鉴权）

---

## 零、格式处理核心原则（澄清最容易混淆的点）

### 两道关键分发判断

后端 `rawHandler` 在执行归档前有**两道不可绕过的分发判断**，决定了请求的命运：

```
HTTP 请求到达 rawHandler
        ↓
【第一道：IsDir 判断】
  ┌───────────────────────────────────────────────────────┐
  │  file.IsDir == false（请求路径是单个文件）              │
  │    → 走 rawFileHandler                                 │
  │    → 直接流式返回文件内容                               │
  │    → ⚠️  完全无视 algo 和 files 参数，不做任何归档       │
  ├───────────────────────────────────────────────────────┤
  │  file.IsDir == true（请求路径是目录）                   │
  │    → 走 rawDirHandler                                  │
  │    → 进入第二道判断                                     │
  └───────────────────────────────────────────────────────┘
        ↓
【第二道：parseQueryAlgorithm 解析 algo】
  ┌───────────────────────────────────────────────────────┐
  │  algo 为空字符串（""）、"zip"、"true"  →  zip 归档     │
  │  algo = "tar"                             →  tar       │
  │  algo = "targz" 等 tar 变体               →  tar.xxx   │
  │  algo = 其他值                           →  报错        │
  └───────────────────────────────────────────────────────┘
```

**核心结论**：
1. **单文件下载永远不会归档**，哪怕 URL 上手动加 `?algo=targz` 也会被忽略
2. **format=null 不等于默认 zip**。format=null 只是前端不附加 algo 参数，实际行为取决于请求的是文件还是目录：
   - 单文件 → 直接返回原文件
   - 目录 → algo 为空 → 进入第二道判断 → 默认 zip

### format 参数与实际下载结果的完整对应

| 前端调用方式 | format 值 | URL 中 algo | 后端目标类型 | 实际处理路径 | 最终结果 |
|------------|----------|------------|------------|------------|---------|
| 选中 1 个非目录文件点击下载 | `null` | **无 algo** | 文件（IsDir=false） | rawFileHandler | 返回原文件（不归档） |
| 选中 1 个目录点击下载 | 用户选择（如"zip"） | `algo=zip` | 目录（IsDir=true） | rawDirHandler | zip 归档 |
| 选中多文件/多目录 | 用户选择（如"targz"） | `algo=targz` | 目录（IsDir=true） | rawDirHandler | tar.gz 归档 |
| 未选中任何文件下载当前目录 | 用户选择 | 有 algo | 目录（IsDir=true） | rawDirHandler | 指定格式归档 |
| 信息面板下载链接（文件） | 不传 | **无 algo** | 文件（IsDir=false） | rawFileHandler | 返回原文件（不归档） |
| 信息面板下载链接（目录） | 不传 | **无 algo** | 目录（IsDir=true） | rawDirHandler | **默认 zip 归档** |

---

## 一、路径 A：普通文件列表下载（完整调用链）

### 1. 调用链总览

```
用户点击下载按钮（HeaderBar Action）
    ↓
FileListing.download() 判断 isDir + selectedCount
    ├─【分支 A1】选中 1 个非目录文件
    │      → api.download(null, item.url)
    │      → URL 无 algo 参数
    │      → 后端 IsDir=false → rawFileHandler → 直接返回原文件
    │
    └─【分支 A2】目录 / 多文件 / 未选中任何文件
           → layoutStore.showHover({ prompt: "download", confirm })
           → Prompts.vue 渲染 Download.vue 弹窗
           → 用户点击格式按钮 → confirm(format) 回调触发
           → 收集文件 url 列表 → api.download(format, ...files)
           → 构造 URL（含 algo、files 查询参数）
           → window.open(url) 触发浏览器下载
           → 后端 handle() → withUser() 中间件（JWT 鉴权）→ rawHandler
           → 权限检查 d.user.Perm.Download
           → IsDir=true → rawDirHandler
           → parseQueryFiles + parseQueryAlgorithm + getFiles + Archive
           → 归档输出到 HTTP Response
```

### 2. 前端 UI 触发

#### 2.1 下载按钮渲染

**文件**：[FileListing.vue](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/frontend/src/views/files/FileListing.vue#L61-L67)

下载按钮在 HeaderBar 的 actions 插槽中渲染，受 `headerButtons.download` 计算属性控制：

```typescript
// FileListing.vue L476-L490
const headerButtons = computed(() => {
  return {
    download: authStore.user?.perm.download,  // 用户权限控制按钮可见性
    // ...
  };
});
```

- 下载按钮仅在 `authStore.user.perm.download === true` 时渲染
- 按钮绑定 `@action="download"` 事件，选中文件数通过 `:counter` 显示

#### 2.2 download() 函数逻辑分支

**文件**：[FileListing.vue](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/frontend/src/views/files/FileListing.vue#L977-L1006)

```typescript
const download = () => {
  if (fileStore.req === null) return;

  // 【分支 A1】：选中 1 个 AND 该文件不是目录
  // → 直接下载，跳过格式选择弹窗
  // → format=null，URL 不附加 algo 参数
  if (
    fileStore.selectedCount === 1 &&
    !fileStore.req.items[fileStore.selected[0]].isDir
  ) {
    api.download(null, fileStore.req.items[fileStore.selected[0]].url);
    return;
  }

  // 【分支 A2】：其他所有情况
  //   a) 选中 1 个目录（isDir=true）
  //   b) 选中 2 个及以上文件/目录
  //   c) 未选中任何文件（下载当前目录）
  // → 弹出格式选择弹窗，用户选择后带 format 参数调用
  layoutStore.showHover({
    prompt: "download",
    confirm: (format: any) => {
      layoutStore.closeHovers();
      const files = [];
      if (fileStore.selectedCount > 0 && fileStore.req !== null) {
        for (const i of fileStore.selected) {
          files.push(fileStore.req.items[i].url);  // 使用 item.url 作为路径
        }
      } else {
        files.push(route.path);  // 未选中任何文件时下载当前目录
      }
      api.download(format, ...files);
    },
  });
};
```

**关键决策点**（⚠️ 注意 format 与实际行为的关系）：

| 条件 | 行为 | format 参数 | 实际结果 |
|------|------|------------|---------|
| `selectedCount === 1` AND `!item.isDir` | 不弹窗，直接调用 API | `null`（URL 无 algo） | 后端 IsDir=false → 返回原文件（**不归档**） |
| `selectedCount === 1` AND `item.isDir` | 弹窗选格式 | 用户选择值（如 "zip"） | 后端 IsDir=true → 指定格式归档 |
| `selectedCount >= 2` | 弹窗选格式 | 用户选择值（如 "targz"） | 后端 IsDir=true → 指定格式归档 |
| `selectedCount === 0` | 弹窗选格式 | 用户选择值（如 "tar"） | 下载当前目录 → 指定格式归档 |

### 3. 格式选择弹窗

**文件**：[Download.vue](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/frontend/src/components/prompts/Download.vue#L31-L41)

```typescript
const formats = {
  zip: "zip",
  tar: "tar",
  targz: "tar.gz",
  tarbz2: "tar.bz2",
  tarxz: "tar.xz",
  tarlz4: "tar.lz4",
  tarsz: "tar.sz",
  tarbr: "tar.br",
  tarzst: "tar.zst",
};
```

弹窗中每个按钮调用 `layoutStore.currentPrompt?.confirm(format)`，其中 `format` 是 `formats` 对象的**键**（如 `"zip"`, `"targz"` 等），**不是**显示的扩展名字符串。

**弹窗的渲染流程**：
1. `layoutStore.showHover({ prompt: "download", confirm })` 将 `{ prompt: "download", confirm }` 推入 `prompts` 栈
2. [Prompts.vue](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/frontend/src/components/prompts/Prompts.vue#L37-L55) 监听 `currentPromptName`，匹配到 `"download"` 后渲染 `Download.vue`
3. 用户点击格式按钮 → `confirm(format)` 被调用 → 触发 `api.download(format, ...files)`

### 4. API 层 URL 构造

**文件**：[files.ts](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/frontend/src/api/files.ts#L82-L104)

```typescript
export function download(format: any, ...files: string[]) {
  let url = `${baseURL}/api/raw`;

  // 单文件/单目录：路径放在 URL path 中（不使用 files 查询参数）
  if (files.length === 1) {
    url += removePrefix(files[0]) + "?";
  }
  // 多文件/多目录：路径以逗号分隔放在 files 查询参数中
  else {
    let arg = "";
    for (const file of files) {
      arg += removePrefix(file) + ",";
    }
    arg = arg.substring(0, arg.length - 1);
    arg = encodeURIComponent(arg);
    url += `/?files=${arg}&`;
  }

  // ⚠️ 关键：format 为 falsy（null / 空字符串 / undefined）时不附加 algo 参数
  if (format) {
    url += `algo=${format}&`;
  }

  window.open(url);  // 新窗口打开触发浏览器下载
}
```

**URL 构造规则与后端实际行为对应**：

| 场景 | URL 示例 | algo | 后端 IsDir | 实际处理 |
|------|---------|------|-----------|---------|
| 单文件下载（选中 1 个 txt） | `/api/raw/path/to/file.txt?` | 无 | false | rawFileHandler → 原文件 |
| 单目录下载（选中 zip 格式） | `/api/raw/path/to/dir?algo=zip&` | zip | true | rawDirHandler → zip 归档 |
| 多文件下载（tar.gz 格式） | `/api/raw/?files=dir/a,dir/b&algo=targz&` | targz | true | rawDirHandler → tar.gz 归档 |
| 下载当前根目录（tar 格式） | `/api/raw/?algo=tar&` | tar | true | rawDirHandler → tar 归档 |
| 信息面板下载目录（无格式） | `/api/raw/path/to/dir?`（见下文） | 无 | true | rawDirHandler → **默认 zip** |

**注意**：`removePrefix` 函数移除路径中的 `/files` 前缀，使其变为相对路径。

### 5. 信息面板下载链接（getDownloadURL）——隐蔽的默认 zip 触发点

**文件**：[Preview.vue](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/frontend/src/views/files/Preview.vue#L285-L291) 和 [files.ts](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/frontend/src/api/files.ts#L211-L217)

```typescript
// Preview.vue
const downloadUrl = computed(() =>
  fileStore.req ? api.getDownloadURL(fileStore.req, false) : ""
);

// files.ts
export function getDownloadURL(file: ResourceItem, inline: any) {
  const params = {
    ...(inline && { inline: "true" }),
  };
  return createURL("api/raw" + file.path, params);
}
```

**⚠️ 重要行为**：
- `getDownloadURL` **永远不附加 algo 参数**
- 当下载对象是**单个文件**时：后端 IsDir=false → rawFileHandler → 直接返回原文件
- 当下载对象是**目录**时：后端 IsDir=true → rawDirHandler → parseQueryAlgorithm 中 algo="" → **默认 zip 归档**
- 用户在信息面板点击下载目录时，**不会弹出格式选择**，直接下载 zip 格式

### 6. 后端鉴权中间件

**文件**：[auth.go](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/auth.go#L85-L111)

`withUser` 中间件在 `rawHandler` 之前执行：

```go
func withUser(fn handleFunc) handleFunc {
    return func(w http.ResponseWriter, r *http.Request, d *data) (int, error) {
        // 1. 从请求中提取 JWT token（X-Auth header 或 auth cookie）
        token, err := request.ParseFromRequest(r, &extractor{}, keyFunc, ...)

        // 2. 验证失败 → 401 Unauthorized
        if (err != nil || !token.Valid) && !renewableErr(err, d) {
            return http.StatusUnauthorized, nil
        }

        // 3. 通过 token 中的 User.ID 从数据库获取完整用户对象
        d.user, err = d.store.Users.Get(d.server.Root, tk.User.ID)

        // 4. 将 d.user 传递给后续处理器
        return fn(w, r, d)
    }
}
```

**JWT 提取顺序**：
1. `X-Auth` 请求头（包含两个 `.` 的字符串才视为 JWT）
2. `auth` Cookie（仅 GET 请求）

**注意**：`window.open(url)` 触发下载时，浏览器会自动携带 Cookie（含 `auth`），但不会携带自定义 `X-Auth` 头。因此普通下载依赖 Cookie 鉴权。

### 7. 后端分发处理

#### 7.1 rawHandler —— 第一道分水岭

**文件**：[raw.go](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/raw.go#L83-L110)

```go
var rawHandler = withUser(func(w http.ResponseWriter, r *http.Request, d *data) (int, error) {
    // 权限检查
    if !d.user.Perm.Download {
        return http.StatusAccepted, nil   // 无下载权限，返回 202（不报错但无内容）
    }

    // 获取文件信息（确定 IsDir 值的关键一步）
    file, err := files.NewFileInfo(&files.FileOptions{
        Fs:         d.user.Fs,
        Path:       r.URL.Path,   // URL path 对应实际文件系统路径
        Modify:     d.user.Perm.Modify,
        Expand:     false,
        ReadHeader: d.server.TypeDetectionByHeader,
        Checker:    d,
    })

    // 命名管道特殊处理
    if files.IsNamedPipe(file.Mode) {
        setContentDisposition(w, r, file)
        return 0, nil
    }

    // ⚠️ 第一道分发判断：决定了 algo 参数是否会被使用
    if !file.IsDir {
        // 【路径 1】单个文件 → 直接流式返回，无视 algo 和 files 参数
        return rawFileHandler(w, r, file)
    }

    // 【路径 2】目录 → 走归档流程，才会解析 algo 参数
    return rawDirHandler(w, r, d, file)
})
```

**`r.URL.Path` 的来源**：
- 前端 `files.length === 1` 时：URL path 包含目标路径（如 `/api/raw/path/to/file.txt`）→ `r.URL.Path` = `/path/to/file.txt`
- 前端 `files.length > 1` 时：URL 基础路径是 `/api/raw/` → `r.URL.Path` = `/`（即用户根目录）
  - 但实际待下载文件列表通过 `files` 查询参数解析（见 parseQueryFiles）

#### 7.2 rawFileHandler —— 单文件直传路径（完全不归档）

**文件**：[raw.go](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/raw.go#L218-L231)

```go
func rawFileHandler(w http.ResponseWriter, r *http.Request, file *files.FileInfo) (int, error) {
    fd, err := file.Fs.Open(file.Path)
    if err != nil {
        return http.StatusInternalServerError, err
    }
    defer fd.Close()

    setContentDisposition(w, r, file)   // 用 file.Name 设置下载文件名（原文件名）
    w.Header().Add("Content-Security-Policy", `script-src 'none';`)
    w.Header().Set("X-Content-Type-Options", "nosniff")
    w.Header().Set("Cache-Control", "private")
    http.ServeContent(w, r, file.Name, file.ModTime, fd)  // 支持 Range 请求
    return 0, nil
}
```

**关键特征**：
- 不调用 `parseQueryAlgorithm`，**完全不解析 algo 参数**
- 不调用 `parseQueryFiles`，**完全不解析 files 参数**
- 输出文件名就是原文件名（不带 `.zip` 等后缀）
- 使用 `http.ServeContent` 支持断点续传（Range 请求）

### 8. 后端归档流程（仅目录路径）

#### 8.1 parseQueryFiles —— 文件列表参数解析

**文件**：[raw.go](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/raw.go#L26-L45)

```go
func parseQueryFiles(r *http.Request, f *files.FileInfo, _ *users.User) ([]string, error) {
    var fileSlice []string
    names := strings.Split(r.URL.Query().Get("files"), ",")

    // ⚠️ 注意：strings.Split("", ",") 返回 [""]（长度为 1）
    // 所以 len(names)==0 仅在 Go 特殊情况下触发，一般用不到
    // 实际生效的分支是：无 files 查询参数时 → 进入 else，name 为空字符串
    if len(names) == 0 {
        fileSlice = append(fileSlice, f.Path)   // 无 files 参数：使用 URL path 对应路径
    } else {
        for _, name := range names {
            name, err := url.QueryUnescape(strings.ReplaceAll(name, "+", "%2B"))
            if err != nil {
                return nil, err
            }

            name = slashClean(name)              // 规范化为以 / 开头的绝对路径
            fileSlice = append(fileSlice, filepath.Join(f.Path, name))
        }
    }
    return fileSlice, nil
}
```

**实际场景分析**：

| 场景 | `?files=` 查询参数值 | strings.Split 结果 | f.Path（URL path） | 最终 fileSlice |
|------|---------------------|-------------------|-------------------|---------------|
| 单目录（无 files） | 空 | `[""]` | `/path/to/dir` | `[/path/to/dir]` |
| 根目录下载当前（无 files） | 空 | `[""]` | `/` | `[/]` |
| 多文件 | `dir/a,dir/b` | `["dir/a", "dir/b"]` | `/` | `[/dir/a, /dir/b]` |
| 多目录 | `proj/src,proj/test` | `["proj/src", "proj/test"]` | `/` | `[/proj/src, /proj/test]` |

**注意**：此函数**仅在 rawDirHandler 中被调用**（即 f.IsDir=true 时）。

#### 8.2 parseQueryAlgorithm —— 格式解析与默认 zip

**文件**：[raw.go](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/raw.go#L47-L70)

```go
func parseQueryAlgorithm(r *http.Request) (string, archives.Archival, error) {
    switch r.URL.Query().Get("algo") {
    // ⚠️ 三者等效：空字符串（无 algo）、"zip"、"true" → 都返回 zip
    case "zip", "true", "":
        return ".zip", archives.Zip{}, nil
    case "tar":
        return ".tar", archives.Tar{}, nil
    case "targz":
        return ".tar.gz", archives.CompressedArchive{Compression: archives.Gz{}, Archival: archives.Tar{}}, nil
    case "tarbz2":
        return ".tar.bz2", archives.CompressedArchive{Compression: archives.Bz2{}, Archival: archives.Tar{}}, nil
    case "tarxz":
        return ".tar.xz", archives.CompressedArchive{Compression: archives.Xz{}, Archival: archives.Tar{}}, nil
    case "tarlz4":
        return ".tar.lz4", archives.CompressedArchive{Compression: archives.Lz4{}, Archival: archives.Tar{}}, nil
    case "tarsz":
        return ".tar.sz", archives.CompressedArchive{Compression: archives.Sz{}, Archival: archives.Tar{}}, nil
    case "tarbr":
        return ".tar.br", archives.CompressedArchive{Compression: archives.Brotli{}, Archival: archives.Tar{}}, nil
    case "tarzst":
        return ".tar.zst", archives.CompressedArchive{Compression: archives.Zstd{}, Archival: archives.Tar{}}, nil
    default:
        return "", nil, errors.New("format not implemented")
    }
}
```

**完整的 algo 参数映射表**（⚠️ 仅当 IsDir=true 时生效）：

| algo 参数 | 扩展名 | archives 库类型 | 触发场景 |
|----------|--------|----------------|---------|
| `""`（空 / 无参数） | `.zip` | `archives.Zip{}` | 信息面板下载目录、format=null 时下载目录 |
| `"zip"` | `.zip` | `archives.Zip{}` | 用户在弹窗选择 zip |
| `"true"` | `.zip` | `archives.Zip{}` | 历史遗留值，前端目前不使用 |
| `"tar"` | `.tar` | `archives.Tar{}` | 用户选择 tar（无压缩） |
| `"targz"` | `.tar.gz` | `CompressedArchive{Gz, Tar}` | 用户选择 tar.gz |
| `"tarbz2"` | `.tar.bz2` | `CompressedArchive{Bz2, Tar}` | 用户选择 tar.bz2 |
| `"tarxz"` | `.tar.xz` | `CompressedArchive{Xz, Tar}` | 用户选择 tar.xz |
| `"tarlz4"` | `.tar.lz4` | `CompressedArchive{Lz4, Tar}` | 用户选择 tar.lz4 |
| `"tarsz"` | `.tar.sz` | `CompressedArchive{Sz, Tar}` | 用户选择 tar.sz |
| `"tarbr"` | `.tar.br` | `CompressedArchive{Brotli, Tar}` | 用户选择 tar.br |
| `"tarzst"` | `.tar.zst` | `CompressedArchive{Zstd, Tar}` | 用户选择 tar.zst |
| 其他任意值 | - | - | 返回错误 "format not implemented" |

#### 8.3 权限检查与归档输出的关联

权限检查贯穿整个调用链，存在三个层级：

| 层级 | 检查点 | 代码位置 | 失败行为 |
|------|--------|---------|---------|
| L1 用户权限 | `d.user.Perm.Download` | [raw.go#L84](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/raw.go#L84) | 返回 202，无输出 |
| L2 规则过滤 | `d.Check(path)` | [raw.go#L113](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/raw.go#L113) | 跳过该文件，不纳入归档 |
| L3 文件系统 | `d.user.Fs.Stat/Open` | [raw.go#L117-L139](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/raw.go#L117-L139) | 返回错误，中断归档 |

**L2 规则过滤的详细逻辑**（[data.go#L37-L64](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/data.go#L37-L64)）：
```go
func (d *data) Check(path string) bool {
    // 1. 如果文件系统被重基（公共分享），还原到用户原始范围
    if d.checkerPrefix != "" {
        path = gopath.Join(d.checkerPrefix, path)
    }
    // 2. 隐藏点文件检查
    if d.user.HideDotfiles && rules.MatchHidden(path) {
        return false
    }
    // 3. 全局规则匹配
    for _, rule := range d.settings.Rules {
        if rule.Matches(path) { allow = rule.Allow }
    }
    // 4. 用户规则匹配（优先级更高）
    for _, rule := range d.user.Rules {
        if rule.Matches(path) { allow = rule.Allow }
    }
    return allow
}
```

**关键影响**：被规则过滤的文件在 `getFiles` 中返回 `nil`，不会中断归档，只是被静默跳过。这意味着用户下载的归档可能比实际目录内容少，但不会报错。

#### 8.4 getFiles —— 递归文件收集

**文件**：[raw.go](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/raw.go#L112-L168)

```
getFiles(d, path, commonPath)
    ↓
d.Check(path) → 无权限则返回 nil（文件被跳过，不报错）
    ↓
d.user.Fs.Stat(path) → 获取文件信息（ScopedFs 安全限制生效）
    ↓
如果 path == commonPath → 不将目录自身加入归档（避免根目录条目）
否则 → 计算 nameInArchive（去掉 commonPath 前缀）
    ↓
构建 archives.FileInfo{
    FileInfo:      info,
    NameInArchive: nameInArchive,    // 归档内路径（做 Zip Slip 防护）
    Open:          func() → d.user.Fs.Open(path),   // 延迟打开
}
    ↓
如果是目录 → Readdirnames 读取子项 → 递归 getFiles 处理
    ↓
返回 []archives.FileInfo（扁平化的文件列表，不含目录项本身）
```

**Zip Slip 防护**（[raw.go#L125-L133](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/raw.go#L125-L133)）：
```go
nameInArchive = filepath.ToSlash(nameInArchive)
// filepath.ToSlash only rewrites the host separator, so on a Linux
// host a stored backslash survives and is emitted verbatim into the
// archive. Windows extractors then treat "\" as a path separator,
// allowing the entry to escape the extraction directory (zip-slip).
// Strip Windows separators regardless of host OS.
nameInArchive = strings.ReplaceAll(nameInArchive, "\\", "/")
```

#### 8.5 rawDirHandler —— 归档输出主流程

**文件**：[raw.go](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/raw.go#L170-L216)

```
1. parseQueryFiles(r, file) → 解析待下载文件列表 filenames
2. parseQueryAlgorithm(r)   → 解析格式，得到 extension（如 ".zip"）和 archiver
3. fileutils.CommonPrefix(filenames...) → 计算所有文件的公共目录前缀
4. 遍历 filenames，对每个 fname 调用 getFiles(d, fname, commonDir)
5. 合并所有 []archives.FileInfo → allFiles
6. 生成归档文件名：
   a. name = basename(commonDir)
   b. 特殊情况（basename 为 "."/"/"/""）→ 用 file.Name 或文件系统根名
   c. 若 filenames 数量 > 1 → name = "_" + name（加下划线前缀区分）
   d. name += extension（追加格式扩展名）
7. Set Content-Disposition: attachment; filename*=utf-8''<name>
8. archiver.Archive(ctx, w, allFiles) → 流式写入 HTTP Response
      （内部会调用 archives.FileInfo.Open() 逐个读取文件内容）
```

**输出文件名示例**：

| 场景 | filenames | commonDir | basename | 最终文件名 |
|------|-----------|-----------|----------|-----------|
| 下载单个目录 `docs` | `[/home/user/docs]` | `/home/user/docs` | `docs` | `docs.zip` |
| 下载两个子目录 | `[/proj/src, /proj/test]` | `/proj` | `proj` | `_proj.tar.gz` |
| 下载根目录（无选中） | `[/]` | `/` | `/` → 取 file.Name | `<rootname>.zip` |
| 下载多个散文件 | `[/a.txt, /b/c.txt]` | `/` | `/` → 取文件系统根名 | `<fsroot>.tar` |

---

## 二、路径 B：公共分享下载（完整调用链）

### 1. 调用链总览

```
用户访问分享链接 /share/{hash}
    ↓
Share.vue 挂载 → fetchData() → api.fetch(url, password)
    ↓
GET /api/public/share/{hash} → publicShareHandler → withHashFile 中间件
    ↓
withHashFile: hash 查找分享 → authenticateShareRequest → 获取分享用户
    ↓
设置 d.user, d.user.Fs = ScopedFs, d.checkerPrefix
    ↓
返回前端（含分享 token，用于后续下载）
    ↓
用户点击下载（三个入口）
    ├─【入口 B1】信息面板下载链接
    │     → api.getDownloadURL(req)
    │     → URL: /api/public/dl/{hash}{path}?token=xxx（无 algo）
    │     → 后端 IsDir=false → 原文件；IsDir=true → 默认 zip 归档
    │
    ├─【入口 B2】选中单个非目录文件点击下载
    │     → Share.download() 判断 isSingleFile()=true
    │     → api.download(null, hash, token, item.path)
    │     → URL 无 algo → 后端 IsDir=false → 直接返回原文件
    │
    └─【入口 B3】选中目录/多文件点击下载
          → Share.download() → 弹窗选格式
          → api.download(format, hash, token, ...files)
          → URL 含 algo、files、token 参数
          → window.open(url) 触发下载
    ↓
后端 publicDlHandler → withHashFile 中间件（hash + token 鉴权）
    ↓
权限检查 user.Perm.Share && user.Perm.Download
    ↓
第一道分发：IsDir=false → rawFileHandler；IsDir=true → rawDirHandler
    ↓
归档输出到 HTTP Response（与路径 A 完全相同的逻辑）
```

### 2. 前端 UI 触发

#### 2.1 Share.vue 的三个下载入口

**文件**：[Share.vue](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/frontend/src/views/Share.vue)

**入口 1 - 信息面板下载链接**（始终可见，无格式选择）：

[Share.vue#L110-L121](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/frontend/src/views/Share.vue#L110-L121) — 直接下载链接：
```html
<a target="_blank" :href="link" class="button button--flat">
  <i class="material-icons">file_download</i>{{ t("buttons.download") }}
</a>
```

其中 `link` 的计算方式（[Share.vue#L355](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/frontend/src/views/Share.vue#L355)）：
```typescript
const link = computed(() => (req.value ? api.getDownloadURL(req.value) : ""));
```

**⚠️ 同普通路径的 getDownloadURL**：不附加 algo 参数。下载目录时后端默认 zip，下载文件时直接返回。

**入口 2 - 预览单个文件时的"原始文件"链接**（仅单文件非目录时显示）：

[Share.vue#L14](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/frontend/src/views/Share.vue#L14)：
```vue
<a v-if="isSingleFile()" target="_blank" :href="raw">原始文件</a>
```

`raw` 的计算（[Share.vue#L356-L362](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/frontend/src/views/Share.vue#L356-L362)）：
```typescript
const raw = computed(() => {
  if (!req.value || !req.value.items[fileStore.selected[0]]) return "";
  return createURL(
    `api/public/dl/${hash.value}${req.value.items[fileStore.selected[0]].path}`,
    { token: token.value }
  );
});
```

**入口 3 - 选中文件后点击 Header 下载按钮**（选中文件后可见）：

[Share.vue#L6-L11](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/frontend/src/views/Share.vue#L6-L11)：
```html
<action
  v-if="fileStore.selectedCount"
  icon="file_download"
  :label="t('buttons.download')"
  @action="download"
  :counter="fileStore.selectedCount"
/>
```

#### 2.2 Share.download() 函数 —— 与 FileListing 相同的分支逻辑

**文件**：[Share.vue](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/frontend/src/views/Share.vue#L441-L476)

```typescript
// 判断条件与 FileListing.download 完全一致：1 个 AND 非目录
const isSingleFile = () =>
  fileStore.selectedCount === 1 &&
  !req.value?.items[fileStore.selected[0]].isDir;

const download = () => {
  if (!req.value) return false;

  // 【分支 B1】：选中单个非目录文件 → 直接下载，不弹窗
  if (isSingleFile()) {
    api.download(
      null,                                    // format=null → 无 algo
      hash.value,
      token.value,
      req.value.items[fileStore.selected[0]].path  // 用 path 而非 url
    );
    return true;
  }

  // 【分支 B2】：目录/多文件 → 弹窗选格式
  layoutStore.showHover({
    prompt: "download",
    confirm: (format: DownloadFormat) => {
      layoutStore.closeHovers();
      const files: string[] = [];
      for (const i of fileStore.selected) {
        files.push(req.value.items[i].path);  // 注意：用 path 而非 url
      }
      api.download(format, hash.value, token.value, ...files);
      return true;
    },
  });
  return true;
};
```

**与 FileListing.download() 的差异**：

| 对比项 | FileListing（普通） | Share（公共分享） |
|--------|-------------------|-----------------|
| 单文件判断条件 | 完全相同（`selectedCount===1 && !isDir`） | 完全相同 |
| format 参数 | 单文件时传 `null` | 单文件时传 `null` |
| 路径参数来源 | `item.url`（含 `/files` 前缀，经 removePrefix 处理） | `item.path`（无前缀） |
| 附加参数 | 无 | 传 `hash` 和 `token` |

### 3. 公共分享 API 层 URL 构造

**文件**：[pub.ts](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/frontend/src/api/pub.ts#L35-L66)

```typescript
export function download(
  format: DownloadFormat,
  hash: string,
  token: string,
  ...files: string[]
) {
  let url = `${baseURL}/api/public/dl/${hash}`;

  // 单文件/单目录：路径追加在 hash 后
  if (files.length === 1) {
    url += files[0] + "?";
  }
  // 多文件/多目录：files 查询参数
  else {
    let arg = "";
    for (const file of files) {
      arg += file + ",";
    }
    arg = arg.substring(0, arg.length - 1);
    arg = encodeURIComponent(arg);
    url += `/?files=${arg}&`;
  }

  // ⚠️ format 为 falsy 时不附加 algo（同普通下载）
  if (format) {
    url += `algo=${format}&`;
  }

  // ⚠️ 分享特有：token 作为查询参数
  if (token) {
    url += `token=${token}&`;
  }

  window.open(url);
}
```

**公共分享 URL 示例**：

| 场景 | URL 示例 |
|------|---------|
| 信息面板下载单文件 | `/api/public/dl/ABC123/path/file.txt?token=xyz` |
| 信息面板下载目录（默认 zip） | `/api/public/dl/ABC123/path/dir?token=xyz` |
| 选中单文件（无 algo） | `/api/public/dl/ABC123/path/file.txt?token=xyz` |
| 选中目录（tar.gz 格式） | `/api/public/dl/ABC123/path/dir?algo=targz&token=xyz` |
| 多文件下载（zip） | `/api/public/dl/ABC123/?files=a,b&algo=zip&token=xyz` |

**getDownloadURL**（[pub.ts#L68-L75](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/frontend/src/api/pub.ts#L68-L75)）：
```typescript
export function getDownloadURL(res: Resource, inline = false) {
  const params = {
    ...(inline && { inline: "true" }),
    ...(res.token && { token: res.token }),   // 有密码保护的分享带 token
  };
  return createURL("api/public/dl/" + res.hash + res.path, params);
}
```

### 4. 后端分享鉴权中间件

#### 4.1 withHashFile 中间件 —— 设置 d.raw 供 publicDlHandler 使用

**文件**：[public.go](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/public.go#L17-L98)

```go
var withHashFile = func(fn handleFunc) handleFunc {
    return func(w http.ResponseWriter, r *http.Request, d *data) (int, error) {
        // 1. 从 URL 解析分享 hash 和子路径
        //    /api/public/dl/ABC123/sub/file.txt → id="ABC123", ifPath="/sub/file.txt"
        id, ifPath := ifPathWithName(r)

        // 2. 通过 hash 查找分享记录（含 UserID、FilePath、PasswordHash、Token）
        link, err := d.store.Share.GetByHash(id)

        // 3. 验证分享密码/token（见 authenticateShareRequest）
        status, err := authenticateShareRequest(r, link)

        // 4. 获取分享所属用户（分享创建者）
        user, err := d.store.Users.Get(d.server.Root, link.UserID)

        // 5. ⚠️ 双重权限检查：必须同时有 Share 和 Download 权限
        if !user.Perm.Share || !user.Perm.Download {
            return http.StatusForbidden, nil
        }

        // 6. 计算分享目录的实际路径（用户范围内）
        basePath := user.ScopedPath(link.Path)

        // 7. 设置用户和作用域文件系统
        d.user = user
        d.user.Fs = files.NewScopedFs(d.user.Fs, basePath)  // 限制在分享目录
        d.checkerPrefix = basePath                           // 规则检查前缀

        // 8. 在 ScopedFs 范围内获取文件信息，存入 d.raw
        //    publicDlHandler 会从 d.raw 读取（而非重新构造）
        d.raw = file

        return fn(w, r, d)
    }
}
```

**d.raw 的关键作用**：
- `withHashFile` 已经在 ScopedFs 上构造了正确的 `files.FileInfo`
- `publicDlHandler` 直接复用 `d.raw`，避免重复构造
- `d.raw.IsDir` 是后续分发判断的依据

#### 4.2 authenticateShareRequest —— Token/密码验证

**文件**：[public.go](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/public.go#L136-L161)

```go
func authenticateShareRequest(r *http.Request, l *share.Link) (int, error) {
    // 无密码保护 → 直接通过
    if l.PasswordHash == "" {
        return 0, nil
    }

    // 方式1：URL 查询参数中的 token（时序安全比较，防时序攻击）
    // 用于 window.open 触发的下载（无法设置自定义 header）
    if subtle.ConstantTimeCompare([]byte(r.URL.Query().Get("token")), []byte(l.Token)) == 1 {
        return 0, nil
    }

    // 方式2：X-SHARE-PASSWORD 请求头中的密码
    // 用于前端 fetch 请求（如首次加载分享页面时）
    password := r.Header.Get("X-SHARE-PASSWORD")
    password, _ = url.QueryUnescape(password)
    if password == "" {
        return http.StatusUnauthorized, nil
    }
    if err := bcrypt.CompareHashAndPassword(...); err != nil {
        return http.StatusUnauthorized, nil
    }

    return 0, nil
}
```

**Token 传递机制**：

| 场景 | 前端传递方式 | 后端验证方式 |
|------|------------|------------|
| 无密码分享 | 不传 token | `l.PasswordHash == ""` → 直接通过 |
| 有密码分享（下载） | URL `?token=xxx` | `subtle.ConstantTimeCompare` |
| 有密码分享（fetch） | `X-SHARE-PASSWORD` header | `bcrypt.CompareHashAndPassword` |

**Token 的生命周期**：
1. 用户首次访问密码保护的分享链接 → 输入密码 → 前端用 `X-SHARE-PASSWORD` 头请求 `/api/public/share/{hash}`
2. 后端验证密码通过 → 返回的 JSON 中包含 `token` 字段（即 `link.Token`）
3. 前端保存 token 到 Vue 状态 → 后续所有下载请求的 URL 都附加 `?token=xxx`

#### 4.3 publicDlHandler —— 从 d.raw 读取并分发

**文件**：[public.go](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/public.go#L115-L134)

```go
var publicDlHandler = withHashFile(func(w http.ResponseWriter, r *http.Request, d *data) (int, error) {
    // ⚠️ 直接使用 withHashFile 预先设置好的 d.raw
    // 而不是像 rawHandler 那样重新构造 files.FileInfo
    file := d.raw

    // 命名管道特殊处理
    if files.IsNamedPipe(file.Mode) {
        setContentDisposition(w, r, file)
        return 0, nil
    }

    // 第一道分发判断：与 rawHandler 完全相同的逻辑
    if !file.IsDir {
        return rawFileHandler(w, r, file)  // 单文件 → 直接返回
    }

    return rawDirHandler(w, r, d, file)    // 目录 → 归档（与普通路径完全相同）
})
```

**与 rawHandler 的对比**：

| 对比项 | rawHandler（普通） | publicDlHandler（分享） |
|--------|------------------|-----------------------|
| FileInfo 来源 | 现场构造 `files.NewFileInfo()` | 复用 `d.raw`（withHashFile 预构造） |
| Fs | 用户原始 Fs | ScopedFs（重基到分享目录） |
| 分发逻辑 | 完全相同 | 完全相同 |
| 调用 rawFileHandler | 是 | 是 |
| 调用 rawDirHandler | 是 | 是 |

### 5. 权限检查与归档输出的关联（公共分享路径）

| 层级 | 检查点 | 代码位置 | 失败行为 |
|------|--------|---------|---------|
| L1 分享验证 | `authenticateShareRequest` | [public.go#L136](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/public.go#L136) | 401 Unauthorized |
| L2 用户权限 | `user.Perm.Share && user.Perm.Download` | [public.go#L35-L37](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/public.go#L35-L37) | 403 Forbidden |
| L3 文件系统 | `ScopedFs` 限制逃逸符号链接 | [public.go#L70](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/public.go#L70) | 操作返回 ErrPermission |
| L4 规则过滤 | `d.Check(path)`（含 checkerPrefix 还原） | [data.go#L37-L64](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/data.go#L37-L64) | 跳过该文件 |
| L5 下载权限 | `d.user.Perm.Download`（在 rawHandler 中不重复检查） | - | publicDlHandler 在 L2 已检查，此处不重复 |

**ScopedFs 的作用**：公共分享时，文件系统被重基到分享目录。这意味着：
- 归档只能包含分享目录内的文件
- 逃逸的符号链接在 `d.user.Fs.Open()` 时被 ScopedFs 拦截（返回 `os.ErrPermission`）
- 在 `getFiles` 中，`d.user.Fs.Stat(path)` 和 `d.user.Fs.Open(path)` 都受 ScopedFs 保护

**checkerPrefix 的作用**：由于文件系统被重基，路径相对于分享根目录。但规则是基于用户原始范围定义的，所以需要通过 `checkerPrefix` 将路径还原后再检查规则。

---

## 三、两条路径的格式处理流程对比

| 对比项 | 路径 A：普通文件列表下载 | 路径 B：公共分享下载 |
|--------|----------------------|-------------------|
| HTTP 端点 | `GET /api/raw/*` | `GET /api/public/dl/{hash}/*` |
| 鉴权方式 | JWT（X-Auth 头 / auth Cookie） | 分享 hash + token（URL 查询参数） |
| 中间件 | `withUser` | `withHashFile` |
| 用户来源 | JWT 中的 `User.ID` | 分享记录中的 `link.UserID` |
| 第一道分发判断依据 | `files.NewFileInfo(...).IsDir` | `d.raw.IsDir`（预构造） |
| 下载权限检查 | `d.user.Perm.Download` | `user.Perm.Share && user.Perm.Download` |
| 文件系统 | 用户原始 `d.user.Fs` | `ScopedFs`（重基到分享目录） |
| 规则检查 | 直接检查路径 | 通过 `checkerPrefix` 还原路径后检查 |
| format=null（单文件） | 无 algo → IsDir=false → 原文件 | 无 algo → IsDir=false → 原文件 |
| format=null（目录） | 一般不触发（目录会弹窗） | 一般不触发（目录会弹窗） |
| getDownloadURL（目录） | 无 algo → 默认 zip | 无 algo → 默认 zip |
| 弹窗格式选择（目录） | algo 指定格式归档 | algo 指定格式归档 |
| files 参数 | 文件的 `url` 属性（相对路径） | 文件的 `path` 属性（绝对路径） |
| rawFileHandler 调用 | 是 | 是 |
| rawDirHandler 调用 | 是 | 是（完全相同代码） |

**核心设计**：两条路径最终都调用相同的 `rawDirHandler` 和 `rawFileHandler`。`withHashFile` 通过预先设置 `d.user`、`d.user.Fs`（ScopedFs）、`d.checkerPrefix` 和 `d.raw`，使公共分享请求在进入归档逻辑时拥有与普通请求同构的数据结构。

---

## 四、格式处理混淆点汇总与澄清

### 混淆点 1：format=null 等于默认 zip？

**❌ 错误理解**：format=null 表示"默认 zip 格式"

**✅ 正确理解**：format=null 表示前端不附加 `algo` 查询参数。后端行为取决于 `IsDir`：
- 如果目标是**文件** → 完全不走归档路径，直接返回原文件
- 如果目标是**目录** → 走归档路径，algo 为空 → 默认 zip

### 混淆点 2：单文件也可以用 algo 指定格式归档？

**❌ 错误理解**：给单文件下载 URL 加 `?algo=tar` 可以返回 tar 包

**✅ 正确理解**：`rawHandler` 的 `IsDir` 判断在 `parseQueryAlgorithm` 之前执行。只要目标路径是文件，algo 参数**完全被忽略**，永远返回原文件。

### 混淆点 3：信息面板下载链接下载目录时是什么格式？

**❌ 错误理解**：不选格式就无法下载目录 / 会报错

**✅ 正确理解**：`getDownloadURL` 不附加 algo，后端收到目录请求后 `parseQueryAlgorithm` 解析 algo 为空 → 命中 `case "", "zip", "true"` → **默认 zip 归档**。用户无感知地下载 zip。

### 混淆点 4：选中 1 个目录点击下载是什么流程？

**❌ 错误理解**：选中 1 个项目 → format=null → 直接下载

**✅ 正确理解**：`selectedCount === 1 && !item.isDir` 两个条件缺一不可。目录的 `isDir=true`，会走到分支 B（弹窗选格式），用户选择后才下载。

### 混淆点 5：parseQueryFiles 中 len(names)==0 有用吗？

**实际情况**：`strings.Split("", ",")` 返回 `[""]`（长度 1），所以 `len(names)==0` 是死代码。实际场景下，无 files 参数时进入 else 分支，name 为空字符串 → 经 slashClean 和 Join 后等效于 `f.Path`。

### 混淆点 6：公共分享的 token 是怎么来的？

**完整流程**：
1. 分享创建时生成 `link.Token`（随机字符串）
2. 首次访问密码保护的分享，前端用密码通过 `X-SHARE-PASSWORD` 头鉴权
3. 后端验证通过后，响应 JSON 中包含 token 字段
4. 前端保存 token，后续所有下载 URL 附加 `?token=xxx`
5. 后端通过 `subtle.ConstantTimeCompare` 做时序安全比较

---

## 五、解压功能

### 当前状态

**项目中尚未实现上传压缩包后自动解压的功能**。

现有的上传相关代码只负责将文件原样写入磁盘，不进行解压处理：

- **普通上传**：[resourcePostHandler](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/resource.go#L125-L178) → `writeFile` 直接写入
- **Tus 分片上传**：[tusPatchHandler](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/tus_handlers.go#L156-L239) 追加写入

### 第三方库能力

虽然没有使用，但 `github.com/mholt/archives` 库本身支持解压操作（`Extract` 方法），如果后续需要实现解压功能，可以基于该库扩展。

---

## 六、安全机制汇总

### 1. 权限检查

| 检查项 | 普通下载 | 公共分享下载 |
|--------|---------|------------|
| 用户下载权限 | `d.user.Perm.Download` | `user.Perm.Share && user.Perm.Download` |
| 规则过滤 | `d.Check(path)` | `d.Check(path)`（含 checkerPrefix 还原） |
| 文件系统限制 | 用户 scope | ScopedFs（更严格） |

### 2. Zip Slip 防护

**文件**：[raw.go](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/raw.go#L125-L133)

在构建归档内路径时：
- 使用 `filepath.ToSlash` 统一路径分隔符
- 显式将所有反斜杠 `\` 替换为 `/`
- 防止恶意构造的路径在 Windows 解压时逃逸出目标目录

### 3. 符号链接安全

- 使用 `ScopedFs` 文件系统包装，在操作前检查符号链接是否指向范围外
- 归档时通过 `d.user.Fs.Open(path)` 打开文件，会受到 ScopedFs 的保护
- 公共分享路径中，逃逸符号链接在 `getFiles` 的 `d.user.Fs.Stat()` 阶段就被拦截
- 测试用例 [public_symlink_test.go](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/public_symlink_test.go#L132-L166) 验证了归档不会包含符号链接逃逸的文件

---

## 七、相关文件列表

| 文件 | 作用 |
|------|------|
| [http/raw.go](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/raw.go) | 压缩下载核心逻辑（两条路径共用） |
| [http/http.go](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/http.go) | HTTP 路由注册 |
| [http/public.go](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/public.go) | 公共分享鉴权与下载处理 |
| [http/auth.go](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/auth.go) | JWT 鉴权中间件 |
| [http/data.go](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/data.go) | data 结构与规则检查 |
| [http/resource.go](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/resource.go) | 资源操作（上传等） |
| [http/tus_handlers.go](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/tus_handlers.go) | Tus 协议上传处理 |
| [frontend/src/views/files/FileListing.vue](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/frontend/src/views/files/FileListing.vue) | 文件列表页面（下载触发入口） |
| [frontend/src/views/files/Preview.vue](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/frontend/src/views/files/Preview.vue) | 文件预览（信息面板下载链接） |
| [frontend/src/views/Share.vue](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/frontend/src/views/Share.vue) | 公共分享页面（分享下载入口） |
| [frontend/src/api/files.ts](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/frontend/src/api/files.ts) | 普通文件 API |
| [frontend/src/api/pub.ts](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/frontend/src/api/pub.ts) | 公共分享 API |
| [frontend/src/components/prompts/Download.vue](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/frontend/src/components/prompts/Download.vue) | 下载格式选择弹窗（两条路径共用） |
| [frontend/src/components/prompts/Prompts.vue](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/frontend/src/components/prompts/Prompts.vue) | 弹窗容器组件 |
| [frontend/src/stores/layout.ts](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/frontend/src/stores/layout.ts) | 弹窗状态管理 |
| [frontend/src/components/header/Action.vue](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/frontend/src/components/header/Action.vue) | 操作按钮组件 |
| [files/scoped.go](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/files/scoped.go) | 作用域文件系统（安全） |
| [fileutils/file.go](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/fileutils/file.go) | 文件工具函数（CommonPrefix 等） |
| [share/share.go](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/share/share.go) | 分享链接数据结构 |
| [users/permissions.go](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/users/permissions.go) | 用户权限结构定义 |
