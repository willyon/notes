# M1 MacBook Air 使用 WinDiskWriter 制作 Windows 11 25H2 x64 UEFI 启动 U 盘教程

本文记录的是一次已经实际完成并验收通过的流程：在 **M1 MacBook Air** 上，使用 **WinDiskWriter** 制作一只给未来 **AMD/x64 AI 工作站** 使用的 **Windows 11 25H2 简体中文 UEFI 启动 U 盘**。

本教程只把本次流程中已经实际验证过的步骤写成事实；未在本次流程中验证的安装、BIOS、驱动、Windows 激活等内容不写成已完成结论。

## 一、目标与实测结果

本次目标：

- 在 M1 MacBook Air 上制作 Windows 11 安装 U 盘。
- 目标安装机器不是 Mac，而是未来的 AMD/x64 AI 工作站。
- 启动方式为现代主板常用的 UEFI。
- U 盘文件系统使用 FAT32，并由 WinDiskWriter 自动处理超过 4GB 的 Windows 安装镜像文件。

最终验收结果：

- U 盘可见 x64 UEFI 启动文件 `bootx64.efi`。
- 原始 `install.wim` 已被拆分为多个 `.swm` 文件，避开 FAT32 单文件 4GB 限制。
- `boot.wim` 存在。
- 启动 U 盘制作流程验收通过。

## 二、本次实际使用的设备与文件

### 1. 制作电脑

- Apple Silicon Mac：**M1 MacBook Air**
- macOS 环境下操作

### 2. Windows ISO

本次使用的 ISO 文件：

```text
Win11_25H2_Chinese_Simplified_x64.iso
```

实测信息：

- 文件大小约 **8GB**
- 手动挂载后，macOS 中出现的 ISO 卷名为：

```text
CCCOMA_X64FRE_ZH-CN_DV9
```

ISO 内的 Windows 安装镜像：

```text
install.wim
```

实测大小约：

```text
7.1GB
```

### 3. 目标 U 盘

本次实际写入的 U 盘：

```text
Kingston DataTraveler 3.0 64GB
```

在磁盘工具或命令行中曾显示容量约：

```text
62GB
```

### 4. 现场同时存在的其他磁盘

本次环境中还连接了一个数据盘：

```text
Samsung PSSD T5 EVO 2TB
```

这是重要数据盘，制作启动盘时 **绝不能误选**。

制作 Windows 启动盘会格式化目标 U 盘。选错磁盘会直接抹掉错误设备上的数据，所以每一步涉及磁盘选择时，都必须确认容量、品牌、名称和设备号。

## 三、为什么 M1 Mac 上仍然下载 x64 Windows ISO

这一步很容易误解。

虽然制作工具运行在 **M1 MacBook Air** 上，但这只 U 盘不是用来给 M1 Mac 安装 Windows 的，而是给未来的 **AMD AI 工作站** 使用。

AMD 9950X 这类桌面平台属于：

```text
x86-64 / AMD64 / x64
```

因此，应该下载和写入：

```text
Windows 11 简体中文 x64 ISO
```

而不是 ARM 版本。

简单说：

- 制作电脑：M1 MacBook Air，Apple Silicon
- 目标电脑：AMD/x64 AI 工作站
- ISO 架构应按目标电脑选择，所以本次使用 **x64 Windows ISO**

## 四、为什么选择 FAT32，以及为什么需要拆分 install.wim

现代 PC 主板的 UEFI 固件通常能很好识别 FAT32 启动盘。为了制作兼容 UEFI 的安装盘，本次选择了 FAT32。

但 FAT32 有一个关键限制：

```text
单个文件最大不能超过 4GB
```

本次 ISO 中的：

```text
sources/install.wim
```

实测约：

```text
7.1GB
```

它超过了 FAT32 单文件 4GB 限制，不能直接完整复制到 FAT32 U 盘中。

WinDiskWriter 在本次流程中自动完成了 WIM 拆分，把 `install.wim` 拆成多个 `.swm` 文件：

```text
install.swm
install2.swm
install3.swm
```

Windows Setup 可以识别这种拆分后的安装镜像格式。

本次最终验收看到：

```text
install.swm   3.3GB
install2.swm  3.5GB
install3.swm  206MB
```

三者都低于 FAT32 的 4GB 单文件限制。

## 五、准备 WinDiskWriter

本次使用的是适配当前系统环境的：

```text
WinDiskWriter Tahoe 26+ Apple Silicon 版本
```

### Gatekeeper 提示的处理方式

如果 macOS 提示 WinDiskWriter 未签名或无法直接打开，本次采用的处理方式是：

1. 打开 **系统设置**
2. 进入 **隐私与安全性**
3. 找到被拦截的 WinDiskWriter
4. 点击 **仍要打开**

注意：

- 本次没有关闭 Gatekeeper。
- 不建议为了运行一个未签名应用而全局关闭 macOS 安全机制。
- 只对确认来源的这个应用单独允许打开。

## 六、先确认磁盘，再格式化 U 盘

### 1. 查看当前磁盘列表

在终端执行：

```bash
diskutil list
```

必须根据输出认真确认：

- 哪个是 Kingston DataTraveler 3.0 64GB U 盘
- 哪个是 Samsung PSSD T5 EVO 2TB 数据盘
- 当前 U 盘对应的设备号是什么

本次实际操作中，Kingston U 盘对应的是：

```text
/dev/disk6
```

但这个编号不是固定的。

每次重新插拔 U 盘、移动硬盘、重启电脑后，设备号都可能变化。下次操作时不能直接照抄 `/dev/disk6`，必须重新运行：

```bash
diskutil list
```

确认无误后再继续。

### 2. 本次实际执行的格式化命令

在确认 Kingston U 盘确实是 `/dev/disk6` 后，本次执行：

```bash
diskutil eraseDisk MS-DOS WIN11 GPT /dev/disk6
```

这条命令的含义：

- `eraseDisk`：抹掉整块磁盘
- `MS-DOS`：格式化为 FAT32
- `WIN11`：给 U 盘卷命名为 WIN11
- `GPT`：使用 GUID 分区表
- `/dev/disk6`：本次确认到的 Kingston U 盘设备号

再次强调：

```text
/dev/disk6 只是本次实测环境中的设备号，不是通用固定值。
```

如果误把 Samsung PSSD T5 EVO 2TB 数据盘选成目标，数据会被抹掉。

## 七、WinDiskWriter 设置

打开 WinDiskWriter 后，本次使用的关键设置如下。

### 1. ISO 类型

选择：

```text
Windows x64 ISO
```

原因是目标机器是 AMD/x64 AI 工作站。

### 2. 目标磁盘

选择：

```text
Kingston DataTraveler 3.0 64GB
```

不要选择：

```text
Samsung PSSD T5 EVO 2TB
```

### 3. 文件系统

选择：

```text
FAT32
```

本次 WinDiskWriter 会自动处理超过 4GB 的 `install.wim`，将其拆分为 `.swm` 文件。

### 4. 不勾选的选项

本次没有勾选：

```text
Patch Installer Requirements
```

也没有勾选：

```text
Install Legacy BIOS Boot Sector
```

原因：

- 本次目标是现代 AMD/x64 工作站的 UEFI 启动盘。
- 本次流程没有验证绕过 Windows 安装要求的补丁。
- 本次流程没有验证 Legacy BIOS 启动。

## 八、本次遇到的问题：ISO 已手动挂载导致写入失败

### 现象

首次尝试时，因为 ISO 已经被手动挂载，系统中已经存在 ISO 卷：

```text
CCCOMA_X64FRE_ZH-CN_DV9
```

WinDiskWriter 再尝试自行挂载 ISO 时，出现类似错误：

```text
hdiutil attach failed - Resource temporarily unavailable
```

### 原因

同一个 ISO 已经被 macOS 手动挂载，WinDiskWriter 再次调用 `hdiutil attach` 时资源被占用。

### 本次解决方式

先在 Finder 中弹出已手动挂载的 ISO 卷：

```text
CCCOMA_X64FRE_ZH-CN_DV9
```

然后回到 WinDiskWriter，让它自己挂载 ISO 并继续写入。

本次这样处理后，制作流程继续完成。

## 九、制作完成后的实际卷名

制作完成后，在 `/Volumes` 中可见：

```text
CCCOMA_X64FRE_ZH-CN_DV9
Macintosh HD
Personal-Files
Time Machine
WDW_KQV9VX4
```

其中：

- `CCCOMA_X64FRE_ZH-CN_DV9`：WinDiskWriter 挂载的 Windows ISO
- `WDW_KQV9VX4`：WinDiskWriter 制作完成后的 U 盘卷

## 十、最终验收命令

制作完成后，本次使用以下命令检查关键文件。

注意：这里的 U 盘卷名是本次制作完成后的实际卷名：

```text
WDW_KQV9VX4
```

执行：

```bash
echo "=== UEFI ===" && \
ls -lh "/Volumes/WDW_KQV9VX4/efi/boot/bootx64.efi" && \
echo "=== INSTALL IMAGE ===" && \
ls -lh "/Volumes/WDW_KQV9VX4/sources/"install* && \
echo "=== BOOT WIM ===" && \
ls -lh "/Volumes/WDW_KQV9VX4/sources/boot.wim"
```

## 十一、本次实际验收输出

本次看到的关键输出为：

```text
=== UEFI ===
-rwx------@ 1 zhangshouchang  staff   2.9M Sep 27 08:39 /Volumes/WDW_KQV9VX4/efi/boot/bootx64.efi

=== INSTALL IMAGE ===
-rwx------@ 1 zhangshouchang  staff   3.3G Sep 27 09:03 /Volumes/WDW_KQV9VX4/sources/install.swm
-rwx------@ 1 zhangshouchang  staff   3.5G Sep 27 09:22 /Volumes/WDW_KQV9VX4/sources/install2.swm
-rwx------@ 1 zhangshouchang  staff   206M Sep 27 09:23 /Volumes/WDW_KQV9VX4/sources/install3.swm

=== BOOT WIM ===
-rwx------@ 1 zhangshouchang  staff   652M Sep 27 08:42 /Volumes/WDW_KQV9VX4/sources/boot.wim
```

这说明：

- `bootx64.efi` 存在，大小约 **2.9MB**，x64 UEFI 启动文件存在。
- `install.wim` 已成功拆分为 `install.swm`、`install2.swm`、`install3.swm`。
- 拆分后的文件大小分别约 **3.3GB**、**3.5GB**、**206MB**，均低于 FAT32 单文件 4GB 限制。
- `boot.wim` 存在，大小约 **652MB**。

因此，本次 Windows 11 25H2 简体中文 x64 UEFI 启动 U 盘制作完成并验收通过。

## 十二、安全弹出

制作完成并验收通过后，本次建议通过 Finder 安全弹出：

```text
WDW_KQV9VX4
```

以及 WinDiskWriter 挂载的 ISO 卷：

```text
CCCOMA_X64FRE_ZH-CN_DV9
```

Finder 中两个卷都弹出后，再拔掉 Kingston U 盘。

也可以在确认设备号仍然正确时使用终端弹出，但因为磁盘编号可能变化，本次最终建议使用 Finder 弹出更稳妥。

## 十三、故障排查

### 1. WinDiskWriter 报 `hdiutil attach failed - Resource temporarily unavailable`

本次实际遇到过。

原因：

```text
ISO 已经被手动挂载，WinDiskWriter 再次尝试挂载时资源被占用。
```

解决：

1. 在 Finder 中弹出已挂载的 ISO 卷 `CCCOMA_X64FRE_ZH-CN_DV9`
2. 回到 WinDiskWriter
3. 让 WinDiskWriter 自己重新挂载 ISO 并继续制作

### 2. 不确定哪个磁盘是 U 盘

不要继续格式化。

先执行：

```bash
diskutil list
```

重点核对：

- 品牌
- 容量
- 是否为 Kingston DataTraveler 3.0 64GB
- 是否不是 Samsung PSSD T5 EVO 2TB

只有确认目标是 Kingston U 盘后，才能执行格式化或写入。

### 3. 找不到 `install.wim`，只看到 `install.swm`

在本次流程中，这是正常结果。

因为 FAT32 不支持超过 4GB 的单个文件，而原始 `install.wim` 约 7.1GB。WinDiskWriter 已经把它拆成：

```text
install.swm
install2.swm
install3.swm
```

这正是本次希望看到的结果。

### 4. 是否应该勾选 Patch Installer Requirements

本次没有勾选，也没有验证该选项。

因此本文不把它写成推荐操作。是否需要绕过 Windows 安装限制，应该等目标 AMD/x64 AI 工作站实际装机时，根据硬件、TPM、Secure Boot、Windows 版本要求再判断。

### 5. 是否需要 Legacy BIOS Boot Sector

本次没有勾选，也没有验证 Legacy BIOS 启动。

本次目标是现代 AMD/x64 AI 工作站的 UEFI 启动，因此只按 UEFI 启动盘流程验收。

## 十四、最终结论

本次制作完成的 U 盘可以标记为：

```text
Windows 11 25H2 简体中文 x64｜UEFI｜AI 工作站安装盘
```

它是在 M1 MacBook Air 上通过 WinDiskWriter 制作，并已通过以下关键点验收：

- x64 UEFI 启动文件存在：`bootx64.efi`，约 2.9MB
- Windows 安装镜像已拆分：`install.swm`、`install2.swm`、`install3.swm`
- Windows PE 安装环境存在：`boot.wim`，约 652MB
- 使用 FAT32 文件系统，适配 UEFI 启动
- 未把 Samsung PSSD T5 EVO 2TB 数据盘作为目标盘

后续真正装机时，应在目标 AMD/x64 AI 工作站上继续验证：

- BIOS/UEFI 中是否能识别该 U 盘
- 是否能从 U 盘启动进入 Windows Setup
- SSD 分区和 Windows 安装流程是否顺利
- AMD 芯片组驱动、NVIDIA 驱动、Windows 更新、WSL2、CUDA/AI 环境等后续配置

这些后续步骤不属于本次已完成流程，因此本文只作为启动 U 盘制作与验收教程。
