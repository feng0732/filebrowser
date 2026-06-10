# FileBrowser 在线编辑器与保存流程实现脉络

本文档从代码层面梳理 FileBrowser 中在线编辑器的**内容读取**、**保存写回**、**冲突处理**三大核心环节的实现逻辑与调用关系。

> 路径说明：本文档中的代码引用均为**仓库相对路径**，以项目根目录为基准。

---

## 一、整体架构概览

```
┌─────────────────────────────────────────────────────────────┐
│                        前端 (Vue + Ace)                      │
│  ┌──────────┐   ┌──────────┐   ┌──────────────────────────┐  │
│  │ Files.vue│──▶│ api/files│──▶│ Editor.vue (Ace编辑器)    │  │
│  └──────────┘   └──────────┘   └──────────────────────────┘  │
│         │              │                    │                 │
│         ▼              ▼                    ▼                 │
│  ┌──────────┐   ┌──────────────┐   ┌──────────────────┐      │
│  │ fileStore│   │ fetchURL()   │   │ 冲突检测/提示组件 │      │
│  └──────────┘   └──────────────┘   └──────────────────┘      │
└──────────────────────────┬───────────────────────────────────┘
                           │ HTTP (REST API)
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                        后端 (Go + afero)                     │
│  ┌──────────────────┐   ┌──────────────┐   ┌──────────────┐  │
│  │ resourceGetHandler│  │resourcePut   │   │ writeFile()  │  │
│  │ (读取)            │  │Handler(保存) │   │ (磁盘写入)   │  │
│  └──────────────────┘   └──────────────┘   └──────────────┘  │
│         │                     │                  │           │
│         ▼                     ▼                  ▼           │
│  ┌──────────────────────────────────────────────────────┐    │
│  │              files.NewFileInfo (文件元信息+内容检测)    │    │
│  └──────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

---

## 二、内容读取流程

### 2.1 前端触发与路由

用户访问文件时，Vue Router 加载 `frontend/src/views/Files.vue`，在 `onMounted` 和 `watch(route)` 中调用 `fetchData()`：

```
Files.vue:fetchData()
  └─▶ api.fetch(url, signal)          // 发起资源获取请求
       └─▶ fileStore.updateRequest(res)  // 结果存入 Pinia 状态
            └─▶ 根据 req.type 决定渲染 Editor 或 Preview
```

关键点：
- `fetchData` 支持 `AbortController` 取消重复请求（路由快速切换时）
- 通过 `computed` 属性 `currentView` 动态决定组件：
  - `text` / `textImmutable` → `frontend/src/views/files/Editor.vue`
  - `csv` 默认走 Preview，`?edit=true` 时走 Editor
  - 其他类型走 Preview

### 2.2 API 层：frontend/src/api/files.ts → fetch()

```typescript
export async function fetch(url: string, signal?: AbortSignal) {
  const encoding = isEncodableResponse(url);  // .csv 文件返回 true
  // 携带 X-Encoding 头标记是否需要二进制传输
  const res = await fetchURL(`/api/resources${url}`, {
    signal,
    headers: { "X-Encoding": encoding ? "true" : "false" },
  });

  // 两种响应格式分支：
  if (res.headers.get("Content-Type") == "application/octet-stream") {
    // 分支A: CSV等需要编码处理的文件 → 二进制流
    data = await makeRawResource(res, url);  // 转为 UTF-8 文本
  } else {
    // 分支B: 普通文件 → JSON（已包含 content 字段）
    data = (await res.json()) as Resource;
  }
}
```

**编码分支设计意图**：CSV 等文件可能含非 UTF-8 编码，后端直接返回二进制，前端通过 `TextDecoder` 解码，避免 JSON 传输时的编码丢失。

### 2.3 后端处理：http/resource.go → resourceGetHandler

```
resourceGetHandler
  ├─▶ files.NewFileInfo()  ← 核心：构建 FileInfo 对象
  │     ├─▶ stat()          // 获取文件元信息（大小、时间、模式）
  │     └─▶ detectType()    // 检测文件类型 + 读取文本内容
  │           ├─▶ MIME 类型判断（扩展名 + 前512字节嗅探）
  │           ├─▶ 文本且≤10MB → 读取全部内容到 FileInfo.Content
  │           └─▶ 无 modify 权限 → type 设为 "textImmutable"
  │
  ├─▶ 分支判断：
  │   ├─▶ 目录 → 排序后返回 JSON
  │   ├─▶ X-Encoding=true 且是文本 → 以 application/octet-stream 返回原始字节
  │   └─▶ 其他 → 以 JSON 返回（content 字段含文本）
  │
  └─▶ checksum 查询参数 → 计算 MD5/SHA 等哈希
```

核心文件：`files/file.go` → `detectType()`

```go
// 文本文件的判断条件：MIME以text开头 或 非二进制 且 大小≤10MB
case (strings.HasPrefix(mimetype, "text") || !isBinary(buffer)) && i.Size <= 10*1024*1024:
    i.Type = "text"
    if !modify {
        i.Type = "textImmutable"  // 只读模式
    }
    if saveContent {
        content, _ := afs.ReadFile(i.Path)  // 一次性读入内存
        i.Content = string(content)
    }
```

### 2.4 编辑器初始化

`frontend/src/views/files/Editor.vue` → `initEditor()` 从 `fileStore.req.content` 读取初始内容：

```typescript
const fileContent = fileStore.req?.content || "";

editor.value = ace.edit("editor", {
  value: fileContent,        // 初始内容注入
  readOnly: fileStore.req?.type === "textImmutable",
  mode: modelist.getModeForPath(fileStore.req!.name).mode,  // 自动语法高亮
  // ...
});
```

---

## 三、保存写回流程

### 3.1 前端触发

保存有三种触发方式：
1. 点击工具栏保存按钮 → `save()`
2. 快捷键 `Ctrl+S` / `Cmd+S` → `keyEvent()` 拦截
3. 路由跳转前未保存 → `onBeforeRouteUpdate` 弹窗确认后自动保存

### 3.2 保存执行：Editor.vue → save()

文件位置：`frontend/src/views/files/Editor.vue`

```typescript
const save = async (throwError?: boolean) => {
  const button = "save";
  buttons.loading("save");

  try {
    // 核心调用：PUT 方法写入内容
    await api.put(route.path, editor.value?.getValue());
    // 成功后标记编辑器为"干净"状态（无未保存修改）
    editor.value?.session.getUndoManager().markClean();
    buttons.success(button);
  } catch (e: any) {
    buttons.done(button);
    $showError(e);  // Toast 提示错误
    if (throwError) throw e;
  }
};
```

### 3.3 API 层：frontend/src/api/files.ts → put()

```typescript
export async function put(url: string, content = "") {
  return resourceAction(url, "PUT", content);
}
```

`resourceAction` 是通用封装，直接将内容作为 PUT 请求的 body 发送到 `/api/resources{path}`。

**注意**：PUT 请求**不携带任何版本标识**（如 `If-Match` 头），也不携带修改时间校验参数。这是导致静默覆盖的根本原因之一。

### 3.4 后端处理：http/resource.go → resourcePutHandler

```
resourcePutHandler
  ├─▶ 权限校验：Perm.Modify + 路径规则检查
  ├─▶ 只允许文件（拒绝目录 PUT）
  ├─▶ 检查文件是否存在（不存在返回 404）
  └─▶ d.RunHook → writeFile()  ← 实际写入
       ├─▶ MkdirAll（确保目录存在）
       ├─▶ OpenFile(O_RDWR|O_CREATE|O_TRUNC)  ← 截断式覆盖
       ├─▶ io.Copy（从请求 body 写入文件）
       ├─▶ file.Sync()  ← 强制刷盘，防止断电丢失
       └─▶ 设置 ETag 响应头（modTime + size 的十六进制）
```

核心写入函数：`http/resource.go` → `writeFile()`

```go
// 关键：O_TRUNC 意味着整体覆盖，不存在增量写入
file, err := afs.OpenFile(dst, os.O_RDWR|os.O_CREATE|os.O_TRUNC, fileMode)
io.Copy(file, in)
file.Sync()  // 确保数据落盘
```

**关键观察**：`resourcePutHandler` 只检查文件**是否存在**，不检查文件**是否被修改过**。没有基于 ETag、修改时间或内容哈希的版本校验，直接全盘覆盖。

### 3.5 ETag 的"名存实亡"

后端生成 ETag：`fmt.Sprintf(`"%x%x"`, info.ModTime().UnixNano(), info.Size())`

但 **编辑器保存流程并未使用 ETag 做乐观并发控制**：
- 前端保存请求不携带 `If-Match` 或 `If-Unmodified-Since` 头
- 后端 `resourcePutHandler` 不校验任何前置条件
- ETag 仅作为响应元信息返回，未参与任何冲突判断

---

## 四、冲突处理机制（核心重点）

FileBrowser 的"冲突"概念实际上包含**三种完全不同的语义**，它们作用于不同层面、检测方式不同、后果也不同。这三者经常被混淆，需要明确区分：

| 冲突类型 | 作用层面 | 检测方式 | 后果 | 是否真的"冲突" |
|---------|---------|---------|------|---------------|
| 未保存更改提示 | 前端本地 | Ace UndoManager 状态 | 用户可能丢失自己的修改 | ❌ 不是真正冲突，是防误操作 |
| 文件操作冲突 | 上传/复制/移动 | 前端预检测 + 后端 409 | 目标文件已存在 | ✅ 是真正冲突 |
| 编辑器并发覆盖 | 多人同时编辑 | **无任何检测** | 后保存者静默覆盖先保存者的内容 | ⚠️ 是真正冲突但完全未处理 |

### 4.1 第一层：未保存更改提示（本地防误操作）

**本质**：这不是真正的"冲突"，而是单用户场景下的**防误操作保护**。

**核心机制**：利用 Ace 编辑器内置的 UndoManager 状态追踪。

```
Ace UndoManager 状态机（纯内存状态）：
  ┌──────────┐    edit     ┌──────────┐
  │  Clean   │────────────▶│  Dirty   │
  │ (无修改) │◀────────────│ (有修改) │
  └──────────┘  markClean  └──────────┘
```

检测时机（三处）：

1. **路由跳转前**：`Editor.vue:onBeforeRouteUpdate`
2. **关闭编辑器时**：`Editor.vue:close()`
3. **页面卸载/刷新**：`Editor.vue:handlePageChange`

检测逻辑：
```typescript
if (!editor.value?.session.getUndoManager().isClean()) {
  // 弹窗 DiscardEditorChanges.vue 让用户选择：保存 / 丢弃 / 取消
  layoutStore.showHover({ prompt: "discardEditorChanges", ... });
}
```

**关键特征**：
- 完全在浏览器内存中运行，不涉及服务器
- 只关心"用户有没有未保存的编辑"，不关心服务器上文件是否变化
- 如果用户打开文件后服务器上文件已被他人修改，这个机制**完全察觉不到**

### 4.2 第二层：文件操作冲突（上传/复制/移动）

**本质**：这是真正的冲突检测——目标位置已存在同名文件。

**适用场景**：文件上传、复制、移动、拖拽操作（**不包括编辑器保存**）。

#### 4.2.1 前端预检测：frontend/src/utils/upload.ts → checkConflict()

```
checkConflict(files, basePath, includeDirectories)
  ├─▶ api.fetchAll(basePath)  // 单次请求获取目标目录完整递归列表
  ├─▶ 构建 serverMap（以相对路径为 key）
  └─▶ 遍历待操作文件，逐一匹配：
       ├─▶ 命中 → 生成 ConflictingResource（含双方时间、大小）
       └─▶ 未命中 → 无冲突
```

关键设计：
- `conflictKey` 使用 `fullPath || name`（原始文件名），**不使用 URL 编码的 `to` 字段**，避免空格、`#`、非 ASCII 字符导致匹配失败
- `includeDirectories`：上传时文件夹允许合并（不报告目录冲突），复制/移动时目录本身也算冲突

#### 4.2.2 用户决策界面：ResolveConflict.vue

文件位置：`frontend/src/components/prompts/ResolveConflict.vue`

提供五种冲突解决策略：

| 操作 | checked 值 | 效果 |
|------|-----------|------|
| 覆盖 (Override) | `["origin"]` | 用源文件替换目标 |
| 跳过 (Skip) | `["dest"]` | 保留目标，放弃当前操作 |
| 重命名 (Rename) | `["origin","dest"]` | 自动加序号后缀保存 |
| 全部覆盖 / 全部跳过 / 全部重命名 | 批量设置所有项 | 快速处理 |
| 续传 (Resume) | 按大小判断 | 服务端文件更小则覆盖（断点续传语义） |

#### 4.2.3 后端兜底：409 Conflict

前端检测可能存在时序窗口（检测后、执行前他人写入），后端做最终校验：

**POST（上传/创建）**：`http/resource.go` → `resourcePostHandler`
```go
file, err := files.NewFileInfo(...)  // 尝试获取目标信息
if err == nil {  // 文件已存在
  if r.URL.Query().Get("override") != "true" {
    return http.StatusConflict, nil  // ← 409 冲突
  }
  // override=true 时继续覆盖写入
}
```

**PATCH（复制/移动）**：`http/resource.go` → `resourcePatchHandler`
```go
if !override && !rename {
  if _, err = d.user.Fs.Stat(dst); err == nil {
    return http.StatusConflict, nil  // ← 409 冲突
  }
}
if rename {
  dst = addVersionSuffix(dst, d.user.Fs)  // 自动生成 file(1).txt, file(2).txt
}
```

自动重命名算法：`http/resource.go` → `addVersionSuffix()` — 从 `(1)` 开始递增，直到找到不存在的文件名。

**为什么编辑器保存不用这套机制？**
- 上传/复制/移动是"创建新文件"语义，目标可能已存在
- 编辑器保存是"更新现有文件"语义，PUT 方法本身就是幂等覆盖
- 设计上假设用户打开文件后立即编辑保存，不会有并发修改

### 4.3 第三层：编辑器并发覆盖（静默覆盖，未处理）

**这是最隐蔽也最危险的冲突：完全没有检测，用户永远不会知道自己的保存覆盖了别人的修改。**

#### 4.3.1 静默覆盖的完整链路分析

```
时间轴 →
用户A                    服务器                     用户B
  │                        │                        │
  ├─ GET /api/resources/f.txt ──▶                   │
  │ ◀──── 返回 content="hello", modTime=T1 ──────── │
  │ 内容存入 Ace，UndoManager=Clean                  │
  │                        │                        │
  │                        │  ◀── GET /api/resources/f.txt ──┤
  │                        │  ──── 返回 content="hello", modTime=T1 ──▶
  │                        │                        │ 内容存入 Ace
  │ 用户编辑为 "hello A"    │                        │ 用户编辑为 "hello B"
  │ UndoManager=Dirty      │                        │ UndoManager=Dirty
  │                        │                        │
  ├─ PUT /api/resources/f.txt ──▶                   │
  │   body: "hello A"     │                        │
  │                        │ writeFile 覆盖写入     │
  │                        │ modTime 更新为 T2      │
  │ ◀──── 200 OK, ETag=T2 ──────────────────────── │
  │ markClean()            │                        │
  │                        │                        │
  │                        │  ◀── PUT /api/resources/f.txt ──┤
  │                        │    body: "hello B"    │
  │                        │  writeFile 覆盖写入   │
  │                        │  modTime 更新为 T3    │
  │                        │  ──── 200 OK, ETag=T3 ──▶
  │                        │                        │ markClean()
  │                        │                        │
  └────────────────────────┴────────────────────────┘
       结果：用户A的修改 "hello A" 被静默覆盖，
            两人都不会收到任何提示。
```

#### 4.3.2 为什么会静默覆盖？三层原因

**第一层：HTTP 语义层面**
- PUT 方法的标准语义就是"替换目标资源"，本身就是覆盖语义
- 没有乐观锁（`If-Match` / `If-Unmodified-Since`）就等于无条件覆盖

**第二层：后端代码层面**
- `resourcePutHandler` 只检查文件是否存在（`afero.Exists`），不检查版本
- 对比一下：`resourcePostHandler` 有 override 参数保护，`resourcePatchHandler` 有 conflict 检测，唯独 PUT 没有
- ETag 生成了但不校验，形同虚设

```go
// resourcePutHandler 的完整逻辑（关键缺失点）
var resourcePutHandler = withUser(func(w http.ResponseWriter, r *http.Request, d *data) (int, error) {
    if !d.user.Perm.Modify || !d.Check(r.URL.Path) {
        return http.StatusForbidden, nil
    }
    if strings.HasSuffix(r.URL.Path, "/") {
        return http.StatusMethodNotAllowed, nil
    }

    exists, err := afero.Exists(d.user.Fs, r.URL.Path)
    // ↑ 只检查"是否存在"，不检查"是否被修改过"
    if err != nil { return http.StatusInternalServerError, err }
    if !exists { return http.StatusNotFound, nil }

    // 直接写入，没有任何版本校验 ← 静默覆盖的根本原因
    err = d.RunHook(func() error {
        info, writeErr := writeFile(d.user.Fs, r.URL.Path, r.Body, ...)
        ...
    }, "save", r.URL.Path, "", d.user)

    return errToStatus(err), err
})
```

**第三层：前端代码层面**
- 保存时只发送内容，不携带任何版本标识
- 保存成功后直接 `markClean()`，不更新文件元信息（如 modTime）
- 没有任何"保存前检查文件是否已被修改"的逻辑

```typescript
// Editor.vue:save() — 保存时只发送纯文本内容
await api.put(route.path, editor.value?.getValue());
// ↑ 没有携带原始修改时间、ETag 或内容哈希
editor.value?.session.getUndoManager().markClean();
```

#### 4.3.3 与"未保存更改提示"的本质区别

| 维度 | 未保存更改提示 | 编辑器并发覆盖 |
|------|-------------|-------------|
| **作用对象** | 单个用户 vs 自己的未保存修改 | 多个用户 vs 彼此的编辑 |
| **检测范围** | 浏览器内存状态 | 服务器文件系统状态 |
| **是否感知** | 用户能看到"脏状态"（星号、弹窗） | 用户完全无感知 |
| **数据风险** | 丢失自己的编辑（可恢复） | 丢失他人的编辑（不可恢复） |
| **技术方案** | Ace UndoManager | 需要乐观锁/悲观锁/协作编辑 |
| **当前状态** | ✅ 已实现 | ❌ 完全未实现 |

一句话总结：
> **未保存更改提示是"你自己还没保存"，并发覆盖是"别人已经改了你不知道"。前者是防呆设计，后者才是真正的并发冲突。**

---

## 五、三大流程的调用关系图

```
                         ┌─────────────────────┐
                         │  用户访问 /files/x.txt │
                         └──────────┬──────────┘
                                    ▼
                ┌───────────────────────────────────┐
                │  【读取】 Files.vue:fetchData()   │
                │    api.fetch() → GET /api/resources│
                │    后端 NewFileInfo + detectType  │
                │    Content 注入 Ace Editor        │
                └───────────────┬───────────────────┘
                                │
          ┌─────────────────────┼─────────────────────┐
          ▼                     ▼                     ▼
┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│ 用户编辑内容      │  │ 用户点击保存/Ctrl+S│  │ 用户导航离开      │
│ 修改状态 Dirty   │  │ 【保存】          │  │ 【本地防呆】      │
│  (本地状态)      │  │                   │  │  UndoManager检测 │
└────────┬─────────┘  └─────────┬────────┘  └─────────┬────────┘
         │                      │                     │
         │                      ▼                     ▼
         │           api.put() → PUT /api/resources  isClean()?
         │                后端 writeFile()            ├─▶ Clean → 直接离开
         │                O_TRUNC 整体覆盖            └─▶ Dirty → 弹窗确认
         │                ❌ 无版本校验                     │
         │                      │                          ▼
         │                      ▼                    save() 或 丢弃
         │              markClean()
         │                      │
         │                      │
         │                      ▼
         │              ⚠️  静默覆盖风险
         │              (多人同时编辑时)
         └──────────────────────┘
                                │
                                ▼
                    ┌───────────────────────┐
                    │ 【文件操作冲突】        │
                    │ 上传/复制/移动前        │
                    │ checkConflict() 预检测 │
                    │ ResolveConflict 弹窗   │
                    │ override/rename/skip   │
                    │ 后端 409 兜底校验       │
                    └───────────────────────┘
```

---

## 六、关键代码文件索引

| 文件路径 | 作用 |
|---------|------|
| `frontend/src/views/files/Editor.vue` | 编辑器组件：初始化、保存、未保存检测 |
| `frontend/src/views/Files.vue` | 路由入口：资源加载、视图分发 |
| `frontend/src/api/files.ts` | 前端 API：fetch/put/资源操作 |
| `frontend/src/utils/upload.ts` | checkConflict() 冲突预检测（上传/复制/移动） |
| `frontend/src/components/prompts/ResolveConflict.vue` | 冲突解决弹窗（上传/复制/移动场景） |
| `frontend/src/components/prompts/DiscardEditorChanges.vue` | 未保存更改提示弹窗 |
| `frontend/src/stores/file.ts` | Pinia 文件状态管理 |
| `frontend/src/utils/encodings.ts` | 编码处理：CSV 等文件的二进制转文本 |
| `http/resource.go` | 后端资源 API：GET/PUT/POST/PATCH + writeFile |
| `http/http.go` | 路由注册：各 HTTP 方法绑定 |
| `files/file.go` | 文件元信息构建：detectType() 文本类型判断与内容读取 |
| `frontend/src/types/file.d.ts` | TypeScript 类型定义：Resource/ConflictingResource 等 |
