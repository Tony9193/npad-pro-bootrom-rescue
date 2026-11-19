# BMAX MaxPad I11 Plus / N-one Npad Pro（Unisoc T616 / UMS9230）玩机与 BootROM 救砖记录

> 记录时间：2026-11-18 ~ 2026-11-19  
> 适用机型：BMAX MaxPad I11 Plus / N-one Npad Pro（`ro.product.model=Npad-Pro`，`ro.product.device=I11_Plus`，**Unisoc T616 + UFS + Android 12 版**）

> ⚠️ **注意区分版本**：后续有换用 **T606 + Android 13** 的新版机型，硬件与固件不同，**切勿混刷**。

---

## 1. 设备硬件与系统信息

| 项目 | 实测信息 |
|---|---|
| 型号 / 代号 | `Npad-Pro` / `I11_Plus`（品牌 `BMAX`，ODM `incar`） |
| SoC | 紫光展锐 **Unisoc Tiger T616（UMS9230-AB）** |
| CPU 架构 | `arm64-v8a`（6× Cortex-A55 @ 1.82GHz + 2× Cortex-A75 @ 1.95GHz） |
| GPU | ARM Mali-G57（OpenGL ES 3.2 / Vulkan 1.1） |
| 内存 / 存储 | 8GB RAM + 128GB **UFS 2.1**（`userdata` 约 113GB） |
| 屏幕 | 10.36" IPS，1200×2000（280 dpi） |
| 出厂系统 | Android 12（API 31，`userdebug`，安全补丁 `2022-09-01`） |
| 分区特性 | Virtual A/B（当前 `_a` 槽，`super` 约 5.6GB），支持 Treble / VNDK 31 / DSU（`gsi_tool`） |
| 解锁状态 | 出厂 Bootloader 已解锁（`flash.locked=0`，`verifiedbootstate=orange`），支持 `adb root` |

---

## 2. 故障背景与原因分析

在尝试为该平板移植 postmarketOS 时：
1. 实测该机型的 `fastboot boot` 为空操作，无法临时引导镜像，因此直接执行了 `fastboot flash boot_a test-boot7.img`。
2. 重启后内核无法挂载 rootfs（`unknown-block(1,0)`）触发 Kernel Panic，设备陷入以下两个画面无限循环：
   - U-Boot 阶段：`INFO: LOCK FLAG IS : UNLOCK!!!` / `WARNNING: LOCK FLAG IS : UNLOCK, SKIP VERIFY!!!`
   - 崩溃转储阶段：`Sysdumping now, keep power on.`
3. 此时无法进入 Android，也无法通过常规组合键稳定停留在 Fastboot。

### 关键踩坑总结

- **插着 USB 长按电源键不会关机**：会直接触发 PMIC 硬件冷重启，并因 USB 供电立刻再次开机进死循环。
- **不要用 Reset 针孔进刷机模式**：该 `userdebug` 工程固件会将 `ext rstn` 识别为硬件异常复位（`Reset mode: ext rstn reboot`），直接跳进 Sysdump。
- **BootROM 握手窗口极短（约 2~3 秒）**：进入 `VID_1782&PID_4D00` 后若电脑端没有提前装好驱动且提前运行监听工具，芯片会自动超时退出并重新开机。
- **USB 集线器（Hub）容易导致枚举失败**：建议直接插到电脑机身原生 USB 口，避免出现 `未知 USB 设备(设备描述符请求失败)`。
- **必须使用 UFS 版 FDL1/FDL2**：本机存储为 UFS，**切勿使用 UMS9230 eMMC 设备的 FDL1/FDL2**。
- **已实测确认的 BootROM 按键**：本机进入芯片底层 BootROM（`VID_1782&PID_4D00`）的实体按键为 **「音量减（Volume-）」键**。

---

## 3. BootROM 救砖步骤（Windows + `spd_dump` + Zadig）

> 前提：手头已提前备份原厂 `boot_a.img`（64MB，强烈建议玩机前先通过 `adb root` 备份 A/B 两槽的 `boot`、`init_boot`、`vendor_boot`、`dtbo`、`vbmeta` 及 `misc`）。

### 准备工具
1. **Zadig**（如 `zadig-2.9.exe`）：用于给 `VID_1782&PID_4D00` 安装 `WinUSB` 驱动。
2. **libusb 版 `spd_dump.exe`**（来自 `TomKing062/action_spd_dump_it` 的 `oldpath` x64 Release，含 `libusb-1.0.dll`）。
3. **UMS9230 UFS 版引导与绕过文件**（来自 `TomKing062/CVE-2022-38694_unlock_bootloader` 的 `ums9230_universal_unlock_UFS` 包）：
   - `custom_exec_no_verify_65015f08.bin`
   - `fdl1-dl.bin`
   - `fdl2-dl.bin`

### 第一步：利用 Sysdump 提前装好 WinUSB 驱动
1. 将平板**直连电脑机身 USB 口**，让其停在 `Sysdump` 界面（此时电脑会稳定枚举出 `Gadget Serial`，USB ID 为 `1782:4D00`，不会超时断开）。
2. 以管理员身份运行 **Zadig**，点击菜单 **Options → List All Devices**。
3. 下拉选中 **`Gadget Serial`**（核对 USB ID 为 `1782` / `4D00`），目标驱动选 **`WinUSB`**，点击 **Install Driver / Replace Driver**。

### 第二步：电脑端先启动 `spd_dump` 守株待兔
在放有 `spd_dump.exe` 与三个 `.bin` 文件的目录打开 PowerShell，提前运行监听命令（等待 300 秒）：

```powershell
.\spd_dump.exe --wait 300 `
  exec_addr 0x65015f08 `
  fdl fdl1-dl.bin 0x65000800 `
  fdl fdl2-dl.bin 0x9efffe00 `
  exec `
  w boot_a "C:\path\to\boot_a.img" `
  reset
```

### 第三步：按住「音量减（Volume-）」键触发 BootROM
1. 拔掉平板 USB 线，让平板黑屏（或在强制重启黑屏瞬间操作）。
2. **按住「音量减（Volume-）」键不放**（不要碰电源键），将 USB 线插回电脑。
3. `spd_dump` 会立刻捕获 BootROM（`BSL_REP_VER: "SPRD3"`），自动完成：
   - 加载 `fdl1-dl.bin`（`0x65000800`）与 `custom_exec_no_verify_65015f08.bin`（`0x65015f08`）
   - 加载 `fdl2-dl.bin`（`0x9efffe00`）并读取 UFS GPT 分区表（共 72 个分区）
   - 将 64MB 原厂 `boot_a.img` 写回 `boot_a`（日志提示 `Force Write boot_a Done`）并自动重启。

### 第四步：重启进入 Android 桌面
写回完成后平板脱离死循环，由原厂内核启动进入 **FastbootD**（`USB\VID_1782&PID_4EE0`）：
- 在平板屏幕上用音量键选择 **`Reboot system now`**，按电源键确认即可回到原厂 Android 12 桌面。

---

## 4. 系统净化记录（ADB 精简）

原厂系统已可直接 `adb root`，可通过以下命令清理预装第三方应用及冗余出厂测试工具（不影响系统核心与 Google Play）：

```bash
# 卸载第三方预装
adb shell pm uninstall --user 0 com.youku.phone
adb shell pm uninstall --user 0 com.qihoo.appstore
adb shell pm uninstall --user 0 com.jadard.testtool

# 为当前用户禁用/隐藏出厂测试与日志工具
for pkg in \
  com.sprd.logmanager \
  com.incar.update \
  com.guanhong.guanhongpcb \
  com.sprd.autoslt \
  com.sprd.validationtools \
  com.sprd.engineermode \
  com.sprd.commlog \
  com.sprd.cameracalibration; do
  adb shell pm uninstall -k --user 0 "$pkg"
done
```

---

## 5. 后续玩机与安全提示

- **最安全的新系统体验方式**：优先使用本机内置的 **DSU（`gsi_tool`）** 临时加载 ARM64 Android 12/13 GSI，不覆盖原系统，失败可直接切回。
- **切勿随意切换 B 槽**：本机 `super` 中 B 槽动态分区为空，切槽会导致无法启动（若误切槽，可通过刷回原厂 `misc` 分区切回 A 槽）。
- **高危分区切勿乱动**：`uboot_*`、`sml_*`、`trustos_*`、`teecfg_*`、`hypervsior_*`、`vbmeta_*`、`l_modem_*`、`l_fixnv*`、`prodnv`、`miscdata`（含 IMEI / Wi-Fi MAC 等校准信息）。

## 6. 推荐“玩花”路线

| 路线 | 好玩程度 | 成熟度 | 建议 |
|---|---:|---:|---|
| Android 12/13 ARM64 GSI（DSU） | ★★★★ | 较高 | 最先尝试，避免永久覆盖原系统；AOSP 可做干净系统，LineageOS GSI 可体验更自由的 Android |
| N-one Npad Pro 专用 TWRP | ★★★★ | 测试版 | 作为恢复/安装底座；作者称 Release 版本可工作，但项目自 2023 年后停止维护，刷前必须保留原厂 `vendor_boot` |
| Android 14+ GSI | ★★★★★ | 中低 | 外观和功能更新，但原厂 vendor 为 Android 12，Wi-Fi、摄像头、音视频可能不兼容 |
| postmarketOS / Alpine 用户态 + 原厂内核 | ★★★★★ | 移植中 | 有 T616 主线基础，但本机 DTS、DRM、触控、UFS、电源管理尚缺完整集成；需要继续解决 initramfs 与 vendor DTB 兼容 |
| Android TV / 复古游戏 / Termux Linux | ★★★★ | 高 | 不必刷底层系统即可长期玩：Kodi/Jellyfin、RetroArch、Debian/proot、Kali，风险最低 |
| Ubuntu Touch / Sailfish / Windows ARM | ★★★ | 很低 | 当前没有本机专用移植，不建议直接刷 |

### 已验证的专用 TWRP 资源

仓库：[ManInTheMirrow/twrp_device_incar_Npad-Pro](https://github.com/ManInTheMirrow/twrp_device_incar_Npad-Pro)

- Release：`vendor_boot_nocrypt.img`
- 大小：`104857600` 字节（100 MiB），与本机 `vendor_boot_a/b` 分区大小完全匹配
- SHA-256：`01DC53956836E0FC91CFB6206F0CBF4D6FC4D197D8E2F5FCC3E7F6699FF57ED2`
- 设备树采用 Android 12 Header v4，并将 recovery ramdisk 放进 `vendor_boot`
- 注意：仓库已停止维护，作者 README 提到部分版本黑屏；即使 image 能启动，也不能假定触控、解密或 MTP 全部可用

### 建议实施顺序

1. 确认 Android 已从救砖后的 FastbootD 正常重启，重新上线 ADB。
2. 先用 DSU 试 Android 12/13 ARM64 GSI；验证 Wi-Fi、触控、旋转、音频、视频硬解与充电。
3. 如需更方便地切换系统，再将本机专用 TWRP 写入 **A 槽的 `vendor_boot_a`**；不动 B 槽，并保留原厂 `vendor_boot_a.img`。
4. TWRP 稳定后才测试 LineageOS GSI 或 Android 14+ GSI。
5. 真正的 postmarketOS 作为长期移植项目单独推进：优先完成原厂 5.4.161 内核 + Alpine rootfs 的最小启动，再解决 DRM、触控、GPU 与电源管理，不把主线内核文件夹直接照搬到本机。
