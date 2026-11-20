# 紫光展锐 T616（UMS9230）平板深度玩机、救砖与跨代系统移植全经验手册

> **适用设备**：N-one Npad Pro / BMAX MaxPad I11 Plus（ODM `incar`，代号 `I11_Plus`，主板 `ums9230_1h10`，**Unisoc T616 + 8GB + 128GB UFS 2.1 + Android 12 版**）  
> **沉淀背景**：从初始硬件摸底、出厂精简、裸机 Linux 引导变砖死循环、芯片级 BootROM 救砖，到最终通过 `pstore` 字节级破案并在 35 秒内无损点亮 Android 15 定制系统（Miku UI）的全链路实战总结。

---

## 目录

1. [硬件底包画像与“天生开挂”体质](#1-硬件底包画像与天生开挂体质)
2. [分区表、启动链与 Boot v4 字节级结构](#2-分区表启动链与-boot-v4-字节级结构)
3. [Fastboot、Sysdump 与实体按键的五大反直觉陷阱](#3-fastbootsysdump-与实体按键的五大反直觉陷阱)
4. [终极不死鸟：BootROM 芯片级救砖实战（`spd_dump` + UFS）](#4-终极不死鸟bootrom-芯片级救砖实战spd_dump--ufs)
5. [跨代刷机神器：DSU 双系统与 Android 14/15/16+ GSI 通用点亮秘籍](#5-跨代刷机神器dsu-双系统与-android-141516-gsi-通用点亮秘籍)
6. [修改 Android 12+ EXT4 系统镜像的三大生死细节](#6-修改-android-12-ext4-系统镜像的三大生死细节)
7. [原生 Linux（postmarketOS / Droidian / Ubuntu Touch）可行性与破案真相](#7-原生-linuxpostmarketos--droidian--ubuntu-touch可行性与破案真相)
8. [底层玩机无损排障方法论（排障三板斧）](#8-底层玩机无损排障方法论排障三板斧)
9. [机器人上位机实战：Root Chroot Ubuntu 22.04 + ROS 2 Humble + Mali-G57 硬件加速与桌面一键启动器](#9-机器人上位机实战root-chroot-ubuntu-2204--ros-2-humble--mali-g57-硬件加速与桌面一键启动器)

---

## 1. 硬件底包画像与“天生开挂”体质

### 1.1 核心硬件实测参数
- **SoC**：紫光展锐 **Unisoc Tiger T616（UMS9230-AB）**，12nm，6× Cortex-A55 @ 1.82GHz + 2× Cortex-A75 @ 1.95GHz
- **GPU**：ARM Mali-G57 MP1（OpenGL ES 3.2 / Vulkan 1.1）
- **存储**：8GB LPDDR4X + **128GB UFS 2.1**（`/dev/block/sda`，共 72 个 GPT 分区，`userdata` 可用约 113GB）
- **显示与触控**：10.36" `1200×2000` IPS（面板 `JD9366`，`lcd_base=0x9e000000`；触控内建 `Himax HX83102` + 模块 `jadard_ts.ko`）
- **内核与架构**：`Linux 5.4.161 aarch64`（64 位 `arm64-v8a`，兼容 32 位 `armeabi-v7a`）

### 1.2 为什么这台机器极适合折腾？
出厂固件（`Npad-Pro_userdebug_W22.24.2_20230202`）实际上是一版**未锁死的工程测试固件（`userdebug` + `test-keys`）**：
1. **出厂已解锁 Bootloader**：`ro.boot.flash.locked=0`、`ro.boot.vbmeta.device_state=unlocked`、`ro.boot.verifiedbootstate=orange`，省去了展锐设备最痛苦的签名令牌解锁流程。
2. **原生自带 Root**：直接插线执行 `adb root` 即可拿到 `uid=0(root)` 权限（系统还自带 `/system/xbin/su`）。
3. **完整支持 Treble 与 DSU**：`ro.treble.enabled=true`、`ro.vndk.version=31`、Dynamic Partitions + Virtual A/B，内置 `/system/bin/gsi_tool` 与 `gsid` 动态双系统服务。

### 1.3 头号防错红线：严防同名硬件改款混刷
BMAX 后续推出过同名的“新版 MaxPad I11 Plus”，处理器降级为 **T606 + Android 13 + 256GB（部分批次为 eMMC）**。两代产品的存储控制器、FDL 引导与设备树完全不通用，**绝对不能混刷 `.pac` 固件或混用 eMMC 版 FDL**。

---

## 2. 分区表、启动链与 Boot v4 字节级结构

### 2.1 紫光展锐安全启动链顺序
```text
芯片只读 BootROM (USB 1782:4D00)
  └─> splloader (256KB, 初始化基础环境)
       └─> sml_a / trustos_a / teecfg_a / hypervsior_a (安全监控与 TEE)
            └─> uboot_a (3MB, 展锐定制 U-Boot, 提供 Fastboot 与显示初始化)
                 └─> vbmeta_a 校验 (orange 解锁态下跳过强校验，打印 SKIP VERIFY)
                      └─> 加载 boot_a + vendor_boot_a + dtbo_a 启动 Linux 内核
```

### 2.2 Boot Header v4 分区职责真相
很多旧教程习惯把内核、DTB、ramdisk 全塞进 `boot.img`，但本机采用 **Android 12 Boot Image v4** 拆分结构：

| 分区 | 容量 | 实测头部与格式 | 真实作用 |
|---|---:|---|---|
| `boot_a` | 64 MB | `ANDROID!` v4（Header `1584B`） | 存放 `47,016,448` 字节内核 `Image` + `2,542,275` 字节 **LZ4 Legacy** 通用 ramdisk（含 `/init` 入口） |
| `init_boot_a` | 8 MB | 全 `0x00` | 本机未启用（空分区） |
| `vendor_boot_a` | 100 MB | `VNDRBOOT` v4（Page `4096B`） | 存放 `23,721,019` 字节 **LZ4 Legacy** 厂商 ramdisk（含内核 `.ko` 模块、`/first_stage_ramdisk`）+ **`162,805` 字节真实 DTB** + 内核 `cmdline` |
| `dtb_a` | 8 MB | 空 | 摆设分区，真实 DTB 在 `vendor_boot_a` 内 |
| `dtbo_a` | 8 MB | DTBO 表 | 板级设备树叠加层（`dtbo_idx=1`） |

### 2.3 `super` 动态分区与“空 B 槽”大坑
- `super` 分区总大小 `5,872,025,600` 字节（约 **5.47 GiB**）。
- 通过 `lpdump` 实测：`system_a`、`system_ext_a`、`vendor_a`（约 644 MiB）、`product_a`、`vendor_dlkm_a` 全部位于 `group_unisoc_a`；而 **`_b` 槽的所有逻辑分区（`system_b`、`vendor_b` 等）Extents 全为空**！
- **切槽必死定律**：千万不要执行 `bootctl set-active-boot-slot 1` 切去 B 槽，切过去一定无法启动。
- **误切槽救回方法**：本机的 A/B 槽位控制块（BCB，`BOOT_CTRL_MAGIC = 0x42414342 "BCAB"`）位于 `misc` 分区（`sda3`）偏移 **`0x800`** 处（`5f 61 00 00` 即 `"_a"`）。只要把备份的原厂 `misc.img` 刷回 `misc` 分区即可强制切回 A 槽。

### 2.4 玩机前必做的“救命备份”与禁区
- **必备份清单**：`boot_a/b`、`vendor_boot_a/b`、`init_boot_a/b`、`dtbo_a/b`、`vbmeta_a/b`、`misc`（合计不到 400MB，导出后务必核对设备端与电脑端 `SHA-256` 一致）。
- **绝对禁区（切勿擦写）**：`prodnv`、`miscdata`、`l_fixnv1/2_*`、`l_runtimenv*`（保存本机唯一 IMEI、Wi-Fi/蓝牙 MAC 及射频校准参数，擦除且无备份将永久丢失基带校准）以及 `splloader`、`uboot_*`、`sml_*`、`trustos_*`。

---

## 3. Fastboot、Sysdump 与实体按键的五大反直觉陷阱

在救砖排障过程中，常规高通/联发科手机的经验在本机上几乎全部翻车，原因如下：

### 陷阱 1：`fastboot boot xxx.img` 是彻头彻尾的“假动作”
本机 U-Boot 的 `fastboot boot` 命令未真正实现内存引导：无论传给它合法的 `boot.img`、截断的残缺镜像还是 4KB 随机垃圾文件，终端都会返回 `Booting OKAY`，随后直接重启去引导闪存里原有的 `boot_a`。**结论：不要用 `fastboot boot` 验证镜像好坏，测试结果完全无效。**

### 陷阱 2：`fastboot getvar` 部分变量会直接挂死 USB 会话
执行 `fastboot getvar all` 会立即返回空，而查询 `fastboot getvar unlocked` 会直接卡死整个 Fastboot USB 管道。在脚本中只应使用 `fastboot devices` 或确定支持的单变量查询。

### 陷阱 3：插着 USB 无法彻底关机，且长按电源键会触发 `manual_dump`
1. 只要插着 USB 线，长按电源键 10~15 秒触发 PMIC 硬复位后，由于 VBUS 有电，平板会立即重新上电，根本不会停留在关机状态。
2. 更反直觉的是：因为本机是 `userdebug` 固件，**长按电源键强制断电重启会被 U-Boot 判定为手动抓日志复位（`Reset mode: manual_dump`）**，开机直接停在黑底白字的 `Sysdumping now, keep power on.` 界面！
   - **破局口诀**：看到 `Reset mode: manual_dump` **不要慌，更不要再长按电源键**（再长按又会再次进 `manual_dump` 循环！）。看屏幕最后一行提示 `check key volumn up pressed to abort`——**只需轻按一下「音量加（Volume+）」键**，就会立即中止转储并正常启动进系统！

### 陷阱 4：捅 `RESET` 针孔不会救砖，反而直送 Sysdump
机器侧边的复位孔触发的是硬件 `ext_rstn` 引脚复位。U-Boot 检测到该复位源后打印 `Reset mode: ext rstn reboot`，同样无条件拦截进 Sysdump。**救砖与卡模式时严禁使用 Reset 孔。**

### 陷阱 5：四种模式对应四种不同 USB ID，且极其挑剔 USB 集线器
Windows 按 `VID + PID` 独立匹配驱动，装了 ADB 驱动不代表 Fastboot / FastbootD / BootROM 能识别：

| 所处模式 | USB 硬件 ID | Windows 推荐驱动 | 用途 |
|---|---|---|---|
| Android 系统 (ADB) | `VID_18D1&PID_4EE8` | ADB Interface | 日常调试、`adb root`、`gsi_tool` |
| U-Boot Bootloader | `VID_18D1&PID_4EE0` | Google `android_winusb.inf` | 底层 `fastboot flash` |
| Userspace FastbootD | `VID_1782&PID_4EE0` | WinUSB / Android Bootloader | 动态分区刷写（注意 VID 变成了展锐 `1782`！） |
| **BootROM / Sysdump** | **`VID_1782&PID_4D00`** | **Zadig 安装 `WinUSB`** | **`spd_dump` 芯片级救砖读写** |

> ⚠️ **物理连接铁律**：通过扩展坞 / USB Hub 连接时，BootROM 与 Sysdump 极易枚举失败变成 `未知 USB 设备(设备描述符请求失败) (VID_0000&PID_0002)`。底层刷机务必**直插电脑主板原生 USB 接口**。

---

## 4. 终极不死鸟：BootROM 芯片级救砖实战（`spd_dump` + UFS）

当 `boot_a` 被刷坏、开机直接 Kernel Panic 进 Sysdump、按键又进不去 Fastboot 时，紫光展锐固化在芯片 ROM 里的 **BootROM 下载模式（`VID_1782&PID_4D00`）** 是绝对刷不死的终极退路。

### 4.1 为什么之前“按遍所有组合键都没反应”？
因为 **UMS9230 的 BootROM 握手窗口只有约 2~3 秒**！
哪怕你按对了按键进入了黑屏 BootROM，只要电脑端：
1. 没提前为 `1782:4D00` 装好 `WinUSB` 驱动，**或者**
2. 没提前把 `spd_dump --wait 300` 挂在后台循环监听发送 `0x7E`（`SPRD3`）握手包，  
芯片等满 2~3 秒收不到下载指令，就会**自动超时退出 BootROM 并正常走闪存启动**，一头撞回坏掉的 `boot_a`！人类肉眼根本来不及在插线后再去点软件或装驱动。

### 4.2 巧借 Sysdump 界面“逃课”装驱动
正常黑屏 BootROM 只活 2 秒很难在 Zadig 里点中，但本机崩溃停在 **`Sysdump` 界面时，USB 枚举出的 `Gadget Serial` 硬件 ID 恰好也是 `VID_1782&PID_4D00`，而且永久停住不超时**！
1. 让平板停在 Sysdump 界面直插电脑；
2. 打开 **Zadig** → `Options -> List All Devices` → 选中 `Gadget Serial (1782:4D00)` → 目标选 **`WinUSB`** → 点击安装；
3. 驱动一次装好后，真正黑屏进 BootROM 的那一瞬间 Windows 就能在几毫秒内完成绑定！

### 4.3 实测验证的 UMS9230 UFS 救砖工具链与命令
- **工具选型**：必须使用支持 `libusb` 后端的 `spd_dump.exe`（`TomKing062/action_spd_dump_it` 的 `oldpath` x64 Release，带 `libusb-1.0.dll`），配合 `ums9230_universal_unlock_UFS` 中的三个文件：
  - `custom_exec_no_verify_65015f08.bin`（CVE-2022-38694 BROM 签名校验绕过载荷，加载地址 `0x65015f08`）
  - `fdl1-dl.bin`（初始化 LPDDR4X 内存，加载地址 `0x65000800`）
  - `fdl2-dl.bin`（初始化 UFS 2.1 存储控制器并读写 GPT 分区，加载地址 `0x9efffe00`）
- **本机 BootROM 触发键（实测确认）**：**单按「音量减（Volume-）」键**。
- **一键写回原厂 `boot_a` 标准流程**：
  1. 电脑端先执行监听（守株待兔）：
     ```powershell
     .\spd_dump.exe --wait 300 `
       exec_addr 0x65015f08 `
       fdl fdl1-dl.bin 0x65000800 `
       fdl fdl2-dl.bin 0x9efffe00 `
       exec `
       w boot_a "C:\path\to\boot_a.img" `
       reset
     ```
  2. 平板拔掉 USB 线黑屏 → **按住「音量减（Volume-）」不放（不按电源键）插回 USB 线**；
  3. `spd_dump` 瞬间捕获 `SPRD3`，开启 `RAW_DATA` 高速模式（约 38 MB/s），2 秒写完 64MB `boot_a` 并重启恢复！

> **经验补充**：`spd_dump` 的 `reboot-fastboot` 命令在本机写完 `misc` 后仍可能落回串口状态，不如直接用 `w <分区名> <镜像>` 直接写回目标分区干脆可靠。

---

## 5. 跨代刷机神器：DSU 双系统与 Android 14/15/16+ GSI 通用点亮秘籍

### 5.1 为什么强烈推荐优先用 DSU（`gsi_tool`）玩花活？
1. **绝对零变砖风险**：原厂 `boot_a`、`vendor_boot_a` 和 `super` 分区一个字节都不动，默认开启 `one_shot_boot`（单次启动保护），就算刷进去的系统彻底死机，重启一次立刻回到原厂 Android 12。
2. **打破 4.8GB `super` 物理容量枷锁**：本机 `super` 扣掉 `vendor` 后最多只能塞约 4.8GB 的 `system.img`，很多大厂移植 GSI（OneUI / HyperOS 等解压后 5~7GB）硬刷 `super` 根本装不下；而 DSU 直接在剩余 90GB+ 的 `/data/gsi/dsu/` 下分配镜像文件，容量随便给！
3. **国内玩机选包铁律**：优先选 **Vanilla（不含 GApps 谷歌套件）** 版本！带 GApps 的高版本 GSI 首次开机必进 Google SetupWizard，国内无透明代理网络会死卡在“正在检查更新…”进不去桌面；Vanilla 版开机直接秒进桌面。

### 5.2 踩坑 1：`gsi_tool install` 报 `Could not commit live image data (rc=70)`
- **现象**：用输入重定向 `gsi_tool install ... < /data/local/tmp/system.img` 安装时，分区创建到 100% 后立刻报错退出（返回码 `70`）。
- **根因（`logcat` 实测）**：`gsi_tool` 会把标准输入的 fd 通过 Binder 传给后台服务 `gsid`（域 `u:r:gsid:s0`）。使用 `< file` 时传入的是 `u:object_r:shell_data_file:s0` 普通文件句柄，被 SELinux 拦截 `avc: denied { read }`！
- **标准解法**：
  1. `adb root` 后先执行 `setenforce 0`；
  2. 改用 **`cat` 匿名管道**传流，且**必须显式指定 `--userdata-size`**（阅读 AOSP `gsi_tool.cpp` 源码发现：虽然 `--help` 声称默认 8GiB，但代码里未传参时实际传 `0`，回退到 `gsid` 内部仅分配 2GiB！）：
     ```sh
     setenforce 0
     sz=$(stat -c %s /data/local/tmp/miku_fixed.img)
     cat /data/local/tmp/miku_fixed.img | gsi_tool install --no-reboot --userdata-size 8589934592 --gsi-size $sz
     ```

### 5.3 踩坑 2：Android 15 GSI 卡死在开机动画（`pstore` 破案全过程）
- **现象**：首次刷入原版 `MikuUI-Vampire_v2-0.6.1-arm64-ab`（Android 15）后，能顺利看到初音未来开机动画，但卡在动画超过 5 分钟进不去系统，且电脑端无任何 USB/ADB 设备枚举。
- **破案过程（提取 `/sys/fs/pstore/pmsg-ramoops-0`）**：
  回到原厂系统后解码 `pmsg-ramoops-0`（内核跨重启保留的持久化 logcat 缓冲区），抓到每秒刷屏一次的死循环日志：
  ```text
  HidlServiceManagement: Waited one second for android.hardware.boot@1.0::IBootControl/default
  hwservicemanager: Tried to start android.hardware.boot@1.0::IBootControl/default as a lazy service, but was unable to.
  ```
  顺藤摸瓜查清完整因果链：
  1. Android 15 的 `/system/etc/init/hw/init.rc` 在 `late-fs` 阶段有一句 **`exec_start update_verifier`**——这是 **PID 1（`init`）的同步阻塞调用**，`update_verifier` 不退出，`init` 就绝对不会往下走！
  2. `update_verifier` 启动后调用 `IBootControl::getService()` 死等 BootControl HAL；
  3. 本机提供该 HAL 的是 `/vendor/bin/hw/vendor.sprd.hardware.boot@1.2-service`，它依赖展锐私有的 `/vendor/lib64/android.hardware.boot@1.0~1.2.so`；在非 `vndklite` 的严格 VNDK 隔离命名空间及 Enforcing SELinux 下，该服务无法正常加载注册；
  4. 于是 `init` 永远冻结在 `late-fs`——此时 `SurfaceFlinger` 已经在放开机动画了，但后面的 **`post-fs-data`（初始化加密 `userdata`）、`zygote`（启动安卓框架）和 `on boot`（触发 `/vendor/etc/init/hw/init.ums9230_1h10.usb.rc` 开启 USB ADB）全部被堵死在门外**！
- **一劳永逸的通用补丁方案（适用于本机所有 Android 14/15/16+ TrebleDroid GSI）**：
  1. 注释掉 `/system/etc/init/hw/init.rc` 中的 `exec_start update_verifier`，并将 `/system/etc/init/update_verifier.rc` 指向 `/system/bin/true` + `disabled`（DSU 与第三方 GSI 根本不需要 OTA 槽位校验）；
  2. 在 `/system/build.prop` 与 `/system/system_ext/etc/build.prop` 中写入 `ro.vndk.lite=true`（让链接器优先使用 `/vendor/lib64` 下的展锐库）；
  3. 在 `/system/etc/init/vndk.rc` 的 `early-init` 阶段利用 TrebleDroid 内置高权限域执行 `exec u:r:phhsu_daemon:s0 root -- /system/bin/setenforce 0`。  
  打入这三处补丁后，**Android 15 Miku UI 从重启到进入桌面仅耗时 35 秒**！

---

## 6. 修改 Android 12+ EXT4 系统镜像的三大生死细节

手工解包修改现代 Android `system.img` 极易导致无法挂载或开机秒崩，必须严守以下三条铁律：

### 细节 1：去尾 + 扩容 + 解开 `shared_blocks` 只读特性
现代 GSI 的 EXT4 镜像普遍开启了块去重特性 `EXT4_FEATURE_RO_COMPAT_SHARED_BLOCKS (0x4000)`，且镜像尾部追加了几十 MB 的 AVB Hashtree + FEC 数据。直接 `mount -o rw` 会报错拒绝。  
**标准处理顺序**（可直接借用平板自身的 `/system/bin/e2fsck` 与 `resize2fs` 在 UFS 上高速处理）：
1. 用 `avbtool.py info_image` 查出纯文件系统的真实长度 `Original image size`，先截断去掉尾部旧 AVB 数据；
2. `e2fsck -y -f system.img`
3. `resize2fs system.img 3300M`（先留出充足空闲块供去重拆解）
4. `e2fsck -E unshare_blocks -y -f system.img`（彻底解除共享块，变为标准可读写 EXT4）
5. `mount -t ext4 -o loop,rw system.img /mnt`

### 细节 2：严禁 `sed -i` / `mv` 破坏 SELinux 扩展属性（`security.selinux`）
这是无数人改坏 `system.img` 的隐形杀手！
- 在挂载的 EXT4 目录中，如果执行 `sed -i 's/a/b/' init.rc` 或 `mv tmp init.rc`，工具会**新建一个临时文件（新 inode）再重命名覆盖旧文件**。新创建的 inode 没有写入 `security.selinux` xattr，标签会变成 `unlabeled`（或继承挂载点标签），开机时 Android `init` 会直接拒绝读取执行！
- **正确无损改法（原位覆写原 inode）**：
  ```sh
  sed 's/exec_start update_verifier/# exec_start update_verifier/g' $MNT/system/etc/init/hw/init.rc > /data/local/tmp/t1
  cat /data/local/tmp/t1 > $MNT/system/etc/init/hw/init.rc
  ```
  用 `cat > 原文件` 只截断覆写数据块，**原 inode、权限位（如 `build.prop` 的 `0600`、`rw-system.sh` 的 `0755`）、UID/GID 以及 `u:object_r:system_file:s0` / `u:object_r:phhsu_exec:s0` 标签 100% 原封不动**！
- 若必须新建文件（例如预置免授权公钥 `/system/product/etc/security/adb_keys`），创建后必须立即显式执行 `chmod` 与 `chcon u:object_r:adb_keys_file:s0`，并在 `umount` 前用 `ls -laZ` 逐一核验！
- **注意根目录软链接陷阱**：GSI 根目录下的 `/adb_keys`、`/product`、`/system_ext` 都是指向 `/product/...` 或 `/system/...` 的**绝对路径软链接**！在宿主平板挂载 `$MNT` 时直接写 `$MNT/adb_keys` 会顺着绝对路径写到宿主机的只读分区上，必须写实际路径 `$MNT/system/product/etc/security/adb_keys`。

### 细节 3：收缩镜像并用 `avbtool.py` 重建合法 `AVBf` 签名尾
修改完成卸载后：
1. 执行 `resize2fs -M system.img` 紧凑收缩体积；
2. 选取一个比当前文件大 64MB 且按 4096 对齐的 `--partition_size`，用官方 `avbtool.py`（配合 Git 自带的 `openssl.exe` 与 AOSP `testkey_rsa2048.pem`，加上 `--do_not_generate_fec` 免去漫长的纠错码计算）重签：
   ```powershell
   python avbtool.py add_hashtree_footer `
     --image miku_fixed.img `
     --partition_name system `
     --partition_size 2704158720 `
     --hash_algorithm sha256 `
     --do_not_generate_fec `
     --algorithm SHA256_RSA2048 `
     --key testkey_rsa2048.pem
   ```
这样生成的镜像同时具备合法的 `AVBf` 尾部与匹配的 SHA-256 Hashtree，`gsid` 与 `FirstStageMountVBootV2` 均能完美放行。

---

## 7. 原生 Linux（postmarketOS / Droidian / Ubuntu Touch）可行性与破案真相

### 7.1 为什么 Droidian 与 Ubuntu Touch GSI 预编译包不能直接跑？
导出本机 `/proc/config.gz` 实测发现：原厂 `5.4.161` 内核虽然开启了 `CONFIG_CGROUPS=y`、`CONFIG_NAMESPACES=y`、`CONFIG_UTS_NS=y`、`CONFIG_NET_NS=y`、`CONFIG_OVERLAY_FS=y`，但**未开启 `CONFIG_PID_NS`、`CONFIG_SYSVIPC`、`CONFIG_DEVTMPFS` 和 `CONFIG_VT`**。
- **Droidian / Ubuntu Touch Halium 容器**强依赖在容器/chroot 内拉起 **`systemd`**（要求 `PID 1` 命名空间 `CONFIG_PID_NS=y`），在不自行重编内核源码的情况下 `systemd` 会立即报错退出。
- 此外，检查 `zerofrip/Ubuntu_Touch_AIDL_GSI` 构建脚本还发现其 GitHub Actions 自动发布的预编译包默认开启了 `CI_MINIMAL_PACKAGES=1`，并未打包完整的 Lomiri 桌面环境。读源码再动手避免了一次无效刷机。

### 7.2 裸机 postmarketOS / Alpine Linux 可行性与 `unknown-block(1,0)` 字节级破案
- **为什么 Alpine / postmarketOS 能绕开内核限制**：它使用 **OpenRC / BusyBox init**（不要求 `PID_NS` 与 `SYSVIPC`），且没有 `CONFIG_DEVTMPFS` 时只需在 `/init` 脚本开头跑一句 `mount -t tmpfs dev /dev && mdev -s` 即可自动建齐所有设备节点。
- **字节级破案此前刷 `boot_a` 报 `Unable to mount root fs on unknown-block(1,0)` 的真因**：
  1. 实测解包原厂 `vendor_boot_a.img` 与 `boot_a.img` 的 ramdisk 起始字节，二者全都是 **`02 21 4c 18`（小端 `0x184C2102`，即 LZ4 Legacy 格式）**！
  2. 在 Android Boot v4 下，U-Boot 会把 `vendor_boot_a` 的 ramdisk 与 `boot_a` 的 ramdisk **在内存中首尾直接拼成一整块缓冲区**传给内核。
  3. Linux 5.4 内核的 `lib/decompress_unlz4.c` 在解压完第一段（`vendor_boot_a`）后，循环读取接下来的 4 字节判断是否为下一个 `0x184C2102` 魔数：
     - 如果你用 **gzip（`1f 8b 08 00`）** 或 **未压缩 `newc` cpio（`30 37 30 37`，即 `"0707"`）** 打包 `boot_a` 的测试 initramfs，`unlz4` 会把 `0x37303730` 当作 **925MB 的块长度**去读，立刻触发解压越界失败并丢弃整个 initramfs！
     - 内核没拿到 `/init`，只能回退去挂载 `cmdline` 里的 `root=/dev/ram0`，从而报出 `VFS: Unable to mount root fs on unknown-block(1,0)` 掉进 Sysdump！
  4. **后续裸机 Linux 正确姿势**：`boot_a` 的自定义 initramfs **必须使用 `lz4 -l`（LZ4 Legacy）压缩**，并在 `/init` 中按依赖顺序 `insmod` `/first_stage_ramdisk/lib/modules/`（或 `/vendor/lib/modules/`）下的 `sprd_drm.ko` 与触控模块即可点亮屏幕。

---

## 8. 底层玩机无损排障方法论（排障三板斧）

总结整个折腾过程，最高效、最不容易翻车的三条工程习惯：

1. **绝不盲猜卡死原因——第一现场永远在 `/sys/fs/pstore/`**：
   本机内核开启了 `CONFIG_PSTORE_RAM=y`、`CONFIG_PSTORE_CONSOLE=y` 与 `PMSG`。只要新系统卡开机动画或崩溃重启，退回原厂后立刻拉取：
   - `/sys/fs/pstore/dmesg-ramoops-0`：看上次内核有没有 Panic；
   - `/sys/fs/pstore/pmsg-ramoops-0`：直接提取可打印字符串，里面保存着**上一次启动死机前最后 32KB 的真实 `logcat` 日志**！一次就能锁定具体是哪个 HAL、哪个 SELinux 规则或哪个服务卡死。
2. **告别“黑盒等开机”——提前把免授权早期 Root ADB 焊进镜像**：
   在刷入任何实验镜像前，先把电脑的 `~/.android/adbkey.pub` 写进镜像的 `/system/product/etc/security/adb_keys`（配好 `u:object_r:adb_keys_file:s0`），并在 `rw-system.sh` / `vndk.rc` 中保持 `ro.adb.secure=0`、`service.adb.root=1` 及 `on boot` 自动 `start adbd`。这样开机 20 秒（哪怕桌面还没出来）电脑就能连上 `adb logcat` 实时看现场。
3. **双层安全网兜底——高层用 DSU 单次启动，底层备好 BootROM 救砖**：
   - 能用 DSU（`gsi_tool`）测的系统绝不先刷物理分区；
   - 必须改 `boot_a` / `vendor_boot_a` 时，确保手头有校验无误的原厂备份 + 装好 `WinUSB` 的 `spd_dump` UFS 命令——哪怕刷成黑砖，按住 **音量减** 插线 10 秒就能满血重来。

---

## 9. 机器人上位机实战：Root Chroot Ubuntu 22.04 + ROS 2 Humble + Mali-G57 硬件加速与桌面一键启动器

在将本机进一步改造为 **ESP32 micro-ROS 移动机器人无线手持上位机** 的过程中，沉淀出以下极具通用价值的嵌入式 Linux 容器、图形加速与安卓系统集成经验：

### 9.1 为什么用“稀疏 EXT4 镜像 + Root Chroot”，而不是解压目录或 `proot`？
1. **拒绝 `proot` 性能损耗**：Termux `proot` 依赖 `ptrace` 拦截所有系统调用，I/O 与编译性能损失严重且破坏 ROS 2 DDS 共享内存；内核级 `chroot` 则是 **100% 原生 ARM64 速度**。
2. **避开 Android `/data` 的 `f2fs` + `nosuid,nodev` 陷阱**：安卓 `/data` 默认以 `nosuid,nodev` 挂载，且 `f2fs` 对 Linux `security.capability` 扩展属性支持不完整（会导致 `dpkg` 解包 `iputils-ping` 等包报错）。通过 `truncate -s 32G /data/ubuntu-ros.img && mke2fs -t ext4` 建立稀疏镜像再以 `loop,rw,suid,dev,exec,noatime` 挂载：
   - 实际占用按需增长（装完 XFCE4 + ROS 2 Humble + Nav2 + SLAM 仅占约 4GB）；
   - 整机备份只需 `adb pull /data/ubuntu-ros.img` 一个文件，彻底卸载只需删除该文件，零分区侵入。

### 9.2 Android 内核网络权限与离线 `AI_ADDRCONFIG (-9)` 怪病
1. **Android `PARANOID_NETWORK` 组权限**：必须在 Ubuntu 内创建 `aid_inet (3003)` 与 `aid_net_raw (3004)` 组，将用户加入组内，且执行 `usermod -g 3003 _apt` 并配置 `APT::Sandbox::User "root";`，否则 `apt` 与普通用户无法创建外网 Socket。
2. **未连 Wi-Fi 时 `127.0.0.1` 报 `Address family for hostname not supported (-9)`**：
   当平板未连 Wi-Fi、仅有 `lo` 回环口存在 IPv4 地址时，glibc 的 `getaddrinfo(..., AI_ADDRCONFIG)` 判定本机“无外部 IPv4 网络”，连 `127.0.0.1` 都会拒绝解析（同样会影响离线单机 ROS 2 节点通信！）。
   - **解法**：当检测到 `wlan0` 未分配 IP 时，自动创建虚拟网卡 `ip link add dummy9 type dummy && ip addr add 10.254.254.1/24 dev dummy9 && ip link set dummy9 up`，离线 DDS 通信与本地回环立即畅通；当 `wlan0` 在线时追加 `ip route add 224.0.0.0/4 dev wlan0 table local` 打通安卓策略路由下的跨机 DDS 组播发现。
3. **`.desktop` 快捷方式找不到 `ros2` 命令**：Ubuntu 默认 `/root/.bashrc` 开头有非交互式早退判断 `case $- in *i*) ;; *) return;; esac`，双击桌面图标执行 `bash -lc` 时会在第 8 行直接 `return`。必须把 `source /opt/ros/humble/setup.bash` 放进 **`/etc/profile.d/ros2-humble.sh`**。

### 9.3 图形界面卡顿根治：从 234% CPU 暴降到 22.5% CPU 的三连优化
初版直接以原生 `2000×1027` 分辨率跑 `llvmpipe` 软渲染时，空载 `rviz2` 独占 **234% CPU** 且拖动窗口严重掉帧。通过以下三步彻底实现 60FPS 丝滑触控：
1. **启用 ARM Mali-G57 物理硬件 GPU 加速（VirGL）**：在宿主侧运行 Termux 的 `virgl_test_server_android --socket-path /data/ubuntu-ros/tmp/.virgl_test`（搭配 `angle-android` 对接 `/vendor/lib64/egl` Mali 驱动），容器内设置 `GALLIUM_DRIVER=virpipe MESA_GL_VERSION_OVERRIDE=4.3 VTEST_SOCK_NAME=/tmp/.virgl_test`。实测 `rviz2` 成功启用 `OpenGL 4.3 (GLSL 4.3)` 硬件渲染，CPU 占用从 **`234%` 骤降至 `22.5%`**！
2. **关闭 `xfwm4` 软件窗口合成器**：在 `xfwm4.xml` 中设 `use_compositing = false`、`box_move = true`，彻底消除全屏透明阴影混合开销。
3. **16:10 等比缩放 `1280×800` 全屏显示**：通过 `com.termux.x11.CHANGE_PREFERENCE` 广播自动设定 `displayResolutionMode=custom`、`displayResolutionCustom=1280x800`、`fullscreen=true`、`showAdditionalKbd=false`，像素填充量减少 **60%**，且按钮文字大小完美契合 10.4 寸手指触控。
4. **切忌用 `[ ! -S X0 ]` 判断 X11 是否存活**：断电重启后镜像内会残留上次未清理的 `/tmp/.X11-unix/X0` 僵尸文件，必须用 `ps -ef | grep CmdEntryPoint` 判断进程存活并在重拉前清理旧 Socket。

### 9.4 深度精简安卓后台 + 断电冷启动“一点即开”桌面 APK 设计
1. **深度精简 50 个无用系统/GMS 包**：国内无谷歌网络环境下，`com.google.android.gms`、Play 商店、语音助手、数字健康、`com.google.android.as` 等服务会在后台疯狂重连唤醒并吃掉超 1GB 内存。通过 `pm disable-user --user 0` 禁用全部 50 个消费娱乐与 GMS 组件并关闭安卓过渡动画后，**开机可用内存从 `769 MB` 暴涨至 `5,850 MB`**，桌面仅留核心入口。
2. **自包含一键启动 APK（`ROS2Launcher.apk`）绕开 Android 12 权限壁垒**：
   - **踩坑 1**：本机 `/system/xbin/su` 权限为 `04750 (root:shell)`，普通第三方 App（`u0_aXXX`）无权直接调用；
   - **踩坑 2**：即使拿 AOSP `platform` 密钥签名并声明 `android:sharedUserId="android.uid.shell"`，Android 12 的 Zygote 也会因 SELinux 禁止 `/data/app` 代码切入 `u:r:shell:s0` 导致进程 `failed to attach` 超时被杀；
   - **终极无损方案**：利用本机 `userdebug` 固件支持 `persist.adb.tcp.port=5555`（Android 12 的 `adbd` 会**同时监听 USB 与本地回环 `127.0.0.1:5555`**）且支持 `--root_seclabel=u:r:su:s0` 的特性，构建一个 `targetSdkVersion="28"` 的自包含 APK（`ROS2Launcher.apk`，注意从镜像站拉取 Termux 依赖包时需将文件名中的 `:` 替换为 `_`，防止触发 Windows NTFS 备用数据流丢文件！），内置 `adb` 运行时并预授权 `/data/misc/adb/adb_keys`。实测在**彻底断电冷启动、SELinux Enforcing、完全脱离电脑**的状态下，在安卓主页轻点一次「**启动 ROS2**」，3~5 秒即可全自动挂载并切入 GPU 加速的 ROS 2 Humble 全屏桌面！

