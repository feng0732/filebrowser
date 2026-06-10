# File Browser 文件系统抽象层深度解析

## 一、整体架构概览

File Browser 的文件系统抽象层由三大核心模块协同工作：

```
HTTP 请求层
    ↓
[路径标准化] stripPrefix → path.Clean → checkerPrefix 修正
    ↓
[权限控制]   粗粒度 Perm → 细粒度 rules.Check (data.Check)
    ↓
[作用域约束] ScopedFs.guard/within → afero.BasePathFs
    ↓
[本地文件操作] afero.OsFs → 操作系统文件系统
```

> **关键认识**：路径标准化、权限控制、作用域约束并非只在入口处各执行一次，而是**贯穿整条调用链、在不同层次以不同形式重复出现**，每层都使用前一层输出的路径作为输入。

核心文件分布：
- [files/scoped.go](files/scoped.go) — 作用域文件系统
- [files/file.go](files/file.go) — 文件信息与操作
- [users/permissions.go](users/permissions.go) — 用户权限定义
- [rules/rules.go](rules/rules.go) — 规则匹配引擎
- [http/data.go](http/data.go) — 请求上下文与权限检查实现
- [http/resource.go](http/resource.go) — 资源增删改查 Handler
- [http/utils.go](http/utils.go) — stripPrefix 等路径工具
- [fileutils/](fileutils/) — 高级文件操作（复制/移动）

---

## 二、本地文件操作层

### 2.1 afero 抽象接口

项目采用 `github.com/spf13/afero` 库作为文件系统抽象层。`afero.Fs` 接口定义了标准文件操作：

```go
type Fs interface {
    Create(name string) (File, error)
    Mkdir(name string, perm os.FileMode) error
    MkdirAll(path string, perm os.FileMode) error
    Open(name string) (File, error)
    OpenFile(name string, flag int, perm os.FileMode) (File, error)
    Remove(name string) error
    RemoveAll(path string) error
    Rename(oldname, newname string) error
    Stat(name string) (os.FileInfo, error)
    Name() string
    Chmod(name string, mode os.FileMode) error
    Chown(name string, uid, gid int) error
    Chtimes(name string, atime, mtime time.Time) error
}
```

最底层是 `afero.NewOsFs()`，直接调用操作系统 API。

### 2.2 ScopedFs — 作用域文件系统

[ScopedFs](files/scoped.go#L19-L184) 是核心安全层，封装了 `afero.BasePathFs`，确保所有操作都限制在用户的根目录（Scope）内。

**构造方式**（[users/users.go:92-96](users/users.go#L92-L96)）：

```go
scope := filepath.Join(baseScope, filepath.Join("/", scope))
u.Fs = files.NewScopedFs(afero.NewOsFs(), scope)
```

**双重防护机制**：

1. **词法层防护**：`afero.BasePathFs` 将所有路径自动拼接 base 前缀，从路径字符串层面防止越界。
2. **符号链接防护**：`ScopedFs.guard()` → `ScopedFs.within()` 通过 `filepath.EvalSymlinks` 解析符号链接的真实目标，确保即使符号链接指向外部，也会被拒绝。

关键方法 [within()](files/scoped.go#L64-L94)：

```go
func (s *ScopedFs) within(p string) (bool, error) {
    // 1. 解析作用域根目录的真实路径（跟随所有符号链接）
    root, err := filepath.EvalSymlinks(afero.FullBaseFsPath(s.base, "/"))

    // 2. 解析目标路径的真实路径
    target := afero.FullBaseFsPath(s.base, p)
    resolved, err := filepath.EvalSymlinks(target)

    // 3. 若目标不存在，递归向上找最近存在的父目录
    for errors.Is(err, fs.ErrNotExist) {
        parent := filepath.Dir(target)
        if parent == target { break }
        target = parent
        resolved, err = filepath.EvalSymlinks(target)
    }

    // 4. 前缀比较：确保 resolved 以 root 为前缀
    prefix := root
    if !strings.HasSuffix(prefix, string(filepath.Separator)) {
        prefix += string(filepath.Separator)
    }
    return resolved == root || strings.HasPrefix(resolved, prefix), nil
}
```

**哪些操作会走 guard()**：
- ✅ 走 guard：`Create`、`Mkdir`、`MkdirAll`、`Open`、`OpenFile`、`Rename`、`Stat`、`Chmod`、`Chown`、`Chtimes`、`LstatIfPossible` — 涉及创建、读内容、改元数据、跟随符号链接
- ❌ 不走 guard：`Remove`、`RemoveAll`、`Name` — 删除不跟随符号链接，且已受 BasePathFs 词法约束

### 2.3 FileInfo — 文件信息模型

[FileInfo](files/file.go#L36-L54) 是面向上层的文件表示对象，包含：

| 字段 | 说明 |
|------|------|
| `Fs afero.Fs` | 关联的文件系统（用于后续操作） |
| `Path` | 相对作用域的路径 |
| `Name` / `Size` / `ModTime` | 基本元数据 |
| `IsDir` / `IsSymlink` | 文件类型标记 |
| `Type` | 检测出的类型（video/audio/image/text/blob） |
| `Content` | 文本文件内容（≤10MB） |
| `Checksums` | md5/sha1/sha256/sha512 校验值 |
| `Listing` | 目录列表（若是目录） |

构造入口 [NewFileInfo()](files/file.go#L77-L107)：

```go
func NewFileInfo(opts *FileOptions) (*FileInfo, error) {
    // 第一步：权限检查
    if !opts.Checker.Check(opts.Path) {
        return nil, os.ErrPermission
    }
    // 第二步：获取文件元信息
    file, err := stat(opts)
    // 第三步：按需展开（目录列表 / 文件类型检测）
    if opts.Expand {
        if file.IsDir {
            file.readListing(...)
        } else {
            file.detectType(...)
        }
    }
    return file, err
}
```

### 2.4 fileutils — 高级操作

[fileutils/](fileutils/) 提供了超越 afero 接口的复杂操作：

- **[Copy()](fileutils/copy.go#L12-L40)**：根据源文件类型自动分发到 CopyFile 或 CopyDir
- **[CopyFile()](fileutils/file.go#L34-L73)**：打开源→创建目标目录→复制内容→复制权限位
- **[CopyDir()](fileutils/dir.go#L13-L63)**：递归遍历目录，逐个复制文件
- **[MoveFile()](fileutils/file.go#L16-L30)**：优先使用 Rename 系统调用，跨卷时回退为 Copy+RemoveAll

---

## 三、权限控制机制

### 3.1 两层权限模型

权限控制分为**粗粒度操作权限**和**细粒度路径规则**两层，二者按顺序依次检查。

**粗粒度操作权限**（[Permissions](users/permissions.go#L4-L13)）：

```go
type Permissions struct {
    Admin    bool  // 管理员权限
    Execute  bool  // 执行命令
    Create   bool  // 创建文件/目录
    Rename   bool  // 重命名/移动
    Modify   bool  // 修改文件内容
    Delete   bool  // 删除
    Share    bool  // 创建分享链接
    Download bool  // 下载
}
```

这些在 HTTP Handler **最开头**直接判断，例如 [resourceDeleteHandler](http/resource.go#L84-L88)：

```go
if r.URL.Path == "/" || !d.user.Perm.Delete {
    return http.StatusForbidden, nil
}
```

### 3.2 细粒度路径规则

[Rule](rules/rules.go#L15-L20) 定义了允许/拒绝特定路径的规则：

```go
type Rule struct {
    Regex  bool    // 是否正则匹配
    Allow  bool    // true=白名单, false=黑名单
    Path   string  // 普通路径模式
    Regexp *Regexp // 正则表达式模式
}
```

匹配逻辑 [Matches()](rules/rules.go#L29-L44)：

1. 正则模式：直接用 regexp 匹配整个路径
2. 普通模式：精确相等，或为路径前缀（自动补全末尾 `/`）

### 3.3 Checker 接口与 data.Check()

[rules.Checker](rules/rules.go#L10-L12) 是权限检查的抽象接口：

```go
type Checker interface {
    Check(path string) bool
}
```

HTTP 请求上下文 [data](http/data.go#L20-L34) 实现了该接口，完整流程 [data.Check()](http/data.go#L37-L64)：

```go
func (d *data) Check(path string) bool {
    // 1. 路径前缀修正（这也是一次"路径标准化"）
    //    公共分享场景下，FS被二次rebase，需要还原到用户原始作用域
    if d.checkerPrefix != "" {
        path = gopath.Join(d.checkerPrefix, path)
    }
    // 2. 隐藏文件过滤
    if d.user.HideDotfiles && rules.MatchHidden(path) {
        return false
    }
    // 3. 全局规则（最后一条匹配生效）
    allow := true
    for _, rule := range d.settings.Rules {
        if rule.Matches(path) {
            allow = rule.Allow
        }
    }
    // 4. 用户规则（覆盖全局规则）
    for _, rule := range d.user.Rules {
        if rule.Matches(path) {
            allow = rule.Allow
        }
    }
    return allow
}
```

**规则覆盖原则**：按顺序遍历，最后一条匹配的规则决定结果。用户规则在全局规则之后执行，因此优先级更高。

---

## 四、路径标准化逻辑

### 4.1 标准化的四个层次

路径标准化不是一次性操作，而是贯穿整条链路的多层处理：

| 层次 | 方法 | 作用 | 发生位置 |
|------|------|------|----------|
| L1 路由前缀剥离 | `stripPrefix()` | 移除 `/api/resources` 等路由前缀，得到资源相对路径 | [http/utils.go:55-80](http/utils.go#L55-L80) |
| L2 虚拟路径归一 | `path.Clean("/" + p)` | 移除 `../`、`./`、多余斜杠，确保以 `/` 开头 | 各 Handler 入口（PATCH/搜索等） |
| L3 规则路径修正 | `gopath.Join(checkerPrefix, path)` | 分享场景下还原到用户原始作用域路径 | [http/data.go:42-44](http/data.go#L42-L44) |
| L4 真实路径解析 | `afero.FullBaseFsPath + filepath.EvalSymlinks` | 虚拟路径→真实磁盘路径→跟随符号链接 | [files/scoped.go:65-94](files/scoped.go#L65-L94) |

### 4.2 L1：路由层前缀剥离 stripPrefix

路由注册时（[http/http.go:60-65](http/http.go#L60-L65)）：

```go
api.PathPrefix("/resources").Handler(monkey(resourceGetHandler, "/api/resources")).Methods("GET")
```

`handle()` → `stripPrefix(prefix, handler)` 将 `/api/resources/docs/file.txt` 剥离为 `/docs/file.txt`，存入 `r.URL.Path`。这是所有资源 Handler 收到的原始路径。

### 4.3 L2：Handler 内显式归一化 slashClean / path.Clean

**并不是所有 Handler 都显式做 L2**：
- **GET /resources**、**DELETE /resources**、**POST /resources**、**PUT /resources**：直接使用路由剥离后的 `r.URL.Path`，主要依赖后续规则检查和文件系统作用域兜底
- **PATCH /resources**（重命名/复制）：显式 `path.Clean("/" + src)` 和 `path.Clean("/" + dst)`（因为 dst 来自 query 参数，不可信）
- **搜索**：`filepath.ToSlash + filepath.Clean + path.Join("/", scope)` 三重归一
- **raw 下载子文件选择**：`slashClean()` 工具函数（[http/raw.go:19-24](http/raw.go#L19-L24)）

```go
func slashClean(name string) string {
    if name == "" || name[0] != '/' {
        name = "/" + name
    }
    return gopath.Clean(name)
}
```

### 4.4 L3：规则检查时的 checkerPrefix 修正

仅公共分享场景生效。当用户 FS 被二次 rebase 到分享子目录时（[http/public.go:70-75](http/public.go#L70-L75)）：

```go
d.user.Fs = files.NewScopedFs(d.user.Fs, basePath)   // FS 的根变成 ./shared/docs
d.checkerPrefix = basePath                            // 例如 /shared/docs
```

后续 `d.Check("/sub/file.txt")` 会在内部拼接为 `/shared/docs/sub/file.txt`，再去匹配用户原始规则。

### 4.5 L4：ScopedFs 内部虚拟→真实路径转换

ScopedFs 内部维护了**虚拟路径**与**真实磁盘路径**的映射：

- 虚拟路径（Handler 视角）：`/docs/report.pdf`（相对于用户 Scope）
- 真实路径（OS 视角）：`/home/user/files/docs/report.pdf`

转换通过 `afero.FullBaseFsPath(baseFs, vpath)` 完成（[files/scoped.go:65-70](files/scoped.go#L65-L70)）：

```go
root, err := filepath.EvalSymlinks(afero.FullBaseFsPath(s.base, "/"))  // 真实根目录
target := afero.FullBaseFsPath(s.base, p)  // 目标文件的真实路径
resolved, err := filepath.EvalSymlinks(target)  // 跟随所有符号链接
```

### 4.6 目录列表中的路径构造

目录遍历时，子路径通过 `path.Join` 构造（[files/file.go:407](files/file.go#L407)）：

```go
fPath := path.Join(i.Path, name)
if !checker.Check(fPath) {
    continue  // 权限不通过的文件直接跳过
}
```

---

## 五、路径标准化 → 权限控制 → 作用域约束：精确顺序关系

### 5.1 通用执行顺序模型

任何一次文件操作都会按以下顺序逐层穿过抽象栈。每一层的输出是下一层的输入：

```
用户输入 (URL Path / Query 参数)
    │
    ▼  【L1 路由层】stripPrefix
    │  剥离 /api/resources 等前缀
    │  输出：/docs/../secret/./file.txt
    │
    ▼  【L2 Handler 层】path.Clean("/" + p)   ← 仅部分 Handler 显式执行
    │  清理 ../ ./ 多余斜杠
    │  输出：/secret/file.txt
    │
    ▼  【权限控制第1关】粗粒度 Perm
    │  d.user.Perm.Delete / Create / Modify ...
    │  不通过 → 直接返回 403
    │
    ▼  【权限控制第2关】细粒度 rules.Check
    │  d.Check(path)：
    │    ├─ L3 标准化：gopath.Join(checkerPrefix, path)  ← 分享场景
    │    ├─ 隐藏文件过滤
    │    ├─ 全局规则匹配
    │    └─ 用户规则匹配
    │  不通过 → 返回 403 或被跳过
    │
    ▼  【作用域约束第1关】ScopedFs.guard()   ← 仅 Create/Open/Stat/Rename 等
    │  within(path)：
    │    ├─ L4 标准化：FullBaseFsPath 虚拟→真实路径
    │    ├─ L4 标准化：EvalSymlinks 跟随符号链接
    │    └─ 前缀比较：真实路径是否仍在作用域根下
    │  不通过 → 返回 os.ErrPermission
    │
    ▼  【作用域约束第2关】afero.BasePathFs
    │  词法层面：自动拼接 base 前缀，禁止 ../ 逃逸
    │
    ▼  【OS 层】afero.OsFs
       实际系统调用
```

**核心规律**：
1. **路径标准化发生在每一层的入口**，并非只在最外层做一次。
2. **权限控制分为"先粗后细"**：先判断操作类型（能不能删除），再判断具体路径（能不能删这个文件）。
3. **作用域约束是最后一道防线**，即使上层所有检查都通过，文件系统层仍会做符号链接越界检查。

### 5.2 为什么有些 Handler 不显式 path.Clean？

观察代码会发现 DELETE/GET/POST/PUT 的主路径直接用 `r.URL.Path`，不额外 `path.Clean`。原因：

1. 路由剥离后 `r.URL.Path` 已经是应用资源路径，普通入口直接把它交给后续权限和文件系统层处理
2. 即便路径里包含 `../` 这类片段，后续进入 `ScopedFs` 时：
   - BasePathFs 会做词法层面的 base 拼接，`../` 无法逃逸
   - `guard()` → `within()` 中 `EvalSymlinks` 会再次做真实路径校验
3. 但 **PATCH 的 dst 参数、搜索的 query 参数、raw 下载的 files 参数**来自 query string，不可信，必须显式归一化。

---

## 六、删除操作完整调用链（逐行追踪）

以 [resourceDeleteHandler](http/resource.go#L84-L123) 为例，逐层追踪每一步：

```
HTTP DELETE /api/resources/docs/old-report.pdf
    │
    ▼
【阶段 0】路由与中间件
│
├─ ① gorilla/mux 匹配路由：PathPrefix("/resources") + Method("DELETE")
├─ ② stripPrefix("/api/resources") → r.URL.Path = "/docs/old-report.pdf"   [L1 标准化]
├─ ③ withUser() 中间件：JWT 解析 → d.store.Users.Get() → 加载 d.user
│     d.user.Fs 是已构造好的 ScopedFs，作用域为用户主目录
│
▼
【阶段 1】粗粒度权限
│
└─ ④ [resource.go:86] 检查操作权限
     if r.URL.Path == "/" || !d.user.Perm.Delete → 403
     （注意：这里直接用 r.URL.Path 判断是否根目录，无需归一，因为 "/" 形式唯一）
│
▼
【阶段 2】细粒度规则 + 文件存在性校验（通过 NewFileInfo 一次完成）
│
└─ ⑤ [resource.go:90-97] 调用 files.NewFileInfo
     {
       Fs:      d.user.Fs,
       Path:    "/docs/old-report.pdf",      ← 输入路径仍为 r.URL.Path
       Checker: d,
       Expand:  false,
     }
     │
     ├─ ⑤-1 [file.go:78] 细粒度规则检查
     │      if !d.Check("/docs/old-report.pdf") → os.ErrPermission
     │        │
     │        └─ data.Check 内部：
     │           ├─ checkerPrefix == ""，跳过 L3 修正
     │           ├─ 隐藏文件判断
     │           ├─ 遍历全局规则
     │           └─ 遍历用户规则 → 返回 allow
     │
     └─ ⑤-2 [file.go:82] stat(opts) 取文件元信息
            │
            ├─ ScopedFs.LstatIfPossible("/docs/old-report.pdf")
            │     │
            │     ├─ guard("/docs/old-report.pdf")   [作用域约束第1关]
            │     │     └─ within()
            │     │          ├─ FullBaseFsPath → "/home/user/files/docs/old-report.pdf"
            │     │          ├─ EvalSymlinks → "/home/user/files/docs/old-report.pdf"
            │     │          └─ 前缀比较：仍在 scope 内 → OK
            │     │
            │     └─ BasePathFs.LstatIfPossible     [作用域约束第2关 + OS 调用]
            │          └─ afero.OsFs.Lstat
            │
            └─ 构造 FileInfo{ Path:"/docs/old-report.pdf", IsDir:false, ... }
│
▼
【阶段 3】清理关联数据
│
├─ ⑥ [resource.go:102] 删除关联分享记录
│     d.store.Share.DeleteWithPathPrefix(file.Path, d.user.ID)
│
└─ ⑦ [resource.go:108] 删除缩略图缓存
      delThumbs(ctx, fileCache, file)
│
▼
【阶段 4】实际文件删除
│
└─ ⑧ [resource.go:113-115] d.RunHook 中执行删除
     d.user.Fs.RemoveAll("/docs/old-report.pdf")
     │
     └─ ScopedFs.RemoveAll()
          │  ⚠️  RemoveAll 不走 guard()（见 2.2 节说明）
          │
          └─ BasePathFs.RemoveAll()     [词法约束：拼接 base，../ 无法逃逸]
               └─ afero.OsFs.RemoveAll → 操作系统调用
│
▼
返回 HTTP 204 No Content
```

**删除操作的关键特征**：
- 粗粒度权限（Perm.Delete）在最前面，挡住无删除权的用户
- `NewFileInfo` 同时完成了「规则校验」和「作用域校验」，并确认文件真实存在
- 最终的 `RemoveAll` 不做符号链接 guard，但 BasePathFs 的词法约束仍在

---

## 七、保存/上传操作完整调用链（逐行追踪）

保存操作有两个入口：
- **POST /api/resources/path** — 创建新文件（或覆盖上传）
- **PUT /api/resources/path** — 修改已有文件内容

以下以 **POST 上传新文件** 为例（[resourcePostHandler](http/resource.go#L125-L178)）：

```
HTTP POST /api/resources/uploads/new-photo.jpg
Body: <二进制文件内容>
    │
    ▼
【阶段 0】路由与中间件
│
├─ ① stripPrefix("/api/resources") → r.URL.Path = "/uploads/new-photo.jpg"
├─ ② withUser() → 加载 d.user 和 d.user.Fs(ScopedFs)
│
▼
【阶段 1】粗粒度 + 细粒度权限
│
└─ ③ [resource.go:127] 双重权限检查（与删除不同，两个检查并列在最开头）
     if !d.user.Perm.Create || !d.Check(r.URL.Path) → 403
        │              │
        │              └─ data.Check("/uploads/new-photo.jpg")   [细粒度规则]
        │                    ├─ checkerPrefix == "" → 跳过
        │                    ├─ 隐藏文件判断
        │                    ├─ 全局规则
        │                    └─ 用户规则
        │
        └─ d.user.Perm.Create == true ?   [粗粒度操作权限]
│
▼
【阶段 2】分支：目录 vs 文件
│
├─ 若路径以 "/" 结尾 → 直接 MkdirAll（略，简单场景）
│
└─ 文件上传场景继续 ↓
│
▼
【阶段 3】文件存在性 & 覆盖权限
│
└─ ④ [resource.go:137-159] 尝试 NewFileInfo 检测冲突
     files.NewFileInfo({ Path: "/uploads/new-photo.jpg", Checker: d, Expand: false })
     │
     ├─ 4-1 Checker.Check(path) → 规则 OK
     ├─ 4-2 stat(opts)
     │     ├─ ScopedFs.LstatIfPossible
     │     │    ├─ guard() → within() → EvalSymlinks → OK   [作用域约束]
     │     │    └─ BasePathFs.LstatIfPossible → OsFs.Lstat
     │     │
     │     └─ 两种结果：
     │        ├─ 文件不存在（err != nil）→ 视为新上传，跳过覆盖检查
     │        └─ 文件已存在（err == nil）→ 继续检查：
     │              ├─ override != true → 返回 409 Conflict
     │              └─ override == true 且 !d.user.Perm.Modify → 返回 403
     │
     └─ 文件已存在且允许覆盖 → delThumbs() 清旧缩略图
│
▼
【阶段 4】实际写入文件
│
└─ ⑤ [resource.go:161-170] d.RunHook 中执行 writeFile
     writeFile(d.user.Fs, "/uploads/new-photo.jpg", r.Body, fileMode, dirMode)
     │
     ├─ 5-1 [resource.go:298] 确保父目录存在
     │      dir, _ := path.Split(dst)   → "/uploads/"
     │      err := afs.MkdirAll(dir, dirMode)
     │        │
     │        └─ ScopedFs.MkdirAll("/uploads/", dirMode)
     │             ├─ guard("/uploads/")     [作用域约束 ①]
     │             │    └─ within(): FullBaseFsPath + EvalSymlinks + 前缀比较
     │             └─ BasePathFs.MkdirAll → OsFs.MkdirAll
     │
     ├─ 5-2 [resource.go:303] 打开/创建目标文件
     │      afs.OpenFile(dst, O_RDWR|O_CREATE|O_TRUNC, fileMode)
     │        │
     │        └─ ScopedFs.OpenFile("/uploads/new-photo.jpg", ...)
     │             ├─ guard("/uploads/new-photo.jpg")   [作用域约束 ②]
     │             │    └─ within(): FullBaseFsPath + EvalSymlinks + 前缀比较
     │             └─ BasePathFs.OpenFile → OsFs.OpenFile → 返回文件句柄
     │
     ├─ 5-3 [resource.go:309] io.Copy(file, in)        写入请求体
     ├─ 5-4 [resource.go:316] file.Sync()               强制落盘
     └─ 5-5 [resource.go:321] file.Stat()               获取最终元信息（生成 ETag）
│
▼
【阶段 5】回滚 & 响应
│
├─ ⑥ 写入失败 → d.user.Fs.RemoveAll(path) 清理半写入文件
└─ ⑦ 成功 → 响应头写入 ETag，返回状态码
```

**PUT 修改已有文件**的流程更简洁（[resourcePutHandler](http/resource.go#L180-L210)）：
- 权限检查用 `Perm.Modify`（不是 `Perm.Create`）
- 用 `afero.Exists()` 检查文件是否存在，不存在返回 404
- 后续 `writeFile` 流程完全相同

**保存操作的关键特征**：
- 权限检查是「粗粒度 Perm.Create + 细粒度 d.Check」并列在 Handler 最开头
- `writeFile` 内部会触发**两次** ScopedFs.guard：一次 MkdirAll、一次 OpenFile
- 每次 guard 内部都会走完整的 L4 路径标准化（FullBaseFsPath + EvalSymlinks + 前缀比较）

---

## 八、其他协作场景

### 8.1 公共分享：双重 ScopedFs + checkerPrefix

[public.go](http/public.go#L17-L98) 展示了更复杂的嵌套：

```
用户原始配置：
  Scope = /home/user/files

分享链接配置：
  link.Path = /shared/docs

嵌套结构：
afero.OsFs
  └─ ScopedFs (作用域 /home/user/files)          ← 用户原始 Fs
       └─ ScopedFs (作用域 /home/user/files/shared/docs)  ← withHashFile 中再次 NewScopedFs
```

同时设置 `d.checkerPrefix = "/shared/docs"`，确保后续 `d.Check("/sub/file.txt")` 会被还原为 `/shared/docs/sub/file.txt` 再匹配规则，避免分享场景下用户规则失效。

### 8.2 重命名/移动：双向路径检查

[resourcePatchHandler](http/resource.go#L212-L262) 中，源和目标路径都要经过完整链路：

```go
dst = path.Clean("/" + dst)   // L2 标准化（dst 来自 query，必须显式归一）
src = path.Clean("/" + src)   // L2 标准化
if !d.Check(src) || !d.Check(dst) {      // 两条路径都过细粒度规则
    return http.StatusForbidden, nil
}
// ...
return d.user.Fs.Rename(oldname, newname)  // ScopedFs.Rename 对两条路径都做 guard()
```

ScopedFs.Rename 的实现（[files/scoped.go:139-147](files/scoped.go#L139-L147)）：

```go
func (s *ScopedFs) Rename(oldname, newname string) error {
    if err := s.guard(oldname); err != nil { return err }  // 源作用域检查
    if err := s.guard(newname); err != nil { return err }  // 目标作用域检查
    return s.base.Rename(oldname, newname)
}
```

---

## 九、断点续传（TUS 协议）完整协作流程

TUS (Transloadit Upload Specification) 是实现大文件断点续传的标准协议。File Browser 实现了完整的 4 个 TUS 端点：

| HTTP 方法 | 端点 | 作用 | 对应 Handler |
|-----------|------|------|-------------|
| POST | `/api/tus/*path` | 创建上传会话（分片初始化） | [tusPostHandler](http/tus_handlers.go#L41-L123) |
| HEAD | `/api/tus/*path` | 查询上传进度（断点恢复用） | [tusHeadHandler](http/tus_handlers.go#L125-L154) |
| PATCH | `/api/tus/*path` | 追加写入分片数据 | [tusPatchHandler](http/tus_handlers.go#L156-L239) |
| DELETE | `/api/tus/*path` | 取消上传（清理部分文件） | [tusDeleteHandler](http/tus_handlers.go#L241-L273) |

### 9.1 UploadCache：上传状态缓存抽象

断点续传的核心在于**在多次 HTTP 请求之间维持上传状态**。[UploadCache](http/upload_cache_memory.go#L17-L32) 是状态存储的抽象接口：

```go
type UploadCache interface {
    Register(filePath string, fileSize int64)   // 登记新上传（预期总大小）
    Complete(filePath string)                  // 上传完成/取消，删除缓存条目
    GetLength(filePath string) (int64, error)  // 查询预期总大小（用于验证会话）
    Touch(filePath string)                     // 刷新 TTL，防止过期
    Close()                                    // 清理资源
}
```

有两种实现：

**1) memoryUploadCache — 单实例内存缓存**（[upload_cache_memory.go:34-74](http/upload_cache_memory.go#L34-L74)）
- 底层使用 `ttlcache.Cache[string, int64]`，key = 真实磁盘路径，value = 预期文件总大小
- TTL = **3 分钟**
- **GetLength 不自动刷新 TTL**（[upload_cache_memory.go:60-66](http/upload_cache_memory.go#L60-L66)）：仅调用 `c.cache.Get(filePath)`，不调用 Touch，因此 HEAD 查询进度不会刷新内存缓存的 TTL
- **过期自动清理**（[upload_cache_memory.go:41-46](http/upload_cache_memory.go#L41-L46)）：`OnEviction` 回调中检测到 `EvictionReasonExpired` 时，调用 `os.Remove(item.Key())` 删除未完成的部分文件。⚠️ 注意：这里直接调用 `os.Remove`（操作系统原生 API），**绕过了 ScopedFs 作用域检查**，但因为只有经过 POST/NewFileInfo 验证的合法路径才会被存入缓存，所以是安全的

**2) redisUploadCache — 多副本 Redis 缓存**（[upload_cache_redis.go:14-L84](http/upload_cache_redis.go#L14-L84)）
- key 格式：`filebrowser:upload:<真实磁盘路径>`
- **GetLength 自动刷新 TTL**（[upload_cache_redis.go:56-73](http/upload_cache_redis.go#L56-L73)）：第 70 行明确调用 `c.Touch(filePath)`，因此 HEAD/PATCH 查询时会自动刷新 TTL
- **无过期文件清理钩子**：Redis 实现没有 `OnEviction` 回调，key 过期自动删除后，对应的磁盘部分文件不会被自动清理。下次 PATCH 命中时会因缓存不存在返回 404，由客户端决定重新上传（新的 POST 会以 O_TRUNC 截断覆盖）

构造入口 [NewUploadCache()](http/upload_cache_memory.go#L76-L85)：配置了 Redis URL 就用 Redis，否则用内存缓存。

### 9.2 keepUploadActive：传输中防过期守护

[tus_handlers.go:19-39](http/tus_handlers.go#L19-L39) 提供了一个在 PATCH 大分片传输期间持续刷新 TTL 的守护 goroutine：

```go
func keepUploadActive(cache UploadCache, filePath string) func() {
    stop := make(chan bool)
    go func() {
        ticker := time.NewTicker(2 * time.Second)   // 每 2 秒
        for {
            select {
            case <-stop: return
            case <-ticker.C: cache.Touch(filePath)   // 刷新 TTL
            }
        }
    }()
    return func() { close(stop) }   // 返回停止函数，PATCH 结束时 defer 调用
}
```

**为什么需要显式保活**：
- 对于内存缓存：GetLength 不自动 Touch，HEAD 查询也不会刷新 TTL。如果只有 HEAD 轮询没有 PATCH，3 分钟后会话会过期。
- 对于 Redis 缓存：虽然 GetLength 自动 Touch，但单个 PATCH 大分片传输可能超过 3 分钟，传输过程中没有其他请求刷新 TTL，需要守护 goroutine 持续刷新。

---

### 9.3 步骤一：分片创建（POST 创建上传会话）

对应 [tusPostHandler](http/tus_handlers.go#L41-L123)，逐阶段追踪：

```
HTTP POST /api/tus/uploads/large-video.mp4?override=true
Headers:
  Upload-Length: 524288000         (文件总大小 500MB)
    │
    ▼
【阶段 0】路由 & 认证
│
├─ ① stripPrefix("/api/tus") → r.URL.Path = "/uploads/large-video.mp4"
├─ ② withUser() → 加载 d.user 和 ScopedFs
│
▼
【阶段 1】权限检查（与普通 POST 上传一致）
│
└─ ③ [tus_handlers.go:43-45](http/tus_handlers.go#L43-L45)
     if !d.user.Perm.Create || !d.Check(r.URL.Path) → 403
        │                    │
        │                    └─ L3 标准化 + 细粒度规则检查
        └─ 粗粒度创建权限
│
▼
【阶段 2】文件存在性分析 & 父目录自动创建
│
└─ ④ [tus_handlers.go:47-65](http/tus_handlers.go#L47-L65) files.NewFileInfo 探测目标路径
     {
       err == afero.ErrFileNotFound
       │  → 取父目录 filepath.Dir(path)
       │     └─ 父目录也不存在？→ d.user.Fs.MkdirAll(dir, DirMode)
       │           └─ MkdirAll 走完整链路：guard → BasePathFs.MkdirAll
       │
       err != nil → return errToStatus(err), err
       │             errToStatus 转换：权限拒绝→403 / 不存在→404 / 其他→500
       err == nil → 文件已存在，进入覆盖判断
     }
│
▼
【阶段 3】覆盖权限 & 打开标志确定
│
└─ ⑤ [tus_handlers.go:67-86](http/tus_handlers.go#L67-L86)
     fileFlags = O_CREATE | O_WRONLY
     if 文件已存在 {
         if 文件是目录 → 400 Bad Request
         // ⚠️  override 来自 URL Query 参数，不是 Header！
         if r.URL.Query().Get("override") != "true" → 409 Conflict
         if !d.user.Perm.Modify → 403   ← 覆盖需要修改权限
         fileFlags |= O_TRUNC            ← 截断覆盖
     }
│
▼
【阶段 4】创建空文件（或截断已有文件）
│
└─ ⑥ [tus_handlers.go:88-92](http/tus_handlers.go#L88-L92)
     openFile, err := d.user.Fs.OpenFile(path, fileFlags, FileMode)
        │
        └─ ScopedFs.OpenFile()
             ├─ guard(path) → within() → FullBaseFsPath + EvalSymlinks + 前缀比较
             │      └─ 作用域内含符号链接 → 越界会被拒绝（见 tus_symlink_test.go 验证）
             └─ BasePathFs.OpenFile → OsFs.OpenFile
                  └─ 此时文件在磁盘上已存在，大小为 0（或被截断为 0）
     defer openFile.Close()
│
▼
【阶段 5】登记上传会话到缓存
│
├─ ⑦ [tus_handlers.go:94-105](http/tus_handlers.go#L94-L105) 再次 NewFileInfo 取 RealPath
│     file.RealPath() → 磁盘绝对路径（如 /home/user/files/uploads/large-video.mp4）
│
├─ ⑧ [tus_handlers.go:107-110](http/tus_handlers.go#L107-L110) getUploadLength(r)
│     解析 Header "Upload-Length" → uploadLength = 524288000
│
└─ ⑨ [tus_handlers.go:113](http/tus_handlers.go#L113) 缓存登记
     cache.Register(file.RealPath(), uploadLength)
        │
        ├─ memoryCache: ttlcache.Set(realPath, 524288000, ttl=3min)
        └─ redisCache:  SET "filebrowser:upload:/home/..." "524288000" EX 180
│
▼
【阶段 6】返回 Location，会话创建完成
│
└─ ⑩ [tus_handlers.go:115-121](http/tus_handlers.go#L115-L121)
     构造 Location Header = <BaseURL>/api/tus/<EscapedPath>
     返回 HTTP 201 Created
```

**POST 关键点**：
- 此时**磁盘上已创建了目标文件（空或被截断）**，上传会话也登记在缓存中
- **缓存 key 用的是 `file.RealPath()`（真实磁盘路径）**，不是虚拟路径，因为跨请求需要唯一且稳定的标识
- 后续 HEAD/PATCH/DELETE 都需要先通过 `NewFileInfo().RealPath()` 查找到对应缓存条目

---

### 9.4 步骤二：查询上传进度（HEAD 断点恢复）

对应 [tusHeadHandler](http/tus_handlers.go#L125-L154)，客户端断线重连时调用：

```
HTTP HEAD /api/tus/uploads/large-video.mp4
    │
    ▼
① [tus_handlers.go:128-130](http/tus_handlers.go#L128-L130)
   权限：!Perm.Create || !d.Check(path) → 403
    │
    ▼
② [tus_handlers.go:132-142](http/tus_handlers.go#L132-L142)
   NewFileInfo(path) → 取得文件当前 file.Size
   │
   └─ 出错 → return errToStatus(err), err
         errToStatus 转换规则：
         ├─ os.ErrPermission → 403 Forbidden（规则或作用域拒绝）
         ├─ os.ErrNotExist   → 404 Not Found（文件不存在，可能被清理）
         └─ 其他错误         → 500 Internal Server Error
    │
    ▼
③ [tus_handlers.go:144-147](http/tus_handlers.go#L144-L147)
   cache.GetLength(file.RealPath())
   ├─ 缓存不存在 → 明确返回 http.StatusNotFound (404)
   │              （会话已过期或从未创建，需要从 POST 重新开始）
   ├─ 内存缓存命中 → 返回 uploadLength，⚠️ 不自动刷新 TTL
   └─ Redis 缓存命中 → 返回 uploadLength，✅ 自动调用 Touch 刷新 TTL
    │
    ▼
④ [tus_handlers.go:149-152](http/tus_handlers.go#L149-L152)
   返回 Headers：
   Upload-Offset: 104857600   ← 当前文件大小 = 已上传进度 100MB
   Upload-Length: 524288000   ← 文件总大小 500MB
   Cache-Control: no-store
   HTTP 200 OK
```

客户端比较 `Upload-Offset` 与自己记录的进度，下次 PATCH 从对应偏移量继续。

**HEAD 行为差异**：使用内存缓存时，HEAD 查询**不会**刷新 TTL；使用 Redis 缓存时，HEAD 查询**会**自动刷新 TTL。

---

### 9.5 步骤三：追加写入（PATCH 分片上传）

对应 [tusPatchHandler](http/tus_handlers.go#L156-L239)，这是实际传输数据的阶段：

```
HTTP PATCH /api/tus/uploads/large-video.mp4
Headers:
  Content-Type: application/offset+octet-stream
  Upload-Offset: 104857600       (从 100MB 处继续)
Body: <10MB 二进制数据>
    │
    ▼
【阶段 1】权限 & Content-Type 校验
│
├─ ① [tus_handlers.go:158-160] !Perm.Create || !d.Check(path) → 403
└─ ② [tus_handlers.go:161-163] Content-Type != "application/offset+octet-stream" → 415
│
▼
【阶段 2】读取偏移量 & 验证文件与会话
│
├─ ③ [tus_handlers.go:165-168] getUploadOffset(r)
│     解析 Header "Upload-Offset" → uploadOffset = 104857600
│
├─ ④ [tus_handlers.go:170-184](http/tus_handlers.go#L170-L184) NewFileInfo(path)
│     │
│     ├─ d.Check(path) → 细粒度规则检查
│     ├─ ScopedFs.LstatIfPossible → guard/within 作用域检查
│     └─ 出错 → return errToStatus(err), err
│           errToStatus 转换：权限拒绝→403 / 不存在→404 / 其他→500
│
├─ ⑤ [tus_handlers.go:186-189](http/tus_handlers.go#L186-L189) cache.GetLength(file.RealPath())
│     ├─ 缓存不存在 → 明确返回 http.StatusNotFound (404)
│     │              （会话过期，需要重新创建 POST）
│     ├─ 内存缓存命中 → uploadLength = 524288000，⚠️ 不自动刷新 TTL
│     └─ Redis 缓存命中 → uploadLength = 524288000，✅ 自动调用 Touch 刷新 TTL
│
└─ ⑥ [tus_handlers.go:192-193] 启动保活
     stop := keepUploadActive(cache, file.RealPath())
     defer stop()   ← 每 2 秒 Touch 一次，防止大分片传输中缓存过期
│
▼
【阶段 3】偏移量一致性校验（TUS 协议核心）
│
└─ ⑦ [tus_handlers.go:195-204] 一致性核对：
     ├─ file.IsDir → 400（不能写入目录）
     └─ file.Size != uploadOffset
          → 409 Conflict
            说明：客户端声称的偏移量 ≠ 文件实际大小
            可能场景：其他客户端已推进进度 / 部分写入被回滚 / 缓存错乱
│
▼
【阶段 4】追加写入数据
│
├─ ⑧ [tus_handlers.go:206-210] 以追加写模式打开文件
│     d.user.Fs.OpenFile(path, O_WRONLY|O_APPEND, FileMode)
│        └─ ScopedFs.OpenFile → guard → within → BasePathFs → OsFs
│
├─ ⑨ [tus_handlers.go:212-215] Seek 到约定偏移量（双重保险，尽管 O_APPEND 总是追加到末尾）
│     openFile.Seek(uploadOffset, 0)
│        └─ 理论上 O_APPEND 会忽略 Seek 直接写末尾，但显式 Seek 到 offset 做一致性验证
│
├─ ⑩ [tus_handlers.go:217-221] 写入分片数据
│     bytesWritten, _ = io.Copy(openFile, r.Body)   ← 将请求体全部写入文件
│        └─ 这一步实际向磁盘追加了 10MB 数据
│
└─ ⑪ [tus_handlers.go:225-227] 强制落盘
     openFile.Sync()
│
▼
【阶段 5】进度更新 & 完成判断
│
├─ ⑫ [tus_handlers.go:229-230] 计算新偏移量
│     newOffset = uploadOffset + bytesWritten = 104857600 + 10485760 = 115343360
│     响应头 Upload-Offset: 115343360
│
└─ ⑬ [tus_handlers.go:232-235] 判断是否传输完成
     if newOffset >= uploadLength:
         cache.Complete(file.RealPath())   ← 删除缓存条目，停止 TTL 计时
         d.RunHook(noop, "upload", ...)   ← 触发 upload 钩子（通知系统上传完成）
│
▼
返回 HTTP 204 No Content
```

**PATCH 关键点**：
- `file.Size != uploadOffset` 检查是**协议级一致性保证**，防止错乱写入
- `keepUploadActive` 每 2 秒显式刷新一次 TTL，无论哪种缓存实现都能有效防止大分片传输中会话意外过期
- 完成判断用 `newOffset >= uploadLength`（而不是 `==`），容忍客户端多发送字节（协议兼容性）
- Redis 缓存时 `GetLength` 自动 Touch，内存缓存时则完全依赖 `keepUploadActive` 守护刷新 TTL

---

### 9.6 步骤四：取消上传（DELETE 清理）

对应 [tusDeleteHandler](http/tus_handlers.go#L241-L273)，客户端主动中断：

```
HTTP DELETE /api/tus/uploads/large-video.mp4
    │
    ▼
【阶段 1】权限
│
└─ ① [tus_handlers.go:243-245](http/tus_handlers.go#L243-L245)
     path == "/" || !d.user.Perm.Delete → 403
     注意：删除权限（不是 Create），因为会删除已创建的部分文件
│
▼
【阶段 2】确认文件与会话存在
│
├─ ② [tus_handlers.go:247-257](http/tus_handlers.go#L247-L257) NewFileInfo(path)
│     d.Check(path) → 细粒度规则
│     ScopedFs.LstatIfPossible → 作用域检查
│     出错 → return errToStatus(err), err
│           errToStatus 转换：权限拒绝→403 / 不存在→404 / 其他→500
│
└─ ③ [tus_handlers.go:259-262](http/tus_handlers.go#L259-L262) cache.GetLength(file.RealPath())
      缓存不存在 → 404（会话已完成或已过期，不允许取消）
│
▼
【阶段 3】实际清理
│
├─ ④ [tus_handlers.go:264-267](http/tus_handlers.go#L264-L267) 删除磁盘文件
│     d.user.Fs.RemoveAll(path)
│        └─ ScopedFs.RemoveAll → BasePathFs.RemoveAll → OsFs.RemoveAll
│           （⚠️ RemoveAll 不走 guard，但上层已通过 NewFileInfo 验证作用域合法）
│
└─ ⑤ [tus_handlers.go:269](http/tus_handlers.go#L269) 删除缓存条目
     cache.Complete(file.RealPath())
│
▼
返回 HTTP 204 No Content
```

---

### 9.7 TUS 各步骤之间的状态流转

```
                (客户端发起)
                     │
                     ▼
        ┌─── POST 创建会话 ───┐
        │  cache.Register()   │
        │  创建空文件(0B)     │
        │  返回 201 + Location│
        └─────────┬───────────┘
                  │
        ┌─────────▼───────────┐
        │                     │
        ▼                     ▼
  HEAD 查询进度          PATCH 追加写
  cache.GetLength()      cache.GetLength()
  返回 Upload-Offset     keepUploadActive 保活
  内存: 不刷新TTL        Offset == file.Size 校验
  Redis: 自动刷新TTL     追加写入 + Sync
  (404=会话过期)         newOffset >= total
                        → cache.Complete() + 钩子
                              │
        ┌─────────────────────┤
        │                     │
        ▼                     ▼
  (继续 PATCH 循环)    DELETE 取消上传
                   cache.GetLength()
                   RemoveAll() 删除文件
                   cache.Complete()
```

**状态缓存的生命周期与触发事件**：

| 事件 | cache 操作 | 内存缓存 TTL 刷新 | Redis 缓存 TTL 刷新 | 磁盘文件变化 |
|------|-----------|-------------------|---------------------|-------------|
| POST 创建 | `Register(realPath, size)` → TTL=3min | ✅ Set 时启动 | ✅ Set 时启动 | 空文件创建 / 截断为 0 |
| HEAD 查询 | `GetLength()` | ❌ 不刷新 | ✅ 自动 Touch | 无 |
| PATCH 开始 | `GetLength()` + `keepUploadActive` 每 2s Touch | ❌ GetLength 不刷新，但后续守护 goroutine 会 | ✅ GetLength 自动 Touch + 守护 goroutine | 无 |
| PATCH 写入 | — | — | — | 追加 + Sync |
| PATCH 完成且 `>=total` | `Complete()` → 删除条目 | — | — | 无（文件已完整） |
| DELETE 取消 | `Complete()` → 删除条目 | — | — | `RemoveAll` 删除 |
| TTL 过期（内存） | `OnEviction` 回调 → `os.Remove(realPath)` | — | — | 删除不完整文件 |
| TTL 过期（Redis） | 自动过期（无回调） | — | — | 保留孤儿文件，下次 PATCH 返回 404 由客户端决定 |

---

### 9.8 TUS 与"路径标准化 / 权限控制 / 作用域约束"的协作

TUS 的 4 个 Handler 都会先经过路径、权限和作用域相关检查，但最后一步按方法分化：POST/PATCH/DELETE 会继续触发文件系统写入或删除，HEAD 只读取文件信息并查询缓存，不再执行写入或删除。

```
TUS Handler 的共通检查与分化：
  1. stripPrefix("/api/tus")            ← L1 路径标准化
  2. withUser() 认证加载 d.user
  3. !d.user.Perm.{Create|Delete}       ← 权限第1关：粗粒度
  4. !d.Check(r.URL.Path)               ← L3 标准化 + 权限第2关：细粒度规则
  5. files.NewFileInfo({                ← L3 标准化 + 权限第2关（第2次验证）
       Checker: d,                      ← 内部 d.Check 再次确认
       Fs: d.user.Fs,                   ← 触发 ScopedFs.LstatIfPossible
       ...                              ← guard/within：L4 标准化 + 作用域约束
     })
  6. POST/PATCH/DELETE 继续执行 OpenFile 或 RemoveAll；HEAD 到缓存查询和响应头返回为止
```

**安全验证**：[TestTusHandlersRejectSymlinkScopeEscape](http/tus_symlink_test.go#L24-L119) 测试专门验证：
- 用户作用域内的目录符号链接指向作用域外 → POST 创建会话被 403 拒绝
- 同一场景下 PATCH 写入也被 403 拒绝
- 即使路径词法上合法，ScopedFs.within 的符号链接解析也会拦截越界

---

## 十、关键安全设计总结

| 安全机制 | 实现位置 | 执行阶段 | 防护目标 |
|----------|----------|----------|----------|
| 路由前缀剥离 | `stripPrefix()` | L1 路径标准化 | 隔离路由命名空间与资源路径 |
| 路径歧义清理 | `path.Clean` + `slashClean` | L2 路径标准化 | 消除 `../` `./` 等路径遍历载体 |
| 粗粒度权限门控 | `d.user.Perm.*` | 权限控制第1关 | 按操作类型（创建/删除/修改）快速拒绝 |
| 细粒度规则检查 | `data.Check()` | 权限控制第2关 | 按路径黑白名单控制可见性与可操作性 |
| 规则路径修正 | `checkerPrefix` + `Join` | L3 路径标准化 | 分享场景下保持规则语义正确 |
| 作用域词法隔离 | `afero.BasePathFs` | 作用域约束 | 从字符串层面防止 `../` 越界 |
| 符号链接越界防护 | `ScopedFs.within()` | 作用域约束 | 阻止作用域内符号链接指向外部目录 |
| 真实路径解析 | `FullBaseFsPath + EvalSymlinks` | L4 路径标准化 | 获取磁盘上真实位置，作为作用域判断依据 |
| 分享路径二次约束 | `public.go` 嵌套 `ScopedFs` | 作用域约束 | 分享链接只能访问被分享的子树 |
| TUS override 参数位置 | `r.URL.Query().Get("override")` | POST 创建阶段 | 覆盖参数来自 URL Query，不是 Header |
| TUS 偏移量一致性 | `file.Size == uploadOffset` | PATCH 阶段 3 | 防止多客户端或错乱写入 |
| TUS 上传会话 TTL | `UploadCache` + TTL 过期清理 | 全流程 | 防止孤儿不完整文件永久残留 |
| TUS 传输保活 | `keepUploadActive` 每 2s Touch | PATCH 大分片期间 | 防止大文件传输中会话意外过期 |
| TUS 内存缓存过期清理 | `os.Remove(item.Key())`（绕过 ScopedFs） | OnEviction 回调 | ⚠️ 直接操作系统 API，但缓存 key 已在 POST 阶段验证合法 |

**多层防护的执行顺序口诀**：

> **先剥离前缀，再归一形式；先判断能不能做，再判断能不能对这个路径做；最后让文件系统自己再把一次关。**

即：L1/L2 路径标准化 → Perm 粗粒度 → rules 细粒度（含 L3 标准化）→ ScopedFs 作用域（含 L4 标准化）→ BasePathFs 词法 → 操作系统调用。
