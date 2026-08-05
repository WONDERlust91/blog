---
title: 手机刷入 LineageOS 后 Root 与修复 Google 信任问题全记录
publishDate: 2026-08-01
tags: ["cellphone", "LineageOS", "Root"]
description: 从密钥链破坏到完整修复 · KernelSU + NeoZygisk + TEESimulator-RS + Tricky-Add-on Enhanced + PlayIntegrityFix-Inject-s 实战笔记
---

从密钥链破坏到完整修复 · KernelSU + NeoZygisk + TEESimulator-RS + Tricky-Add-on Enhanced + PlayIntegrityFix-Inject-s 实战笔记

---

## 一、问题起源

在安卓手机上刷入第三方纯净系统 LineageOS 后，系统密钥链被破坏，手机不再被 Google Play 信任。随之而来的是一连串业务失效：

- **京东**：无法在苹果店消费，领不了高价值优惠券
- **淘宝**：无法进入闪购页面，无法在苹果店购物
- **美团**：被直接断网
- **12306**：无法完成人脸识别

想彻底解决这些问题，只有两条路：要么换回原厂系统并重新锁回 Bootloader；要么继续走 Root 路线，把被破坏的信任链一块块修回来。

---

## 二、Root 三大路线

目前主流 Root 方案大致分为三条路线，每条路线下又有各种衍生分支：

- **Magisk**：曾经的王者，如今已不推荐（检测对抗能力下降、维护方向变化）
- **APatch**：更新节奏偏慢，社区活跃度一般
- **KernelSU**：隐藏性好、模块生态健康，是当前更推荐的选择

本文后续全部基于 **KernelSU** 展开。

---

## 三、Root 一句话原理

> 下载当前系统对应的 `boot.img` → 在手机上安装 KernelSU → 把 `boot.img` 传到手机，用 KernelSU 修补成带 Root 的版本 → 回传到电脑 → 用 fastboot 线刷回手机。

整条链路非常清晰，不需要复杂的 payload 或额外分区操作。

---

## 四、刷机注意事项

### 4.1 获取 boot.img 与内核版本

LineageOS 官网可以直接下载对应机型的 `boot.img`，同时也能查到自己手机的内核版本，方便在 KernelSU 修补时选择正确版本。KernelSU 官网的说明已经比较老旧，直接按上面的「一句话原理」操作即可。

LineageOS Wiki 也提供了到 Android 官网的链接，可下载最新的 fastboot 工具和 USB 驱动。

### 4.2 USB 驱动强制安装（「张冠李戴」法）

Windows 经常出现 USB 驱动安装失败的情况。官方驱动中可能没有你的手机型号，因此无法识别，此时可以采用强制「张冠李戴」法，直接指定设备类型：

1. 右键带感叹号的设备 → 更新驱动程序 → 浏览我的电脑以查找驱动程序。
2. 点击「让我从计算机上的可用驱动程序列表中选取」。
3. 如果列表里让你选择设备类型，**不要选「显示所有设备」**，而是往下拉找到 **Android 设备（Android Device）** 或 **通用串行总线设备**，双击进入。
4. 左侧厂商选择 **Google, Inc.**，右侧型号选择 **Android Bootloader Interface**。
5. 点击下一步。Windows 会弹出非常严厉的不兼容警告窗口，不用理会，直接点「是 / 确定」强行写入。

> **注意**：这种方法跳过了硬件 ID 匹配检查，属于强制指定。如果之后设备异常，可在设备管理器中卸载驱动后重新插拔尝试。

另外提醒：不要去下载完整的 Android Studio 只为了装驱动，体积太大，得不偿失。直接从 Android 官网下载独立的 USB 驱动即可。

### 4.3 电脑 USB 烧录 Bug

部分电脑在插了太多 USB 设备后，会出现无法烧录的报错。解决方法很简单：重启电脑，然后只插手机再进行烧录操作。

---

## 五、刷机结束后的模块选型

### 5.1 Zygisk 选型

Zygisk 原本只适配 Magisk。后来出现的 Zygisk-Next 已改为闭源且带有痕迹。最终选择了技术更新更积极的 **NeoZygisk**。不过作者精力有限，目前主要集中在维护 **Vector**（LSPosed 的重构版）。

### 5.2 整体架构关系

理解各组件的层级关系非常重要：

```txt
KernelSU                  ← Root 基座
    ↓
Zygisk (NeoZygisk)        ← 中间层，可视为特殊的 KernelSU 模块，其他模块常依赖它
    ↓
LSPosed / Vector          ← 既是 KernelSU 模块，也是应用层模块的基座
    ↓
HMA 等应用层模块          ← 运行在 LSPosed / Vector 之上（或直接作为 KernelSU 模块）
```

### 5.3 LSPosed 与 Vector 的现状

LSPosed 原作者已不再维护。现有团队接手后近期有积极维护，但因转为闭源，最终没有选择。NeoZygisk 作者开发了技术更先进的开源版 **Vector**，需要安装应用层模块时可考虑它。

隐藏应用列表模块 **HMA** 是几乎必装的，因此通常需要 Vector。但因为原版 HMA 也改为闭源，最终选择了开源版 **HMA-OSS**。开源版已不再是应用层模块，而是直接做成了 KernelSU 模块，因此也无需再装 Vector。

### 5.4 HMA-OSS

HMA-OSS 用于隐藏应用列表，使某些应用在另一些应用中不可见。比原版 HMA 更好用，而且省去了安装 LSPosed 的麻烦。

---

## 六、推荐工具

### 6.1 文件管理：MiXplorer

Root 环境下强烈推荐使用 **MiXplorer** 进行文件浏览与管理。

- 免费版可在 XDA 论坛和官网下载
- 收费的 **Silver 版**可在 Google Play 商店购买，功能更完整（支持更多压缩格式、云盘、加密等）

### 6.2 执行脚本：Termux

执行 `.sh` 文件推荐使用 **Termux**。

需要注意：Termux **默认不走系统 VPN**，如果需要通过代理访问网络，必须手动设置 `http_proxy` 参数，例如：

```bash
export http_proxy=http://127.0.0.1:端口号
export https_proxy=http://127.0.0.1:端口号
```

否则即使手机已开启 VPN，Termux 内的网络请求仍会直连。

---

## 七、Bootloader 与证书链修复

这是整个修复链路中最关键的部分。尝试了多种方案后的结论如下：

- Tricky Store 更新频率已明显降低
- TrickyStore-OSS 已一年未更新
- Google 近期加强了验证，旧库已经跟不上。即使有作者做插件（如 Tricky Add-on），也无法通过最新验证

NeoZygisk / Vector 作者用新技术开发了 **TEE-Simulator**，但因精力都放在 Vector 重构上，维护不足。社区有人维护了第三方版 **TEESimulator-RS**，该作者还额外开发了 **TA Enhanced**，可让证书链的 `Keybox.xml` 自动维护，实现了目前最好的 TEE / Bootloader 修复效果。TA Enhanced 安装后作为 TEESimulator-RS 的插件使用。

---

## 八、PlayIntegrityFix

PlayIntegrityFix 主要修复 TEE 证书以外的 system prop 类参数。例如 LineageOS 的 `build.type` 是 `userdebug`，在正式发布的系统中这属于风险信号，PlayIntegrityFix 会把它改成正常值。

原版 PlayIntegrityFix 已不维护。目前较知名的两个分支：

- **PlayIntegrityFixFork**：可自定义较多，但需要手动改配置文件并执行命令行，下载过程经常失败，最终放弃
- **PlayIntegrityFix-Inject-s**：有图形界面，能完成基本修复，虽然自定义能力不如 Fork，但稳定好用，最终采用此方案

---

## 九、检测修复结果

检测工具很多，但多数已闭源且停止开发。目前推荐仍在更新且开源的：

- **Duck-Detector-Refactoring**：检测能力强，持续更新
- **Play Integrity Check**（Play 商店可下）：用于简单的 Play Integrity 检测

---

## 十、正确保留 Root 的 OTA 更新步骤

现代手机基本都是 A/B 分区，KernelSU 官方支持「安装到另一个分区」：

1. 打开 系统更新 → 下载并**安装**本周的新 LineageOS 包。
2. 安装完成后先不要重启（非常重要）。
3. 打开 KernelSU 管理器。
4. 点击 Working 状态项 进入「安装」→ 选择 「安装到另一个分区（After OTA）」 / Install to inactive slot。
5. 等待修补完成。
6. 重启（或者回到系统更新界面，从那里重启也行）。

重启后就会进入新系统，并且 Root 仍然保留。

这是 KernelSU 官方文档里明确推荐的 OTA 后操作方式。

## 十一、一点观察

一圈折腾下来，明显感觉到这个圈子里很多开源作者被滥用、黑产以及不良用户逼得选择了闭源，转而在 Telegram 渠道发布。真正还在坚持开源的作者已经越来越少。

---

## 十二、参考链接

1. [KernelSU 官方文档](https://kernelsu.org/zh_CN/)
2. [KernelSU 模块仓库](https://modules.kernelsu.org/)
3. [LineageOS Marble 刷机指南（额外分区）](https://wiki.lineageos.org/devices/marble/install/variant3/#flashing-additional-partitions)
4. [LineageOS Marble 构建下载](https://download.lineageos.org/devices/marble/builds)
5. [LineageOS ADB/Fastboot 指南](https://wiki.lineageos.org/adb_fastboot_guide#installing-adb-and-fastboot)
6. [Android 官方 Windows USB 驱动](https://developer.android.com/studio/run/win-usb)
7. [NeoZygisk](https://github.com/JingMatrix/NeoZygisk)
8. [TEESimulator](https://github.com/JingMatrix/TEESimulator)
9. [Vector](https://github.com/JingMatrix/Vector)
10. [PlayIntegrityFix](https://github.com/KOWX712/PlayIntegrityFix)
11. [TEESimulator-RS](https://github.com/Enginex0/TEESimulator-RS)
12. [TA Enhanced](https://github.com/Enginex0/tricky-addon-enhanced)
13. [HMA-OSS](https://github.com/frknkrc44/HMA-OSS)
14. [Detection 相关工具](https://github.com/rushiranpise/detection)
15. [Duck-Detector-Refactoring](https://github.com/eltavine/Duck-Detector-Refactoring)
16. [MiXplorer](https://xdaforums.com/t/app-2-3-mixplorer-v6-x-released-fully-featured-file-manager.1523691/#post-23109280)
17. [termux](https://termux.dev/en/)

---

_本文根据个人实际操作整理，仅供学习交流。Root 与刷机有风险，请自行做好备份并评估后果。_
