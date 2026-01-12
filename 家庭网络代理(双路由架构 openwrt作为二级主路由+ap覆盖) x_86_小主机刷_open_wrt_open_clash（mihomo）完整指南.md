# x86 小主机刷 OpenWrt + OpenClash（Mihomo）完整指南

## 运营商路由器（主路由 / 上游 NAT）→ OpenWrt（二级主路由 / DHCP / 网关 / 代理）→ 华硕路由器（AP 模式）

> 适用对象：N150 / 倍控类 x86 小主机（NVMe/SSD），官方 OpenWrt 24.10.x（x86_64），安装 OpenClash（Mihomo/Meta）实现全家透明代理。
>
> 本文包含 **三份指南**：
>
> 1. **完整详细操作指南**（从 0 到成功）
> 2. **极简执行清单**（照抄即可）
> 3. **故障排查对照表**（一步卡住就查）

---

## ⚠️ 重要安全提醒

- **刷盘会覆盖硬盘所有数据**（原 Ubuntu/Windows 会被清空）。
- `dd` 一旦写错盘，数据不可逆。
- 操作建议使用 **有线连接** 到 OpenWrt 的 LAN 口。

---

# 一、完整详细操作指南（从 0 到成功）

## 1. 准备材料

- x86 小主机（N150/倍控类）+ NVMe/SSD
- U 盘（≥8GB）
- 一台电脑（Windows 或 macOS）
- 固件：`openwrt-24.10.x-x86-64-generic-ext4-combined-efi.img`

---

## 2. 制作 OpenWrt 启动 U 盘

- Windows：Rufus / balenaEtcher / Win32DiskImager（**镜像写入**）
- macOS/Linux：`dd` 写入

> 说明：该 `.img` **可直接写入 U 盘并 UEFI 启动**。

---

## 3. BIOS 设置

- Boot Mode：UEFI
- **关闭 Secure Boot**
- 启动顺序：USB 优先

保存退出，插 U 盘开机。

---

## 4. 从 U 盘启动 OpenWrt（临时系统）

- 出现提示 “There is no root password defined…” 即表示已进入 OpenWrt。

### 4.1 如何进入 OpenWrt 管理页面（LuCI，**新手必看**）

#### ⚠️ 关于 x86 小主机“物理 LAN 口 ≠ 逻辑 LAN/WAN”的重要提醒（必读）

> **很多 x86 小主机虽然外壳上标着两个 LAN 口，但在 OpenWrt 默认配置中：**
>
> - 实际只有 **一个是 LAN（内网口）**
> - 另一个会被系统默认当作 **WAN（外网口）**

也就是说：

- ✅ **只有连接到 LAN 口的电脑，才能访问 `192.168.1.1`**
- ❌ **如果电脑误插在 WAN 口，管理页面一定打不开**
  > 这是**正常现象**，不是 OpenWrt 安装失败，也不是系统坏了

这是很多新手在**第一次调试 x86 软路由时最容易踩的坑之一**。

##### 如何判断哪个口是 LAN、哪个是 WAN？

**方法一：默认行为判断（最简单，推荐新手）**

- 插上网线后：
  - 能自动获取 `192.168.1.x` IP 的口 → **LAN**
  - 完全获取不到 IP 的口 → **WAN**

**方法二：在 LuCI 中查看（系统已能访问时）**

- 进入：`网络 → 接口`
- 查看接口绑定关系：
  - `br-lan` 绑定的物理接口 → **LAN 口**
  - `wan` 绑定的物理接口 → **WAN 口**

**方法三：经验总结（非常重要）**

- 小主机外壳上的 `LAN1 / LAN2`、`ETH0 / ETH1` **不一定可信**
- 一切以 **OpenWrt 中接口绑定关系**为准

> ⚠️ **新手强烈建议：**
>
> - 第一次配置 OpenWrt
> - 一定要反复确认：**电脑插在 LAN 口，而不是 WAN 口**

这是**很多新手最容易忽略，但实际最重要的一步**。

#### 方式一：通过浏览器访问（最常用）

1. 用 **网线** 将电脑连接到 OpenWrt 的 **LAN 口**
2. 在电脑浏览器中打开：

```
http://192.168.1.1
```

3. 第一次进入时：

   - 用户名：`root`
   - 密码：**留空**（首次尚未设置）

4. 登录后即可进入 OpenWrt Web 管理界面（LuCI）

> 默认 LAN IP 为 `192.168.1.1`，这是官方 x86 OpenWrt 的默认设置。

#### 方式二：通过 SSH 命令行访问（辅助方式）

如果浏览器暂时无法访问，也可以通过 SSH：

```bash
ssh root@192.168.1.1
```

- 首次登录同样 **无密码**
- 成功后可在命令行中执行所有配置与安装操作

#### 常见问题排查

- ❌ 浏览器打不开 `192.168.1.1`
  - 确认电脑 IP 是否为 `192.168.1.x`
  - 确认网线是否插在 **LAN 口** 而非 WAN
  - 可尝试手动给电脑设置临时 IP（如 `192.168.1.2`）

---

### 4.2 是否必须用网线访问 OpenWrt 管理页面？（非常重要）

> 结论先行：**不一定，但强烈建议第一次配置时使用网线直连 OpenWrt 的 LAN 口。**

下面按实际常见场景说明。

#### 场景一：可以直接用 Wi‑Fi 访问（成立条件较苛刻）

```
路由器 A（主路由）
├─ Wi‑Fi：电脑 / 手机
└─ LAN：OpenWrt 小主机（只是普通设备）
```

- 路由器 A 是唯一主路由
- 所有设备都在同一网段（如 `192.168.1.0/24`）
- OpenWrt 只是被当作一台普通主机

✅ 此时，电脑即使通过 Wi‑Fi，也可以访问 OpenWrt 的管理 IP。

---

#### 场景二：软路由作为主路由（**最常见、也是本指南采用的结构**）

```
OpenWrt 小主机（主路由 192.168.1.1）
├─ LAN → 路由器 A（AP 模式）→ Wi‑Fi：电脑 / 手机
└─ WAN → 光猫 / 上级网络
```

- OpenWrt 才是主路由
- 路由器 A 仅作为 AP / 交换机使用
- Wi‑Fi 设备实际位于 OpenWrt 的 LAN 内

✅ **只要 AP 未开启“客户端隔离”，Wi‑Fi 设备可以访问 \*\***192.168.1.1\***\*。**

---

#### 场景三：双路由 / 双 NAT（新手最容易误判的情况）

```
路由器 A（主路由 192.168.1.1）
├─ Wi‑Fi：电脑（192.168.1.x）
└─ LAN → OpenWrt（次级路由 192.168.2.1）
```

- 路由器 A 和 OpenWrt 都在做 NAT
- 两个不同网段（`192.168.1.x` / `192.168.2.x`）

❌ 电脑通过 Wi‑Fi **无法直接访问 OpenWrt 的管理页面**。

---

#### 为什么“第一次配置”强烈建议用网线？

即便在理论上可以通过 Wi‑Fi 访问，也仍建议第一次配置时使用网线直连，因为：

1. 初期会频繁修改 LAN / DHCP / IP 设置
2. 一旦配置错误，Wi‑Fi 可能立即中断
3. 有线直连可以避免 AP 隔离、双网段、路由策略干扰

> **工程经验总结**：
>
> - 第一次配置：用网线
> - 系统稳定后：Wi‑Fi / 远程访问都可以

---

## 5. 识别磁盘（关键）

OpenWrt BusyBox 环境可能没有 `lsblk`，请用：

```sh
ls /dev/sd*
cat /proc/partitions
```

**判断规则**：

- U 盘容量小（8/16/32G）→ 常见 `/dev/sda`
- NVMe/SSD 容量大（128G/256G/512G）→ 常见 `/dev/nvme0n1`

---

## 6. 将 OpenWrt 写入 SSD（永久安装）

```sh
dd if=/dev/sda of=/dev/nvme0n1 bs=1M
sync
poweroff
```

- 等待完成（BusyBox 可能无进度条）
- 关机 → 拔 U 盘 → 开机

> `bs` 是每次拷贝的数据块大小（非“速度”），常用 `1M/4M`。

---

## 7. 拔 U 盘后是否要进 BIOS？

- 通常 **不需要**。
- 若无法启动：BIOS → 启动顺序 → 将 NVMe/UEFI OS 置顶。

---

## 8. 基本网络拓扑（推荐）

- OpenWrt 小主机：主路由（DHCP/NAT）
- 华硕路由器：AP（桥接，关闭 DHCP）
- 手机/Mac 连接华硕 Wi‑Fi → 实际走 OpenWrt 出口

### ⚠️ AP 路由器接线与模式补充说明（非常容易踩坑）

在「小主机软路由（OpenWrt） + 普通路由器作为 AP」的方案中，有一个**最常见、也最容易被忽略的错误点**，会导致网络表现“看起来很玄学”：

#### ✅ 软路由 与 AP 路由器必须使用 **LAN 口 → LAN 口** 连接

#### 为什么不能接 WAN 口？

在本指南的网络结构中：

- **小主机 OpenWrt 是全网唯一的主路由**，负责：

  - 拨号 / DHCP / NAT
  - DNS 解析
  - OpenClash 分流与代理规则

- **AP 路由器只负责二层工作**：
  - 交换转发
  - 无线信号覆盖

如果将网线误接到 AP 的 **WAN 口**，或 AP 没有彻底切换到 AP 模式，通常会出现以下问题：

- DHCP 抢答、网关混乱
- 双重 NAT，导致端口映射、内网互访、策略路由异常

**典型症状包括：**

- 刚开始可以正常访问国内和国外网站
- 使用一段时间后变成只能访问国内
- 部分设备（尤其是电脑）可能直接无法上网

---

#### AP 路由器应处于的正确状态

- ✅ 工作模式：**AP 模式（Access Point）**
- ✅ DHCP：关闭
- ✅ NAT / 防火墙：关闭或不启用
- ✅ 接线方式：**LAN → LAN**
- ✅ 管理 IP：建议手动固定在 OpenWrt 所在网段  
  （如 `192.168.1.2`，避免后续找不到管理页面）

> **经验口诀：**  
> **谁拨号，谁是路由；不做路由，只用 LAN。**

````text
软路由 LAN 口  ─────────  AP 路由器 LAN 口

---

## 9. OpenClash 前置依赖

```sh
opkg update
opkg install bash curl ca-bundle ip-full kmod-tun kmod-inet-diag unzip luci-compat
````

---

## 10. 处理 dnsmasq 冲突（必做）

```sh
opkg remove dnsmasq
opkg install dnsmasq-full
reboot
```

- 提示保留 `/etc/config/dhcp` 属于 **正常**。
- 重启后能访问 `192.168.1.1` 且能上网即成功。

---

## 11. 安装 OpenClash

### 11.1 Ruby 依赖

```sh
opkg install ruby ruby-yaml
```

### 11.2 安装 ipk

- LuCI → 系统 → 软件包 → 上传 `luci-app-openclash_*.ipk`
- 若提示 `XHR request aborted`：

```sh
opkg status luci-app-openclash
```

看到 `install ok installed` 即成功。

---

## 12. 安装 Clash Core（Mihomo / Meta）— 手动稳妥方案

### 12.1 传 core 到 OpenWrt（macOS）

```bash
scp -O clash-linux-amd64-v1.tar.gz root@192.168.1.1:/tmp/
```

> `-O` 强制旧 scp 协议，避免 macOS 默认 SFTP 报错。

### 12.2 解压、放置、重命名、授权

```sh
cd /tmp
tar -xzf clash-linux-amd64-v1.tar.gz
mkdir -p /etc/openclash/core
mv clash /etc/openclash/core/clash_meta
chmod +x /etc/openclash/core/clash_meta
```

### 12.3 验证

```sh
/etc/openclash/core/clash_meta -v
```

能输出版本号即成功。

---

## 13. 导入订阅并生成配置

- OpenClash → **Config Subscribe**
- 粘贴订阅链接

**正确点击顺序（关键）**：

1. **Commit Settings**
2. **Update Config**

> 若首次生成配置，OpenClash 可能 **自动 Switch 并 Enable**，属正常。

---

## 14. 成功状态核对

- Overviews：**Meta Running**
- Running Mode：Fake‑IP + Enhance
- Proxy Mode：Rule
- 可执行 **Flush DNS**

---

# 二、极简执行清单（照抄版）

```text
1) 写 U 盘：openwrt-24.10.x-x86-64-generic-ext4-combined-efi.img
2) BIOS：UEFI + 关 Secure Boot + USB 优先
3) 启动 OpenWrt
4) 识别盘：ls /dev/sd* ; cat /proc/partitions
5) 写盘：dd if=/dev/sda of=/dev/nvme0n1 bs=1M ; sync ; poweroff
6) 拔 U 盘重启
7) opkg update
8) opkg install bash curl ca-bundle ip-full kmod-tun kmod-inet-diag unzip luci-compat
9) opkg remove dnsmasq ; opkg install dnsmasq-full ; reboot
10) opkg install ruby ruby-yaml
11) 安装 luci-app-openclash.ipk
12) scp -O clash-linux-amd64-v1.tar.gz root@192.168.1.1:/tmp/
13) tar -xzf /tmp/clash-linux-amd64-v1.tar.gz
14) mv clash /etc/openclash/core/clash_meta ; chmod +x /etc/openclash/core/clash_meta
15) /etc/openclash/core/clash_meta -v
16) Config Subscribe → Commit Settings → Update Config
17) Overviews：Meta Running → Flush DNS
```

---

# 三、故障排查对照表

## Q1. `lsblk: not found`

- 原因：BusyBox 精简
- 解法：`ls /dev/sd*`、`cat /proc/partitions`

## Q2. `dd` 没进度

- 原因：BusyBox dd
- 结论：正常，等待完成

## Q3. 安装 `dnsmasq-full` 冲突

- 解法：先 `opkg remove dnsmasq` 再安装

## Q4. OpenClash 缺 `ruby / ruby-yaml`

- 解法：`opkg install ruby ruby-yaml`

## Q5. LuCI 提示 `XHR request aborted`

- 解法：`opkg status luci-app-openclash` 核验安装状态

## Q6. Core 显示 `Unknown`

- 原因：在线下载失败但不报错
- 解法：**手动安装 core**（见上文）

## Q7. macOS scp 报 `sftp-server not found`

- 解法：`scp -O 文件 root@IP:/tmp/`

## Q8. 订阅点 Update 没反应

- 原因：未保存
- 解法：先 **Commit Settings** 再 **Update Config**

## Q9. 未手动 Switch 却自动 Running

- 结论：正常（首次生成配置自动应用）

## Q10. 国外不走代理

- 检查：Meta Running → 是否有 yaml → Rule 模式 → Flush DNS

---

## ✅ 成功判定

- OpenClash：**Meta Running**
- Fake‑IP + Enhance
- Rule 分流
- 终端设备无需客户端即可生效

---

## 附录 A：Clash Core（Mihomo / Meta）来源说明（基于实操回忆）

> 本附录用于解释 **clash-linux-amd64-v1.tar.gz** 的真实来源、命名差异，以及为什么需要手动安装。

### A.1 本次成功使用的 Core 文件信息

- 文件名：`clash-linux-amd64-v1.tar.gz`
- 架构：`linux / amd64 / v1`（对应 x86_64）
- Core 类型：**Clash Meta / Mihomo 技术路线**

### A.2 Core 的实际下载来源（重要）

本次使用的 `clash-linux-amd64-v1.tar.gz` **并非手动在 GitHub 搜索下载**，而是来源于：

```
OpenClash → Plugin Settings → Version Update → [Meta] Latest Core → Download
```

即：**OpenClash 内置的 Core 下载功能**。

OpenClash 会根据当前设备信息：

- CPU Architecture：x86_64
- Compiled Version：linux-amd64-v1

自动匹配并从 **MetaCubeX 官方发布源** 拉取对应的 Core 压缩包，并由浏览器保存为：

```
clash-linux-amd64-v1.tar.gz
```

### A.3 官方技术来源说明

虽然文件名以 `clash-` 开头，但其技术来源是：

- MetaCubeX（Clash Meta / Mihomo）项目
- 常见官方仓库包括：
  - `MetaCubeX/mihomo`
  - `MetaCubeX/clash-meta`

OpenClash 会根据自身版本和兼容性自动选择合适的发布源。

### A.4 为什么“点了 Download 但 UI 仍显示 Unknown”？

这是 OpenClash 在国内网络环境下的**常见现象**：

- Download 按钮被点击
- 浏览器成功下载了文件
- 但以下步骤**并未自动完成**：
  - Core 文件未被放入 `/etc/openclash/core/`
  - 文件名未重命名为 `clash_meta`
  - 未设置可执行权限

因此 UI 中：

```
[Meta] Current Core: Unknown
```

仍然存在。

### A.5 本次最终生效的正确方式（推荐）

使用 \*\*已下载好的 \*\***clash-linux-amd64-v1.tar.gz**，手动完成：

```sh
cd /tmp
tar -xzf clash-linux-amd64-v1.tar.gz
mkdir -p /etc/openclash/core
mv clash /etc/openclash/core/clash_meta
chmod +x /etc/openclash/core/clash_meta
```

验证：

```sh
/etc/openclash/core/clash_meta -v
```

---

## 附录 B：luci-app-openclash_0.47.046_all.ipk 下载来源说明

> 本附录用于记录 **OpenClash LuCI 插件本体**的可靠下载来源，避免误下第三方或过期版本。

### B.1 本次使用的插件版本

- 插件名：`luci-app-openclash_0.47.046_all.ipk`
- 类型：LuCI 管理界面插件（不包含 Core）
- 适配：OpenWrt 24.10.x / x86_64

### B.2 官方下载来源（推荐且已验证）

本次使用的 ipk 来源于 **OpenClash 官方 GitHub Releases 页面**：

```
https://github.com/vernesong/OpenClash/releases
```

在 Releases 页面中：

- 选择对应版本（如 v0.47.046）
- 下载：

```
luci-app-openclash_0.47.046_all.ipk
```

### B.3 为什么推荐使用 GitHub Releases？

- 由 OpenClash 作者 `vernesong` 直接发布
- 与 OpenClash 插件代码完全一致
- 明确标注版本号与变更日志
- 避免第三方论坛二次打包、缺依赖或版本错配

### B.4 安装方式回顾（本次成功路径）

- 下载 `luci-app-openclash_0.47.046_all.ipk` 到本地电脑
- 通过 LuCI：

```
系统 → 软件包 → 上传 → 安装
```

- 若提示缺少依赖：

```sh
opkg install ruby ruby-yaml
```

- LuCI 若提示 `XHR request aborted by browser`：
  - 使用以下命令确认安装状态：

```sh
opkg status luci-app-openclash
```

---

## 进阶建议

- Apple 服务直连优化（App Store / iCloud）
- Netflix / YouTube / ChatGPT 分流
- 指定设备不走代理（按 IP/MAC）

---

> 文档完结。
