# FileBrowser 压缩解压处理流程

## 概述

FileBrowser 的压缩解压功能主要围绕**文件归档下载**展开，使用第三方库 `github.com/mholt/archives` 实现多种格式的归档打包。目前项目中**仅实现了压缩（归档下载）功能**，尚未实现上传压缩包后自动解压的功能。

存在两条独立的下载路径，最终都汇聚到同一套后端归档逻辑：
- **路径 A**：普通文件列表下载（需 JWT 鉴权）
- **路径 B**：公共分享下载（需分享 hash + 可选 token 鉴权）

---

## 一、路径 A：普通文件列表下载（完整调用链）

### 1. 调用链总览

```
用户点击下载按钮（HeaderBar Action）
    ↓
FileListing.download() 判断是否为单文件
    ├─ 单文件 → api.download(null, item.url) → 直接下载，不选格式
    └─ 目录/多文件 → layoutStore.showHover({ prompt: "download", confirm })
            ↓
    Prompts.vue 渲染 Download.vue 弹窗
            ↓
    用户点击格式按钮 → confirm(format) 回调触发
            ↓
    收集选中文件的 url 列表 → api.download(format, ...files)
            ↓
    构造 /api/raw URL（含 algo、files 查询参数）
            ↓
    window.open(url) 触发浏览器下载
            ↓
    后端 handle() → withUser() 中间件（JWT 鉴权）→ rawHandler
            ↓
    权限检查 d.user.Perm.Download
            ↓
    文件 → rawFileHandler ｜ 目录 → rawDirHandler
            ↓
    归档输出到 HTTP Response
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

  // 分支1：选中单个文件（非目录）→ 直接下载，跳过格式选择
  if (
    fileStore.selectedCount === 1 &&
    !fileStore.req.items[fileStore.selected[0]].isDir
  ) {
    api.download(null, fileStore.req.items[fileStore.selected[0]].url);
    return;
  }

  // 分支2：目录或多文件 → 弹出格式选择弹窗
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

**关键决策点**：
| 条件 | 行为 | format 参数 |
|------|------|------------|
| 选中 1 个非目录文件 | 直接下载，不弹窗 | `null`（后端默认 zip） |
| 选中目录或多文件 | 弹出格式选择弹窗 | 用户选择的格式键名 |
| 未选中任何文件 | 下载当前目录 | 用户选择的格式键名 |

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

弹窗中每个按钮调用 `layoutStore.currentPrompt?.confirm(format)`，其中 `format` 是 `formats` 对象的键（如 `"zip"`, `"targz"` 等），**不是**显示的扩展名字符串。

**弹窗的渲染流程**：
1. `layoutStore.showHover({ prompt: "download", confirm })` 将 `{ prompt: "download", confirm }` 推入 `prompts` 栈
2. [Prompts.vue](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/frontend/src/components/prompts/Prompts.vue#L37-L55) 监听 `currentPromptName`，匹配到 `"download"` 后渲染 `Download.vue`
3. 用户点击格式按钮 → `confirm(format)` 被调用 → 触发 `api.download(format, ...files)`

### 4. API 层 URL 构造

**文件**：[files.ts](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/frontend/src/api/files.ts#L82-L104)

```typescript
export function download(format: any, ...files: string[]) {
  let url = `${baseURL}/api/raw`;

  if (files.length === 1) {
    url += removePrefix(files[0]) + "?";   // 单文件：路径放在 URL path 中
  } else {
    let arg = "";
    for (const file of files) {
      arg += removePrefix(file) + ",";
    }
    arg = arg.substring(0, arg.length - 1);
    arg = encodeURIComponent(arg);
    url += `/?files=${arg}&`;              // 多文件：路径放在查询参数中
  }

  if (format) {
    url += `algo=${format}&`;              // 格式参数
  }

  window.open(url);                        // 新窗口打开触发下载
}
```

**URL 构造规则**：

| 场景 | URL 示例 |
|------|---------|
| 单文件下载（无格式） | `/api/raw/path/to/file.txt?` |
| 单目录下载（zip） | `/api/raw/path/to/dir?algo=zip&` |
| 多文件下载（tar.gz） | `/api/raw/?files=dir/a,dir/b&algo=targz&` |

**注意**：`removePrefix` 函数移除路径中的 `/files` 前缀，使其变为相对路径。

### 5. 后端鉴权中间件

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

### 6. 后端归档处理

#### 6.1 rawHandler 入口

**文件**：[raw.go](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/raw.go#L83-L110)

```go
var rawHandler = withUser(func(w http.ResponseWriter, r *http.Request, d *data) (int, error) {
    // 权限检查
    if !d.user.Perm.Download {
        return http.StatusAccepted, nil   // 无下载权限，返回 202（不报错但无内容）
    }

    // 获取文件信息
    file, err := files.NewFileInfo(&files.FileOptions{
        Fs:      d.user.Fs,
        Path:    r.URL.Path,
        Checker: d,                       // d 实现了 rules.Checker 接口
    })

    // 分发处理
    if !file.IsDir {
        return rawFileHandler(w, r, file)  // 单文件：直接流式返回
    }
    return rawDirHandler(w, r, d, file)    // 目录：归档后返回
})
```

#### 6.2 权限检查与归档输出的关联

权限检查贯穿整个调用链，存在三个层级的检查：

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

#### 6.3 parseQueryFiles - 文件列表参数解析

**文件**：[raw.go](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/raw.go#L26-L45)

```go
func parseQueryFiles(r *http.Request, f *files.FileInfo, _ *users.User) ([]string, error) {
    names := strings.Split(r.URL.Query().Get("files"), ",")

    if len(names) == 0 {
        fileSlice = append(fileSlice, f.Path)   // 无 files 参数：使用当前路径
    } else {
        for _, name := range names {
            name, err := url.QueryUnescape(...)  // URL 解码
            name = slashClean(name)              // 路径规范化
            fileSlice = append(fileSlice, filepath.Join(f.Path, name))
        }
    }
    return fileSlice, nil
}
```

**前端-后端参数对应**：

| 前端 | 后端 | 说明 |
|------|------|------|
| `files` 查询参数 | `parseQueryFiles()` | 逗号分隔的相对路径列表 |
| `algo` 查询参数 | `parseQueryAlgorithm()` | 压缩格式键名 |
| URL path 部分 | `r.URL.Path` | 单文件时路径在 path 中 |

#### 6.4 parseQueryAlgorithm - 格式映射

**文件**：[raw.go](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/raw.go#L47-L70)

| algo 参数 | 扩展名 | archives 库类型 | 说明 |
|----------|--------|----------------|------|
| `zip` / `true` / `""` | `.zip` | `archives.Zip{}` | 默认格式 |
| `tar` | `.tar` | `archives.Tar{}` | 无压缩 |
| `targz` | `.tar.gz` | `archives.CompressedArchive{Gz, Tar}` | gzip 压缩 |
| `tarbz2` | `.tar.bz2` | `archives.CompressedArchive{Bz2, Tar}` | bzip2 压缩 |
| `tarxz` | `.tar.xz` | `archives.CompressedArchive{Xz, Tar}` | xz 压缩 |
| `tarlz4` | `.tar.lz4` | `archives.CompressedArchive{Lz4, Tar}` | LZ4 压缩 |
| `tarsz` | `.tar.sz` | `archives.CompressedArchive{Sz, Tar}` | Snappy 压缩 |
| `tarbr` | `.tar.br` | `archives.CompressedArchive{Brotli, Tar}` | Brotli 压缩 |
| `tarzst` | `.tar.zst` | `archives.CompressedArchive{Zstd, Tar}` | Zstandard 压缩 |
| 其他 | - | - | 返回错误 "format not implemented" |

#### 6.5 getFiles - 递归文件收集

**文件**：[raw.go](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/raw.go#L112-L168)

```
getFiles(d, path, commonPath)
    ↓
d.Check(path) → 无权限则返回 nil（文件被跳过）
    ↓
d.user.Fs.Stat(path) → 获取文件信息
    ↓
计算 nameInArchive（去掉 commonPath 前缀）
    ↓
构建 archives.FileInfo{
    FileInfo:      info,
    NameInArchive: nameInArchive,    // 归档内路径
    Open:          func() → d.user.Fs.Open(path),
}
    ↓
如果是目录 → 递归处理子项
    ↓
返回 []archives.FileInfo
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

#### 6.6 rawDirHandler - 归档输出

**文件**：[raw.go](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/raw.go#L170-L216)

```
1. parseQueryFiles → 解析文件列表
2. parseQueryAlgorithm → 解析压缩格式
3. CommonPrefix → 计算公共路径前缀
4. 循环 getFiles → 收集所有文件的 archives.FileInfo
5. 确定归档文件名：
   - basename(commonDir) + extension（单目录）
   - "_" + basename(commonDir) + extension（多文件/目录）
6. 设置 Content-Disposition: attachment; filename*=utf-8''...
7. archiver.Archive(ctx, w, allFiles) → 流式写入 HTTP Response
```

**输出文件名生成逻辑**：
- `filepath.Base(commonDir)` 获取公共目录名
- 如果基名为 `.` 或空或 `/`，则使用 `file.Name` 或文件系统根名
- 多个文件时加 `_` 前缀（如 `_mydir.zip`）
- 追加格式扩展名

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
用户点击下载按钮（两种入口）
    ├─ 入口1：信息面板下载链接 → api.getDownloadURL → /api/public/dl/{hash}{path}?token=xxx
    └─ 入口2：选中文件后点击下载 → Share.download() → 弹窗选格式
            ↓
    api.download(format, hash, token, ...files)
            ↓
    构造 /api/public/dl/{hash} URL（含 algo、files、token 查询参数）
            ↓
    window.open(url) 触发下载
            ↓
    后端 publicDlHandler → withHashFile 中间件（hash + token 鉴权）
            ↓
    权限检查 user.Perm.Share && user.Perm.Download
            ↓
    文件 → rawFileHandler ｜ 目录 → rawDirHandler
            ↓
    归档输出到 HTTP Response
```

### 2. 前端 UI 触发

#### 2.1 Share.vue 的下载入口

**文件**：[Share.vue](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/frontend/src/views/Share.vue)

Share 页面有两个下载入口：

**入口 1 - 信息面板下载链接**（始终可见）：

[Share.vue#L110-L121](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/frontend/src/views/Share.vue#L110-L121) — 直接下载链接，使用 `api.getDownloadURL` 构造 URL：
```html
<a target="_blank" :href="link" class="button button--flat">
  <i class="material-icons">file_download</i>{{ t("buttons.download") }}
</a>
```

其中 `link` 的计算方式（[Share.vue#L355](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/frontend/src/views/Share.vue#L355)）：
```typescript
const link = computed(() => (req.value ? api.getDownloadURL(req.value) : ""));
```

**入口 2 - 选中文件后的下载按钮**（选中文件后可见）：

[Share.vue#L6-L11](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/frontend/src/views/Share.vue#L6-L11) — 选中文件后出现下载按钮：
```html
<action
  v-if="fileStore.selectedCount"
  icon="file_download"
  :label="t('buttons.download')"
  @action="download"
  :counter="fileStore.selectedCount"
/>
```

#### 2.2 Share.download() 函数

**文件**：[Share.vue](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/frontend/src/views/Share.vue#L445-L476)

```typescript
const download = () => {
  if (!req.value) return false;

  // 分支1：选中单个文件（非目录）→ 直接下载
  if (isSingleFile()) {
    api.download(null, hash.value, token.value, req.value.items[fileStore.selected[0]].path);
    return true;
  }

  // 分支2：目录或多文件 → 弹出格式选择弹窗
  layoutStore.showHover({
    prompt: "download",
    confirm: (format: DownloadFormat) => {
      layoutStore.closeHovers();
      const files: string[] = [];
      for (const i of fileStore.selected) {
        files.push(req.value.items[i].path);  // 注意：使用 item.path 而非 item.url
      }
      api.download(format, hash.value, token.value, ...files);
      return true;
    },
  });
  return true;
};
```

### 3. 公共分享 API 层 URL 构造

**文件**：[pub.ts](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/frontend/src/api/pub.ts#L35-L66)

```typescript
export function download(format: DownloadFormat, hash: string, token: string, ...files: string[]) {
  let url = `${baseURL}/api/public/dl/${hash}`;

  if (files.length === 1) {
    url += files[0] + "?";              // 单文件：路径追加在 hash 后
  } else {
    let arg = "";
    for (const file of files) {
      arg += file + ",";
    }
    arg = arg.substring(0, arg.length - 1);
    arg = encodeURIComponent(arg);
    url += `/?files=${arg}&`;           // 多文件：路径放在查询参数中
  }

  if (format) {
    url += `algo=${format}&`;           // 格式参数
  }

  if (token) {
    url += `token=${token}&`;           // 分享 token（用于密码保护的分享）
  }

  window.open(url);
}
```

**与普通下载 URL 的对比**：

| 对比项 | 普通下载 | 公共分享下载 |
|--------|---------|------------|
| 基础路径 | `/api/raw` | `/api/public/dl/{hash}` |
| 鉴权方式 | JWT（Cookie/Header） | hash + token |
| token 参数 | 无 | `?token=xxx` |
| algo 参数 | 有 | 有 |
| files 参数 | 有 | 有 |

**getDownloadURL**（[pub.ts#L68-L75](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/frontend/src/api/pub.ts#L68-L75)）用于构造信息面板的下载链接：
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

#### 4.1 withHashFile 中间件

**文件**：[public.go](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/public.go#L17-L98)

```go
var withHashFile = func(fn handleFunc) handleFunc {
    return func(w http.ResponseWriter, r *http.Request, d *data) (int, error) {
        // 1. 从 URL 解析分享 hash 和子路径
        id, ifPath := ifPathWithName(r)

        // 2. 通过 hash 查找分享记录
        link, err := d.store.Share.GetByHash(id)

        // 3. 验证分享密码/token
        status, err := authenticateShareRequest(r, link)

        // 4. 获取分享所属用户
        user, err := d.store.Users.Get(d.server.Root, link.UserID)

        // 5. 检查用户权限（必须同时有 Share 和 Download 权限）
        if !user.Perm.Share || !user.Perm.Download {
            return http.StatusForbidden, nil
        }

        // 6. 设置用户和作用域文件系统
        d.user = user
        d.user.Fs = files.NewScopedFs(d.user.Fs, basePath)  // 限制在分享目录范围
        d.checkerPrefix = basePath                           // 规则检查前缀

        // 7. 获取文件信息并传递给后续处理器
        d.raw = file
        return fn(w, r, d)
    }
}
```

#### 4.2 authenticateShareRequest - Token/密码验证

**文件**：[public.go](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/public.go#L136-L161)

```go
func authenticateShareRequest(r *http.Request, l *share.Link) (int, error) {
    // 无密码保护 → 直接通过
    if l.PasswordHash == "" {
        return 0, nil
    }

    // 方式1：URL 查询参数中的 token（时序安全比较）
    if subtle.ConstantTimeCompare([]byte(r.URL.Query().Get("token")), []byte(l.Token)) == 1 {
        return 0, nil
    }

    // 方式2：X-SHARE-PASSWORD 请求头中的密码
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
| 有密码分享 | URL `?token=xxx` | `subtle.ConstantTimeCompare(queryToken, link.Token)` |
| 前端 fetch 请求 | `X-SHARE-PASSWORD` header | `bcrypt.CompareHashAndPassword` |

**注意**：`window.open(url)` 下载时，token 通过 URL 查询参数传递（`?token=xxx`），而非请求头。这是因为在密码保护分享首次加载时，前端通过 `X-SHARE-PASSWORD` 头获取到 `token`，后续下载请求直接将此 token 附加到 URL。

#### 4.3 ifPathWithName - URL 路径解析

**文件**：[public.go](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/public.go#L100-L113)

```go
func ifPathWithName(r *http.Request) (id, filePath string) {
    pathElements := strings.Split(r.URL.Path, "/")
    switch len(pathElements) {
    case 1:
        return r.URL.Path, "/"         // 只有 hash，路径为根
    default:
        return pathElements[0], path.Join("/", path.Join(pathElements[1:]...))
    }
}
```

例如 `/api/public/dl/ABC123/sub/file.txt` 经过 `stripPrefix("/api/public/dl/")` 后变成 `ABC123/sub/file.txt`，解析为 `id="ABC123"`, `filePath="/sub/file.txt"`。

### 5. 权限检查与归档输出的关联（公共分享路径）

公共分享路径中的权限检查比普通下载更严格：

| 层级 | 检查点 | 代码位置 | 失败行为 |
|------|--------|---------|---------|
| L1 分享验证 | `authenticateShareRequest` | [public.go#L136](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/public.go#L136) | 401 Unauthorized |
| L2 用户权限 | `user.Perm.Share && user.Perm.Download` | [public.go#L35-L37](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/public.go#L35-L37) | 403 Forbidden |
| L3 文件系统 | `ScopedFs` 限制 | [public.go#L70](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/public.go#L70) | 逃逸符号链接被拦截 |
| L4 规则过滤 | `d.Check(path)`（含 checkerPrefix 还原） | [data.go#L37-L64](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/data.go#L37-L64) | 跳过该文件 |
| L5 下载权限 | `d.user.Perm.Download`（在 rawHandler 中） | [raw.go#L84](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/raw.go#L84) | 返回 202 |

**ScopedFs 的作用**：公共分享时，文件系统被重基到分享目录。这意味着：
- 归档只能包含分享目录内的文件
- 逃逸的符号链接在 `d.user.Fs.Open()` 时被 ScopedFs 拦截（返回 `os.ErrPermission`）
- 在 `getFiles` 中，`d.user.Fs.Stat(path)` 和 `d.user.Fs.Open(path)` 都受 ScopedFs 保护

**checkerPrefix 的作用**：由于文件系统被重基，路径相对于分享根目录。但规则是基于用户原始范围定义的，所以需要通过 `checkerPrefix` 将路径还原后再检查规则。

---

## 三、两条路径的对比

| 对比项 | 路径 A：普通文件列表下载 | 路径 B：公共分享下载 |
|--------|----------------------|-------------------|
| HTTP 端点 | `GET /api/raw/*` | `GET /api/public/dl/{hash}/*` |
| 鉴权方式 | JWT（X-Auth 头 / auth Cookie） | 分享 hash + token（URL 查询参数） |
| 中间件 | `withUser` | `withHashFile` |
| 用户来源 | JWT 中的 `User.ID` | 分享记录中的 `link.UserID` |
| 下载权限检查 | `d.user.Perm.Download` | `user.Perm.Share && user.Perm.Download` + `d.user.Perm.Download` |
| 文件系统 | 用户原始 `d.user.Fs` | `ScopedFs`（重基到分享目录） |
| 规则检查 | 直接检查路径 | 通过 `checkerPrefix` 还原路径后检查 |
| 格式选择 | 单文件跳过弹窗，目录/多文件弹窗 | 单文件跳过弹窗，目录/多文件弹窗 |
| files 参数 | 文件的 `url` 属性（相对路径） | 文件的 `path` 属性（绝对路径） |
| 最终归档逻辑 | `rawDirHandler` | `rawDirHandler`（完全相同） |

**核心设计**：两条路径最终都调用 `rawDirHandler` 和 `rawFileHandler`，区别仅在于鉴权和文件系统作用域的处理。`withHashFile` 通过设置 `d.user`、`d.user.Fs` 和 `d.checkerPrefix`，使公共分享请求在进入 `rawDirHandler` 时拥有与普通请求相同的数据结构。

---

## 四、解压功能

### 当前状态

**项目中尚未实现上传压缩包后自动解压的功能**。

现有的上传相关代码只负责将文件原样写入磁盘，不进行解压处理：

- **普通上传**：[resourcePostHandler](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/resource.go#L125-L178) → `writeFile` 直接写入
- **Tus 分片上传**：[tusPatchHandler](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/tus_handlers.go#L156-L239) 追加写入

### 第三方库能力

虽然没有使用，但 `github.com/mholt/archives` 库本身支持解压操作（`Extract` 方法），如果后续需要实现解压功能，可以基于该库扩展。

---

## 五、安全机制汇总

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

## 六、相关文件列表

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
