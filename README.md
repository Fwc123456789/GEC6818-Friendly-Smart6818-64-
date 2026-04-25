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
| LCD | 已完成第一轮适配 | Linux 已从 NanoPi3 LVDS/X710 链路切到 GEC6818 RGB/AT070TN92 链路，已加入 LCD_EN 与 PWM 背光 |
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

### 9.4 新增 GEC6818 DTS / DTSI

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

## 10. LCD 驱动移植：从 NanoPi3 LVDS/X710 切换到 GEC6818 RGB/AT070TN92

本节记录 GEC6818 LCD 第一轮移植过程。目标不是简单“让屏幕亮”，而是把 Linux 显示链路从原先继承自 NanoPi3 / Smart6818 的 **LVDS + X710 + 1024x600**，切换为 GEC6818 实际硬件使用的 **RGB + AT070TN92 + 800x480**。

### 10.1 问题现象

移植前，板子可以进入系统，但 LCD 显示异常，表现为乱码、错位或画面不稳定。

串口日志中可以看到一个关键矛盾：

```text
U-Boot: RGB / 800x480 / AT070TN92
Linux : LVDS / 1024x600 / X710
```

U-Boot 阶段已经按 GEC6818 的 7 寸 RGB 屏初始化：

```text
RGB:   dp.0
LCD:   [LCD] dp.0.1 800x480 32bpp FB:0x46000000
```

但 Linux 阶段仍然走 NanoPi3 / FriendlyElec 的默认 LCD 生态：

```text
Display: X710 selected
[drm] Load LVDS panel
[drm] Bind LVDS panel
[drm] framebuffer width(1024), height(600) and bpp(32) buffers(3)
```

这说明 Linux 实际使用的是：

```text
X710 + LVDS + 1024x600
```

而 GEC6818 板载 LCD 实际应为：

```text
AT070TN92 + RGB TTL + 800x480
```

这就是 LCD 乱码的主要原因。

### 10.2 移植前的关键判断

移植前先确认几件事：

1. `lcd=AT070TN92,128dpi` 已经通过 cmdline 传给内核。
2. 但当前 `panel-friendlyelec.c` 的面板列表中没有 `AT070TN92`。
3. 驱动匹配失败后，默认落回 `X710`。
4. 当前 DTS 启用的是 `dp_drm_lvds`，不是 `dp_drm_rgb`。
5. 当前 DTS 中仍残留 NanoPi3 的 Wi-Fi、蓝牙、onewire、LVDS、HDMI 等配置。
6. GEC6818 的 LCD_EN 是 `GPIOB24`，但该 GPIO 曾被 NanoPi3 风格的 Wi-Fi 节点占用。
7. GEC6818 背光更符合 `PWM0 + pwm-backlight` 模型，而不是 FriendlyElec 外接屏模块的 `onewire_pwm` 模型。

因此，本轮 LCD 移植不能只改一个分辨率参数，而要同时处理：

```text
panel 驱动识别
DTS 输出链路
RGB pinctrl
LCD_EN GPIO
PWM 背光
NanoPi3 残留冲突
```

### 10.3 第一阶段移植目标

本轮 LCD 移植目标如下：

```text
Display: AT070TN92 selected
Load/Bind RGB panel
framebuffer 800x480
LCD 不再乱码
背光可通过 PWM 控制
```

不在本轮处理的内容：

```text
触摸驱动
LCD reset GPIO
panel power-supply 精确建模
HDMI 同屏/双屏输出
LVDS 输出
```

原因是这些内容会引入更多变量。LCD bring-up 第一阶段应先保证 RGB 显示链路稳定。

### 10.4 关闭冲突节点：LVDS、HDMI、Wi-Fi、蓝牙、onewire 残留

GEC6818 当前 DTS 是从 NanoPi3 / Smart6818 相关 DTS 迁移而来，里面有不少残留节点。LCD 移植前，需要先关闭或隔离这些可能抢 GPIO、抢输出链路、抢 panel 绑定关系的节点。

重点冲突如下：

| 残留节点 / 功能 | 问题 |
|---|---|
| `dp_drm_lvds` | Linux 原先走 LVDS 输出，和 GEC6818 RGB LCD 不符 |
| HDMI 输出 | 第一阶段会干扰 DRM 输出路径判断，建议先关闭 |
| `wlan` | 可能占用 `GPIOB24`，而 GEC6818 的 `GPIOB24` 是 `LCD_EN` |
| `bt_bcm` | 可能占用 `GPIOB8`，而 GEC6818 LCD 区域中 `GPIOB8` 标为 `WAKE` |
| `onewire_pwm` | FriendlyElec 外接屏模块的 one-wire 背光/ID 机制，不适合 GEC6818 板载 AT070TN92 |
| `onewire_ts` / Friendly 触摸节点 | Friendly 外接屏触摸生态残留，GEC6818 触摸应另行按原理图处理 |

建议在 `s5p6818-gec6818-common.dtsi` 中先将无关节点置为 disabled。具体节点名以当前 DTS 为准，示意如下：

```dts
&dp_drm_lvds {
	status = "disabled";
};

&dp_drm_hdmi {
	status = "disabled";
};

/* NanoPi3 / FriendlyElec 残留：GEC6818 第一阶段 LCD 移植不使用 */
&wlan {
	status = "disabled";
};

&bt_bcm {
	status = "disabled";
};

&onewire_pwm {
	status = "disabled";
};
```

如果某些节点没有 label，就直接在原节点中加：

```dts
status = "disabled";
```

为什么要先关这些节点：

- 避免 `GPIOB24` 被 Wi-Fi 节点提前申请，导致 panel 的 `enable-gpios` 申请失败。
- 避免 Linux 继续绑定 LVDS panel。
- 避免 one-wire 背光和 GEC6818 的 PWM 背光混在一起。
- 降低变量数量，先把 RGB LCD 主链路跑通。

### 10.5 修改 panel 驱动：添加 AT070TN92 面板描述

文件位置：

```text
drivers/gpu/drm/panel/panel-friendlyelec.c
```

原问题：

```text
cmdline 已经传入 lcd=AT070TN92,128dpi
但 panel-friendlyelec.c 的 panel_lcd_list[] 中没有 AT070TN92
所以内核最终落回 X710
```

处理方法：在 `panel-friendlyelec.c` 中新增 `AT070TN92` 的 panel descriptor，并加入面板列表。

参数优先参考 GEC6818 U-Boot 和官方 32 位 BSP 中已经验证过的 AT070TN92 配置：

```text
分辨率: 800x480
物理尺寸: 约 155mm x 93mm
色深: RGB888 / 24bpp
接口: RGB TTL
h_fp = 80
h_bp = 36
h_sw = 10
v_fp = 22
v_bp = 15
v_sw = 8
```

示意代码如下，字段名需要按你当前 `panel-friendlyelec.c` 的实际结构体定义调整：

```c
static struct lcd_desc at070tn92 = {
	.width= 800,
	.height = 480,
	.p_width = 155,
	.p_height = 93,
	.bpp = 24,
	.freq = 61,

	.timing = {
		.h_fp = 80,
		.h_bp = 36,
		.h_sw = 10,
		.v_fp = 22,
		.v_fpe = 1,
		.v_bp = 15,
		.v_bpe = 1,
		.v_sw = 8,
	},
	.polarity = {
		.rise_vclk = 0,
		.inv_hsync = 1,
		.inv_vsync = 1,
		.inv_vden = 0,
	},
};
```

然后把它加入当前驱动的 panel list，例如：

```c
static struct {
	char *name;
	struct lcd_desc *lcd;
	int ctp;
} panel_lcd_list[] = {
	{ "AT070TN92",	&at070tn92,	CTP_ITE7260 },
	{ "X710",	&x710,	CTP_ITE7260 },
	{ "HD101B",	&hd101,	CTP_GOODIX  },
	{ "HD101",	&hd101,	1 },
	{ "HD700",	&hd700,	1 },
	{ "HD702",	&hd700,	CTP_GOODIX  },
	{ "H70",	&hd700,	0 },
	{ "HD900",	&hd900,	CTP_ST1572  },
	{ "S70",	&s70,	1 },
	{ "S701",	&s70,	CTP_GOODIX  },
	{ "S702",	&s702,	1 },
	{ "S70D",	&s70d,	0 },
	{ "S430",	&s430,	CTP_HIMAX   },
	{ "K101",	&hd101,	CTP_FT5X06  },
	{ "U101A",	&hd101,	CTP_GOODIX  },
	{ "W500",	&w500,	CTP_GOODIX  },
	{ "YZ43",	&yz43,	CTP_GT9XXA  },
	{ "S430B",	&s430b,	CTP_GT9XXA  },

	{ "H43",	&h43,	0 },
	{ "P43",	&p43,	0 },
	{ "W35",	&w35,	0 },

	/* TODO: Testing */
	{ "W50",	&w50,	0 },
	{ "W101",	&w101,	1 },
	{ "A97",	&a97,	0 },
	{ "LQ150",	&lq150,	1 },
	{ "L80",	&l80,	1 },
	{ "BP101",	&bp101,	1 },

	{ "HDMI",	&hdmi_def,	0 },	/* Pls keep it at last */
};
```

关键目标是让启动日志从：

```text
Display: X710 selected
```

变成：

```text
Display: AT070TN92 selected
```

只要这里没有变，后面 DTS 即使切 RGB，也仍可能继续使用错误的 panel mode。

### 10.6 DTS：从 LVDS 输出切换到 RGB 输出

当前 GEC6818 DTS 原先走的是：

```text
panel-friendlyelec -> lcd_panel endpoint -> dp_drm_lvds -> LVDS output
```

但 GEC6818 AT070TN92 应该走：

```text
panel-friendlyelec -> lcd_panel endpoint -> dp_drm_rgb -> RGB output
```

所以需要禁用 LVDS，启用 RGB。

在 `s5p6818-gec6818-common.dtsi` 中处理显示输出节点。

禁用 LVDS：

```dts
&dp_drm_lvds {
	status = "disabled";
};
```

启用 RGB：

```dts
/* RGB 输出配置 */
&dp_drm_rgb  {
	remote-endpoint = <&lcd_panel>;
	status = "ok";
	/*status = "disabled";*/

	display-timing {
		clock-frequency = <33000000>;
		hactive = <800>;
		vactive = <480>;
		hfront-porch = <80>;
		hback-porch = <36>;
		hsync-len = <10>;
		vfront-porch = <22>;
		vback-porch = <15>;
		vsync-len = <8>;
	};

	dp_control {
		clk_src_lv0 = <3>;
		clk_div_lv0 = <16>;
		clk_src_lv1 = <7>;
		clk_div_lv1 = <1>;
		out_format = <2>;
	};
};
```

注意：

- `remote-endpoint = <&lcd_panel>;` 用来把 RGB 输出 backend 接到 panel。
- 第一阶段不要同时启用 LVDS 和 RGB，避免 DRM 绑定路径混乱。
- HDMI 也建议先关掉，等 RGB LCD 稳定后再考虑多输出。

### 10.7 DTS：恢复 RGB pinctrl

RGB TTL 屏需要 LCD 数据线和同步信号 pinmux 正确输出。

GEC6818 使用 RGB24，涉及：

```text
LCD_R0~R7
LCD_G0~G7
LCD_B0~B7
LCD_CLK
LCD_HSYNC
LCD_VSYNC
LCD_DE
```

如果当前 DTS 中 RGB pinctrl 被注释，需要恢复：

```dts
pinctrl-names = "default";
pinctrl-0 = <&dp_rgb_vclk &dp_rgb_vsync &dp_rgb_hsync
	     &dp_rgb_de &dp_rgb_R &dp_rgb_G &dp_rgb_B>;
```

可以放在 `panel` 节点，也可以放在 `dp_drm_rgb` 相关节点。具体放置位置以当前 Nexell DRM DTS 示例为准，关键是最终运行时 pinctrl 必须生效。

建议验证：

```bash
cat /sys/kernel/debug/pinctrl/*/pinmux-pins | grep -iE "lcd|rgb|dp"
```

如果系统没有开启 pinctrl debug，也可以先通过画面是否正常、dmesg 是否报 pinctrl 错误来判断。

### 10.8 DTS：panel 节点加入 lcd 名称与 LCD_EN

GEC6818 原理图中：

```text
LCD_EN -> GPIOB24
```

而 `panel-friendlyelec.c` 会读取 `enable-gpios`，因此 panel 节点应加入：

```dts
	/* LCD 面板节点。Weicheng已经适配！ */
	panel: panel-friendlyelec {
		compatible = "lcds";
		lcd = "AT070TN92";
		enable-gpios = <&gpio_b 24 GPIO_ACTIVE_HIGH>;/* 10.8加这一行 */
		backlight = <&backlight>;
		status = "okay";

		/*
		 * 如果后续确认 RGB 引脚复用需要在 panel 节点显式声明，
		 * 可以恢复下面这组 pinctrl。
		 * Weicehng已经恢复！
		 */
		pinctrl-names = "default";
		pinctrl-0 = <&dp_rgb_vclk &dp_rgb_vsync &dp_rgb_hsync
			&dp_rgb_de &dp_rgb_R &dp_rgb_G &dp_rgb_B>;
		

		port {
			lcd_panel: endpoint {
			};
		};
	};
```

如果你还没有包含 GPIO flag 宏，需要确认 DTS 文件或上级 include 是否已经引入：

```dts
#include <dt-bindings/gpio/gpio.h>
```

否则可以先沿用数字 flag：

```dts
enable-gpios = <&gpio_b 24 0>;
```

但从可读性上，更推荐使用：

```dts
GPIO_ACTIVE_HIGH
```

加入 `enable-gpios` 后，启动时应检查是否还有 GPIO 冲突：

```bash
dmesg | grep -iE "gpio|panel|friendlyelec"
cat /sys/kernel/debug/gpio | grep -iE "gpio-.*24|panel|enable|lcd"
```

如果 `GPIOB24` 仍被 wlan 节点占用，说明前面的 NanoPi3 残留没有清干净。

### 10.9 DTS：添加 PWM 背光

GEC6818 板载 LCD 背光线索是：

```text
MCU_BACKLIGHT_PWM -> GPIOD1 / PWM0
```

官方 32 位 BSP 中也指向 PWM0，频率为 10 kHz。因此背光应优先建模为 `pwm-backlight`，而不是沿用 FriendlyElec 的 `onewire_pwm`。

建议加入：

```dts
	/* Weicheng 新增 pwm-backlight 节点 */
	backlight: pwm-backlight {
    compatible = "pwm-backlight";
    pwms = <&pwm 0 100000 0>; /* PWM0, 10kHz */
    brightness-levels = <0 8 16 32 64 128 192 255>;
    default-brightness-level = <5>;
    status = "okay";
	};
```

然后在 panel 中引用：

```dts
	/* LCD 面板节点。Weicheng已经适配！ */
	panel: panel-friendlyelec {
		compatible = "lcds";
		lcd = "AT070TN92";
		enable-gpios = <&gpio_b 24 GPIO_ACTIVE_HIGH>;/* 10.8加这一行 */
		backlight = <&backlight>;/* 10.9加这一行 */
		status = "okay";

		/*
		 * 如果后续确认 RGB 引脚复用需要在 panel 节点显式声明，
		 * 可以恢复下面这组 pinctrl。
		 * Weicehng已经恢复！
		 */
		pinctrl-names = "default";
		pinctrl-0 = <&dp_rgb_vclk &dp_rgb_vsync &dp_rgb_hsync
			&dp_rgb_de &dp_rgb_R &dp_rgb_G &dp_rgb_B>;
		

		port {
			lcd_panel: endpoint {
			};
		};
	};
```

如果当前 `&pwm` 节点没有启用，需要确认并打开：

```dts
&pwm {
	status = "okay";
};
```

如果有 pinctrl，需要确保 PWM0 pin 生效，例如：

```dts
/* PWM 公共配置。LCD 背光常常会直接依赖这里的 PWM0。 */
&pwm {
	status = "okay";
	pinctrl-0 = <&pwm0_pin &pwm1_pin &pwm2_pin>;
	samsung,pwm-outputs = <0>, <1>, <2>, <3>;
};
```

具体 pinctrl label 以当前 `s5p6818.dtsi` / `s5p6818-gec6818-common.dtsi` 中已有定义为准。

验证背光：

```bash
ls /sys/class/backlight
cat /sys/class/backlight/*/max_brightness
cat /sys/class/backlight/*/brightness
```

尝试调低和调高亮度：

```bash
echo 3 > /sys/class/backlight/*/brightness
sleep 1
echo 8 > /sys/class/backlight/*/brightness
```

> **注意：如果亮度写入后无变化，不要立刻乱改 PWM 极性。先确认 PWM0 pinctrl、背光电源、LCD_EN、背光芯片 EN 脚是否已经正确。**


### 10.10 编译与替换

回到 Kernel 根目录：

```bash
cd /path/to/linux-4.4.172-audiofix
touch .scmversion
make ARCH=arm64 gec6818_linux_defconfig
make ARCH=arm64 -j"$(nproc)"
```

重点产物：

```text
arch/arm64/boot/Image
arch/arm64/boot/dts/nexell/s5p6818-gec6818-rev01.dtb
```

替换 SD 卡 boot 分区中的：

```text
Image
s5p6818-gec6818-rev01.dtb
```

> **警告：替换 SD 卡 boot 分区文件前，建议先备份原来的 `Image` 和 `.dtb`。不要在 LCD 第一轮调试时直接写 eMMC。**

### 10.11 上板验证

启动后先看 cmdline：

```bash
cat /proc/cmdline
```

确认包含：

```text
lcd=AT070TN92,128dpi
```

检查 DRM / panel 日志：

```bash
dmesg | grep -Ei "Display:|Load .*panel|Bind .*panel|framebuffer|splash|panel-friendlyelec|LVDS|RGB|onewire|gpio|backlight"
```

理想日志方向：

```text
Display: AT070TN92 selected
[drm] Load RGB panel
[drm] Bind RGB panel
[drm] framebuffer width(800), height(480)
```

检查 framebuffer：

```bash
cat /sys/class/graphics/fb0/virtual_size
cat /sys/class/graphics/fb0/bits_per_pixel
```

期望类似：

```text
800,480
32
```

检查背光：

```bash
ls /sys/class/backlight
cat /sys/class/backlight/*/brightness
cat /sys/class/backlight/*/max_brightness
```

检查 GPIO 占用：

```bash
cat /sys/kernel/debug/gpio | grep -iE "panel|lcd|enable|gpio.*24|backlight|pwm"
```

如果结果满足：

```text
Display: AT070TN92 selected
RGB panel 被加载
framebuffer 为 800x480
LCD 画面不再乱码
背光可亮
```

说明 LCD 第一轮移植成功。

### 10.12 本轮 LCD 移植结论

本轮 LCD 调试的关键不是单纯改分辨率，而是修正整条显示链路：

```text
原错误链路：
panel-friendlyelec -> X710 -> LVDS -> 1024x600

修正后链路：
panel-friendlyelec -> AT070TN92 -> RGB -> 800x480
```

实际完成的核心改动：

1. 关闭冲突节点：LVDS、HDMI、蓝牙、Wi-Fi、onewire 残留。
2. 在 `panel-friendlyelec.c` 中新增 `AT070TN92` panel 描述，并加入 panel list。
3. 在 `s5p6818-gec6818-common.dtsi` 中将显示输出从 `dp_drm_lvds` 改为 `dp_drm_rgb`。
4. 恢复 RGB pinctrl。
5. 加入 `LCD_EN -> GPIOB24`。
6. 添加 `pwm-backlight`，使用 PWM0 方向适配 GEC6818 板载背光。
7. 暂不绑定 `panel power-supply`，避免错误建模。

这一步完成后，GEC6818 的 Linux 显示路径已经从 NanoPi3/FriendlyElec 的默认 LVDS 屏迁移到 GEC6818 板载 RGB 屏，为后续触摸、声卡、网卡等外设适配提供了更可靠的板级 DTS 基线。


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
- [x] 适配 LCD RGB/AT070TN92 第一轮显示链路
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
