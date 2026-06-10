# File Browser 文件系统抽象层深度解析

## 一、整体架构概览

File Browser 的文件系统抽象层由三大核心模块协同工作：

```
HTTP 请求层
    ↓
[权限检查] rules.Checker → data.Check()
    ↓
[路径归一] path.Clean / filepath.EvalSymlinks
    ↓
[作用域约束] ScopedFs (封装 afero.BasePathFs)
    ↓
[本地文件操作] afero.OsFs → 操作系统文件系统
```

核心文件分布：
- [files/scoped.go](files/scoped.go) — 作用域文件系统
- [files/file.go](files/file.go) — 文件信息与操作
- [users/permissions.go](users/permissions.go) — 用户权限定义
- [rules/rules.go](rules/rules.go) — 规则匹配引擎
- [http/data.go](http/data.go) — 请求上下文与权限检查实现
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

[ScopedFs](files/scoped.go) 是核心安全层，封装了 `afero.BasePathFs`，确保所有操作都限制在用户的根目录（Scope）内。

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

多数会解引用符号链接或创建新节点的操作（Create/Open/Stat/Chmod/Mkdir/Rename 等）会先调用 `guard()` 检查，失败则返回 `os.ErrPermission`；`Remove` 和 `RemoveAll` 直接委托给底层 BasePathFs，仍受词法作用域约束，但不额外解析符号链接目标。

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

## 三、权限检查机制

### 3.1 两层权限模型

权限检查分为**粗粒度操作权限**和**细粒度路径规则**两层。

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

这些在 HTTP Handler 层直接判断，例如 [resourceDeleteHandler](http/resource.go#L84-L88)：

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
    // 1. 路径前缀修正（公共分享场景下，FS被二次rebase，需要还原到用户原始作用域）
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

## 四、路径归一化逻辑

### 4.1 归一化的三个层次

| 层次 | 方法 | 作用 | 位置 |
|------|------|------|------|
| 路径字符串清理 | `path.Clean("/" + p)` | 移除 `../`、`./`、多余斜杠，确保以 `/` 开头 | HTTP Handler 入口 |
| 路径拼接 | `path.Join(...)` / `filepath.Join(...)` | 正确处理多段路径合并 | 多处 |
| 符号链接解析 | `filepath.EvalSymlinks(...)` | 跟随所有符号链接，得到磁盘上的真实路径 | ScopedFs.within() |

### 4.2 HTTP 入口处的路径归一

HTTP 层并不是所有入口都显式清理路径：读文件等常规入口直接使用路由剥离后的 `r.URL.Path`，写入、移动、复制、搜索等会在各自流程里按需要归一化，确保权限判断和文件系统调用使用一致的虚拟路径。

**示例：重命名/移动操作**（[resourcePatchHandler](http/resource.go#L212-L262)）：

```go
src := r.URL.Path
dst := r.URL.Query().Get("destination")
dst, err := url.QueryUnescape(dst)
dst = path.Clean("/" + dst)   // ← 归一化：加前缀"/"，清理 ".."
src = path.Clean("/" + src)   // ← 同上
if !d.Check(src) || !d.Check(dst) {  // ← 归一后再做权限检查
    return http.StatusForbidden, nil
}
```

**搜索操作**（[search.Search()](search/search.go#L22-L77)）：

```go
scope = filepath.ToSlash(filepath.Clean(scope))
scope = path.Join("/", scope)
// 遍历中每个路径也做同样归一
fPath = filepath.ToSlash(filepath.Clean(fPath))
fPath = path.Join("/", fPath)
```

关键步骤：
1. `filepath.Clean` — 清理原生路径（处理 Windows `\` 等）
2. `filepath.ToSlash` — 统一转为 POSIX 斜杠 `/`
3. `path.Join("/", ...)` — 确保绝对路径格式

### 4.3 ScopedFs 内部的路径转换

ScopedFs 内部维护了**虚拟路径**与**真实磁盘路径**的映射：

- 虚拟路径：`/docs/report.pdf`（相对于用户 Scope）
- 真实路径：`/home/user/files/docs/report.pdf`（OS 实际路径）

转换通过 `afero.FullBaseFsPath(baseFs, vpath)` 完成（[scoped.go:65-70](files/scoped.go#L65-L70)）：

```go
root, err := filepath.EvalSymlinks(afero.FullBaseFsPath(s.base, "/"))  // 真实根目录
target := afero.FullBaseFsPath(s.base, p)  // 目标文件的真实路径
```

### 4.4 GET 与目录列表中的路径构造

GET 资源详情入口会把路由剥离后的 `r.URL.Path` 交给 `NewFileInfo`，不在该入口额外调用 `path.Clean`。目录遍历时，子路径通过 `path.Join` 构造，确保列表内子项继续保持统一的虚拟路径格式（[file.go:407](files/file.go#L407)）：

```go
fPath := path.Join(i.Path, name)
if !checker.Check(fPath) {
    continue  // 权限不通过的文件直接跳过
}
```

---

## 五、三者协作流程

### 5.1 典型读流程：获取文件信息

以 [resourceGetHandler](http/resource.go#L25-L82) 为例：

```
HTTP GET /api/resources/docs/report.pdf
    │
    ▼
① r.URL.Path                             路由剥离后的虚拟路径
    │
    ▼
② files.NewFileInfo({
     Fs:      d.user.Fs,                  用户的 ScopedFs
     Path:    r.URL.Path,                 当前资源虚拟路径
     Checker: d,                          data.Check() 实现
     Expand:  true,
   })
    │
    ├─→ ③ Checker.Check(path)            rules 规则检查（隐藏文件+全局规则+用户规则）
    │        │
    │        └─ 不通过 → 返回 os.ErrPermission → HTTP 403
    │
    ├─→ ④ stat(opts)
    │        │
    │        ├─ ScopedFs.LstatIfPossible(path)
    │        │      │
    │        │      ├─→ guard(path) → within(path)
    │        │      │        └─ filepath.EvalSymlinks 检查符号链接是否越界
    │        │      └─→ afero.BasePathFs.LstatIfPossible → afero.OsFs.Lstat
    │        │
    │        └─ 构造 FileInfo（Name/Size/IsDir/Mode...）
    │
    └─→ ⑤ detectType(...)                MIME 检测 → 读前 512 字节 → 确定 Type
             │
             └─ 文本文件且 saveContent=true → ReadFile 读内容
    │
    ▼
⑥ renderJSON(w, r, file)                 返回 JSON 响应
```

### 5.2 典型写流程：上传文件

以 [resourcePostHandler](http/resource.go#L125-L178) 为例：

```
HTTP POST /api/resources/upload/new.txt
    │
    ▼
① 权限检查
   ├─ d.user.Perm.Create ?                                  粗粒度：创建权限
   └─ d.Check(r.URL.Path) ?                                 细粒度：路径规则
    │
    ▼
② 目录则 MkdirAll，文件则继续
    │
    ▼
③ 若文件已存在且 override=false → 409 Conflict
   若 override=true 且 !d.user.Perm.Modify → 403 Forbidden
    │
    ▼
④ writeFile(d.user.Fs, path, body, ...)
    │
    ├─ MkdirAll(dir, dirMode)                               确保父目录存在
    │      └─ ScopedFs.MkdirAll → guard → BasePathFs.MkdirAll → OsFs.MkdirAll
    │
    ├─ OpenFile(path, O_RDWR|O_CREATE|O_TRUNC, fileMode)    打开/创建文件
    │      └─ ScopedFs.OpenFile → guard → BasePathFs.OpenFile → OsFs.OpenFile
    │
    ├─ io.Copy(file, in)                                    写入内容
    ├─ file.Sync()                                          落盘
    └─ file.Stat()                                          获取元数据
    │
    ▼
⑤ d.RunHook(..., "upload", ...)                             执行用户配置的命令钩子
```

### 5.3 公共分享场景：双重 ScopedFs

[public.go](http/public.go#L17-L98) 展示了更复杂的嵌套：

```
原始用户作用域: /home/user/files
分享链接路径:   /home/user/files/shared/docs
                        ↑
               分享的基准路径 basePath

嵌套结构：
afero.OsFs
  └─ ScopedFs (用户作用域 /home/user/files)    ← 用户原始 Fs
       └─ ScopedFs (分享作用域 ./shared/docs)  ← withHashFile 中再次 NewScopedFs
```

同时设置 `d.checkerPrefix = basePath`，确保规则检查仍基于用户原始作用域的路径进行匹配，避免分享场景下规则失效。

### 5.4 重命名/移动：双向检查

[resourcePatchHandler](http/resource.go#L212-L262) 中，源和目标路径都要检查：

```go
dst = path.Clean("/" + dst)
src = path.Clean("/" + src)
if !d.Check(src) || !d.Check(dst) {      // 两条路径都要过规则
    return http.StatusForbidden, nil
}
// ...
return d.user.Fs.Rename(oldname, newname)  // ScopedFs.Rename 对两个路径都做 guard()
```

ScopedFs.Rename 的实现（[scoped.go:139-147](files/scoped.go#L139-L147)）：

```go
func (s *ScopedFs) Rename(oldname, newname string) error {
    if err := s.guard(oldname); err != nil { return err }  // 源路径越界检查
    if err := s.guard(newname); err != nil { return err }  // 目标路径越界检查
    return s.base.Rename(oldname, newname)
}
```

---

## 六、关键安全设计总结

| 安全机制 | 实现位置 | 防护目标 |
|----------|----------|----------|
| 作用域词法隔离 | afero.BasePathFs | 防止 `../` 路径遍历攻击 |
| 符号链接越界防护 | ScopedFs.within() | 阻止作用域内的符号链接指向外部目录 |
| 规则访问控制 | data.Check() | 按路径黑白名单控制可见性 |
| 操作权限门控 | HTTP Handler 入口 | 防止未授权的创建/修改/删除/分享 |
| 路径归一化 | path.Clean + path.Join | 消除歧义路径，统一比较基准 |
| 分享路径二次约束 | public.go 嵌套 ScopedFs | 分享链接只能访问被分享的子树 |

这几层机制层层递进，从底层文件系统到上层业务规则形成完整的安全防护链。
