+++
title = "为raspberry pi 4 交叉编译, 构建u-boot 和 linux 内核"
date = 2021-06-27

[taxonomies]
tags = ["raspberry pi", "u-boot", "kernel"]
+++

1.  编译u-boot 启动管理器：
```bash
git   clone  git://git.denx.de/u-boot.git
cd  u-boot  
make  rpi_4_defconfig 
CROSS_COMPILE=aarch64-rpi4-linux-gnu-  make -j8
```
编译后生成u-boot 相关文件:
```bash
u-boot  u-boot.bin  u-boot.cfg  u-boot.cfg.configs  u-boot.lds  u-boot.map  u-boot-nodtb.bin  u-boot.srec  u-boot.sym
```
其中的u-boot.bin  是需要放到SD卡启动分区的启动管理器二进制文件

2.  编译raspberry pi foundation 的内核:
```bash
git clone --depth=1 -b rpi-5.10.y  https://github.com/raspberrypi/linux.git
cd  linux
make ARCH=arm64 CROSS_COMPILE=aarch64-rpi4-linux-gnu-   bcm2711_defconfig
make ARCH=arm64 CROSS_COMPILE=aarch64-rpi4-linux-gnu-  -j8
```

3. 准备SD卡启动分区中的文件:
```bash
cd  ~
sudo apt install subversion
svn export https://github.com/raspberrypi/firmware/trunk/boot
rm boot/kernel*
rm boot/*.dtb
rm boot/overlays/*.dtbo
```
安装svn， 使用svn下载树莓派官方不开源的二进制启动文件(奇怪，git 没什么便利的方法下载github仓库中某个指定目录),  我们需要boot 目录中的start4*elf ,   fixup4*dat；   删除boot目录中已有的kernel 文件， 设备树blob, overlay 文件。

```bash
cd  linux
cp arch/arm64/boot/Image    ../boot/kernel8.img
cp arch/arm64/boot/dts/overlays/*.dtbo    ../boot/overlays/
cp arch/arm64/boot/dts/broadcom/*.dtb     ../boot/
```
把刚才编译的新内核，设备树相关文件 copy  到  boot 目录

```bash
cd  ~/boot

cat    << EOF   >  config.txt
enable_uart=1
arm_64bit=1
kernel=u-boot.bin
EOF

cat << EOF   >    cmdline.txt
console=serial0,115200 console=tty1 root=/dev/mmcblk0p2 rootwait
EOF
```
在boot 目录中生成config.txt (start4*elf 的配置文件) , 内核命令行参数文件cmdline.txt 

```bash
cd  ~
cat  <<'EOF'  > boot.scr
fatload mmc 0:1 ${kernel_addr_r}   kernel8.img
fatload mmc 0:1 ${fdt_addr} bcm2711-rpi-4-b.dtb
setenv bootargs 8250.nr_uarts=1 console=ttyS0,115200 root=/dev/mmcblk0p2 rootwait rw
booti ${kernel_addr_r} - ${fdt_addr}
EOF

~/u-boot/tools/mkimage -A arm64 -O linux -T script -C none -n boot.scr  -d boot.scr  boot.scr.uimg

cp  ./boot.scr.uimg      ~/boot
cp  ~/u-boot/u-boot.bin     ~/boot
```

准备u-boot 启动脚本 boot.scr的内容， 使用u-boot 的mkimage 工具把 boot.scr 转换成 boot.scr.uimg;   最后把boot.scr.uimg 和u-boot.bin copy 到 boot 目录.

4. 分区格式化SD卡,  拷贝boot 目录内容 到SD卡的 fat 分区:
{{ image(src="https://oscimg.oschina.net/oscnet/up-c5fa385b8f03e818debce15169330d39087.png",  position="left") }}

```bash
cp   -a  ~/boot/*     /media/jianyu/BOOT  
```

5.  准备一个简单的init 程序， 放在根文件系统，防止内核因找不到init 程序而panic ：

loopInit.go:
```golang
package main

import (
	"time"
	//"fmt"
)

func main(){
	for range  time.Tick(time.Second){
		print("\u001Bc", time.Now().Local().String(), " hello, i'm init process!")
	}
}
```
```bash
CGO_ENABLED=0  GOARCH="arm64"   go build  -o loopInit loopInit.go

mkdir -p  /media/jianyu/root/bin
mv  loopInit    /media/jianyu/root/bin/init 
```

6. 启动系统:

使用TTL-USB 连接pi 和pc , 启动gtkterm：
```bash
sudo  gtkterm -p /dev/ttyUSB0 -s 115200
```
插上pi 的电源,u-boot 启动信息刷屏后，出现唯一的用户态程序:

{{ image(src="https://oscimg.oschina.net/oscnet/up-c67b2c145d8b0cda01c20e92687a04437bd.png", position="left") }}


