# GEC6818 FriendlyCore ARM64 移植记录

> 将 FriendlyElec Smart6818 的 ARM64 FriendlyCore 系统迁移到 GEC6818 开发板的实践记录。  
> 本仓库目前以「跑通启动链路 + 记录移植过程 + 后续驱动适配」为目标，不保证所有外设已经完整可用。

## 1. 项目背景

GEC6818 开发板常见资料以 32 位系统为主，内核和用户态环境相对较旧。但 S5P6818 处理器本身是 Cortex-A53 架构，具备运行 ARM64 Linux 系统的条件。

因此，本项目尝试基于 FriendlyElec Smart6818 的 ARM64 FriendlyCore 系统，在 GEC6818 开发板上完成以下工作：

- 启动 ARM64 FriendlyCore 系统
- 适配 GEC6818 的 U-Boot 启动流程
- 添加 GEC6818 对应的 Kernel defconfig
- 添加 GEC6818 对应的 DTS / DTB
- 记录 LCD、声卡、网卡等外设适配问题
- 为后续驱动移植和板级适配留下可复现流程

## 2. 适用环境

| 项目 | 内容 |
|---|---|
| 开发板 | GEC6818 |
| SoC | Samsung / Nexell S5P6818 |
| CPU 架构 | ARMv8-A / Cortex-A53 / AArch64 |
| 宿主机 | WSL2 Ubuntu 20.04 LTS |
| 目标系统 | FriendlyCore ARM64 |
| Kernel | Linux 4.4.172 |
| Toolchain | aarch64-linux-gcc 6.4 |
| 启动介质 | SD 卡 |
| 串口调试 | 必需 |

> 说明：本文档优先记录 SD 卡启动流程。eMMC 烧写风险更高，建议在 SD 卡流程完全跑通后再处理。

## 3. 当前状态

| 模块 | 状态 | 说明 |
|---|---|---|
| SD 卡启动 | 已跑通 | 基于 FriendlyCore ARM64 SD 镜像 |
| U-Boot | 已验证 | 基于已有 `u-boot_gec6818` 适配 |
| Kernel | 可编译 | 基于 FriendlyElec Linux 4.4.172 分支 |
| DTB | 可生成 | 新增 `s5p6818-gec6818-rev01.dtb` |
| LCD | 部分适配 | 需要修改 U-Boot LCD 参数 |
| 声卡 | 待适配 | 需要继续检查 DTS / 驱动 |
| 网卡 | 待适配 | 需要继续检查 DTS / 驱动 |
| eMMC 启动 | 未验证 | 暂不建议直接烧写 |

## 4. 参考来源

本项目不是完全从零移植，而是基于以下公开资料和已有工程进行整理、验证和二次适配：

- FriendlyElec Smart6818 Wiki  
  <https://wiki.friendlyelec.com/wiki/index.php/Smart6818/zh>
- FriendlyARM prebuilts 工具链仓库  
  <https://github.com/friendlyarm/prebuilts>
- FriendlyARM Linux Kernel 仓库  
  <https://github.com/friendlyarm/linux>
- FriendlyARM SD 镜像打包工具  
  <https://github.com/friendlyarm/sd-fuse_s5p6818>
- celeron633 的 GEC6818 U-Boot 适配仓库  
  <https://github.com/celeron633/u-boot_gec6818>
- GEC6818 原理图、手册等资料  
  <https://pan.baidu.com/s/1WsLTYM0sY2A7xuVaUlddLg?pwd=22yR>

## 5. 工具链安装

### 5.1 下载 FriendlyARM 预编译工具链

```bash
git clone https://github.com/friendlyarm/prebuilts.git -b master --depth 1
cd prebuilts/gcc-x64
```

### 5.2 解压工具链

> **警告：以下命令会使用 `sudo` 将工具链解压到 `/opt/FriendlyARM`。执行前请确认当前目录和压缩包来源可信。**

```bash
cat toolchain-6.4-aarch64.tar.gz* | sudo tar xz -C /
```

### 5.3 配置环境变量

```bash
cat >> ~/.bashrc <<'EOF'
export PATH=/opt/FriendlyARM/toolchain/6.4-aarch64/bin:$PATH
export GCC_COLORS=auto
EOF
```

让配置立即生效：

```bash
. ~/.bashrc
```

### 5.4 验证工具链

```bash
aarch64-linux-gcc -v
```

示例输出：

```text
Using built-in specs.
COLLECT_GCC=aarch64-linux-gcc
Target: aarch64-cortexa53-linux-gnu
Thread model: posix
gcc version 6.4.0 (ctng-1.23.0-150g-FA)
```

如果能看到 `Target: aarch64-cortexa53-linux-gnu` 和 `gcc version 6.4.0`，说明工具链基本可用。

## 6. 快速启动 FriendlyCore ARM64

### 6.1 下载 SD 卡镜像

从 FriendlyElec 资料源下载 Smart6818 / S5P6818 对应的 FriendlyCore ARM64 SD 卡镜像。

当前验证使用的镜像路径：

```text
01_系统固件/01_SD卡固件/s5p6818-sd-friendlycore-xenial-4.4-arm64-20260101.img.gz
```

### 6.2 烧录 SD 卡

推荐使用 balenaEtcher 将 `.img` 镜像烧录到 SD 卡。

> **警告：烧录镜像会清空 SD 卡原有数据。执行前请确认目标设备是 SD 卡，不要选错磁盘。**

烧录完成后：

1. 将 SD 卡插入 GEC6818 的 SD0 卡槽
2. 连接串口线
3. 上电启动
4. 观察 U-Boot 和 Kernel 启动日志

> 注意：GEC6818 只有指定 SD 卡槽支持启动，本项目验证使用的是 SD0。

## 7. 临时复制 DTB 让系统先启动

首次启动时，U-Boot 可能能运行，但 Kernel 无法启动，串口日志会提示缺少某个 `.dtb` 文件。

临时处理方法：

1. 将 SD 卡重新插回电脑
2. 打开 boot 分区
3. 找到类似 `s5p6818-nanopi3-rev03.dtb` 的设备树文件
4. 复制一份并重命名为启动日志提示缺失的文件名

例如：

```bash
cp s5p6818-nanopi3-rev03.dtb s5p6818-nanopi3-rev00.dtb
cp s5p6818-nanopi3-rev03.dtb s5p6818-nanopi3-rev04.dtb
```

如果后续 U-Boot 期望的是 GEC6818 自己的 DTB，则复制为：

```bash
cp s5p6818-nanopi3-rev03.dtb s5p6818-gec6818-rev01.dtb
```

> 说明：这是临时跑通方法。真正适配应当在 Kernel 源码中新增 GEC6818 对应的 DTS，并编译生成自己的 DTB。

启动后如果出现声卡、网卡等报错，通常是因为设备树仍然沿用了 NanoPi3 / Smart6818 的外设描述，和 GEC6818 实际硬件连接不完全一致。

## 8. U-Boot 适配

### 8.1 获取源码

```bash
git clone https://github.com/celeron633/u-boot_gec6818.git
cd u-boot_gec6818
```

### 8.2 编译 U-Boot

```bash
source profile_env.sh
make s5p6818_gec6818_defconfig
make -j"$(nproc)"
```

编译完成后，重点关注生成的：

```text
fip-nonsecure.img
```

### 8.3 重新打包 SD 卡镜像

获取 FriendlyARM 的 SD 镜像打包工具：

```bash
git clone https://github.com/friendlyarm/sd-fuse_s5p6818 -b master --single-branch sd-fuse_s5p6818
cd sd-fuse_s5p6818
```

下载 FriendlyCore ARM64 分区镜像：

```bash
wget http://112.124.9.243/dvdfiles/s5p6818/images-for-eflasher/friendlycore-arm64-images.tgz
tar xvzf friendlycore-arm64-images.tgz
```

解压后会得到：

```text
friendlycore-arm64/
```

将自己编译得到的 `fip-nonsecure.img` 替换到相关目录中。

建议同时检查并替换以下位置：

```text
prebuilt/fip-nonsecure.img
friendlycore-arm64/fip-nonsecure.img
```

> **警告：不要在尚未确认启动链路前直接烧写 eMMC。尤其是原板仍使用 32 位 U-Boot 2014.07 时，直接替换启动组件可能导致开发板无法正常启动。建议优先使用 SD 卡验证。**

打包 SD 卡镜像：

```bash
./mk-sd-image.sh friendlycore-arm64
```

成功后会生成类似文件：

```text
out/s5p6818-sd-friendly-core-xenial-4.4-arm64-YYYYMMDD.img
```

将该镜像重新烧录到 SD 卡，上电后观察串口日志。

## 9. Kernel / DTS 适配

### 9.1 获取 Kernel 源码

```bash
git clone https://github.com/friendlyarm/linux.git -b nanopi2-v4.4.y linux-4.4-full
cd linux-4.4-full
```

### 9.2 确认 Kernel 版本

```bash
grep '^SUBLEVEL ?=' Makefile
git log -1 --oneline --decorate
```

期望 Kernel 版本为：

```text
4.4.172
```

> 说明：Kernel Image、DTB、U-Boot 以及 rootfs 之间存在版本耦合。版本不匹配时，可能出现启动失败、驱动 probe 失败或用户态模块不兼容等问题。

### 9.3 新增 GEC6818 defconfig

进入配置目录：

```bash
cd arch/arm64/configs
cp nanopi3_linux_defconfig gec6818_linux_defconfig
```

`gec6818_linux_defconfig` 是 GEC6818 的板级内核配置文件，后续可以在这里裁剪或打开驱动选项。

### 9.4 新增 GEC6818 DTS / DTSI （不想整下面的就直接去Release里下载v0.1.0-dts-bringup）

进入设备树目录：

```bash
cd arch/arm64/boot/dts/nexell
```

复制公共设备树：

```bash
cp s5p6818-nanopi3-common.dtsi s5p6818-gec6818-common.dtsi
```

复制板级 DTS：

```bash
cp s5p6818-nanopi3-rev03.dts s5p6818-gec6818-rev01.dts
```

修改 `s5p6818-gec6818-rev01.dts` 的 include：

```dts
/dts-v1/;
#include "s5p6818-gec6818-common.dtsi"

/ {
	model = "GEC6818 rev01";
	compatible = "weicheng,gec6818-rev01", "nexell,s5p6818";
};

&mach {
	hwrev = <1>;
	model = "GEC6818_Common_Board1";
};
```

### 9.5 修改 DTS Makefile

编辑：

```text
arch/arm64/boot/dts/nexell/Makefile
```

加入：

```makefile
dtb-$(CONFIG_ARCH_S5P6818) += s5p6818-gec6818-rev01.dtb
```

这样编译 Kernel 时会生成 GEC6818 对应的 DTB。

### 9.6 编译 Kernel 和 DTB

回到 Kernel 根目录：

```bash
cd /path/to/linux-4.4-full
touch .scmversion
make ARCH=arm64 gec6818_linux_defconfig
make ARCH=arm64 -j"$(nproc)"
```

编译产物位置：

```text
arch/arm64/boot/Image
arch/arm64/boot/dts/nexell/s5p6818-gec6818-rev01.dtb
```

### 9.7 替换 boot 分区文件

将生成的文件替换到 SD 卡 boot 分区：

```text
Image
s5p6818-gec6818-rev01.dtb
```

然后重新上电，使用串口抓取日志，确认启动日志中出现：

```text
GEC6818 rev01
GEC6818_Common_Board1
```

如果串口日志能看到自定义 `model` 字段，说明当前启动使用的确实是自己新增的 GEC6818 DTS。

## 10. LCD 显示问题

目前发现 U-Boot 阶段 LCD 分辨率可能不正确，导致显示错位。

相关文件：

```text
board/s5p6818/gec6818/lcd.s
```

以 7 寸 AT070TN92 面板为例，可尝试确认面板分辨率是否为：

```text
800x480
```

修改后重新编译 U-Boot，并替换 `fip-nonsecure.img` 后验证。

> 注意：LCD 参数不仅包括分辨率，还可能包括 pixel clock、hsync、vsync、前后肩参数等。只改分辨率能解决部分显示错位问题，但不一定能完全解决显示稳定性问题。

## 11. 常见问题与排查思路

### 11.1 U-Boot 能启动，但 Kernel 不启动

优先检查：

```text
1. boot 分区是否存在 U-Boot 期望的 DTB 文件
2. DTB 文件名是否和启动脚本 / U-Boot 环境变量一致
3. Kernel Image 是否为 ARM64 Image
4. Kernel 版本是否与 rootfs、模块目录匹配
5. 串口日志中是否有 FDT / DTB / bad magic / unable to mount rootfs 报错
```

### 11.2 系统能启动，但声卡 / 网卡报错

优先检查：

```text
1. GEC6818 原理图中的外设连接
2. DTS 中对应节点的 compatible、reg、interrupts、clocks、gpios
3. Kernel 是否打开对应驱动
4. dmesg 中对应驱动 probe 失败原因
5. /proc/device-tree 中实际加载的设备树内容
```

常用命令：

```bash
dmesg | grep -iE 'sound|audio|eth|net|phy|dtb|fdt'
ls /proc/device-tree
cat /proc/device-tree/model
```

### 11.3 如何确认当前运行的是自己的 DTB

```bash
cat /proc/device-tree/model
```

期望输出：

```text
GEC6818 rev01
```

也可以检查 compatible：

```bash
tr '\0' '\n' < /proc/device-tree/compatible
```

期望包含：

```text
weicheng,gec6818-rev01
nexell,s5p6818
```

## 12. 后续计划

- [ ] 整理完整串口启动日志
- [ ] 增加 U-Boot 编译补丁
- [ ] 增加 Kernel DTS 补丁
- [ ] 适配 LCD 参数
- [ ] 排查声卡驱动
- [ ] 排查网卡驱动
- [ ] 整理 SD 卡镜像打包脚本
- [ ] 验证 eMMC 启动方案
- [ ] 增加图片和启动效果截图

## 13. 建议的仓库结构

后续建议将仓库整理为：

```text
.
├── README.md
├── docs/
│   ├── boot-log.md
│   ├── uboot.md
│   ├── kernel-dts.md
│   └── known-issues.md
├── patches/
│   ├── u-boot/
│   └── linux-4.4.172/
├── scripts/
│   ├── build-kernel.sh
│   └── pack-sd-image.sh
└── images/
    └── boot-success.jpg
```

## 14. 免责声明

本项目仍处于验证和整理阶段，部分外设尚未完成适配。本文档中的命令和路径基于当前测试环境记录，其他宿主机、Kernel 版本、U-Boot 版本或硬件批次可能需要调整。

涉及 `sudo`、镜像烧录、启动镜像替换和 eMMC 写入的操作均有风险。执行前请确认目标设备、分区路径和备份状态。
