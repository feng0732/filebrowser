# FileBrowser 压缩解压处理流程

## 概述

FileBrowser 的压缩解压功能主要围绕**文件归档下载**展开，使用第三方库 `github.com/mholt/archives` 实现多种格式的归档打包。目前项目中**仅实现了压缩（归档下载）功能**，尚未实现上传压缩包后自动解压的功能。

---

## 一、压缩（归档下载）流程

### 1. 整体流程概览

```
前端用户选择文件/目录
        ↓
前端调用 download() 函数，构造 /api/raw 请求
        ↓
HTTP 路由匹配到 rawHandler
        ↓
rawHandler 判断是文件还是目录
        ├─ 文件 → rawFileHandler（直接下载，不压缩）
        └─ 目录 → rawDirHandler（打包压缩后下载）
                ↓
        解析查询参数（files、algo）
                ↓
        递归收集所有待归档文件信息
                ↓
        调用 archives 库的 Archive 方法
                ↓
        流式输出到 HTTP 响应
```

### 2. 前端入口

**文件**：[frontend/src/api/files.ts](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/frontend/src/api/files.ts#L82-L104)

核心函数 `download(format, ...files)`：
- 构造 `/api/raw` 端点的 URL
- `files` 参数：指定要下载的文件路径列表（逗号分隔）
- `algo` 参数：指定压缩格式
- 通过 `window.open(url)` 触发下载

**支持的格式**（前端定义）：
- `zip` → `.zip`
- `tar` → `.tar`
- `targz` → `.tar.gz`
- `tarbz2` → `.tar.bz2`
- `tarxz` → `.tar.xz`
- `tarlz4` → `.tar.lz4`
- `tarsz` → `.tar.sz`
- `tarbr` → `.tar.br`
- `tarzst` → `.tar.zst`

### 3. HTTP 路由

**文件**：[http/http.go](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/http.go#L82)

```go
api.PathPrefix("/raw").Handler(monkey(rawHandler, "/api/raw")).Methods("GET")
```

路由将所有 `/api/raw/*` 的 GET 请求交由 `rawHandler` 处理。

### 4. 核心处理函数

#### 4.1 rawHandler - 入口处理器

**文件**：[http/raw.go](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/raw.go#L83-L110)

处理逻辑：
1. 检查用户下载权限 `d.user.Perm.Download`
2. 创建 `FileInfo` 对象获取文件/目录信息
3. 如果是命名管道，设置 Content-Disposition 后返回
4. 如果是**文件**，调用 `rawFileHandler`
5. 如果是**目录**，调用 `rawDirHandler`

#### 4.2 parseQueryFiles - 解析待下载文件列表

**文件**：[http/raw.go](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/raw.go#L26-L45)

从 URL 查询参数 `files` 中解析出所有待下载的文件路径：
- 如果没有 `files` 参数，默认下载当前路径
- 如果有 `files` 参数，按逗号分割并 URL 解码
- 所有路径都经过 `slashClean` 规范化处理

#### 4.3 parseQueryAlgorithm - 解析压缩算法

**文件**：[http/raw.go](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/raw.go#L47-L70)

根据 `algo` 查询参数返回对应的归档器：

| algo 参数 | 扩展名 | 归档器类型 |
|----------|--------|-----------|
| `zip` / `true` / 空 | `.zip` | `archives.Zip{}` |
| `tar` | `.tar` | `archives.Tar{}` |
| `targz` | `.tar.gz` | `archives.CompressedArchive{Gz, Tar}` |
| `tarbz2` | `.tar.bz2` | `archives.CompressedArchive{Bz2, Tar}` |
| `tarxz` | `.tar.xz` | `archives.CompressedArchive{Xz, Tar}` |
| `tarlz4` | `.tar.lz4` | `archives.CompressedArchive{Lz4, Tar}` |
| `tarsz` | `.tar.sz` | `archives.CompressedArchive{Sz, Tar}` |
| `tarbr` | `.tar.br` | `archives.CompressedArchive{Brotli, Tar}` |
| `tarzst` | `.tar.zst` | `archives.CompressedArchive{Zstd, Tar}` |

#### 4.4 getFiles - 递归收集文件

**文件**：[http/raw.go](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/raw.go#L112-L168)

递归遍历目录，收集所有需要归档的文件信息：

1. 先通过 `d.Check(path)` 检查规则权限，无权限则跳过
2. 获取文件/目录的 `fs.FileInfo`
3. 计算文件在归档中的名称（去掉公共前缀）
4. 对于目录，递归读取子目录中的所有文件
5. 返回 `[]archives.FileInfo` 切片

**安全特性**（防止 Zip Slip 漏洞）：
- 将所有路径分隔符统一为 `/`
- 移除 Windows 风格的反斜杠 `\`，防止跨平台路径遍历攻击

#### 4.5 rawDirHandler - 目录归档处理

**文件**：[http/raw.go](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/raw.go#L170-L216)

主要步骤：
1. 调用 `parseQueryFiles` 解析待下载文件列表
2. 调用 `parseQueryAlgorithm` 解析压缩格式
3. 计算所有文件的公共目录前缀 `commonDir`
4. 调用 `getFiles` 递归收集所有文件信息
5. 确定输出文件名：
   - 单文件/目录：使用公共目录的基名
   - 多文件：文件名前加 `_` 前缀
   - 追加格式扩展名
6. 设置 `Content-Disposition` 响应头
7. 调用 `archiver.Archive(r.Context(), w, allFiles)` 流式输出归档文件

### 5. 公共分享下载

**文件**：[http/public.go](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/public.go#L127-L134)

公共分享链接也支持归档下载：
- `publicDlHandler` 处理 `/api/public/dl/{hash}/*` 路径
- 如果是文件，调用 `rawFileHandler`
- 如果是目录，调用 `rawDirHandler`
- 分享文件系统使用 `ScopedFs` 限制在分享目录范围内

---

## 二、解压功能

### 当前状态

**项目中尚未实现上传压缩包后自动解压的功能**。

现有的上传相关代码只负责将文件原样写入磁盘，不进行解压处理：

- **普通上传**：[resourcePostHandler](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/resource.go#L125-L178) → `writeFile` 直接写入
- **Tus 分片上传**：[tusPatchHandler](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/tus_handlers.go#L156-L239) 追加写入

### 第三方库能力

虽然没有使用，但 `github.com/mholt/archives` 库本身支持解压操作（`Extract` 方法），如果后续需要实现解压功能，可以基于该库扩展。

---

## 三、安全机制

### 1. 权限检查

- 下载前检查 `d.user.Perm.Download` 权限
- 每个文件都经过 `d.Check(path)` 规则检查
- 公共分享使用 `ScopedFs` 限制访问范围

### 2. Zip Slip 防护

**文件**：[http/raw.go](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/raw.go#L124-L134)

在构建归档内路径时：
- 使用 `filepath.ToSlash` 统一路径分隔符
- 显式将所有反斜杠 `\` 替换为 `/`
- 防止恶意构造的路径在 Windows 解压时逃逸出目标目录

### 3. 符号链接安全

- 使用 `ScopedFs` 文件系统包装，在操作前检查符号链接是否指向范围外
- 归档时通过 `d.user.Fs.Open(path)` 打开文件，会受到 ScopedFs 的保护
- 测试用例 [public_symlink_test.go](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/public_symlink_test.go) 验证了符号链接不会导致数据泄露

---

## 四、相关文件列表

| 文件 | 作用 |
|------|------|
| [http/raw.go](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/raw.go) | 压缩下载核心逻辑 |
| [http/http.go](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/http.go) | HTTP 路由注册 |
| [http/public.go](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/public.go) | 公共分享下载处理 |
| [http/resource.go](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/resource.go) | 资源操作（上传等） |
| [http/tus_handlers.go](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/http/tus_handlers.go) | Tus 协议上传处理 |
| [frontend/src/api/files.ts](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/frontend/src/api/files.ts) | 前端文件 API |
| [frontend/src/components/prompts/Download.vue](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/frontend/src/components/prompts/Download.vue) | 下载格式选择弹窗 |
| [files/scoped.go](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/files/scoped.go) | 作用域文件系统（安全） |
| [fileutils/file.go](file:///d:/fz/0601/solo-dogfeeding/code/173-filebrowser/fileutils/file.go) | 文件工具函数 |
