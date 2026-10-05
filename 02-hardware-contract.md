# BHNSSCAN Hardware Contract

版本：v0.1  
状态：Baseline  
适用范围：Brother DCP-T536DW + Debian 12 生产环境

---

## 1. 目的

本文档只记录真实硬件环境中已经验证，或者由实际设备明确声明的能力。

本文档不得依据以下内容自行扩展：

- 型号相近设备的能力；
- Brother 其他型号文档；
- SANE 后端的一般能力；
- Coding Agent 的推测；
- Mock/FakeScanner 的行为。

未验证内容必须明确标记为：

```text
UNKNOWN
```

如果实现依赖 UNKNOWN 项，应暂停该依赖部分，并由 orchestrator 提供真实机器 probe。

---

# 2. 生产服务器

已验证：

```text
OS:
  Debian GNU/Linux 12 (bookworm)

Kernel:
  Linux 6.1.0-27-amd64

Architecture:
  amd64 / x86_64

LAN interface:
  br0

Server LAN address:
  192.168.3.43/24
```

服务器还存在 Docker、虚拟机、隧道等其他接口。

扫描仪通信不得依赖“默认只有一个网卡”的假设。

---

# 3. 扫描设备

已验证：

```text
Vendor:
  Brother

Model:
  DCP-T536DW

Printer/scanner LAN address:
  192.168.3.177
```

服务器能够通过 LAN 与设备通信。

设备与服务器处于：

```text
192.168.3.0/24
```

同一局域网。

---

# 4. 已安装扫描相关组件

生产服务器已验证存在：

```text
sane-utils
libsane1
sane-airscan
brscan5
brscan-skey
ImageMagick
img2pdf
```

关键版本：

```text
sane-utils / libsane:
  Debian package based on 1.2.1

scanimage:
  sane-backends 1.1.1-debian

sane-airscan:
  0.99.27-1+b1

Brother brscan5:
  1.7.0-0

Brother brscan-skey:
  0.3.5-0
```

应用核心功能不得依赖 Brother `brscan5`。

---

# 5. 扫描接口发现

真实设备当前可以被 SANE 通过多种 backend 发现：

```text
brother5:net1;dev0
escl:https://192.168.3.177:443
airscan:e*:Brother DCP-T536DW
```

BHNSSCAN 生产环境固定选择：

```text
sane-airscan
```

通过：

```text
scanimage
    ↓
sane-airscan
    ↓
eSCL
    ↓
Brother DCP-T536DW
```

进行主动 Web 扫描。

以下接口不作为生产实现：

```text
brother5:
SANE built-in escl:
Brother skey-scanimage
```

其中 `brscan-skey` 会在未来作为打印机面板主动提交任务的独立入口使用，但不属于 Web ScannerDriver。

---

# 6. Device ID 规则

`sane-airscan` 当前设备字符串形式：

```text
airscan:e*:Brother DCP-T536DW
```

其中：

```text
e0
e1
...
```

属于 discovery index。

应用不得把例如：

```text
airscan:e0:Brother DCP-T536DW
```

作为长期配置常量。

应用配置使用逻辑设备名：

```text
Brother DCP-T536DW
```

真实 ScannerDriver 在启动或首次使用时通过：

```bash
scanimage -f '%d%n'
```

发现 `airscan:` 设备。

匹配规则：

```text
backend prefix = airscan:
model/name contains configured scanner name
```

匹配结果：

```text
0:
  scanner unavailable

1:
  use this device

>1:
  configuration/ambiguity error
```

实际 device identifier 可以在进程生命周期中缓存。

---

# 7. sane-airscan 声明能力

通过实际：

```bash
scanimage -d <airscan-device> --all-options
```

已经成功读取以下能力。

## 7.1 Scan source

```text
Flatbed
```

当前未发现 ADF。

应用第一版不得展示 ADF 选项。

---

## 7.2 Color mode

已声明：

```text
Color
Gray
```

第一版支持：

```text
color
gray
```

映射：

```text
color -> --mode Color
gray  -> --mode Gray
```

第一版不需要 Black/White/Lineart。

---

## 7.3 Native resolution

设备通过 sane-airscan 明确声明：

```text
100 dpi
200 dpi
300 dpi
600 dpi
```

第一版 UI 允许：

```text
100
200
300
600
```

默认：

```text
300 dpi
```

推荐高质量：

```text
600 dpi
```

---

# 8. 400 dpi 特殊情况

已进行真实扫描验证。

以下请求：

```text
--resolution 400
```

在仅设置参数、不实际扫描时可能被接受：

```text
scanimage --dont-scan
```

但实际 acquisition 会失败：

```text
Invalid argument
```

结论：

```text
400 dpi native acquisition:
  NOT SUPPORTED
```

应用不得向用户提供“原生 400 dpi”。

未来如果需要约 400 dpi 输出，可以实现：

```text
600 dpi physical acquisition
→ software downscale
→ ~400 dpi derived artifact
```

该功能属于 Image Pipeline，而非 ScannerDriver。

---

# 9. Geometry

sane-airscan 已声明可指定 scan area：

```text
left:
  0 .. 215.9 mm

top:
  0 .. ~296.926 mm

width:
  up to 215.9 mm

height:
  up to ~296.926 mm
```

约等于：

```text
A4 flatbed
```

ScannerDriver 的 ScanRequest 可以保留 optional region 字段。

v0.1 普通扫描默认：

```text
full flatbed
```

证件扫描第一版也默认：

```text
full scan
→ software crop/perspective correction
```

暂不依赖硬件 region scan 实现证件裁切。

---

# 10. Enhancement options

sane-airscan 曾声明：

```text
brightness
contrast
shadow
highlight
analog gamma
negative
```

这些能力目前不是 v0.1 产品要求。

ScannerDriver 不需要在第一阶段公开它们。

后续如果启用，必须先由真实硬件 probe 确认实际 acquisition 行为。

---

# 11. Raw acquisition format

BHNSSCAN ScannerDriver 第一版固定输出：

```text
TIFF
```

调用语义：

```text
scanimage
--format=tiff
--output-file=<destination>
```

每次：

```text
scan_page()
```

只产生一张物理扫描页。

Web 多页不是 scanner backend batch scan。

Web 多页由 ScanJob 逐页调用：

```text
scan_page()
```

完成。

---

# 12. 当前 Brother ScanKey 状态

已验证打印机面板：

```text
Scan to PC
```

能够触发 Debian 上的 `brscan-skey`。

当前已经工作的链路包括：

```text
Brother LCD
→ brscan-skey
→ multi-page TIFF
→ PNG pages
→ combined PDF
→ msmtp
→ email
```

当前配置：

```text
brscan-skey network interface:
  br0
```

Scan-to-email 使用自定义脚本：

```text
/usr/local/libexec/brother-scan-email-png
```

该现有链路必须在 Web MVP 阶段保持可用。

第一阶段不得重构或替换它。

---

# 13. 多页 TIFF

已在真实设备上验证：

打印机面板执行多页扫描时，Brother ScanKey 路径可以生成：

```text
multi-page TIFF
```

其文件大小随页数增加。

该事实仅适用于：

```text
Brother ScanKey flow
```

不能据此假设 Web/eSCL 单次 acquisition 同样提供 multi-page TIFF。

Web 多页明确采用：

```text
one scan_page call per physical page
```

---

# 14. Scanner 并发

真实硬件只允许 BHNSSCAN 同时执行：

```text
1 physical acquisition
```

不允许多个 `scanimage` 子进程同时访问设备。

应用必须通过 ScannerManager：

```text
Lock / single active scan
```

进行互斥。

---

# 15. 错误模型

ScannerDriver 至少应能够把真实执行结果映射为以下应用级错误：

```text
ScannerUnavailable
ScannerBusy
InvalidScanRequest
ScanFailed
ScanCancelled
ScanTimeout
```

第一阶段不要求精准解析所有 SANE stderr。

未知错误允许首先归类为：

```text
ScanFailed
```

同时保留诊断信息到日志。

不得把未经处理的 shell stderr 直接作为前端错误页面。

---

# 16. 取消行为

真实 sane-airscan acquisition 的取消语义尚未完整验证。

状态：

```text
UNKNOWN
```

v0.1 实现允许：

```text
terminate scanimage
→ short grace period
→ kill if required
```

该行为必须能够通过 FakeScanner 自动测试。

真实硬件验证在 production integration milestone 进行。

---

# 17. Timeout

真实设备最大合理扫描时间尚未测量。

状态：

```text
UNKNOWN
```

第一阶段不得依赖具体的“正常扫描一定在 N 秒内完成”假设。

Timeout 应可配置。

---

# 18. 远程开发环境

Coding Agent 所处环境：

```text
cannot access real Brother DCP-T536DW
```

因此：

```text
FakeScanner
```

是开发和测试的默认 ScannerDriver。

Agent 不允许因为真实设备不可访问而：

- 修改硬件契约；
- 假设打印机响应；
- 添加未经验证的型号能力；
- 用网络 mock 假装 eSCL 行为已经验证。

需要硬件事实时，应在回传中明确写：

```text
HARDWARE PROBE REQUIRED
```

并描述需要确认的具体事实。

由 orchestrator 编写 probe，由用户在真实 Debian 主机执行。

---

# 19. Probe 约定

以后真实设备 probe：

```text
temporary working directory:
  /tmp/bhnsscan-probe-*
```

或：

```text
/var/tmp/bhnsscan-probe-*
```

不得把临时 probe 脚本或输出直接堆放在：

```text
$HOME/
```

根目录。

Probe 默认必须：

- 尽量只读；
- 不覆盖现有生产配置；
- 不泄漏 SMTP/API 密钥；
- 明确说明是否会触发实际扫描；
- 输出足够的 exit code / stderr / capability 信息。

---

# 20. 架构结论

生产 ScannerDriver：

```text
SaneAirscanScanner
```

开发 ScannerDriver：

```text
FakeScanner
```

业务代码只依赖：

```text
ScannerDriver
```

不依赖具体 SANE backend。

如果未来 sane-airscan 出现不可解决的兼容性问题：

```text
replace ScannerDriver implementation
```

而不是重构 ScanJob、Web、Imaging 或 Sharing 层。
