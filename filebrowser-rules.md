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
