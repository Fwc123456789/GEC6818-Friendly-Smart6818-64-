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
- 记录 LCD、触摸、声卡、网卡等外设适配问题
- 用 GitHub Release 固化每个阶段的驱动移植结果
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
| LCD | 已完成第一轮适配 | 详见 `v0.2.0-lcd`，Linux 已从 NanoPi3 LVDS/X710 链路切到 GEC6818 RGB/AT070TN92 链路 |
| 触摸 | 已完成第一轮适配 | 详见 `v0.3.0-touch`，FT5206 已在 `i2c_1 / 0x38` 上完成 probe 与 input 事件验证 |
| 声卡 | 待适配 | 后续继续检查 DTS、codec、I2S、时钟和驱动 probe |
| 网卡 | 待适配 | 后续继续检查 MAC/PHY、供电、reset GPIO、时钟和驱动 probe |
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

## 10. 驱动移植进度与版本索引

本仓库的 README 只保留驱动移植的**总路线、版本索引和验证入口**。具体某个驱动的完整教程、代码片段、调试日志和踩坑记录，不再全部塞进 README，避免主文档过长、难维护。

后续每完成一个驱动阶段，就单独发布一个 GitHub Release，并在 Release notes 或 `docs/` 文档中记录完整过程。建议阅读顺序如下：

| 版本 | 主题 | 当前状态 | 建议用途 |
|---|---|---|---|
| `v0.1.0-dts-bringup` | DTS 第一轮供电链路与 MMC / USB OTG 验证 | 已发布 | 理解 GEC6818 DTS 从 NanoPi3 残留中收敛的第一步 |
| `v0.2.0-lcd` | LCD 显示链路适配 | 已完成第一轮 | 学习如何把 Linux 显示从 `LVDS + X710 + 1024x600` 切到 `RGB + AT070TN92 + 800x480` |
| `v0.3.0-touch` | FT5206 触摸驱动适配 | 已完成第一轮 | 学习如何把触摸从旧的 `i2c_2 + GPIOC16 + one-wire` 路径迁移到 `i2c_1 + GPIOB26 + GPIOB8 + FT5206` |
| 后续版本 | 声卡、网卡、eMMC 等 | 待更新 | 每个驱动单独拆分、单独验证、单独发布 |

Release 入口：

```text
https://github.com/Fwc123456789/GEC6818-Friendly-Smart6818-64-/releases
```

Tags 入口：

```text
https://github.com/Fwc123456789/GEC6818-Friendly-Smart6818-64-/tags
```

### 10.1 驱动移植总体原则

GEC6818 的外设适配不是简单把 NanoPi3 / Smart6818 的 DTS 复制过来，而是要逐项确认：

```text
原理图事实 -> DTS 节点 -> pinctrl / GPIO / clock / regulator -> 驱动 probe -> 运行时设备树 -> dmesg -> /sys 或 /dev 验证
```

每个驱动都按下面顺序推进：

1. **确认硬件事实**
   - 看 GEC6818 核心板 / 底板原理图。
   - 确认信号名称、GPIO、I2C/SPI/SDIO/UART 总线、供电、reset、interrupt、clock。

2. **识别上游残留**
   - 找出从 NanoPi3 / Smart6818 继承来的节点。
   - 判断哪些节点是可复用的，哪些是错误残留。
   - 对冲突节点先禁用，不要一边保留旧节点一边加新节点。

3. **做最小可验证修改**
   - 一次只改一个方向。
   - 不要同时大改 DTS、驱动、U-Boot、rootfs、启动参数。
   - 先让驱动干净 probe，再处理性能、坐标、音量、背光曲线、休眠恢复等细节。

4. **上板验证**
   - 串口日志必须保存。
   - `dmesg`、`/proc/device-tree`、`/sys`、`/dev` 的结果都要交叉验证。
   - 不能只凭“现象好像对了”判断移植成功。

5. **固化为版本**
   - 代码、文档、patch、日志一起提交。
   - 用 Git tag / GitHub Release 记录阶段状态。
   - 每个版本写清楚：改了什么、为什么改、怎么验证、还剩什么问题。

### 10.2 LCD 移植教程请看 `v0.2.0-lcd`

LCD 适配过程已经从总 README 中抽出，不再在这里展开完整教程。需要实践 LCD 移植时，请直接看 `v0.2.0-lcd` 对应的 Release notes 或 `docs/` 文档。

LCD 第一轮移植的核心结论：

```text
错误链路：
panel-friendlyelec -> X710 -> LVDS -> 1024x600

修正后链路：
panel-friendlyelec -> AT070TN92 -> RGB -> 800x480
```

该阶段主要完成：

- 关闭 LVDS / HDMI / Wi-Fi / 蓝牙 / one-wire 等冲突残留。
- 在 `drivers/gpu/drm/panel/panel-friendlyelec.c` 中添加 `AT070TN92` 面板描述。
- 将 `s5p6818-gec6818-common.dtsi` 中的显示输出从 `dp_drm_lvds` 切换到 `dp_drm_rgb`。
- 恢复 RGB pinctrl。
- 添加 `LCD_EN -> GPIOB24`。
- 添加 `pwm-backlight`，按 PWM0 方向控制背光。
- 暂不盲目绑定 `panel power-supply`，避免错误建模。

实践 LCD 移植时，建议至少验证：

```bash
cat /proc/cmdline

dmesg | grep -Ei "Display:|Load .*panel|Bind .*panel|framebuffer|splash|panel-friendlyelec|LVDS|RGB|onewire|gpio|backlight"

cat /sys/class/graphics/fb0/virtual_size
cat /sys/class/graphics/fb0/bits_per_pixel

ls /sys/class/backlight
cat /sys/class/backlight/*/brightness
cat /sys/class/backlight/*/max_brightness
```

预期方向：

```text
Display: AT070TN92 selected
RGB panel 被加载
framebuffer 为 800x480
LCD 画面不再乱码
背光节点可见
```

### 10.3 触摸移植教程请看 `v0.3.0-touch`

触摸适配过程同样不在 README 中展开完整教程。需要实践 FT5206 触摸移植时，请看 `v0.3.0-touch` 对应的 Release notes 或 `docs/` 文档。

触摸第一轮移植的核心结论：

```text
错误旧路径：
i2c_2 + GPIOC16 + one-wire / 多候选触摸驱动

修正后路径：
i2c_1 + GPIOB26(INT) + GPIOB8(WAKE) + FT5206 / edt-ft5x06
```

该阶段主要完成：

- 确认实物触摸芯片为 `FT5206 / FT5x06`。
- 将触摸节点挂到 `i2c_1`，运行时对应 `i2c-1 / c00a5000.i2c`。
- 确认 I2C 地址为 `0x38`。
- 使用 `GPIOB26` 作为触摸中断。
- 使用 `GPIOB8` 作为 WAKE。
- 在 `panel-friendlyelec.c` 中将 `AT070TN92` 默认触摸类型从 `CTP_ITE7260` 修正为 `CTP_FT5X06`。
- 不再依赖临时启动参数 `ctp=2`。
- 验证 `EP01200M09` input 设备注册成功。
- 验证 `/dev/input/event0` 有触摸事件流。

实践触摸移植时，建议至少验证：

```bash
cat /proc/cmdline

dmesg | grep -Ei "EP01200|0038|c00a5000|i2c-1|touch|ft5|edt|onewire"

cat /proc/bus/input/devices
ls -l /dev/input/

sudo hexdump /dev/input/event0

find -L /proc/device-tree -name "*touch*" -o -name "*ft*" -o -name "*onewire*" -o -name "*1wire*" -o -name "*ts*"
```

预期方向：

```text
/proc/cmdline 中不再需要 ctp=2
Display: AT070TN92 selected
input: EP01200M09 as .../i2c-1/1-0038/input/input0
/proc/bus/input/devices 中出现 EP01200M09
/dev/input/eventX 有触摸事件数据流
```

注意：当前旧的 `fa_ts_input` / one-wire 虚拟触摸残留可能仍存在。只要 `EP01200M09` 已经注册且事件流正常，FT5206 主触摸链路就已经跑通。后续应单独做 one-wire 清理版本。

### 10.4 后续驱动更新计划

后续驱动不会继续堆在 README 里，而是按版本逐个补充：

| 计划版本 | 目标 | 当前判断 |
|---|---|---|
| `v0.4.0-touch-cleanup` | 清理 one-wire / `fa_ts_input` / 旧 i2c_2 触摸候选 | 待处理 |
| `v0.5.0-audio` | 声卡适配 | 待分析 codec、I2S、时钟、DTS 和 probe 日志 |
| `v0.6.0-ethernet` | 网卡适配 | 待分析 MAC/PHY、reset GPIO、供电、时钟和驱动 |
| `v0.7.0-emmc` | eMMC 启动与供电链路 | 待确认 `VDD_EMMC` 与启动稳定性 |
| `v1.0.0-basic-board` | 基础板级支持 | LCD + 触摸 + 基础外设达到可复现状态 |

每个后续驱动版本建议至少包含：

```text
1. 问题现象
2. 硬件事实
3. 旧 DTS / 旧驱动残留
4. 最小修改集
5. 编译与替换
6. 上板验证命令
7. 成功日志
8. 已知问题
9. 下一步计划
```

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
- [x] 适配 LCD RGB/AT070TN92 第一轮显示链路，详见 `v0.2.0-lcd`
- [x] 适配 FT5206 触摸第一轮驱动链路，详见 `v0.3.0-touch`
- [ ] 清理 one-wire / `fa_ts_input` 旧触摸残留
- [ ] 排查声卡驱动
- [ ] 排查网卡驱动
- [ ] 整理 SD 卡镜像打包脚本
- [ ] 验证 eMMC 启动方案
- [ ] 增加图片和启动效果截图

## 13. 免责声明

本项目仍处于验证和整理阶段，部分外设尚未完成适配。本文档中的命令和路径基于当前测试环境记录，其他宿主机、Kernel 版本、U-Boot 版本或硬件批次可能需要调整。

涉及 `sudo`、镜像烧录、启动镜像替换和 eMMC 写入的操作均有风险。执行前请确认目标设备、分区路径和备份状态。
