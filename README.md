# GEC6818-Friendly-Smart6818-64-
因为GEC6818系统太老了（32位） 内核也是  但是这个处理器支持64位的系统！于是决定移植Friendly-Smart6818的64位数Linux系统!
开放流程！

# 一 环境
我用的wsl2
Ubuntu 20.04 LTS
以下来源Friendly Wiki （https://wiki.friendlyelec.com/wiki/index.php/Smart6818/zh）
## 1安装交叉编译器aarch64-linux-gcc 6.4
首先下载并解压编译器:
git clone https://github.com/friendlyarm/prebuilts.git -b master --depth 1
cd prebuilts/gcc-x64
cat toolchain-6.4-aarch64.tar.gz* | sudo tar xz -C /

## 2然后将编译器的路径加入到PATH中
用vi编辑vi ~/.bashrc，在末尾加入以下内容：
export PATH=/opt/FriendlyARM/toolchain/6.4-aarch64/bin:$PATH
export GCC_COLORS=auto

执行一下~/.bashrc脚本让设置立即在当前shell窗口中生效，注意"."后面有个空格：
. ~/.bashrc

这个编译器是64位的，不能在32位的Linux系统上运行，安装完成后，你可以快速的验证是否安装成功：
aarch64-linux-gcc -v
Using built-in specs.
COLLECT_GCC=aarch64-linux-gcc
COLLECT_LTO_WRAPPER=/opt/FriendlyARM/toolchain/6.4-aarch64/libexec/gcc/aarch64-cortexa53-linux-gnu/6.4.0/lto-wrapper
Target: aarch64-cortexa53-linux-gnu
Configured with: /work/toolchain/build/aarch64-cortexa53-linux-gnu/build/src/gcc/configure --build=x86_64-build_pc-linux-gnu
--host=x86_64-build_pc-linux-gnu --target=aarch64-cortexa53-linux-gnu --prefix=/opt/FriendlyARM/toolchain/6.4-aarch64
--with-sysroot=/opt/FriendlyARM/toolchain/6.4-aarch64/aarch64-cortexa53-linux-gnu/sysroot --enable-languages=c,c++
--enable-fix-cortex-a53-835769 --enable-fix-cortex-a53-843419 --with-cpu=cortex-a53
...
Thread model: posix
gcc version 6.4.0 (ctng-1.23.0-150g-FA)

# 二 开发流程 
## 1火速下载跑通
正常的开发流程 u-boot kernel rootfs 
但是这里移植 优先使板子跑起来 所以  找到正确的sd卡镜像启动包 跑起来！
这里引用friendly的百度网盘链接（https://pan.baidu.com/s/1kVySP6Z#list/path=%2F）
路径是Friendly...>01_系统固件>01_SD卡固件 s5p6818-sd-friendlycore-xenial-4.4-arm64-20260101.img.gz

### 1.2烧录
下载完解压 然后把img文件烧录到SD卡！（我用的是balenaEtcher烧录软件 Windows版本的）（需要读卡器!）
烧录完成之后 就可以把SD卡插在开发板SD0的卡槽（只有这个卡槽支持启动！）然后连接串口线 上电！
哦对了 这里上电会发现 进了uboot但是不进系统 报错 说 缺少dtb文件！ 这时候要把SDka用读卡器连接电脑

### 1.3复制DTB
这里笔者建议 去用个DiskGenius这样的软件 把s5p6818-nanopi3-rev03.dtb下载到桌面 （因为这个和gec6818最相似）
路径就是<img width="1024" height="293" alt="SnowShot_2026-04-24_15-54-07" src="https://github.com/user-attachments/assets/7978bc05-18db-474a-8cb0-f169a055a7ba" />
选中右键复制到桌面就行
然后复制多一份 改名 （缺少什么就改成什么名字）印象中是s5p6818-nanopi3-rev00.dtb和s5p6818-nanopi3-rev04.dtb
然后再把改好的文件直接拖拽到那群dtb文件之中 根据提示把文件正确的复制进SD卡正确的分区路径下（就是boot下）
这时候 再一次上电 就可以开机进系统了 由于驱动存在不一样 所以即便启动成功 也会报错 比如说 声卡报错 网卡报错 这些想要改进 就要修改dtsi源码了！

## 2uboot适配
不建议自己从零开始移植 因为这里面很多门道 所以这里直接参考一个大哥移植好的uboot（https://github.com/celeron633/u-boot_gec6818）
编译方法：
### 2.1下载源码
git clone https://github.com/celeron633/u-boot_gec6818.git（如果慢就改手动下载然后自己解压到正确的Ubuntu文件夹下面）

### 2.2编译
source profile_env.sh
make s5p6818_gec6818_defconfig
make -j 12

### 2.3制作镜像并烧录（目前仅用SD卡开发 后续烧emmc请要认真注意！若现在使用的uboot 2014.07版本的32位请不要直接烧，会砖掉）
重新打包SD卡运行固件
注: 这里以friendlycore-arm64系统为例进行说明
下载本仓库到本地, 然后下载并解压friendlycore-arm64系统的分区镜像文件压缩包, 由于http服务器带宽的关系, wget命令可能会比较慢, 推荐从网盘上下载同名的文件:

git clone https://github.com/friendlyarm/sd-fuse_s5p6818 -b master --single-branch sd-fuse_s5p6818
cd sd-fuse_s5p6818
wget http://112.124.9.243/dvdfiles/s5p6818/images-for-eflasher/friendlycore-arm64-images.tgz
tar xvzf friendlycore-arm64-images.tgz
解压后, 会得到一个名为friendlycore-arm64的目录, 可以根据项目需要, 对目录里的文件进行修改, 例如把rootfs.img替换成自已修改过的文件系统镜像, 或者自已编译的内核和uboot等

重点！这里用编译产物的fip-nonsecure.img替换prebuilt目录下面原来的再烧写即可
（这是大哥的原话 我亲测 建议把friendlycore-arm64里面的fip-nonsecure.img也替换掉！看脚本似乎就是拿这个去打包的！）

打包成可用于SD卡烧写的单一镜像文件:
./mk-sd-image.sh friendlycore-arm64
命令执行成功后, 将生成以下文件, 此文件可烧写到SD卡运行:

out/s5p6818-sd-friendly-core-xenial-4.4-arm64-YYYYMMDD.img
然后烧录SD卡运行！这时候 就会提示缺少dtb 根据上面1.3复制dtb到桌面改名s5p6818-gec6818-rev01.dtb就行了
然后就能进系统了

## 3驱动移植（Smart6818移植到GEC6818）
这里需要GEC6818资料（https://pan.baidu.com/s/1WsLTYM0sY2A7xuVaUlddLg?pwd=22yR） 后续开发需要这里面的原理图手册等
###　3.1拉“完整仓库”
git clone https://github.com/friendlyarm/linux.git -b nanopi2-v4.4.y linux-4.4-full
cd linux-4.4-full

### 3.2确认当前版本
grep '^SUBLEVEL ?=' Makefile
git log -1 --oneline --decorate
确保是4.4.172(不然编译出来的image会不兼容当前uboot)

### 3.3在源码添加GEC6818板子
在5-Linux-4.4.172-gec\arch\arm64\configs
复制nanopi3_linux_defconfig
并重命名为gec6818_linux_defconfig（这就是你的板级配置文件　后续在这里面可以更改很多可以决定你板子的功能）

在5-Linux-4.4.172-gec\arch\arm64\boot\dts\nexell
在这里复制s5p6818-nanopi3-common.dtsi
并重命名为s5p6818-gec6818-common.dtsi（这就是你的板级设备树文件　驱动主要在这里修改）

在5-Linux-4.4.172-gec\arch\arm64\boot\dts\nexell
在这里复制s5p6818-nanopi3-rev03.dts
并重命名为s5p6818-gec6818-rev01.dts（这就是你的板级设备树追加文件）
打开s5p6818-gec6818-rev01.dts 切到自己的 common.dtsi　
把#include "s5p6818-nanopi3-common.dtsi"
改成#include "s5p6818-gec6818-common.dtsi"


在5-Linux-4.4.172-gec\arch\arm64\boot\dts\nexell
Makefile这个文件里面　第1行加入
dtb-$(CONFIG_ARCH_S5P6818) += s5p6818-gec6818-rev01.dtb
（这个确保生成正确的s5p6818-gec6818-rev01.dtb）

### 3.4编译之前改日志输出
编译之前　建议修改s5p6818-gec6818-rev01.dts已确保我们的产出区别与nanopi3
原来
/dts-v1/;
#include "s5p6818-nanopi3-common.dtsi"
/ {
	model = "FriendlyElec Smart6818";
	compatible = "friendlyelec,smart6818", "nexell,s5p6818";
};
&mach {
	hwrev = <3>;
	model = "Smart6818";
};

改成你自己想要输出的信息　这样后续在串口日志里面抓取到这些信息　证明这个板级文件配置算是成功了
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


### 3.5编译！
在5-Linux-4.4.172-gec之下
touch .scmversion
make ARCH=arm64 gec6818_linux_defconfig
make ARCH=arm64　
或者指定8线程编译（最好你的电脑满足　你也可以拉满　这样不用等很久）　
make ARCH=arm64 -j 8


