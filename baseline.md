# BHNSSCAN 架构基线 v0.1

## 1. 项目定位

BHNSSCAN 是运行于 Debian 小型服务器上的单机扫描服务，为 Brother DCP-T536DW 提供比打印机原生功能更完整的 Web 扫描工作流。

首要能力包括：

- Web 发起普通单页扫描；
- Web 管理多页扫描会话；
- 分辨率、颜色模式、输出格式等基本设置；
- 每页独立 PNG 与合并 PDF；
- 扫描历史记录；
- 证件边框自动识别；
- 用户在 Web 上拖动四角修正边框；
- 高分辨率透视矫正和裁切；
- 创建可撤销、可过期、可设置密码的分享链接；
- 后期接入现有 Brother ScanKey 面板扫描。

项目定位为家庭/个人单机服务，不设计成通用企业扫描平台。

---

# 2. 总体原则

采用 **Modular Monolith（模块化单体）**。

不拆微服务，不引入 Redis、RabbitMQ、Celery 等额外基础设施。

生产环境核心组件只有：

```text
Browser
   │
   │ HTTP / SSE
   ▼
┌──────────────────────────────────┐
│          BHNSSCAN Server         │
│                                  │
│ FastAPI                          │
│ ├── Scan Job                     │
│ ├── Scanner Manager              │
│ ├── Image Pipeline               │
│ ├── History                      │
│ ├── Share                        │
│ └── Static Frontend              │
└──────────────┬───────────────────┘
               │
               │ ScannerDriver
               ▼
       SaneAirscanScanner
               │
               │ scanimage
               ▼
          sane-airscan
               │
               │ eSCL
               ▼
      Brother DCP-T536DW
```

数据侧：

```text
SQLite
  └── metadata / jobs / pages / shares

Filesystem
  └── TIFF / PNG / preview / PDF
```

---

# 3. 固定技术栈

## Backend

Python 3.11 + FastAPI。

主要原因：

- Debian 12 原生环境友好；
- `asyncio.create_subprocess_exec()` 很适合控制 `scanimage`；
- Pillow / OpenCV / img2pdf 图像生态成熟；
- SSE 实现简单；
- 后续 ScanKey ingest 也容易接入。

数据库采用：

```text
SQLite
+
SQLAlchemy 2.x
+
Alembic
```

不使用 PostgreSQL。

## Frontend

采用：

```text
React
TypeScript
Vite
```

证件四点调整使用 SVG overlay 实现，暂不引入 Fabric.js 等大型 Canvas 框架。

开发时 Vite 独立运行。

生产构建后：

```text
frontend/dist/
```

由 FastAPI 或部署层直接提供静态资源。

---

# 4. Scanner 边界

业务代码不得直接出现：

```text
scanimage
airscan:e0:...
eSCL
Brother
```

所有真实扫描必须经过一个非常薄的接口：

```python
class ScannerDriver(Protocol):

    async def get_capabilities(
        self,
    ) -> ScannerCapabilities:
        ...

    async def scan_page(
        self,
        request: ScanRequest,
        destination: Path,
    ) -> ScanResult:
        ...

    async def cancel(
        self,
    ) -> None:
        ...
```

第一版只有两个实现：

```text
SaneAirscanScanner
FakeScanner
```

不实现：

```text
BrotherScanner
EsclScanner
AirscanScannerFactory
BackendResolver
```

生产环境固定使用：

```text
SaneAirscanScanner
→ scanimage
→ sane-airscan
→ eSCL
```

开发和自动测试使用：

```text
FakeScanner
```

---

# 5. 已冻结的硬件契约

真实设备：

```text
Brother DCP-T536DW
IP: 192.168.3.177
```

真实服务器：

```text
Debian 12 amd64
LAN: br0
Server IP: 192.168.3.43
```

生产扫描 backend：

```text
sane-airscan
```

设备当前发现形式：

```text
airscan:e*:Brother DCP-T536DW
```

应用不依赖 `e0` 这样的 discovery index。

启动时 Scanner Driver 应根据：

```text
backend == airscan
+
device name == Brother DCP-T536DW
```

从：

```bash
scanimage -f ...
```

解析实际 device identifier，并缓存。

必须满足：

```text
0 个匹配 → scanner unavailable
1 个匹配 → 正常使用
>1 个匹配 → configuration error
```

---

# 6. 已验证扫描能力

通过 sane-airscan / eSCL：

```text
Source:
Flatbed

Modes:
Color
Gray

Resolution:
100
200
300
600 dpi
```

真实扫描时：

```text
400 dpi → INVALID / 不支持
```

因此第一版不得向用户宣称支持原生 400 dpi。

UI 可以暴露：

```text
100
200
300
600
```

普通默认：

```text
300 dpi
```

高清：

```text
600 dpi
```

如果以后需要“约 400 dpi”模式，应实现为：

```text
600 dpi acquisition
→ software downscale
→ approximately 400 dpi output
```

这是 Image Pipeline 功能，不属于 Scanner Driver。

---

# 7. 原始扫描格式

Scanner Driver 的职责只到：

> 从物理扫描仪获得一页未经业务处理的原始扫描。

第一版规定：

```text
scanimage --format=tiff
```

即每次物理扫描生成一张单页 TIFF。

Web 多页扫描不是一次生成 multi-page TIFF，而是：

```text
scan page 1 → 0001.tif
换纸
scan page 2 → 0002.tif
换纸
scan page 3 → 0003.tif
```

这样多页逻辑完全由 BHNSSCAN 控制。

以后 Brother ScanKey ingest 如果收到 multi-page TIFF，再由 ingest pipeline 拆页。

---

# 8. Scan Job 是整个系统的核心领域对象

所有扫描最终都表示为：

```text
ScanJob
```

一个 Job 包含：

```text
ScanJob
├── scan settings
├── state
├── pages[]
├── artifacts[]
├── created_at
├── completed_at
└── error
```

Web 普通扫描、Web 多页扫描以及未来 ScanKey 导入，最终都进入同一个 Job 模型。

状态机冻结为：

```text
CREATED
   │
   ▼
QUEUED
   │
   ▼
SCANNING
   │
   ├─────────────┐
   ▼             │
WAITING_NEXT_PAGE│
   │             │
   └── scan ─────┘
   │
   │ finish
   ▼
PROCESSING
   │
   ▼
READY
```

任何有效阶段都可能进入：

```text
FAILED
CANCELLED
```

---

# 9. 多页扫描语义

Web 多页由浏览器控制，不调用打印机 LCD 上的多页工作流。

典型流程：

```text
Create Job
    ↓
Scan Page
    ↓
Page 1 ready
    ↓
WAITING_NEXT_PAGE
    ↓
用户换纸
    ↓
Scan Next Page
    ↓
Page 2 ready
    ↓
WAITING_NEXT_PAGE
    ↓
Finish
    ↓
PROCESSING
    ↓
READY
```

因此用户可以在 Finish 前：

```text
重新扫描某页
删除某页
调整页序
旋转某页
```

具体 API 在独立的 API Contract 文档定义。

---

# 10. Scanner 并发模型

物理扫描仪是单实例资源。

整个 BHNSSCAN 进程只能存在：

```text
1 个 active physical scan
```

ScannerManager 负责全局互斥。

第一版使用：

```text
asyncio.Lock
```

或等价的单 worker 内部队列。

禁止同时启动多个 `scanimage` 进程竞争同一扫描仪。

不引入外部消息队列。

---

# 11. 子进程模型

真实扫描使用：

```python
asyncio.create_subprocess_exec(...)
```

而不是 shell command string。

禁止：

```python
os.system(...)
subprocess(..., shell=True)
```

扫描参数必须作为独立 argv 传入。

这同时避免 shell injection。

取消扫描时：

```text
cancel Job
→ terminate scanimage process
→ 等待退出
→ 必要时 kill
→ Job = CANCELLED
```

具体超时策略后续定义。

---

# 12. Image Pipeline

扫描和输出格式完全分离。

硬件层产生：

```text
raw TIFF
```

Image Pipeline 产生：

```text
PNG
PDF
preview
cropped image
perspective-corrected image
```

推荐工具：

```text
Pillow
img2pdf
OpenCV
```

ImageMagick 不作为核心业务依赖，必要时才使用。

处理关系：

```text
RAW TIFF
   │
   ├── PNG page
   │
   ├── preview WebP
   │
   └── processed page
            │
            ├── crop
            ├── rotate
            └── perspective correction

processed pages
      │
      ▼
combined PDF
```

---

# 13. 原始文件不可修改

这是强制架构原则。

```text
raw/
```

下的内容创建后 immutable。

任何操作：

```text
裁切
旋转
透视
增强
重新调整证件四角
```

都必须重新从 raw source 渲染。

禁止：

```text
processed
→ 再处理 processed
→ 再处理 processed
```

以避免累计质量损失。

---

# 14. 文件存储布局

根目录默认：

```text
/var/lib/bhnsscan/
```

Job：

```text
/var/lib/bhnsscan/jobs/<job-id>/
├── raw/
│   ├── 0001.tif
│   ├── 0002.tif
│   └── 0003.tif
│
├── processed/
│   ├── 0001.png
│   ├── 0002.png
│   └── 0003.png
│
├── preview/
│   ├── 0001.webp
│   ├── 0002.webp
│   └── 0003.webp
│
└── export/
    └── document.pdf
```

SQLite 只存：

```text
metadata
state
settings
relative paths
sharing information
```

禁止把 TIFF/PNG/PDF 存成 SQLite BLOB。

Job ID 第一版直接使用 UUID4。

---

# 15. 输出格式

输出格式属于 Export 配置，不属于 Scanner 设置。

例如：

```json
{
  "scan": {
    "resolution": 300,
    "mode": "color"
  },
  "output": {
    "page_png": true,
    "combined_pdf": true
  }
}
```

第一版支持：

```text
每页 PNG
合并 PDF
PNG + PDF
```

JPEG 可以以后增加，但不是核心格式。

---

# 16. Preview 与原图分离

浏览器不得直接加载 300/600 dpi 完整 TIFF/PNG 用于交互。

每页生成：

```text
preview WebP
```

最长边控制在适合 Web UI 的尺寸。

证件边框识别也首先基于 preview 或降采样图运行。

最终裁切/透视处理必须回到 raw 原图执行。

---

# 17. 证件扫描模型

证件模式本质上仍然是普通 ScanJob。

不同点只是 Page 上附加：

```text
DocumentTransform
```

核心字段：

```json
{
  "corners": [
    [0.12, 0.18],
    [0.84, 0.17],
    [0.86, 0.73],
    [0.11, 0.75]
  ]
}
```

坐标统一使用：

```text
0.0 ~ 1.0 normalized coordinates
```

不得使用浏览器像素尺寸作为后端数据。

浏览器：

```text
preview + SVG four-point overlay
```

后端：

```text
normalized coordinates
→ 映射 raw pixels
→ OpenCV perspective transform
```

自动边框识别只产生初始建议。

用户永远可以手动修正。

---

# 18. 实时状态

浏览器控制命令使用普通 REST。

例如：

```text
create
scan-next-page
finish
cancel
```

服务端状态推送使用：

```text
SSE
```

而不是 WebSocket。

原因是当前通信模式主要是：

```text
Server → Browser
```

浏览器断开再连接时：

```text
GET current job state
+
重新订阅 SSE
```

SSE 不承担持久化职责。

---

# 19. 分享系统边界

分享是 ScanJob 已生成 Artifact 的只读访问层。

路径类似：

```text
/s/<random-token>
```

分享接口绝对不能：

```text
启动扫描
修改 scanner settings
调用 ScannerDriver
```

分享链接仅允许读取授权 Artifact。

Token 使用密码学安全随机值。

数据库保存：

```text
token hash
```

而非明文 token。

可选分享密码使用：

```text
Argon2id
```

支持：

```text
expiration
revocation
preview permission
download permission
```

详细安全模型另写文档。

---

# 20. Scanner Web API 与 Share API 权限隔离

逻辑上分为：

```text
/api/*
```

登录用户使用；

和：

```text
/s/*
```

公开分享使用。

分享认证绝不能获得 `/api/*` 权限。

即便未来对公网暴露分享页面，scanner control API 也可以进一步限制为 LAN/VPN/登录用户。

---

# 21. Brother ScanKey 的定位

当前已经正常工作的：

```text
Brother LCD
→ brscan-skey
→ email
```

暂时保持不动。

第一阶段不重构。

未来架构：

```text
Brother ScanKey
      │
      ▼
local ingest command
      │
      ▼
BHNSSCAN internal ingest
      │
      ▼
ScanJob
```

即：

```text
Web scan ──────┐
               │
ScanKey ingest ├──→ ScanJob
               │
Future API ────┘
```

但这是后期 milestone。

不得为了第一版 Web 功能破坏目前已经可用的 Scan-to-Email。

---

# 22. FakeScanner

远程 Agent 无法访问真实打印机，因此 FakeScanner 是一级能力，不是临时 hack。

测试素材：

```text
tests/fixtures/scanner/
├── document.tif
├── id-card.tif
├── skewed-id-card.tif
└── photo.tif
```

FakeScanner：

```text
scan_page()
→ 等待可配置的短暂延迟
→ 从 fixture 复制一份 TIFF
→ 返回 ScanResult
```

还必须支持模拟：

```text
scanner unavailable
scan failure
scan timeout
cancel
```

因此整个 Web/Job/Image/Share 开发可以在完全没有打印机的环境下完成。

---

# 23. 项目目录基线

```text
bhnsscan/
├── backend/
│   ├── app/
│   │   ├── api/
│   │   ├── domain/
│   │   ├── scanner/
│   │   ├── jobs/
│   │   ├── imaging/
│   │   ├── sharing/
│   │   ├── persistence/
│   │   ├── config/
│   │   └── main.py
│   │
│   ├── migrations/
│   ├── tests/
│   └── pyproject.toml
│
├── frontend/
│   ├── src/
│   └── package.json
│
├── tests/
│   └── fixtures/
│       └── scanner/
│
├── docs/
│   ├── 00-project-overview.md
│   ├── 01-architecture.md
│   ├── 02-hardware-contract.md
│   ├── 03-domain-model.md
│   ├── 04-api-contract.md
│   ├── 05-image-pipeline.md
│   ├── 06-security-model.md
│   └── 07-deployment.md
│
├── tasks/
│
├── AGENTS.md
└── README.md
```

模块之间允许同进程直接调用 Python interface。

不得人为拆 HTTP 微服务。

---

# 24. 配置原则

源码不能包含真实打印机 IP、SMTP 密码、分享 secret 等环境信息。

开发环境默认：

```text
scanner.kind = fake
```

生产环境：

```text
scanner.kind = sane-airscan
scanner.name = Brother DCP-T536DW
```

真实 device string 在运行时发现。

生产数据：

```text
/var/lib/bhnsscan
```

生产配置最终放置在：

```text
/etc/bhnsscan/
```

具体配置格式在 Deployment 文档确定。

---

# 25. 部署模型

第一版目标：

```text
一个 backend systemd service
+
SQLite
+
本地 filesystem
```

不要求 Docker。

前端生产 build 可由后端直接提供。

应用默认只监听本机或 LAN。

公网访问、TLS 和域名交给外部 reverse proxy，不耦合进核心扫描服务。

---

# 26. 明确不做的事情

v0.1 架构不实现：

```text
直接自己实现 eSCL HTTP 协议
Brother 私有扫描协议
微服务
Redis
Celery
RabbitMQ
PostgreSQL
Kubernetes
多扫描仪调度
多服务器
OCR
云存储
完整用户/组织权限系统
```

这些以后只有真实需求出现时再加入。

---

# 27. 架构变更规则

Coding Agent 可以实现既定架构，但不得自行改变下列边界：

```text
ScannerDriver abstraction
sane-airscan production backend
FakeScanner development backend
ScanJob state model
SQLite + filesystem split
raw immutable
REST + SSE
modular monolith
share/scanner permission isolation
```

如果实现过程中发现上述设计不可行，Agent 应：

```text
停止扩大改动范围
记录问题
给出建议
回传 orchestrator
```

由 orchestrator 决定是否修改架构。

Agent 不得自行实施跨架构变更。

---

# 28. 当前推荐实施顺序

第一阶段不连接真实打印机。

首先完成：

```text
project bootstrap
→ config
→ domain models
→ ScannerDriver
→ FakeScanner
→ basic tests
```

然后：

```text
ScanJob lifecycle
→ SaneAirscanScanner
→ Image Pipeline
→ Web UI
→ ID document workflow
→ History
→ Sharing
→ ScanKey ingest
→ Production deployment
```

每一步由独立 Agent 完成，完成后回传，由 orchestrator review 后再下发下一阶段任务。
