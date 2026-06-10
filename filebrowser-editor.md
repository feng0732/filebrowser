# FileBrowser 在线编辑器与保存流程实现脉络

本文档从代码层面梳理 FileBrowser 中在线编辑器的**内容读取**、**保存写回**、**冲突处理**三大核心环节的实现逻辑与调用关系。

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

用户访问文件时，Vue Router 加载 [Files.vue](file:///d:/fz/0601/solo-dogfeeding/code/172-filebrowser/frontend/src/views/Files.vue)，在 `onMounted` 和 `watch(route)` 中调用 `fetchData()`：

```
Files.vue:fetchData()
  └─▶ api.fetch(url, signal)          // 发起资源获取请求
       └─▶ fileStore.updateRequest(res)  // 结果存入 Pinia 状态
            └─▶ 根据 req.type 决定渲染 Editor 或 Preview
```

关键点：
- `fetchData` 支持 `AbortController` 取消重复请求（路由快速切换时）
- 通过 `computed` 属性 `currentView` 动态决定组件：
  - `text` / `textImmutable` → [Editor.vue](file:///d:/fz/0601/solo-dogfeeding/code/172-filebrowser/frontend/src/views/files/Editor.vue)
  - `csv` 默认走 Preview，`?edit=true` 时走 Editor
  - 其他类型走 Preview

### 2.2 API 层：[files.ts:fetch()](file:///d:/fz/0601/solo-dogfeeding/code/172-filebrowser/frontend/src/api/files.ts#L8-L50)

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

### 2.3 后端处理：[resource.go:resourceGetHandler](file:///d:/fz/0601/solo-dogfeeding/code/172-filebrowser/http/resource.go#L25-L82)

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

核心文件：[file.go:detectType()](file:///d:/fz/0601/solo-dogfeeding/code/172-filebrowser/files/file.go#L222-L287)

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

[Editor.vue:initEditor()](file:///d:/fz/0601/solo-dogfeeding/code/172-filebrowser/frontend/src/views/files/Editor.vue#L221-L241) 从 `fileStore.req.content` 读取初始内容：

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

### 3.2 保存执行：[Editor.vue:save()](file:///d:/fz/0601/solo-dogfeeding/code/172-filebrowser/frontend/src/views/files/Editor.vue#L269-L282)

```typescript
const save = async (throwError?: boolean) => {
  buttons.loading("save");
  try {
    // 核心调用：PUT 方法写入内容
    await api.put(route.path, editor.value?.getValue());
    // 成功后标记编辑器为"干净"状态（无未保存修改）
    editor.value?.session.getUndoManager().markClean();
    buttons.success(button);
  } catch (e) {
    buttons.done(button);
    $showError(e);  // Toast 提示错误
  }
};
```

### 3.3 API 层：[files.ts:put()](file:///d:/fz/0601/solo-dogfeeding/code/172-filebrowser/frontend/src/api/files.ts#L78-L80)

```typescript
export async function put(url: string, content = "") {
  return resourceAction(url, "PUT", content);
}
```

`resourceAction` 是通用封装，直接将内容作为 PUT 请求的 body 发送到 `/api/resources{path}`。

### 3.4 后端处理：[resource.go:resourcePutHandler](file:///d:/fz/0601/solo-dogfeeding/code/172-filebrowser/http/resource.go#L180-L210)

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

核心写入函数：[resource.go:writeFile()](file:///d:/fz/0601/solo-dogfeeding/code/172-filebrowser/http/resource.go#L296-L327)

```go
// 关键：O_TRUNC 意味着整体覆盖，不存在增量写入
file, err := afs.OpenFile(dst, os.O_RDWR|os.O_CREATE|os.O_TRUNC, fileMode)
io.Copy(file, in)
file.Sync()  // 确保数据落盘
```

### 3.5 ETag 的作用

后端生成 ETag：`fmt.Sprintf(`"%x%x"`, info.ModTime().UnixNano(), info.Size())`

但 **编辑器保存流程并未使用 ETag 做乐观并发控制**，PUT 请求不会携带 `If-Match` 头。ETag 目前仅作为响应元信息返回，未参与冲突检测。

---

## 四、冲突处理机制

FileBrowser 的冲突处理分为**两个独立层面**：

| 层面 | 作用场景 | 实现方式 |
|------|---------|---------|
| 本地未保存更改 | 用户编辑后未保存就离开 | Ace UndoManager 状态检测 |
| 多用户文件操作冲突 | 上传/复制/移动时目标已存在 | 前端预检测 + 后端 409 兜底 |

### 4.1 本地未保存更改检测

**核心机制**：利用 Ace 编辑器内置的 UndoManager 状态追踪。

```
Ace UndoManager 状态机：
  ┌──────────┐    edit     ┌──────────┐
  │  Clean   │────────────▶│  Dirty   │
  │ (无修改) │◀────────────│ (有修改) │
  └──────────┘  markClean  └──────────┘
```

检测时机（三处）：

1. **路由跳转前**：[Editor.vue:onBeforeRouteUpdate](file:///d:/fz/0601/solo-dogfeeding/code/172-filebrowser/frontend/src/views/files/Editor.vue#L201-L219)
2. **关闭编辑器时**：[Editor.vue:close()](file:///d:/fz/0601/solo-dogfeeding/code/172-filebrowser/frontend/src/views/files/Editor.vue#L298-L317)
3. **页面卸载/刷新**：[Editor.vue:handlePageChange](file:///d:/fz/0601/solo-dogfeeding/code/172-filebrowser/frontend/src/views/files/Editor.vue#L260-L267)

检测逻辑：
```typescript
if (!editor.value?.session.getUndoManager().isClean()) {
  // 弹窗 [DiscardEditorChanges.vue] 让用户选择：保存 / 丢弃 / 取消
  layoutStore.showHover({ prompt: "discardEditorChanges", ... });
}
```

### 4.2 多用户文件操作冲突

#### 4.2.1 前端预检测：[upload.ts:checkConflict()](file:///d:/fz/0601/solo-dogfeeding/code/172-filebrowser/frontend/src/utils/upload.ts#L43-L89)

适用于：文件上传、复制、移动、拖拽操作。

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

#### 4.2.2 用户决策界面：[ResolveConflict.vue](file:///d:/fz/0601/solo-dogfeeding/code/172-filebrowser/frontend/src/components/prompts/ResolveConflict.vue)

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

**POST（上传/创建）**：[resource.go:resourcePostHandler](file:///d:/fz/0601/solo-dogfeeding/code/172-filebrowser/http/resource.go#L125-L178)
```go
file, err := files.NewFileInfo(...)  // 尝试获取目标信息
if err == nil {  // 文件已存在
  if r.URL.Query().Get("override") != "true" {
    return http.StatusConflict, nil  // ← 409 冲突
  }
  // override=true 时继续覆盖写入
}
```

**PATCH（复制/移动）**：[resource.go:resourcePatchHandler](file:///d:/fz/0601/solo-dogfeeding/code/172-filebrowser/http/resource.go#L212-L262)
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

自动重命名算法：[resource.go:addVersionSuffix()](file:///d:/fz/0601/solo-dogfeeding/code/172-filebrowser/http/resource.go#L278-L294) — 从 `(1)` 开始递增，直到找到不存在的文件名。

### 4.3 编辑器保存的"缺失"：并发覆盖风险

需要特别注意：**文本编辑器的 PUT 保存没有任何并发冲突检测**。

```
用户A打开文件 test.txt → 内容 "hello"
用户B同时打开 test.txt → 内容 "hello"
用户A修改为 "hello A" → 保存成功 ✅
用户B修改为 "hello B" → 保存成功 ✅（A 的修改被静默覆盖！）
```

原因：
- PUT 请求未携带 `If-Match: <etag>` 头
- 后端 `resourcePutHandler` 只检查文件是否存在，不校验版本
- 整个保存流程遵循"最后写入者获胜"（Last Write Wins）策略

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
│ 修改状态 Dirty   │  │ 【保存】          │  │ 【本地冲突检测】  │
└────────┬─────────┘  └─────────┬────────┘  └─────────┬────────┘
         │                      │                     │
         │                      ▼                     ▼
         │           api.put() → PUT /api/resources  UndoManager.isClean()
         │                后端 writeFile()            ├─▶ Clean → 直接离开
         │                O_TRUNC 整体覆盖            └─▶ Dirty → 弹窗确认
         │                      │                          │
         │                      ▼                          ▼
         │              markClean()                    save() 或 丢弃
         │                      │
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

| 文件 | 作用 |
|------|------|
| [Editor.vue](file:///d:/fz/0601/solo-dogfeeding/code/172-filebrowser/frontend/src/views/files/Editor.vue) | 编辑器组件：初始化、保存、未保存检测 |
| [Files.vue](file:///d:/fz/0601/solo-dogfeeding/code/172-filebrowser/frontend/src/views/Files.vue) | 路由入口：资源加载、视图分发 |
| [files.ts](file:///d:/fz/0601/solo-dogfeeding/code/172-filebrowser/frontend/src/api/files.ts) | 前端 API：fetch/put/资源操作 |
| [upload.ts](file:///d:/fz/0601/solo-dogfeeding/code/172-filebrowser/frontend/src/utils/upload.ts) | checkConflict() 冲突预检测 |
| [ResolveConflict.vue](file:///d:/fz/0601/solo-dogfeeding/code/172-filebrowser/frontend/src/components/prompts/ResolveConflict.vue) | 冲突解决弹窗 |
| [DiscardEditorChanges.vue](file:///d:/fz/0601/solo-dogfeeding/code/172-filebrowser/frontend/src/components/prompts/DiscardEditorChanges.vue) | 未保存更改提示 |
| [resource.go](file:///d:/fz/0601/solo-dogfeeding/code/172-filebrowser/http/resource.go) | 后端资源 API：GET/PUT/POST/PATCH + writeFile |
| [file.go](file:///d:/fz/0601/solo-dogfeeding/code/172-filebrowser/files/file.go) | 文件元信息构建：detectType() 文本类型判断与内容读取 |
| [http.go](file:///d:/fz/0601/solo-dogfeeding/code/172-filebrowser/http/http.go) | 路由注册：各 HTTP 方法绑定 |
