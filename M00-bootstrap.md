# M00 — Project Bootstrap + Scanner Abstraction

状态：Ready for Agent

---

# 1. 任务目标

建立 BHNSSCAN 第一版可运行项目骨架。

本任务只完成：

```text
项目结构
配置系统
基础 FastAPI app
领域基础类型
ScannerDriver interface
FakeScanner
基础 ScannerManager
自动测试
```

本任务不连接真实打印机。

本任务不实现：

```text
真实 sane-airscan driver
ScanJob REST API
SQLite persistence
图像处理
React UI
分享
Brother ScanKey
```

---

# 2. 必须先阅读

开始编码前必须完整阅读：

```text
AGENTS.md
docs/00-project-overview.md
docs/01-architecture.md
docs/02-hardware-contract.md
docs/03-domain-model.md
```

如果其中存在冲突：

```text
不要自行选择
不要修改架构
在回传中报告
```

---

# 3. 技术要求

Backend：

```text
Python 3.11+
FastAPI
Pydantic v2
pytest
pytest-asyncio
```

项目依赖管理使用现代：

```text
pyproject.toml
```

不要加入：

```text
Poetry-specific runtime dependency
Redis
Celery
SQLAlchemy
Alembic
OpenCV
React
Docker
```

这些不属于本 milestone。

可使用：

```text
uv
pip
venv
```

但仓库不能要求开发者必须拥有某一个非标准包管理器才能运行。

---

# 4. 目标目录

完成后至少：

```text
backend/
├── app/
│   ├── __init__.py
│   ├── main.py
│   │
│   ├── config/
│   │   ├── __init__.py
│   │   └── settings.py
│   │
│   ├── domain/
│   │   ├── __init__.py
│   │   ├── scan.py
│   │   └── errors.py
│   │
│   └── scanner/
│       ├── __init__.py
│       ├── base.py
│       ├── fake.py
│       └── manager.py
│
├── tests/
│   ├── test_health.py
│   ├── test_fake_scanner.py
│   └── test_scanner_manager.py
│
└── pyproject.toml

tests/
└── fixtures/
    └── scanner/
```

可以合理增加少量辅助文件。

不得提前创建大量空模块。

---

# 5. FastAPI App

实现：

```text
backend/app/main.py
```

必须可以启动：

```bash
uvicorn app.main:app
```

至少提供：

```http
GET /health
```

返回：

```json
{
  "status": "ok"
}
```

不要在 `/health` 中尝试连接真实扫描仪。

---

# 6. 配置系统

实现：

```text
Settings
```

最少字段：

```text
scanner_kind
data_dir
fake_scanner_fixture
fake_scanner_delay_ms
```

开发默认：

```text
scanner_kind = fake
```

data dir 默认应适合开发环境，例如：

```text
./var
```

不要在开发默认配置中写：

```text
/var/lib/bhnsscan
```

避免测试/本地开发需要 root。

生产路径以后由 deployment milestone 配置。

---

# 7. Domain Types

实现当前任务所需的最小 domain types。

至少：

```text
ScanColorMode
ScanRegion
ScanRequest
ScanResult
ScannerCapabilities
```

建议语义：

```python
ScanColorMode:
    COLOR
    GRAY
```

`ScanRequest`：

```text
resolution_dpi
color_mode
region optional
```

`ScanRegion`：

```text
x_mm
y_mm
width_mm
height_mm
```

`ScanResult` 至少包含：

```text
output_path
width_px optional
height_px optional
```

不要在本 milestone 实现完整 ScanJob ORM/model。

---

# 8. ScannerDriver

在：

```text
scanner/base.py
```

定义非常薄的协议/抽象边界。

语义：

```python
async get_capabilities() -> ScannerCapabilities

async scan_page(
    request: ScanRequest,
    destination: Path,
) -> ScanResult

async cancel() -> None
```

可以使用：

```text
typing.Protocol
```

优先于复杂 class hierarchy。

业务层不得依赖具体 FakeScanner 类型。

---

# 9. Scanner Errors

在：

```text
domain/errors.py
```

至少定义：

```text
ScannerError
ScannerUnavailable
ScannerBusy
InvalidScanRequest
ScanFailed
ScanCancelled
ScanTimeout
```

不要建立过深继承树。

都可直接继承：

```text
ScannerError
```

---

# 10. FakeScanner

实现：

```text
scanner/fake.py
```

要求：

## 10.1 Capabilities

返回与真实生产设备兼容的能力集合：

```text
resolutions:
100
200
300
600

color modes:
COLOR
GRAY

source:
flatbed semantics
```

不要加入真实设备未验证能力。

## 10.2 scan_page

接收：

```text
ScanRequest
destination
```

行为：

1. 验证 request；
2. 模拟 configurable delay；
3. 创建 destination parent directory；
4. 把 fixture 内容复制到 destination；
5. 返回 ScanResult。

输出必须是：

```text
TIFF
```

## 10.3 Fixture

测试仓库中必须存在至少一个合法、小体积 TIFF fixture。

可以程序生成一个非常简单的 TIFF 测试文件。

不要加入大体积 binary asset。

## 10.4 Error simulation

FakeScanner 至少提供测试可控方式模拟：

```text
unavailable
fail next scan
```

cancel 也必须有可测试语义。

实现方式由 Agent 决定，但不要引入复杂 dependency injection framework。

---

# 11. ScannerManager

实现：

```text
scanner/manager.py
```

职责：

```text
持有 ScannerDriver
保证单次 physical scan
把并发 scan_page 串行化或拒绝
转发 cancel
```

第一版使用：

```text
asyncio.Lock
```

即可。

要求：

同一时间不能有两个 driver acquisition。

具体策略选择：

```text
queue/wait
```

优先于：

```text
immediate ScannerBusy
```

但 active scan cancellation 必须可实现。

不要构建消息队列。

---

# 12. Scanner Factory / Construction

不要实现大型 Factory hierarchy。

允许非常小的应用 bootstrap 函数：

```text
build_scanner(settings)
```

v0.1 仅支持：

```text
fake
```

如果 scanner_kind 不是：

```text
fake
```

可以显式抛出配置错误。

真实：

```text
sane-airscan
```

由下一 milestone 实现。

---

# 13. 测试要求

必须至少覆盖：

## health

```text
GET /health → 200
```

## FakeScanner capabilities

验证：

```text
100/200/300/600
COLOR/GRAY
```

## Successful scan

验证：

```text
destination exists
non-empty
valid expected fixture copy
```

## Invalid resolution

例如：

```text
400 dpi
```

必须失败为：

```text
InvalidScanRequest
```

注意：

这同时保护已验证硬件契约：

```text
native 400 dpi unsupported
```

## Fake unavailable

模拟：

```text
ScannerUnavailable
```

## Fake failure

模拟：

```text
ScanFailed
```

## Concurrency

发起两个并发 scan。

验证 ScannerManager 不会同时进入 driver physical acquisition。

## Cancel

验证 active FakeScanner scan 能被取消，并得到明确的 cancellation outcome。

---

# 14. Code Quality

要求：

```text
type hints
async correctness
clear module boundaries
no shell commands
no real network calls
no production device assumptions
```

不要使用：

```text
Any
```

逃避核心 domain typing。

测试中可以适量使用 monkeypatch/mock。

---

# 15. README

本 milestone 可以更新根 README 或 backend README，至少说明：

```text
如何创建环境
如何安装 backend
如何运行测试
如何启动 FastAPI
```

必须保证一名新 Agent 可以依据 README 在无打印机环境完成运行。

---

# 16. 不允许修改的架构决定

本 Agent 不得更改：

```text
FastAPI backend
ScannerDriver abstraction
sane-airscan future production backend
FakeScanner development backend
TIFF raw format
100/200/300/600 verified resolution contract
400 dpi unsupported
modular monolith
```

不得为了“方便”改成：

```text
直接调用 scanimage
直接用 Brother backend
Mock HTTP eSCL service
```

---

# 17. 不要做的额外工作

明确禁止顺手实现：

```text
SaneAirscanScanner
SQLite
ScanJob database
Alembic
PNG conversion
PDF generation
React
SSE
Share
Authentication
Dockerfile
systemd
```

这些属于后续独立 milestone。

控制 scope 很重要。

---

# 18. 验收标准

任务完成必须满足：

```text
1. Backend installs successfully.
2. pytest passes.
3. FastAPI starts.
4. GET /health returns OK.
5. FakeScanner produces TIFF.
6. Invalid 400 dpi is rejected.
7. FakeScanner failure/unavailable modes are tested.
8. ScannerManager prevents concurrent physical acquisition.
9. Active fake scan can be cancelled.
10. No code attempts to access real scanner/network.
```

---

# 19. Agent 回传格式

完成后不要只说“done”。

必须回传：

## A. Summary

简述实现了什么。

## B. Files changed

列出新增/修改文件。

## C. Tests

提供实际执行命令和结果，例如：

```text
pytest ...
XX passed
```

## D. Architecture deviations

如果没有：

```text
None
```

如果有，逐条列出，但不得未经批准自行扩大架构。

## E. Open questions

需要 orchestrator 决策的问题。

## F. Hardware assumptions

列出实现中是否产生任何新的硬件假设。

正常结果应该是：

```text
None
```

## G. Suggested next task

可以建议，但不得自行开始下一 milestone。
