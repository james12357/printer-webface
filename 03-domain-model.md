# BHNSSCAN Domain Model

版本：v0.1  
状态：Baseline

---

# 1. 设计目标

领域模型围绕：

```text
ScanJob
```

组织。

所有扫描来源最终统一进入 ScanJob。

来源可以是：

```text
WEB
SCANKEY
IMPORT
```

v0.1 只真正实现：

```text
WEB
```

但数据模型允许以后增加其他来源。

---

# 2. 核心对象关系

```text
ScanJob
  │
  ├── ScanSettings
  │
  ├── Page 1
  │     └── PageTransform
  │
  ├── Page 2
  │     └── PageTransform
  │
  ├── ...
  │
  ├── Artifacts
  │     ├── page PNG
  │     ├── previews
  │     └── combined PDF
  │
  └── Shares
```

---

# 3. ScanJob

ScanJob 代表一次逻辑扫描会话。

建议字段：

```text
id: UUID
source: ScanSource
status: ScanJobStatus

resolution_dpi: int
color_mode: ScanColorMode

created_at: datetime
updated_at: datetime
completed_at: datetime | null

error_code: str | null
error_message: str | null
```

可选：

```text
title: str | null
```

第一版不强制用户命名。

---

# 4. ScanSource

```text
WEB
SCANKEY
IMPORT
```

v0.1：

```text
WEB
```

必须完整支持。

其他枚举值可以暂不使用。

---

# 5. ScanJobStatus

冻结状态：

```text
CREATED
QUEUED
SCANNING
WAITING_NEXT_PAGE
PROCESSING
READY
FAILED
CANCELLED
```

含义：

## CREATED

Job metadata 已建立，但尚未进入物理扫描。

## QUEUED

等待 ScannerManager 获取独占扫描资源。

## SCANNING

正在执行真实或 FakeScanner acquisition。

## WAITING_NEXT_PAGE

至少一页扫描成功。

系统等待用户：

```text
scan next
finish
delete/reorder/etc.
```

## PROCESSING

不再接受新物理页面。

正在：

```text
render page outputs
generate preview
generate PDF
```

## READY

任务完成。

Artifacts 可下载、查看、分享。

## FAILED

不可恢复错误终止。

## CANCELLED

用户或系统主动取消。

---

# 6. 合法状态转换

核心路径：

```text
CREATED
→ QUEUED
→ SCANNING
→ WAITING_NEXT_PAGE
```

追加页：

```text
WAITING_NEXT_PAGE
→ QUEUED
→ SCANNING
→ WAITING_NEXT_PAGE
```

完成：

```text
WAITING_NEXT_PAGE
→ PROCESSING
→ READY
```

失败：

```text
CREATED
QUEUED
SCANNING
WAITING_NEXT_PAGE
PROCESSING
→ FAILED
```

取消：

```text
CREATED
QUEUED
SCANNING
WAITING_NEXT_PAGE
→ CANCELLED
```

第一版不要求：

```text
PROCESSING → CANCELLED
```

但未来可以加入。

禁止：

```text
READY → SCANNING
FAILED → SCANNING
CANCELLED → SCANNING
```

需要重新扫描时创建新 job，或者在 READY 之前 rescan page。

---

# 7. ScanSettings

逻辑结构：

```text
resolution_dpi
color_mode
region
```

## resolution_dpi

整数。

生产环境当前合法：

```text
100
200
300
600
```

FakeScanner 可以接受相同集合。

## color_mode

```text
COLOR
GRAY
```

## region

第一版：

```text
null
```

为主。

结构预留：

```text
x_mm
y_mm
width_mm
height_mm
```

不得把浏览器像素坐标混入 ScanSettings。

---

# 8. Page

一张 Page 对应一个逻辑页面。

建议字段：

```text
id: UUID
job_id: UUID

position: int

raw_path: relative path
processed_path: relative path | null
preview_path: relative path | null

width_px: int | null
height_px: int | null

created_at: datetime
updated_at: datetime
```

`position` 表示用户定义的页序。

例如：

```text
1
2
3
```

而不是依赖文件名排序。

---

# 9. Raw Page

每次成功 physical acquisition 后：

```text
raw/<page-id>.tif
```

立即成为 immutable。

任何后续行为不得覆盖 raw。

如果用户点击：

```text
重新扫描此页
```

正确行为是：

1. 扫描产生新的 raw file；
2. Page 指向新的 raw；
3. 旧 raw 根据文件生命周期策略延迟删除或标记 orphan。

不得在旧 raw 文件路径上覆盖写入。

---

# 10. PageTransform

每页可以有一组声明式 transform。

第一版预留：

```text
rotation
crop/perspective corners
```

建议结构：

```text
rotation_degrees:
  0 | 90 | 180 | 270

perspective_corners:
  null
  or normalized four-point coordinates
```

normalized points：

```json
[
  [0.12, 0.18],
  [0.84, 0.17],
  [0.86, 0.73],
  [0.11, 0.75]
]
```

约定顺序：

```text
top-left
top-right
bottom-right
bottom-left
```

范围：

```text
0.0 <= x <= 1.0
0.0 <= y <= 1.0
```

最终渲染永远：

```text
raw
+
PageTransform
→ processed
```

不得：

```text
processed
+
new transform
→ processed again
```

---

# 11. Document edge detection

自动证件检测的结果不是不可变真值。

它仅产生：

```text
SuggestedPerspective
```

前端可以展示给用户调整。

用户确认后写入：

```text
PageTransform.perspective_corners
```

因此领域上应区别：

```text
algorithm suggestion
```

与：

```text
committed transform
```

第一版实现时可以暂时不持久化 suggestion。

---

# 12. Artifact

Artifact 是面向用户的派生文件。

类型第一版：

```text
PAGE_PNG
COMBINED_PDF
PREVIEW
```

建议字段：

```text
id: UUID
job_id: UUID
page_id: UUID | null

kind: ArtifactKind
relative_path: str
mime_type: str

size_bytes: int | null
created_at: datetime
```

其中：

```text
PAGE_PNG:
  page_id != null

PREVIEW:
  page_id != null

COMBINED_PDF:
  page_id == null
```

---

# 13. Artifact 生成原则

Artifacts 都是可再生数据。

理论 source of truth：

```text
raw pages
+
page order
+
PageTransform
+
export settings
```

因此删除 processed/preview/export 后，可以重新构建。

SQLite 不保存二进制 artifact。

---

# 14. ExportSettings

第一版：

```text
page_png: bool
combined_pdf: bool
```

默认：

```text
page_png = true
combined_pdf = true
```

因此典型结果：

```text
page-01.png
page-02.png
...
document.pdf
```

JPEG 暂不属于 v0.1 必需能力。

---

# 15. Job 完成语义

用户执行：

```text
finish
```

之后：

```text
WAITING_NEXT_PAGE
→ PROCESSING
```

此后禁止继续追加物理页面。

Processing：

1. 按 position 读取 Page；
2. 从 raw + transform 渲染 processed page；
3. 生成 PNG；
4. 生成 preview；
5. 按页序生成 combined PDF；
6. 完成全部 DB artifact metadata；
7. 状态置为 READY。

如果其中关键导出失败：

```text
PROCESSING → FAILED
```

不得把部分完成误报为 READY。

---

# 16. Page reorder

只修改：

```text
Page.position
```

不重命名 raw。

后续生成 PDF 时按照：

```text
position ASC
```

排序。

---

# 17. Page deletion

在 READY 前：

```text
delete page
```

允许。

至少必须保留：

```text
1 page
```

才能 finish。

如果所有页面被删除，Job 返回：

```text
CREATED
```

或维持：

```text
WAITING_NEXT_PAGE
```

第一版选择：

```text
WAITING_NEXT_PAGE
```

直到用户扫描新页或 cancel。

---

# 18. Page rescan

语义：

```text
rescan page N
```

不是创建新位置。

流程：

```text
existing Page
→ physical scan
→ new raw TIFF
→ replace Page.raw reference
→ invalidate derived page artifacts
```

页序保持不变。

失败时应保留旧 raw，避免因重扫失败损坏已有页面。

因此实际 replacement 必须具有事务语义：

```text
new scan succeeds
→ atomically switch Page.raw_path
```

---

# 19. ScannerManager

ScannerManager 不属于领域实体，但属于核心 domain service。

职责：

```text
single scanner ownership
queue / lock
execute scan
cancel active scan
map driver errors
```

它不负责：

```text
PNG conversion
PDF creation
Share
HTTP
```

---

# 20. ImagePipeline

ImagePipeline 属于 domain/application service。

接口方向：

```text
raw Page
+
PageTransform
→ rendered Page

pages
→ PDF
```

不允许 ImagePipeline：

```text
调用 ScannerDriver
写 Share token
处理 HTTP request
```

---

# 21. Share

后期加入：

```text
Share
```

建议字段：

```text
id: UUID
job_id: UUID

token_hash
password_hash | null

expires_at | null
revoked_at | null

allow_preview: bool
allow_download: bool

created_at
```

Share 只能引用：

```text
READY ScanJob
```

禁止分享：

```text
SCANNING
PROCESSING
FAILED
CANCELLED
```

---

# 22. Persistence Boundary

Repository interface 负责：

```text
ScanJob metadata
Page metadata
Artifact metadata
Share metadata
```

Filesystem service 负责：

```text
raw files
processed files
preview files
exports
```

应用逻辑不应手工拼接任意未经校验的外部路径。

数据库保存相对路径。

实际路径必须始终限制在配置的数据根目录内。

---

# 23. FakeScanner 行为

FakeScanner 必须表现得像 ScannerDriver，而不是绕过领域逻辑。

调用：

```text
scan_page(request, destination)
```

产生：

```text
TIFF at destination
```

并返回：

```text
ScanResult
```

FakeScanner 至少支持配置：

```text
fixture image
artificial delay
fail next scan
unavailable
```

cancel 需要可自动测试。

---

# 24. 时间

所有数据库时间统一使用：

```text
UTC
```

API 输出 ISO 8601。

浏览器负责本地化显示。

文件名不作为 authoritative timestamp。

---

# 25. ID

第一版统一使用：

```text
UUID4
```

适用于：

```text
ScanJob
Page
Artifact
Share
```

公开分享 URL 不直接使用 Share UUID 作为安全 token。

---

# 26. 错误信息

领域层区分：

```text
machine-readable error code
human-readable message
```

例如：

```json
{
  "code": "SCANNER_UNAVAILABLE",
  "message": "Scanner is currently unavailable."
}
```

日志可包含详细底层诊断。

API 返回不得泄漏：

```text
服务器绝对文件路径
shell command
环境变量
secret
完整内部 traceback
```

---

# 27. v0.1 非目标

领域模型当前不解决：

```text
OCR
document text search
multiple users/organizations
multiple scanners
cloud storage
version history
full audit log
distributed workers
```

出现真实需求再扩展。
