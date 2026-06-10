# File Browser 规则引擎与正则匹配机制分析

## 一、核心概念与架构

### 1.1 规则是什么
规则（Rule）是 File Browser 中用于控制文件访问权限的核心机制，决定用户能否看到或操作某个路径下的文件。规则分为两种类型：
- **路径规则（Path Rule）**：基于路径前缀进行匹配
- **正则规则（Regex Rule）**：基于正则表达式进行匹配

每条规则都有一个 `Allow` 属性，标记这是一条「允许」规则还是「拒绝」规则。

### 1.2 核心数据结构

#### Rule 结构体
定义位置：[rules/rules.go](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/rules/rules.go#L15-L20)

```go
type Rule struct {
    Regex  bool    `json:"regex"`   // 是否为正则规则
    Allow  bool    `json:"allow"`   // 允许还是拒绝
    Path   string  `json:"path"`    // 路径规则的路径
    Regexp *Regexp `json:"regexp"`  // 正则规则的表达式
}
```

#### Regexp 包装类型
定义位置：[rules/rules.go](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/rules/rules.go#L48-L51)

```go
type Regexp struct {
    Raw    string         `json:"raw"`    // 原始正则表达式字符串
    regexp *regexp.Regexp                  // 编译后的正则对象（延迟初始化）
}
```

`Regexp` 是对标准库 `regexp.Regexp` 的包装，采用**懒加载**策略：只有在第一次调用 `MatchString` 时才编译正则表达式。

#### Checker 接口
定义位置：[rules/rules.go](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/rules/rules.go#L10-L12)

```go
type Checker interface {
    Check(path string) bool
}
```

`Checker` 是规则检查器的抽象接口。任何实现了 `Check(path string) bool` 方法的类型都可以作为规则检查器使用。这是典型的**策略模式**应用。

---

## 二、规则匹配算法

### 2.1 单条规则的匹配逻辑

`Rule.Matches` 方法是单条规则匹配的核心，定义在 [rules/rules.go](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/rules/rules.go#L29-L44)。

```go
func (r *Rule) Matches(path string) bool {
    if r.Regex {
        return r.Regexp.MatchString(path)
    }

    if path == r.Path {
        return true
    }

    prefix := r.Path
    if prefix != "/" && !strings.HasSuffix(prefix, "/") {
        prefix += "/"
    }

    return strings.HasPrefix(path, prefix)
}
```

#### 匹配逻辑详解：

1. **正则规则**：直接调用正则表达式的 `MatchString` 方法进行匹配
2. **路径规则**：采用「精确匹配 + 前缀匹配」的组合策略
   - 精确匹配：路径完全相等时匹配成功
   - 前缀匹配：确保规则路径以 `/` 结尾（根路径 `/` 除外），然后检查目标路径是否以该前缀开头

#### 路径匹配的边界处理：

这个前缀处理逻辑非常关键，它避免了「兄弟路径前缀混淆」的问题：

| 规则路径 | 目标路径 | 结果 | 说明 |
|---------|---------|------|------|
| `/uploads` | `/uploads` | ✅ 匹配 | 精确匹配 |
| `/uploads` | `/uploads/file.txt` | ✅ 匹配 | 子路径，补全前缀后为 `/uploads/` |
| `/uploads` | `/uploads_backup/secret.txt` | ❌ 不匹配 | 兄弟路径，不会误匹配 |
| `/` | `/anything` | ✅ 匹配 | 根路径匹配所有 |

更多测试用例可参考 [rules/rules_test.go](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/rules/rules_test.go#L5-L34)。

### 2.2 隐藏文件匹配

`MatchHidden` 函数用于匹配隐藏文件（文件名以 `.` 开头）：

定义位置：[rules/rules.go](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/rules/rules.go#L24-L26)

```go
func MatchHidden(path string) bool {
    return path != "" && strings.HasPrefix(filepath.Base(path), ".")
}
```

它只检查路径的**基名**（即最后一个路径段）是否以 `.` 开头。

---

## 三、规则检查器（Checker）的工作机制

### 3.1 核心检查逻辑

HTTP 层的 `data` 结构体实现了 `Checker` 接口，这是规则生效的核心入口。

定义位置：[http/data.go](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/http/data.go#L37-L64)

```go
func (d *data) Check(path string) bool {
    // 处理公共分享的路径前缀
    if d.checkerPrefix != "" {
        path = gopath.Join(d.checkerPrefix, path)
    }

    // 隐藏文件检查（优先级最高）
    if d.user.HideDotfiles && rules.MatchHidden(path) {
        return false
    }

    // 全局规则遍历
    allow := true
    for _, rule := range d.settings.Rules {
        if rule.Matches(path) {
            allow = rule.Allow
        }
    }

    // 用户规则遍历
    for _, rule := range d.user.Rules {
        if rule.Matches(path) {
            allow = rule.Allow
        }
    }

    return allow
}
```

### 3.2 检查流程详解

整个检查过程遵循「**后者覆盖前者**」的原则：

```
初始状态: allow = true
   ↓
应用全局规则（按顺序遍历，每条匹配的规则都会覆盖 allow）
   ↓
应用用户规则（按顺序遍历，每条匹配的规则都会覆盖 allow）
   ↓
返回最终的 allow 值
```

#### 关键特性：

1. **顺序敏感**：规则的顺序非常重要。后面的规则会覆盖前面规则的结果。
2. **两层规则**：全局规则先执行，用户规则后执行。用户规则可以覆盖全局规则的结果。
3. **隐藏文件优先**：如果用户开启了 `HideDotfiles`，隐藏文件会被直接拒绝，不经过规则链。

### 3.3 公共分享的特殊处理

`checkerPrefix` 字段用于处理公共分享场景。当用户访问一个共享链接时，文件系统会被重新定位到共享目录。为了让规则仍然基于用户原始的作用域（scope）进行匹配，需要在路径前加上 `checkerPrefix`。

定义位置：[http/public.go](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/http/public.go#L70-L75)

```go
// 将文件系统根目录重设为共享文件/文件夹
d.user.Fs = files.NewScopedFs(d.user.Fs, basePath)

// 设置 checkerPrefix，使规则检查基于原始作用域路径
d.checkerPrefix = basePath
```

这样设计的目的是：防止共享链接绕过共享根路径以下的拒绝规则。

---

## 四、规则的组织与存储

### 4.1 规则的存储位置

规则存储在两个层级：

1. **全局规则**：存储在 `Settings.Rules` 中
   - 定义位置：[settings/settings.go](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/settings/settings.go#L36)

2. **用户规则**：存储在 `User.Rules` 中
   - 定义位置：[users/users.go](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/users/users.go#L35)

两者都是 `[]rules.Rule` 类型的切片。

### 4.2 规则的管理命令

通过 CLI 命令可以管理规则，相关代码在 `cmd/` 目录下：

| 命令 | 文件 | 功能 |
|------|------|------|
| `rules add` | [cmd/rules_add.go](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/cmd/rules_add.go) | 添加规则 |
| `rules ls` | [cmd/rules_ls.go](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/cmd/rules_ls.go) | 列出规则 |
| `rules rm` | [cmd/rule_rm.go](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/cmd/rule_rm.go) | 删除规则 |

#### 添加规则的流程：

1. 解析 `--allow` / `-a` 标志（是否为允许规则）
2. 解析 `--regex` / `-r` 标志（是否为正则规则）
3. 如果是正则规则，先用 `regexp.MustCompile` 验证表达式有效性
4. 创建 `Rule` 对象并追加到对应的规则切片中
5. 保存到存储

相关代码参考 [cmd/rules_add.go](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/cmd/rules_add.go#L19-L66)。

#### 规则的用户/全局区分：

`runRules` 函数（[cmd/rules.go](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/cmd/rules.go#L32-L69)）根据命令行参数决定操作的是用户规则还是全局规则：
- 指定了 `--username` 或 `--id` → 操作用户规则
- 未指定 → 操作全局规则

---

## 五、规则在各场景中的应用

### 5.1 文件信息获取

`NewFileInfo` 函数是获取文件信息的入口，它在最开始就会进行规则检查。

定义位置：[files/file.go](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/files/file.go#L77-L107)

```go
func NewFileInfo(opts *FileOptions) (*FileInfo, error) {
    if !opts.Checker.Check(opts.Path) {
        return nil, os.ErrPermission
    }
    // ... 后续处理
}
```

如果规则检查不通过，直接返回 `os.ErrPermission` 错误。

### 5.2 目录列表遍历

在读取目录列表时，会对每个条目逐一进行规则检查，过滤掉不允许访问的文件。

定义位置：[files/file.go](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/files/file.go#L393-L475)

```go
func (i *FileInfo) readListing(checker rules.Checker, ...) error {
    dir, err := readDir(i.Fs, i.Path)
    // ...
    
    for _, f := range dir {
        fPath := path.Join(i.Path, name)
        if !checker.Check(fPath) {
            continue  // 不满足规则，跳过该条目
        }
        // ... 处理该文件
    }
}
```

### 5.3 递归文件列表

在递归遍历目录树时，同样会应用规则检查。如果目录被拒绝，会使用 `filepath.SkipDir` 跳过整个子树。

定义位置：[http/resource.go](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/http/resource.go#L408-L424)

```go
afero.Walk(d.user.Fs, rootPath, func(fPath string, info os.FileInfo, err error) error {
    // ...
    if !d.Check(fPath) {
        if info.IsDir() {
            return filepath.SkipDir  // 跳过整个目录
        }
        return nil  // 跳过单个文件
    }
    // ...
})
```

### 5.4 文件操作（创建/修改/删除/移动）

各种文件操作在执行前都会进行规则检查：

| 操作 | 代码位置 | 检查点 |
|------|---------|--------|
| 上传/创建文件 | [http/resource.go](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/http/resource.go#L127) | `resourcePostHandler` 第127行 |
| 修改文件 | [http/resource.go](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/http/resource.go#L181) | `resourcePutHandler` 第181行 |
| 重命名/复制 | [http/resource.go](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/http/resource.go#L220) | `resourcePatchHandler` 第220行（源和目标都要检查） |

### 5.5 搜索功能

搜索功能也会应用规则进行过滤。搜索使用 `conditions` 机制来处理搜索条件，包括文件类型过滤等。

定义位置：[search/conditions.go](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/search/conditions.go)

搜索条件（condition）也是一种 `func(path string) bool` 类型的函数，与规则检查的思路一致。支持的条件包括：
- `type:image` - 只搜索图片
- `type:audio` / `type:music` - 只搜索音频
- `type:video` - 只搜索视频
- `type:xxx` - 按扩展名搜索

---

## 六、完整调用链路

### 6.1 HTTP 请求的规则检查链路

以获取文件资源为例，完整的调用链路如下：

```
HTTP 请求到达
    ↓
[http/http.go] NewHandler 注册路由
    ↓
[http/data.go] handle 函数创建 data 上下文
    ↓
[http/auth.go] withUser 中间件：
    - 解析 JWT Token
    - 从存储中加载用户（包含用户规则）
    - 从存储中加载设置（包含全局规则）
    ↓
[http/resource.go] resourceGetHandler
    ↓
[files/file.go] NewFileInfo
    ↓
[http/data.go] data.Check(path)  ← 规则检查入口
    ├─ 处理 checkerPrefix
    ├─ 检查隐藏文件
    ├─ 遍历全局规则（Settings.Rules）
    └─ 遍历用户规则（User.Rules）
    ↓
返回是否允许访问
```

### 6.2 规则检查的关键入口点

规则检查通过 `Checker` 接口解耦，实际调用点很多，但核心都是调用 `d.Check(path)`。主要入口包括：

1. **文件信息获取**：`NewFileInfo`（[files/file.go#L78](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/files/file.go#L78)）
2. **目录列表**：`readListing`（[files/file.go#L409](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/files/file.go#L409)）
3. **递归列表**：`resourceGetRecursiveHandler`（[http/resource.go#L419](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/http/resource.go#L419)）
4. **资源操作**：POST/PUT/PATCH/DELETE 处理器

---

## 七、设计特点总结

### 7.1 优点

1. **接口抽象**：`Checker` 接口使得规则检查逻辑与具体实现解耦，便于测试和扩展。
2. **两层规则**：全局规则 + 用户规则的设计，既满足全局管控需求，又支持用户个性化配置。
3. **顺序覆盖**：后者覆盖前者的设计简单直观，用户可以通过调整规则顺序来实现复杂的权限控制。
4. **路径前缀安全**：路径规则自动补全结尾斜杠，避免了兄弟路径前缀混淆的安全隐患。
5. **懒加载正则**：正则表达式延迟编译，避免了不必要的性能开销。
6. **公共分享适配**：`checkerPrefix` 机制确保了共享链接场景下规则的正确应用。

### 7.2 注意事项

1. **规则顺序很重要**：由于后匹配的规则会覆盖前面的结果，规则顺序直接影响最终结果。
2. **默认允许**：初始状态 `allow = true`，即默认允许访问。如果需要默认拒绝，需要在规则列表开头放一条全局拒绝规则。
3. **隐藏文件独立检查**：`HideDotfiles` 选项不受规则链影响，优先级最高。
4. **正则性能**：正则规则每次匹配都会调用 `MatchString`，对于大量文件的目录列表可能有性能影响。

---

## 八、代码参考索引

| 模块 | 文件 | 主要内容 |
|------|------|---------|
| 规则核心 | [rules/rules.go](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/rules/rules.go) | Rule 结构体、匹配逻辑、Checker 接口 |
| 规则测试 | [rules/rules_test.go](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/rules/rules_test.go) | 匹配逻辑的测试用例 |
| HTTP 检查器 | [http/data.go](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/http/data.go) | data.Check 方法、规则检查入口 |
| 公共分享 | [http/public.go](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/http/public.go) | checkerPrefix 机制、共享场景 |
| 文件信息 | [files/file.go](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/files/file.go) | NewFileInfo、readListing 中的规则应用 |
| 资源操作 | [http/resource.go](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/http/resource.go) | 各种文件操作的规则检查点 |
| 用户模型 | [users/users.go](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/users/users.go) | 用户规则存储 |
| 设置模型 | [settings/settings.go](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/settings/settings.go) | 全局规则存储 |
| 搜索条件 | [search/conditions.go](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/search/conditions.go) | 搜索条件的条件函数 |
| CLI 管理 | [cmd/rules.go](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/cmd/rules.go) | 规则命令的通用逻辑 |
| CLI 添加 | [cmd/rules_add.go](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/cmd/rules_add.go) | rules add 命令 |
| CLI 删除 | [cmd/rule_rm.go](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/cmd/rule_rm.go) | rules rm 命令 |
| 原始下载 | [http/raw.go](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/http/raw.go) | raw/rawFileHandler/rawDirHandler |
| 搜索 | [search/search.go](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/search/search.go) | Search 函数、规则过滤 |
| 错误码 | [errors/errors.go](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/errors/errors.go) | 权限错误定义 |
| 错误映射 | [http/utils.go](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/http/utils.go) | errToStatus 错误→HTTP状态码 |

---

## 九、各操作路径的规则生效详细分析

### 9.1 删除操作

#### 触发入口

- **HTTP 路由**：`DELETE /api/resources/*path`
- **路由注册**：[http/http.go#L62](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/http/http.go#L62)
- **处理函数**：[resourceDeleteHandler](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/http/resource.go#L84-L123)

#### 执行顺序

```
1. withUser 中间件认证
   ├─ 解析 JWT Token，加载用户信息
   └─ 失败 → 401 Unauthorized

2. 权限前置检查（第86行）
   ├─ r.URL.Path == "/" → 403 Forbidden（禁止删除根目录）
   └─ !d.user.Perm.Delete → 403 Forbidden（无删除权限）

3. NewFileInfo 规则检查（第90-97行）  ← 规则生效点
   ├─ 创建 FileOptions，Checker = d（data 实例）
   ├─ NewFileInfo 内部调用 d.Check(r.URL.Path)
   │   ├─ 检查 HideDotfiles（隐藏文件直接拒绝）
   │   ├─ 遍历全局规则
   │   └─ 遍历用户规则
   └─ 规则拒绝 → 返回 os.ErrPermission

4. errToStatus 错误映射（第98-100行）
   └─ os.ErrPermission → 403 Forbidden

5. 删除关联分享记录（第102行）

6. 删除缩略图缓存（第107行）

7. 执行 Hook + 实际删除操作（第113-115行）
```

#### 拒绝后的返回

| 拒绝阶段 | 返回状态码 | 说明 |
|---------|-----------|------|
| JWT 认证失败 | 401 | 未登录或 Token 过期 |
| 路径为根目录 `/` | 403 | 禁止删除根目录 |
| 无删除权限 | 403 | `user.Perm.Delete == false` |
| **规则拒绝** | **403** | `d.Check(path)` 返回 false → `os.ErrPermission` → `errToStatus` 映射为 403 |

#### 关键代码路径

规则拒绝时的传播链：
```go
// files/file.go#L78
if !opts.Checker.Check(opts.Path) {
    return nil, os.ErrPermission  // 规则检查失败，返回权限错误
}

// http/resource.go#L98-99
if err != nil {
    return errToStatus(err), err  // os.ErrPermission → 403
}

// http/utils.go#L34-35
case os.IsPermission(err):
    return http.StatusForbidden   // 最终返回 403
```

---

### 9.2 原始文件下载（Raw Download）

#### 触发入口

- **HTTP 路由**：`GET /api/raw/*path`
- **路由注册**：[http/http.go#L82](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/http/http.go#L82)
- **处理函数**：[rawHandler](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/http/raw.go#L83-L110)

#### 执行顺序

```
1. withUser 中间件认证
   └─ 失败 → 401 Unauthorized

2. 下载权限检查（第84-86行）
   └─ !d.user.Perm.Download → 202 Accepted（注意：不是403！）

3. NewFileInfo 规则检查（第88-95行）  ← 规则生效点
   ├─ 创建 FileOptions，Checker = d
   ├─ Expand = false（不展开目录内容，只检查路径本身）
   ├─ NewFileInfo 内部调用 d.Check(r.URL.Path)
   │   ├─ 检查 HideDotfiles
   │   ├─ 遍历全局规则
   │   └─ 遍历用户规则
   └─ 规则拒绝 → 返回 os.ErrPermission → 403 Forbidden

4. 命名管道检查（第100-103行）
   └─ 是命名管道 → 设置 Content-Disposition，返回 200 空内容

5. 分支处理
   ├─ 普通文件 → rawFileHandler（第105-106行）
   │   └─ 直接以 http.ServeContent 流式传输文件内容
   └─ 目录 → rawDirHandler（第108-109行）
       └─ 进入打包下载流程（见9.3节）
```

#### 拒绝后的返回

| 拒绝阶段 | 返回状态码 | 说明 |
|---------|-----------|------|
| JWT 认证失败 | 401 | 未登录 |
| 无下载权限 | 202 | `user.Perm.Download == false`，注意不是403 |
| **规则拒绝** | **403** | `d.Check(path)` 返回 false → 403 |

#### 特别注意

无下载权限时返回 **202 Accepted** 而非 403，这是一个有意的设计。202 表示请求已被接受但尚未完成，前端可能据此做特殊处理（如弹出提示而非报错）。这与规则拒绝返回 403 有语义上的区别。

---

### 9.3 打包下载（目录下载）

#### 触发入口

- **HTTP 路由**：`GET /api/raw/*path`（目标是目录时）
- **入口函数**：[rawDirHandler](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/http/raw.go#L170-L216)
- **递归收集文件**：[getFiles](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/http/raw.go#L112-L168)

#### 执行顺序

```
1. rawHandler 前置流程（同9.2节）
   ├─ 认证、权限、NewFileInfo 规则检查（检查目录路径本身）
   └─ 目标路径通过规则检查后，判断是目录，进入 rawDirHandler

2. rawDirHandler 解析请求参数（第171-178行）
   ├─ parseQueryFiles：解析 ?files=a.txt,b.txt 参数
   │   └─ 无 files 参数时，默认打包整个目录
   └─ parseQueryAlgorithm：解析 ?algo=zip|tar|targz|... 参数

3. 计算公共路径前缀（第181行）
   └─ fileutils.CommonPrefix 确定归档包内的目录结构

4. 逐文件/目录递归收集  ← 规则生效点
   └─ 对每个 filename 调用 getFiles(d, fname, commonDir)

5. getFiles 内部流程（第112-168行）
   ├─ 对当前路径调用 d.Check(path)（第113行）
   │   ├─ 规则拒绝 → 返回 nil, nil（静默跳过，不报错）
   │   └─ 规则允许 → 继续处理
   ├─ Stat 获取文件信息
   ├─ 如果是文件且不是 commonPath → 加入归档列表
   └─ 如果是目录 → 递归处理子目录
       └─ 对每个子项递归调用 getFiles（第158行）
           ├─ 规则拒绝的子项 → 静默跳过
           └─ 规则允许的子项 → 加入归档列表

6. 执行归档打包（第211行）
   └─ archiver.Archive 将所有收集的文件写入 HTTP Response
```

#### 拒绝后的返回

打包下载中的规则拒绝行为与其他操作**截然不同**：

| 拒绝阶段 | 行为 | 说明 |
|---------|------|------|
| 目录路径本身被规则拒绝 | 403 | 在 rawHandler 的 NewFileInfo 中被拦截 |
| **目录内子文件/子目录被规则拒绝** | **静默跳过** | `getFiles` 返回 `nil, nil`，不报错、不包含在归档中 |
| 归档中部分文件被拒绝 | 返回不完整的归档 | 只有允许的文件被打包，被拒绝的文件无提示地消失 |

#### 关键代码分析

```go
// http/raw.go#L112-L115
func getFiles(d *data, path, commonPath string) ([]archives.FileInfo, error) {
    if !d.Check(path) {
        return nil, nil  // 规则拒绝 → 返回空列表，无错误
    }
    // ...
}
```

这是一个**过滤式**的规则应用方式，与删除操作的**拦截式**应用方式不同：
- **拦截式**（删除）：规则拒绝 → 返回错误 → 中断操作 → 返回 403
- **过滤式**（打包下载）：规则拒绝 → 跳过该条目 → 继续处理其他条目 → 返回不完整结果

同时注意，`getFiles` 中对目录的递归处理：如果子目录被规则拒绝，该目录及其**所有子内容**都不会被包含在归档中（因为不会递归进入被拒绝的目录）。

---

### 9.4 搜索过滤

#### 触发入口

- **HTTP 路由**：`GET /api/search?query=xxx`
- **路由注册**：[http/http.go#L86](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/http/http.go#L86)
- **HTTP 处理函数**：[searchHandler](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/http/search.go#L17-L82)
- **核心搜索逻辑**：[search.Search](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/search/search.go#L22-L77)

#### 执行顺序

```
1. withUser 中间件认证
   └─ 失败 → 401 Unauthorized

2. searchHandler 创建流式响应通道（第18-21行）
   └─ 使用 Server-Sent Events 风格的流式响应

3. 调用 search.Search（第61行）
   └─ 传入 d（data 实例）作为 checker

4. search.Search 内部流程
   ├─ 解析搜索查询（parseSearch）
   │   ├─ 提取搜索条件（type:image, type:video 等）
   │   └─ 提取搜索关键词
   ├─ afero.Walk 递归遍历文件系统
   └─ 对每个文件/目录：
       ├─ 跳过根路径自身（第38-40行）
       ├─ 规则检查（第42-44行）  ← 规则生效点
       │   ├─ checker.Check(fPath) 返回 false → return nil（跳过）
       │   └─ checker.Check(fPath) 返回 true → 继续
       ├─ 搜索条件过滤（第46-58行）
       │   └─ 不满足任何 condition → return nil（跳过）
       ├─ 关键词匹配（第61-73行）
       │   └─ 文件名不包含任何关键词 → return nil（跳过）
       └─ 匹配成功 → 调用 found 回调发送结果

5. 流式发送搜索结果（第62-69行）
   └─ 每找到一个匹配结果，立即通过 response channel 发送
```

#### 拒绝后的返回

| 拒绝阶段 | 行为 | 说明 |
|---------|------|------|
| JWT 认证失败 | 401 | 未登录 |
| **文件/目录被规则拒绝** | **静默跳过** | 不出现在搜索结果中 |
| 被拒绝的目录 | 不递归进入 | 但与打包下载不同，这里 Walk 会继续进入子目录 |
| 搜索条件不满足 | 静默跳过 | 不是规则拒绝，而是搜索条件过滤 |
| 关键词不匹配 | 静默跳过 | 不是规则拒绝，而是搜索匹配过滤 |

#### 关键代码分析

```go
// search/search.go#L42-L44
if !checker.Check(fPath) {
    return nil  // 规则拒绝 → 返回 nil，Walk 继续遍历其他路径
}
```

**重要差异**：搜索中的规则拒绝，与打包下载中 `getFiles` 的行为类似，是**过滤式**的。但有一个关键区别：

- **打包下载**：被拒绝的目录不会被递归进入，整个子树都被排除
- **搜索**：被拒绝的路径 `return nil`，`afero.Walk` **仍会递归进入**该目录的子目录

这意味着在搜索中，如果 `/data/private` 被规则拒绝：
- `/data/private` 本身不会出现在搜索结果中
- 但 `/data/private/subfile.txt` 仍会被 Walk 访问到，如果该子路径没有被单独的规则拒绝，它仍可能出现在搜索结果中

这与递归列表 `resourceGetRecursiveHandler` 的行为不同——后者在目录被拒绝时使用 `filepath.SkipDir` 跳过整个子树。

#### 搜索条件与规则的关系

搜索条件（conditions）和规则（rules）是两个独立的过滤层：

```
文件路径
  ↓
第1层：规则检查（checker.Check） — 基于路径的权限控制
  ↓ 通过
第2层：搜索条件（conditions） — 基于文件类型/扩展名的过滤
  ↓ 通过
第3层：关键词匹配（terms） — 基于文件名的文本搜索
  ↓ 通过
返回搜索结果
```

---

### 9.5 公共分享

公共分享是规则应用中最复杂的场景，因为它涉及文件系统的重定位（rebase）和规则路径的回溯映射。

#### 触发入口

- **分享查看**：`GET /api/public/share/{hash}[/path]`
- **分享下载**：`GET /api/public/dl/{hash}[/path]`
- **路由注册**：[http/http.go#L90-L91](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/http/http.go#L90-L91)
- **中间件**：[withHashFile](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/http/public.go#L17-L98)
- **分享查看处理**：[publicShareHandler](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/http/public.go#L115-L125)
- **分享下载处理**：[publicDlHandler](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/http/public.go#L127-L134)

#### 执行顺序

```
1. withHashFile 中间件（核心，第17-98行）
   │
   ├─ 1.1 从 URL 解析分享 hash 和子路径（第19行）
   │   └─ ifPathWithName：hash 为 URL 第一段，后续为子路径
   │
   ├─ 1.2 查找分享记录（第20-23行）
   │   └─ share.GetByHash(id) → 不存在 → 404/500
   │
   ├─ 1.3 分享认证（第25-28行）
   │   ├─ 无密码分享 → 通过
   │   ├─ URL token 认证 → 通过
   │   ├─ X-SHARE-PASSWORD 头认证 → 通过
   │   └─ 认证失败 → 401 Unauthorized
   │
   ├─ 1.4 加载分享所有者用户（第30-33行）
   │   └─ 获取用户信息（包含用户规则）
   │
   ├─ 1.5 权限检查（第35-37行）
   │   ├─ !user.Perm.Share → 403 Forbidden
   │   └─ !user.Perm.Download → 403 Forbidden
   │
   ├─ 1.6 设置 d.user = user（第39行）
   │   └─ 此后 d.Check 将使用分享所有者的规则
   │
   ├─ 1.7 第一次 NewFileInfo — 检查分享根路径（第41-53行）  ← 规则生效点①
   │   ├─ Path = link.Path（分享链接指向的原始路径）
   │   ├─ Checker = d（使用分享所有者的规则）
   │   ├─ d.Checker.Check(link.Path) 检查分享根路径
   │   └─ 规则拒绝 → os.ErrPermission → 403 Forbidden
   │
   ├─ 1.8 重定位文件系统（第66-75行）  ← 关键步骤
   │   ├─ d.user.Fs = files.NewScopedFs(d.user.Fs, basePath)
   │   │   └─ 文件系统根目录重设为分享路径
   │   └─ d.checkerPrefix = basePath
   │       └─ 规则检查时，路径会加上此前缀，回溯到原始作用域
   │
   ├─ 1.9 第二次 NewFileInfo — 检查实际访问路径（第77-87行）  ← 规则生效点②
   │   ├─ Path = filePath（分享内的子路径）
   │   ├─ Checker = d（此时 checkerPrefix 已设置）
   │   ├─ d.Check(filePath) 内部：
   │   │   ├─ path = gopath.Join(d.checkerPrefix, filePath)
   │   │   │   └─ 将子路径还原为用户原始作用域的完整路径
   │   │   ├─ 检查 HideDotfiles
   │   │   ├─ 遍历全局规则（Settings.Rules）
   │   │   └─ 遍历用户规则（User.Rules，分享所有者的规则）
   │   └─ 规则拒绝 → os.ErrPermission → 403 Forbidden
   │
   └─ 1.10 将 FileInfo 存入 d.raw，调用业务处理函数（第95-96行）

2. 业务处理函数
   ├─ publicShareHandler：返回目录/文件的 JSON 信息
   │   └─ readListing 内部对每个子项也会调用 checker.Check
   └─ publicDlHandler：下载文件或打包目录
       ├─ 普通文件 → rawFileHandler（直接下载）
       └─ 目录 → rawDirHandler → getFiles（递归，每个文件都过 checker.Check）
```

#### 拒绝后的返回

| 拒绝阶段 | 返回状态码 | 说明 |
|---------|-----------|------|
| 分享记录不存在 | 404 | 分享 hash 无效 |
| 分享认证失败 | 401 | 密码错误或未提供 |
| 所有者无 Share 权限 | 403 | `user.Perm.Share == false` |
| 所有者无 Download 权限 | 403 | `user.Perm.Download == false` |
| **分享根路径被规则拒绝** | **403** | 第一次 NewFileInfo，规则生效点① |
| **分享内子路径被规则拒绝** | **403** | 第二次 NewFileInfo，规则生效点② |
| 目录列表中子项被规则拒绝 | 静默过滤 | readListing 中 `continue` 跳过 |
| 打包下载中子文件被规则拒绝 | 静默跳过 | getFiles 返回 `nil, nil` |

#### checkerPrefix 机制详解

这是公共分享场景中最关键的设计，用一个具体例子说明：

```
假设：
  用户 scope = "/srv/data"
  用户规则：{Allow: false, Path: "/projects/private"}
  分享链接：{Hash: "abc123", Path: "/projects"}

访问 /api/public/share/abc123/private/secret.txt 时：

1. 第一次 NewFileInfo
   - Path = "/projects"（分享根路径）
   - Check("/projects") → 规则不匹配 → allow = true → 通过

2. 设置 checkerPrefix
   - d.user.Fs = NewScopedFs(fs, "/projects")  // 文件系统根变为 /projects
   - d.checkerPrefix = "/projects"

3. 第二次 NewFileInfo
   - Path = "private/secret.txt"（相对于分享根的子路径）
   - Check 内部：
     - path = Join("/projects", "private/secret.txt") = "/projects/private/secret.txt"
     - 规则匹配：/projects/private 是前缀 → Allow = false → 拒绝！
   - 返回 403 Forbidden
```

**如果没有 checkerPrefix**：`Check("private/secret.txt")` 不会匹配规则 `/projects/private`，该文件就会被错误地放行。

这个机制确保了即使文件系统被重定位，规则仍然基于用户原始作用域的完整路径进行匹配，防止通过分享链接绕过规则限制。

相关测试用例参见 [http/public_test.go#L151-L255](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/http/public_test.go#L151-L255)。

---

## 十、规则生效方式对比总结

### 10.1 两种生效方式

| 生效方式 | 机制 | 拒绝结果 | 适用场景 |
|---------|------|---------|---------|
| **拦截式** | `NewFileInfo` 中 `Check` 失败 → 返回 `os.ErrPermission` | 中断操作，返回 HTTP 403 | 删除、下载（文件本身）、分享根路径 |
| **过滤式** | 遍历中 `Check` 失败 → `continue`/`return nil` | 静默跳过，不影响其他条目 | 目录列表、打包下载子文件、搜索、递归列表 |

### 10.2 各操作对比

| 操作 | 路由 | 规则生效点 | 生效方式 | 被拒后的返回 |
|------|------|-----------|---------|------------|
| 删除 | `DELETE /api/resources/*` | NewFileInfo | 拦截式 | 403 Forbidden |
| 原始文件下载 | `GET /api/raw/*`（文件） | NewFileInfo | 拦截式 | 403 Forbidden |
| 打包下载（目录本身） | `GET /api/raw/*`（目录） | NewFileInfo | 拦截式 | 403 Forbidden |
| 打包下载（子文件） | 同上 | getFiles | 过滤式 | 静默跳过，归档不完整 |
| 搜索（整体） | `GET /api/search` | search.Search | 过滤式 | 结果中不出现 |
| 公共分享（根路径） | `GET /api/public/share/*` | 第一次 NewFileInfo | 拦截式 | 403 Forbidden |
| 公共分享（子路径） | 同上 | 第二次 NewFileInfo | 拦截式 | 403 Forbidden |
| 公共分享（目录列表子项） | 同上 | readListing | 过滤式 | 静默跳过 |
| 公共分享下载（子文件） | `GET /api/public/dl/*` | getFiles | 过滤式 | 静默跳过，归档不完整 |
| 递归列表 | `GET /api/resources/recursive` | Walk 回调 | 过滤式 | 目录用 SkipDir，文件静默跳过 |

### 10.3 搜索 vs 递归列表的目录跳过行为差异

| 场景 | 目录被规则拒绝时 | 代码位置 |
|------|----------------|---------|
| 递归列表 | `filepath.SkipDir` — 跳过整个子树 | [http/resource.go#L420-L422](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/http/resource.go#L420-L422) |
| 搜索 | `return nil` — 只跳过目录本身，**子目录仍会被遍历** | [search/search.go#L42-L44](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/search/search.go#L42-L44) |
| 打包下载 | `return nil, nil` — 跳过整个子树（因为不递归进入被拒目录） | [http/raw.go#L113-L115](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/http/raw.go#L113-L115) |

这意味着：在搜索场景下，如果一个目录被规则拒绝但其子路径没有被单独的规则覆盖，搜索仍可能找到该目录下的文件。这是一个潜在的规则绕过点，需要通过在规则中使用路径前缀匹配（如 `/data/private` 规则会同时匹配 `/data/private` 及其所有子路径）来避免。

---

## 十一、目录被规则拒绝后的遍历逻辑深度分析

### 11.1 afero.Walk 的标准行为

`afero.Walk` 的行为与标准库 `filepath.Walk` 完全一致，其 WalkFunc 返回值的语义如下：

| 返回值 | 行为 |
|-------|------|
| `nil` | 继续正常遍历，包括进入当前目录的子目录 |
| `filepath.SkipDir` | **跳过当前目录的所有子目录**，但继续遍历同层级的其他路径 |
| 其他 error | 立即终止整个 Walk，返回该错误 |

这是理解所有遍历差异的基础。四种场景中，只有**递归列表**使用了 `filepath.SkipDir`，其他三种都返回 `nil` 或直接不递归。

---

### 11.2 四种目录遍历场景的逐行对比

#### 场景一：搜索（search.Search）

**代码位置**：[search/search.go#L29-L76](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/search/search.go#L29-L76)

核心代码：
```go
return afero.Walk(fs, scope, func(fPath string, f os.FileInfo, _ error) error {
    // ...
    if fPath == scope {
        return nil           // ① 根目录：返回 nil → 继续进入子目录
    }

    if !checker.Check(fPath) {
        return nil           // ② 规则拒绝：返回 nil → 继续进入子目录！
    }
    // ...
})
```

**遍历行为详解**：

```
目录结构：
/data/
  ├── public/
  │   └── readme.txt
  └── private/          ← 规则：Allow=false, Path="/data/private"
      └── secret.txt
```

Walk 执行过程：
```
1. 访问 "/data"
   → fPath == scope → return nil
   → ✅ Walk 继续进入子目录

2. 访问 "/data/public"
   → Check("/data/public") → 通过
   → return nil
   → ✅ Walk 继续进入子目录

3. 访问 "/data/public/readme.txt"
   → Check 通过 → 返回搜索结果
   → return nil

4. 访问 "/data/private"
   → Check("/data/private") → 规则匹配 → Allow=false
   → return nil                    ← 关键：返回 nil，不是 SkipDir
   → ✅ Walk 继续进入子目录！

5. 访问 "/data/private/secret.txt"
   → Check("/data/private/secret.txt")
     → 规则 "/data/private" 的前缀匹配：
       prefix = "/data/private/"
       strings.HasPrefix("/data/private/secret.txt", "/data/private/") → true
     → 规则匹配 → Allow=false
   → return nil
   → 不出现在搜索结果中
```

**关键点**：
- 搜索在目录被拒时 `return nil`，Walk **会继续递归进入子目录**
- 对于**路径规则**（Path Rule）：由于路径规则使用前缀匹配，子路径（如 `/data/private/secret.txt`）仍然会被同一条规则匹配到，所以实际效果上子路径也会被拒绝——只是多做了一次无用的 Walk
- 对于**正则规则**（Regex Rule）：如果正则只匹配目录本身（如 `^/data/private$`），则子路径**不会**被匹配，会泄露到搜索结果中

---

#### 场景二：递归列表（resourceGetRecursiveHandler）

**代码位置**：[http/resource.go#L408-L434](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/http/resource.go#L408-L434)

核心代码：
```go
afero.Walk(d.user.Fs, rootPath, func(fPath string, info os.FileInfo, err error) error {
    // ...
    if !d.Check(fPath) {
        if info.IsDir() {
            return filepath.SkipDir   // ① 目录被拒：返回 SkipDir → 跳过整个子树
        }
        return nil                   // ② 文件被拒：返回 nil → 继续其他路径
    }
    // ...
})
```

**遍历行为详解**：

同样的目录结构，Walk 执行过程：
```
1. 访问 "/data" → 跳过根目录 → return nil
   → ✅ Walk 继续进入子目录

2. 访问 "/data/public"
   → Check 通过 → 加入列表 → return nil
   → ✅ Walk 继续进入子目录

3. 访问 "/data/public/readme.txt"
   → Check 通过 → 加入列表 → return nil

4. 访问 "/data/private"
   → Check 失败，且 info.IsDir() == true
   → return filepath.SkipDir          ← 关键：返回 SkipDir
   → ❌ Walk **不会进入** "/data/private" 的子目录！

5. "/data/private/secret.txt" —— **根本不会被访问**
```

**关键点**：
- 递归列表是四种场景中**唯一正确使用 `filepath.SkipDir`** 的
- 被拒绝的目录及其**整个子树**都会被跳过，Walk 根本不会访问子路径
- 无论规则是路径规则还是正则规则，子目录都不会泄露
- 性能最优：避免了访问必然被拒绝的子路径

---

#### 场景三：打包下载（getFiles 手动递归）

**代码位置**：[http/raw.go#L112-L168](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/http/raw.go#L112-L168)

核心代码：
```go
func getFiles(d *data, path, commonPath string) ([]archives.FileInfo, error) {
    if !d.Check(path) {
        return nil, nil           // ① 规则拒绝：直接返回空列表，不递归
    }
    // ...
    if info.IsDir() {
        // ...
        for _, name := range names {
            fPath := filepath.Join(path, name)
            subFiles, err := getFiles(d, fPath, commonPath)  // ② 只有通过检查才递归
            // ...
        }
    }
    return archiveFiles, nil
}
```

**遍历行为详解**：

同样的目录结构，`getFiles` 执行过程：
```
1. getFiles("/data", ...)
   → Check("/data") → 通过
   → 是目录 → 读取子项 ["public", "private"]

2. getFiles("/data/public", ...)
   → Check 通过
   → 是目录 → 读取子项 ["readme.txt"]

3. getFiles("/data/public/readme.txt", ...)
   → Check 通过
   → 加入归档列表

4. getFiles("/data/private", ...)
   → Check("/data/private") → 规则拒绝
   → return nil, nil                ← 关键：直接返回，不递归
   → ❌ **不会调用** getFiles("/data/private/secret.txt", ...)

5. "/data/private/secret.txt" —— **根本不会被访问**
```

**关键点**：
- 打包下载不使用 `afero.Walk`，而是手动实现递归
- 规则拒绝的路径直接 `return nil, nil`，**不会继续递归调用**
- 效果与 `filepath.SkipDir` 等价：被拒绝的目录及其整个子树都被排除
- 无论路径规则还是正则规则，子目录都不会泄露

---

#### 场景四：目录列表（readListing）

**代码位置**：[files/file.go#L393-L475](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/files/file.go#L393-L475)

核心代码：
```go
func (i *FileInfo) readListing(checker rules.Checker, ...) error {
    dir, err := readDir(i.Fs, i.Path)  // 只读取当前层级
    // ...
    for _, f := range dir {
        fPath := path.Join(i.Path, name)
        if !checker.Check(fPath) {
            continue             // 规则拒绝：跳过当前条目
        }
        // ... 加入列表
    }
    return nil
}
```

**遍历行为详解**：

`readListing` 只读取**单一层级**的目录内容，不做递归。用户在界面上逐层点击进入时，每次进入一个目录都会调用一次 `NewFileInfo` → `readListing`。

```
1. 请求 "/data"
   → NewFileInfo → Check("/data") → 通过
   → readListing 列出 ["public", "private"]
     → "/data/public" → Check 通过 → 显示
     → "/data/private" → Check 失败 → continue（不显示）

2. 用户尝试直接请求 "/data/private"
   → NewFileInfo → Check("/data/private") → 失败
   → 返回 403 Forbidden（拦截式）

3. 用户尝试直接请求 "/data/private/secret.txt"
   → NewFileInfo → Check("/data/private/secret.txt")
     → 路径规则前缀匹配 → 失败
   → 返回 403 Forbidden（拦截式）
```

**关键点**：
- `readListing` 本身不递归，只过滤当前层级
- 但访问子目录/子文件需要通过 `NewFileInfo`，后者会做**拦截式**规则检查
- 实际效果上子路径不会泄露，因为每个路径访问都会被独立检查

---

### 11.3 四种场景对比总表

| 场景 | 遍历方式 | 目录被拒时的处理 | 子目录是否被访问 | 泄露风险（路径规则） | 泄露风险（正则规则） | 性能 |
|------|---------|----------------|----------------|-------------------|-------------------|------|
| **搜索** | afero.Walk | `return nil` | ✅ **是**，Walk 继续进入 | 无*（前缀匹配覆盖子路径） | ⚠️ **有** — 正则不匹配子路径则泄露 | 较差（访问无用子目录） |
| **递归列表** | afero.Walk | `filepath.SkipDir` | ❌ 否，跳过整个子树 | 无 | 无 | 最优 |
| **打包下载** | 手动递归 | `return nil, nil`，不递归 | ❌ 否，整个子树不访问 | 无 | 无 | 良好 |
| **目录列表** | 单层读取 | `continue` 跳过 | 不递归，但子路径独立检查 | 无* | 无* | 良好 |

\* 路径规则由于自带前缀匹配机制，子路径天然被同一条规则覆盖。目录列表场景下子路径访问需通过 `NewFileInfo` 的拦截式检查。

---

### 11.4 正则规则在搜索中的泄露风险详解

这是搜索场景独有的风险。用一个具体例子说明：

**配置**：
- 用户规则（正则）：
  ```json
  {
    "regex": true,
    "allow": false,
    "regexp": {"raw": "^/data/private$"}
  }
  ```
  注意：这个正则使用 `$` 结尾，**只精确匹配 `/data/private` 本身**，不匹配子路径。

**文件系统**：
```
/data/
  └── private/
      └── secret.txt
```

**搜索执行过程**：
```
1. Walk 访问 "/data/private"
   → Check("/data/private")
     → 正则匹配："^/data/private$" 匹配 "/data/private" → true
     → Allow = false → 拒绝
   → return nil
   → Walk **继续进入子目录**

2. Walk 访问 "/data/private/secret.txt"
   → Check("/data/private/secret.txt")
     → 正则匹配："^/data/private$" 不匹配 "/data/private/secret.txt" → false
     → 无其他规则匹配 → allow 保持默认 true
   → ✅ 规则检查通过！
   → 关键词匹配 → 出现在搜索结果中！  ← ⚠️ 泄露！
```

**如何避免**：
1. 使用**路径规则**而非正则规则：`{Path: "/data/private", Allow: false}`，路径规则的前缀匹配会自动覆盖子路径。
2. 正则规则要写成前缀匹配形式：`{"raw": "^/data/private(/|$)"}` 或 `{"raw": "^/data/private"}`。

---

### 11.5 规则匹配对子路径的覆盖机制

#### 路径规则（Path Rule）的前缀覆盖

`Rule.Matches` 方法的实现保证了路径规则天然具有前缀匹配能力：

**代码位置**：[rules/rules.go#L29-L44](file:///d:/fz/0601/solo-dogfeeding/code/175-filebrowser/rules/rules.go#L29-L44)

```go
func (r *Rule) Matches(path string) bool {
    // ...
    prefix := r.Path
    if prefix != "/" && !strings.HasSuffix(prefix, "/") {
        prefix += "/"          // 自动补全结尾斜杠
    }
    return strings.HasPrefix(path, prefix)  // 前缀匹配
}
```

| 规则路径 | 自动补全后 | 目标路径 | 匹配结果 |
|---------|----------|---------|---------|
| `/data/private` | `/data/private/` | `/data/private` | ✅（精确匹配分支） |
| `/data/private` | `/data/private/` | `/data/private/secret.txt` | ✅（前缀匹配） |
| `/data/private` | `/data/private/` | `/data/private_backup/file.txt` | ❌（斜杠避免兄弟前缀混淆） |

所以只要使用路径规则，子路径一定被覆盖，不存在泄露问题。

#### 正则规则（Regex Rule）的完全匹配

正则规则没有前缀匹配的特殊处理，完全依赖正则表达式本身：

```go
func (r *Rule) Matches(path string) bool {
    if r.Regex {
        return r.Regexp.MatchString(path)  // 完全依赖正则语义
    }
    // ...
}
```

正则规则的覆盖范围完全取决于表达式如何书写：

| 正则表达式 | 匹配 `/data/private` | 匹配 `/data/private/secret.txt` | 说明 |
|-----------|---------------------|-------------------------------|------|
| `^/data/private$` | ✅ | ❌ | 只匹配目录本身 |
| `^/data/private` | ✅ | ✅ | 前缀匹配（无 `$`） |
| `^/data/private(/\|$)` | ✅ | ✅ | 精确前缀，避免匹配 `/data/private_backup` |
| `/private` | ✅ | ✅ | 子串匹配，但可能误匹配其他路径 |

---

### 11.6 行为差异的潜在原因推测

搜索为什么不像递归列表那样使用 `filepath.SkipDir`？可能的原因：

1. **正则规则考虑**：搜索中目录被拒绝可能是正则精确匹配（`^/dir$`），开发者可能期望子目录仍被独立检查。但这实际上带来了泄露风险。
2. **搜索条件考虑**：目录被拒绝可能只是因为不满足搜索条件（conditions），而不是规则拒绝——但代码中规则检查在条件检查之前，且两者都返回 `nil`。
3. **实现简化**：搜索的 WalkFunc 对规则拒绝、条件不满足、关键词不匹配三种情况统一返回 `nil`，没有区分目录和文件的处理差异。

相比之下，递归列表 `resourceGetRecursiveHandler` 的实现明确区分了目录和文件：目录被拒返回 `SkipDir`，文件被拒返回 `nil`。这是更合理的实现。
