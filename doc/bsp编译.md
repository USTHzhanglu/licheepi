# Licheepi zero  BSP编译指南

编译Licheepi zero的BSP（板级支持包）的启动镜像分步：
1：环境配置。
2：配置内核。
3：编译linux。
4：修改uboot启动选项。
5：制作sd启动盘

# 环境配置

## Docker

使用docker可以任意修改环境而不担心系统崩溃。

### Docker安装

```
sudo apt-get install docker.io
docker version
```

安装成功后可见版本信息

```
Client version: 1.6.2
Client API version: 1.18
Go version (client): go1.2.1
Git commit (client): 7c8fca2
OS/Arch (client): linux/amd64
FATA[0000] Get http:///var/run/docker.sock/v1.18/version: dial unix /var/run/docker.sock: permission denied. Are you trying to connect to a TLS-enabled daemon without TLS?
```

默认情况下会报后面的错误，如果使用sudo就不会报错。不想每次都sudo的话，可以把用户加入到docker组。

```
//如果还没有 docker group 就添加一个(默认安装后已经有了)
//sudo groupadd docker
//将用户加入该 group 内。然后退出并重新登录就生效啦。
sudo gpasswd -a ${your_user_name} docker
//重启 docker 服务
sudo service docker restart
//切换当前会话到新 group, 或者关掉终端重新连接也会生效
//newgrp - docker
```

### 安装荔枝派开发镜像

```
docker pull zepan/licheepi
docker run -d -p 6666:22 zepan/licheepi /usr/sbin/sshd -D
```

这样就安装并开启的容器ssh服务，只需连接主机的6666端口，以root用户，licheepi密码登录即可进行开发操作。

### 修改Docker

Docker中的BSP似乎有些问题，直接用会报`arch/arm/kernel/setup.c:60:29: fatal error: mach/sunxi-chip.h: No such file or directory #include <mach/sunxi-chip.h>`，你可以尝试修复此问题，也可以重新下载linux3.4内核（超链接，指向linux3.4-tar.gz)后替换文件。

### BSP 内核剥离

在lichee/linux-3.4目录下执行`bash ./scripts/build_tiger-cdr.sh` 

执行过程中会生成内核文件：

```
Image Name:   Linux-3.4.39
Created:      Tue Sep  5 06:55:03 2017
Image Type:   ARM Linux Kernel Image (uncompressed)
Data Size:    2275400 Bytes = 2222.07 kB = 2.17 MB
Load Address: 40008000
Entry Point:  40008000
  Image arch/arm/boot/uImage is ready
'arch/arm/boot/Image' -> 'output/bImage'
'arch/arm/boot/uImage' -> 'output/uImage'
'arch/arm/boot/zImage' -> 'output/zImage'
Building modules

compile Kernel successful
```

从主线uboot启动内核，一般使用uImage。

## Ubuntu20.04

### 获取bsp内核源码

bsp内核即lichee linux： http://pan.baidu.com/s/1eRJrViy

解压到你喜欢的目录。

### BSP内核剥离

BSP内核对摄像头驱动支持较好，所以在摄像头应用中有必要使用BSP内核。

官方SDK中camdriod与lichee linux内核绑定，而camdriod比较庞大，所以我们只需抽取lichee linux内核，而抛弃camdriod代码。

单独使用lichee linux的方法是：

解压 *buildroot/dl/gcc-linarno.tar.gz* ，并加入环境变量

```
tar -xvf ~/lichee/buildroot/dl/gcc-linaro.tar.bz2  -C ~/
vim ~/.bashrc
#add export PATH=$PATH:~/gcc-linaro/bin
source ~/.bashrc
```

 对于Ubuntu20.04，还需要安装32位支持库

 ```
sudo apt-get install lib32ncurses5-dev lib32z1
sudo apt-get install lib32ncurses5 lib32z1 #ubuntu16.04
 ```

不然会报`make: arm-linux-gnueabi-gcc: No such file or directory`这样的错误。

使用`arm-linux-gnueabi-gcc -v`验证是否生效

```
Thread model: posix
gcc version 4.6.3 20120201 (prerelease) (crosstool-NG linaro-1.13.1-2012.02-20120222 - Linaro GCC 2012.02) 

```

在lichee/linux-3.4目录下执行`bash ./scripts/build_tiger-cdr.sh` 

执行过程中会生成内核文件：

```
Image Name:   Linux-3.4.39
Created:      Tue Sep  5 06:55:03 2017
Image Type:   ARM Linux Kernel Image (uncompressed)
Data Size:    2275400 Bytes = 2222.07 kB = 2.17 MB
Load Address: 40008000
Entry Point:  40008000
  Image arch/arm/boot/uImage is ready
'arch/arm/boot/Image' -> 'output/bImage'
'arch/arm/boot/uImage' -> 'output/uImage'
'arch/arm/boot/zImage' -> 'output/zImage'
Building modules

compile Kernel successful
```

从主线uboot启动内核，一般使用uImage。

# BSP内核配置

BSP内核源码在lichee/linux-3.4下。

通过 `make ARCH=arm menuconfig` 来配置内核。

## 使能USB摄像头驱动

```
-> Device Drivers
  x       -> Multimedia support (MEDIA_SUPPORT [=y])
  x         -> Video capture adapters (VIDEO_CAPTURE_DRIVERS [=y])
  x           -> V4L USB devices (V4L_USB_DRIVERS [=y])
  x                          -><M>   USB Video Class (UVC)  CONFIG_USB_VIDEO_CLASS
```

## 使能DVP/MIPI摄像头

```
-> Device Drivers
  x       -> Multimedia support (MEDIA_SUPPORT [=y])
  x         -> Video capture adapters (VIDEO_CAPTURE_DRIVERS [=y])
  x           -> V4L platform devices (V4L_PLATFORM_DRIVERS [=y])
  x             -> SoC camera support (SOC_CAMERA [=y])
  x x                          <M>     ov2640 camera support
  x x                          <M>     ov5647 camera support
  x x                          <M>   sunxi video front end (camera and etc)driver
  x x                          <M>     v4l2 driver for SUNXI
   <*>   sunxi video encoder and decoder support
```

## 额外添加ext4支持

由于camdriod原始的内核配置是为了在spi nor flash上运行而配置的，没有ext4支持，所以需要额外添加ext4支持：

```
File systems  --->  
  x				<*> The Extended 4 (ext4) filesystem
  x x                          [*]   Use ext4 for ext2/ext3 file systems (NEW)
  x x                          [*]   Ext4 extended attributes (NEW)
  x x                          [ ]     Ext4 POSIX Access Control Lists (NEW)
  x x                          [ ]     Ext4 Security Labels (NEW)
  x x                          [ ]   EXT4 debugging support (NEW)
  x x                          [ ] JBD2 (ext4) debugging support (NEW)

```

另外还要加上CONFIG_LBDAF（大文件支持，否则无法挂载文件系统）

```
-> Enable the block layer (BLOCK [=y])
[*]   Support for large (2TB+) block devices and files
```

再加上CGROUPS支持：

```
-> General setup
 [*] Control Group support  --->
```

如果在文件系统（如debian）中使用了SWAP等特性，则还需要在内核中开启SWAP。

```
-> General setup
 [*] Support for paging of anonymous memory (swap)
```

debian下还需要开启 FHANDLE 特性，否则会出现以下错误

```
A start job is running for dev-ttyS0.device
timeout
```

```
-> General setup
 [*] open by fhandle syscalls
```

## 添加wifi驱动

如果需要使用wifi功能，则还需要勾选RTL8723BS的支持（注意需要选择模块方式），cfg80211的支持，和AW_RF_PM选项。

```
-> Device Drivers                                           x	-> Network device support (NETDEVICES [=n])             x    	-> Wireless LAN (WLAN [=n])    
x x 		<M>   Realtek 8723B SDIO WiFi 
```

```
-> Networking support  
     -*-   Wireless  --->      
         <*>   cfg80211 - wireless configuration API 
         <*>   Generic IEEE 802.11 Networking Stack (mac80211) 
```

```
-> Device Drivers                                            x    	-> Misc devicess
x x 		 [*] Allwinner rf module pm drivers
```

如果需要使用GPIO,还需要勾选/sys/class/gpio/... (sysfs interface)  

```
-> Device Drivers                                           x	 -*- GPIO Support  --->                      
x    	-> Wireless LAN (WLAN [=n])    
x x 		[*]   /sys/class/gpio/... (sysfs interface)  
```

以及下节所说的fex修改。

# 编译linux

## sh编译

在lichee/linux-3.4目录下执行`bash ./scripts/build_tiger-cdr.sh` （警告，会吃掉你的所有CPU和Memory！）

执行过程中会生成内核文件：

```
Image Name:   Linux-3.4.39
Created:      Fri Jul 23 02:44:59 2021
Image Type:   ARM Linux Kernel Image (uncompressed)
Data Size:    2621832 Bytes = 2560.38 KiB = 2.50 MiB
Load Address: 40008000
Entry Point:  40008000
  Image arch/arm/boot/uImage is ready
'arch/arm/boot/Image' -> 'output/bImage'
'arch/arm/boot/uImage' -> 'output/uImage'
'arch/arm/boot/zImage' -> 'output/zImage'
Building modules

 compile Kernel successful

```

## make编译

### 修改makefile

修改顶层Makefile第195行，指定编译器位置：

```
#ARCH           ?= $(SUBARCH)
#CROSS_COMPILE  ?= $(CONFIG_CROSS_COMPILE:"%"=%)
ARCH           ?=arm
CROSS_COMPILE  ?=arm-linux-gnueabi-
```

### make

```
make uImage -j16
make -j16 INSTALL_MOD_PATH=out modules
make -j16 INSTALL_MOD_PATH=out modules_install
```

执行中会生成内核文件：

```
Image Name:   Linux-3.4.39
Created:      Fri Jul 23 02:56:41 2021
Image Type:   ARM Linux Kernel Image (uncompressed)
Data Size:    2621832 Bytes = 2560.38 KiB = 2.50 MiB
Load Address: 40008000
Entry Point:  40008000
  Image arch/arm/boot/uImage is ready

```

# uboot启动BSP内核

使用主线uboot启动BSP内核，需要修改下启动脚本，放入BSP内核需要的 *script.bin* 配置文件（相当于主线linux的dtb）

## 配置boot.scr

修改boot.cmd:

```
vim ~/u-boot/boot.cmd
```

```
setenv bootargs console=ttyS0,115200 panic=5 rootwait root=/dev/mmcblk0p2 earlyprintk rw
setenv bootm_boot_mode sec
setenv machid 1029
load mmc 0:1 0x41000000 uImage
load mmc 0:1 0x41d00000 script.bin
bootm 0x41000000
```



重新生成boot.scr:

```
cd ~/u-boot/
mkimage -C none -A arm -T script -d boot.cmd boot.scr
```

## 配置script.bin

复制一份 *lichee/tools/pack/chips/sun8iw8p1/configs/tiger-cdr/sys_config.fex*

```
cp /root/lichee/tools/pack/chips/sun8iw8p1/configs/tiger-cdr/sys_config.fex /root/u-boot
vim /root/u-boot/sys_config.fex
```

首先修改SD卡检测策略，设置为不检测，默认插入

`[776]:sdc_detmode=3`

使能RTL8723bs无线网卡的话，需要使能mmc1，也设置为不检测sd卡。

```
[790]:[mmc1_para]
[791]:sdc_used          = 1
[792]:sdc_detmode       = 3
```

设置摄像头型号，csi0是mipi摄像头，csi1是dvp摄像头。

这里默认以mipi摄像头为ov5647, dvp摄像头为ov2640 为例:

```
[csi0]
vip_used                 = 1
vip_mode                 = 0
vip_dev_qty              = 1
vip_define_sensor_list   = 0
vip_csi_mck              = port:PE20<3><default><default><default>
vip_csi_sck              = port:PE21<2><default><default><default>
vip_csi_sda              = port:PE22<2><default><default><default>
vip_dev0_mname           = "ov5647_mipi"
vip_dev0_pos             = "rear"
vip_dev0_lane            = 1
vip_dev0_twi_id          = 0
vip_dev0_twi_addr        = 0x6c
vip_dev0_isp_used        = 1
vip_dev0_fmt             = 1
vip_dev0_stby_mode       = 0
vip_dev0_vflip           = 0
vip_dev0_hflip           = 0
vip_dev0_iovdd           = ""
vip_dev0_iovdd_vol       = 3000000
vip_dev0_avdd            = ""
vip_dev0_avdd_vol        = 3000000
vip_dev0_dvdd            = ""
vip_dev0_dvdd_vol        = 3000000
vip_dev0_afvdd           = ""
vip_dev0_afvdd_vol       = 2800000
vip_dev0_power_en        =
vip_dev0_reset           = port:PB02<1><default><default><default>
vip_dev0_pwdn            = port:PB03<1><default><default><default>
vip_dev0_flash_en        =
vip_dev0_flash_mode      =
vip_dev0_af_pwdn         =
vip_dev0_act_used        = 0
vip_dev0_act_name        = "dw9714_act"
vip_dev0_act_slave       = 0x18

[csi1]
vip_used                 = 1
vip_mode                 = 0
vip_dev_qty              = 1
vip_define_sensor_list   = 0
vip_csi_pck              = port:PE00<2><default><default><default>
vip_csi_mck              = port:PE01<2><default><default><default>
vip_csi_hsync            = port:PE02<2><default><default><default>
vip_csi_vsync            = port:PE03<2><default><default><default>
vip_csi_d0               = port:PE04<2><default><default><default>
vip_csi_d1               = port:PE05<2><default><default><default>
vip_csi_d2               = port:PE06<2><default><default><default>
vip_csi_d3               = port:PE07<2><default><default><default>
vip_csi_d4               = port:PE08<2><default><default><default>
vip_csi_d5               = port:PE09<2><default><default><default>
vip_csi_d6               = port:PE10<2><default><default><default>
vip_csi_d7               = port:PE11<2><default><default><default>
vip_csi_d8               = port:PE12<2><default><default><default>
;vip_csi_d9               = port:PE13<2><default><default><default>
vip_csi_d10               = port:PE14<2><default><default><default>
vip_csi_d11               = port:PE15<2><default><default><default>

vip_csi_sck               = port:PE21<2><default><default><default>
vip_csi_sda               = port:PE22<2><default><default><default>

vip_dev0_mname           = "ov2640"
vip_dev0_pos             = "front"
vip_dev0_twi_id          = 1
vip_dev0_twi_addr        = 0x60
vip_dev0_isp_used        = 0
vip_dev0_fmt             = 0
vip_dev0_stby_mode       = 0
vip_dev0_vflip           = 0
vip_dev0_hflip           = 0
vip_dev0_iovdd           = ""
vip_dev0_iovdd_vol       = 2800000
vip_dev0_avdd            = ""
vip_dev0_avdd_vol        = 2800000
vip_dev0_dvdd            = ""
vip_dev0_dvdd_vol        = 1500000
vip_dev0_afvdd           = ""
vip_dev0_afvdd_vol       = 2800000
vip_dev0_power_en        =
vip_dev0_reset           = 
vip_dev0_pwdn            = 
vip_dev0_flash_en        =
vip_dev0_flash_mode      =
vip_dev0_af_pwdn         =

vip_dev0_act_used        = 0
vip_dev0_act_name        = "ad5820_act"
vip_dev0_act_slave       = 0x18
```

将其中的摄像头信息改成自己使用的摄像头信息

（实际驱动还需要修改其他文件，详见[BSP内核的摄像头使用 — Lichee zero 文档](http://zero.lichee.pro/系统开发/bsp_cam.html#bsp)，并且在使用2640时，要注意与lcd的引脚冲突

![](https://whycan.cn/files/members/1974/_20190918163840.png)

用 devmem 设置寄存器

```
devmem 0x01c20898 32 0x02277777
devmem 0x01c20894 32 0x22222222
devmem 0x01c20890 32 0x77772212
```

后即可使用。）

注释掉165 166两行,不然会报`E: sys_config.fex:165: invalid character at 4.`错误。

保存，并使用 `fex2bin sys_config.fex script.bin` 生成script.bin文件。

先安装fex2bin

```
//安装依赖包（不安装会报错）
sudo apt-get install pkg-config pkgconf zlib1g-dev libusb-1.0-0-dev
//获取源码
git clone -b v3s https://github.com/Icenowy/sunxi-tools.git
//进入源码文件夹
cd sunxi-tools
//编译和安装
make && sudo make install
```

生成script.bin

`fex2bin sys_config.fex script.bin`

# 烧录镜像

首先烧录好u-boot，可以参考[Licheepi zero plus sd 编译启动指南 - Sipeed 开源社区](https://bbs.sipeed.com/thread/813)，然后把script.bin,uImage,boot.scr放入第一分区

使用buildroot建立文件系统，然后将生成的rootfs.tar解压到第二分区

文件情况如下

第一分区：

```
lithromantic@ubuntu:~/lichee$ cd /media/lithromantic/BOOT
lithromantic@ubuntu:/media/lithromantic/BOOT$ ls
 boot.scr   script.bin  'System Volume Information'   uImage

```

第二分区：

```
lithromantic@ubuntu:/media/lithromantic$ cd rootfs/
lithromantic@ubuntu:/media/lithromantic/rootfs$ ls
bin  dev  etc  lib  lib32  linuxrc  lost+found  media  mnt  opt  proc  root  run  sbin  sys  tmp  usr  var

```

上电启动信息：

```
U-Boot SPL 2017.01-rc2-00073-gdd6e874 (Jul 20 2021 - 01:55:52)
DRAM: 64 MiB
Trying to boot from MMC1

U-Boot 2017.01-rc2-00073-gdd6e874 (Jul 20 2021 - 01:55:52 +0000) Allwinner Technology

CPU:   Allwinner V3s (SUN8I 1681)
Model: Lichee Pi Zero
DRAM:  64 MiB
MMC:   SUNXI SD/MMC: 0
SF: unrecognized JEDEC id bytes: 00, 00, 00
*** Warning - spi_flash_probe() failed, using default environment

In:    serial@01c28000
Out:   serial@01c28000
Err:   serial@01c28000


U-Boot 2017.01-rc2-00073-gdd6e874 (Jul 20 2021 - 01:55:52 +0000) Allwinner Technology

CPU:   Allwinner V3s (SUN8I 1681)
Model: Lichee Pi Zero
DRAM:  64 MiB
MMC:   SUNXI SD/MMC: 0
SF: unrecognized JEDEC id bytes: 00, 00, 00
*** Warning - spi_flash_probe() failed, using default environment

In:    serial@01c28000
Out:   serial@01c28000
Err:   serial@01c28000
Net:   No ethernet found.
starting USB...
No controllers found
Hit any key to stop autoboot:  2  1  0 
switch to partitions #0, OK
mmc0 is current device
Scanning mmc 0:1...
Found U-Boot script /boot.scr
reading /boot.scr
290 bytes read in 15 ms (18.6 KiB/s)
## Executing script at 41900000
reading uImage
2641640 bytes read in 170 ms (14.8 MiB/s)
reading script.bin
32468 bytes read in 26 ms (1.2 MiB/s)
## Booting kernel from Legacy Image at 41000000 ...
   Image Name:   Linux-3.4.39
   Image Type:   ARM Linux Kernel Image (uncompressed)
   Data Size:    2641576 Bytes = 2.5 MiB
   Load Address: 40008000
   Entry Point:  40008000
   Verifying Checksum ... OK
   Loading Kernel Image ... OK
Using machid 0x1029 from environment

Starting kernel ...

[    0.000000] Booting Linux on physical CPU 0
[    0.000000] Linux version 3.4.39 (root@0e71897a2d15) (gcc version 4.6.3 20120201 (prerelease) (crosstool-NG linaro-1.13.1-2012.02-20120222 - Linaro GCC 2012.02) ) #7 Thu Jul 22 09:41:43 UTC 2021
[    0.000000] Initialized persistent memory from 41d20800-41d307ff
[    0.000000] Kernel command line: console=ttyS0,115200 panic=5 rootwait root=/dev/mmcblk0p2 earlyprintk rw
[    0.000000] PID hash table entries: 256 (order: -2, 1024 bytes)
[    0.000000] Dentry cache hash table entries: 8192 (order: 3, 32768 bytes)
[    0.000000] Inode-cache hash table entries: 4096 (order: 2, 16384 bytes)
[    0.000000] Memory: 64MB = 64MB total
[    0.000000] Memory: 30016k/30016k available, 35520k reserved, 0K highmem
[    0.000000] Virtual kernel memory layout:
[    0.000000]     vector  : 0xffff0000 - 0xffff1000   (   4 kB)
[    0.000000]     fixmap  : 0xfff00000 - 0xfffe0000   ( 896 kB)
[    0.000000]     vmalloc : 0xc4800000 - 0xff000000   ( 936 MB)
[    0.000000]     lowmem  : 0xc0000000 - 0xc4000000   (  64 MB)
[    0.000000]     modules : 0xbf000000 - 0xc0000000   (  16 MB)
[    0.000000]       .text : 0xc0008000 - 0xc04d8000   (4928 kB)
[    0.000000]       .init : 0xc04d8000 - 0xc04fb000   ( 140 kB)
[    0.000000]       .data : 0xc04fc000 - 0xc05411a0   ( 277 kB)
[    0.000000]        .bss : 0xc05411c4 - 0xc05dc024   ( 620 kB)
[    0.000000] NR_IRQS:544
[    0.000000] Architected local timer running at 24.00MHz.
[    0.000000] Switching to timer-based delay loop
[    0.000000] sched_clock: 32 bits at 24MHz, resolution 41ns, wraps every 178956ms
[    0.000000] Console: colour dummy device 80x30
[    0.000154] Calibrating delay loop (skipped), value calculated using timer frequency.. 4800.00 BogoMIPS (lpj=24000000)
[    0.000177] pid_max: default: 32768 minimum: 301
[    0.000311] Mount-cache hash table entries: 512
[    0.000850] CPU: Testing write buffer coherency: ok
[    0.001116] Setting up static identity map for 0x403a4278 - 0x403a42d0
[    0.001762] devtmpfs: initialized
[    0.003396] pinctrl core: initialized pinctrl subsystem
[    0.003878] NET: Registered protocol family 16
[    0.004179] DMA: preallocated 128 KiB pool for atomic coherent allocations
[    0.004239] script_sysfs_init success
[    0.004997] gpiochip_add: registered GPIOs 0 to 223 on device: sunxi-pinctrl
[    0.005888] sunxi-pinctrl sunxi-pinctrl: initialized sunXi PIO driver
[    0.006258] gpiochip_add: registered GPIOs 1024 to 1031 on device: axp-pinctrl
[    0.007282] persistent_ram: uncorrectable error in header
[    0.007300] persistent_ram: no valid data in buffer (sig = 0x55503537)
[    0.011230] console [ram-1] enabled
[    0.012111] Not Found clk pll_isp in script 
[    0.012224] Not Found clk pll_video in script 
[    0.012328] Not Found clk pll_ve in script 
[    0.012517] Not Found clk pll_periph0 in script 
[    0.012620] Not Found clk pll_de in script 
[    0.016545] bio: create slab <bio-0> at 0
[    0.017001] pwm module init!
[    0.019159] SCSI subsystem initialized
[    0.019503] usbcore: registered new interface driver usbfs
[    0.019757] usbcore: registered new interface driver hub
[    0.019993] usbcore: registered new device driver usb
[    0.020149] twi_chan_cfg()340 - [twi0] has no twi_regulator.
[    0.020261] twi_chan_cfg()340 - [twi1] has no twi_regulator.
[    0.021142] sunxi_i2c_do_xfer()985 - [i2c0] incomplete xfer (status: 0x20, dev addr: 0x34)
[    0.021267] axp20_board 0-0034: failed reading at 0x03
[    0.021488] axp20_board: probe of 0-0034 failed with error -70
[    0.021639] Linux video capture interface: v2.00
[    0.021929] gpiochip_add: gpios 1024..1028 (axp_pin) failed to register
[    0.022358] Advanced Linux Sound Architecture Driver Version 1.0.25.
[    0.023430] cfg80211: Calling CRDA to update world regulatory domain
[    0.024488] Switching to clocksource arch_sys_counter
[    0.029970] NET: Registered protocol family 2
[    0.029970] IP route cache hash table entries: 1024 (order: 0, 4096 bytes)
[    0.029970] TCP established hash table entries: 2048 (order: 2, 16384 bytes)
[    0.029974] TCP bind hash table entries: 2048 (order: 1, 8192 bytes)
[    0.030148] TCP: Hash tables configured (established 2048 bind 2048)
[    0.030343] TCP: reno registered
[    0.030452] UDP hash table entries: 256 (order: 0, 4096 bytes)
<6[    0.031019] NET: Registered protocol family 1
[    0.031452] standby_mode = 1. 
[    0.031644] wakeup src cnt is : 3. 
[    0.031766] pmu1_enable = 0x1. 
[    0.031869] pmux_id = 0x1. 
[    0.032064] config_pmux_para: script_parser_fetch err. 
[    0.032168] pmu2_enable = 0x0. 
[    0.032272] add_sys_pwr_dm: get ldo name failed
[    0.032459] add_sys_pwr_dm: get ldo name failed
[    0.032562] add_sys_pwr_dm: get ldo name failed
[    0.032667] add_sys_pwr_dm: get ldo name failed
[    0.032769] add_sys_pwr_dm: get ldo name failed
[    0.032873] add_sys_pwr_dm: get ldo name failed
[    0.033062] add_sys_pwr_dm: get ldo name failed
[    0.033166] add_sys_pwr_dm: get ldo name failed
[    0.033269] add_sys_pwr_dm: get ldo name failed
[    0.033456] add_sys_pwr_dm: get ldo name failed
[    0.033557] after inited: sys_mask config = 0x0. 
[    0.033746] dynamic_standby enalbe = 0x0. 
[    0.033901] sunxi_reg_init enter
[    0.035925] squashfs: version 4.0 (2009/01/31) Phillip Lougher
[    0.036166] jffs2: version 2.2. (NAND) (SUMMARY)  漏 2001-2006 Red Hat, Inc.
[    0.036454] msgmni has been set to 58
[    0.037591] io scheduler noop registered
[    0.037705] io scheduler deadline registered
[    0.037975] io scheduler cfq registered (default)
[    0.038478] [DISP]disp_module_init
[    0.038953] cmdline,disp=
[    0.039610] [DISP] disp_get_rotation_sw,line:68:disp 0 out of range? g_rot_sw=0
[    0.039902] [DISP] disp_init_connections,line:289:NULL pointer: 0, 0
[    0.041866] [DISP] Fb_map_kernel_logo,line:924:Fb_map_kernel_logo failed!
[    0.044355] [DISP] disp_sys_power_enable,line:387:some error happen, fail to get regulator 
[    0.045439] [DISP]disp_module_init finish
[    0.045842] sw_uart_get_devinfo()1503 - uart0 has no uart_regulator.
[    0.046378] uart0: ttyS0 at MMIO 0x1c28000 (irq = 32) is a SUNXI
[    0.046496] sw_uart_pm()890 - uart0 clk is already enable
[    0.046699] sw_console_setup()1233 - console setup baud 115200 parity n bits 8, flow n
[    0.160333] console [ttyS0] enabled
[    0.687208] sunxi_spi_chan_cfg()1376 - [spi-0] has no spi_regulator.
[    0.695150] spi spi0: master is unqueued, this is deprecated
[    0.701832] m25p_probe()982 - Use the Dual Mode Read.
[    0.707639] m25p80 spi0.0: found m25p05-nonjedec, expected at25df641
[    0.714908] m25p80 spi0.0: m25p05-nonjedec (64 Kbytes)
[    0.722206] partitions_register()865 - m25p80_read() ret 0, PartCnt: 0
[    0.729645] m25p80: probe of spi0.0 failed with error -22
[    0.737191] Failed to alloc md5
[    0.740869] eth0: Use random mac address
[    0.745464] ehci_hcd: USB 2.0 'Enhanced' Host Controller (EHCI) Driver
[    0.773068] sunxi-ehci sunxi-ehci.1: SW USB2.0 'Enhanced' Host Controller (EHCI) Driver
[    0.782281] sunxi-ehci sunxi-ehci.1: new USB bus registered, assigned bus number 1
[    0.791000] sunxi-ehci sunxi-ehci.1: irq 104, io mem 0xf1c1a000
[    0.810035] sunxi-ehci sunxi-ehci.1: USB 0.0 started, EHCI 1.00
[    0.817474] hub 1-0:1.0: USB hub found
[    0.821794] hub 1-0:1.0: 1 port detected
[    0.826684] sunxi-ehci sunxi-ehci.1: remove, state 1
[    0.832423] usb usb1: USB disconnect, device number 1
[    0.838825] sunxi-ehci sunxi-ehci.1: USB bus 1 deregistered
[    0.855344] ohci_hcd: USB 1.1 'Open' Host Controller (OHCI) Driver
[    0.882419] sunxi-ohci sunxi-ohci.1: SW USB2.0 'Open' Host Controller (OHCI) Driver
[    0.891142] sunxi-ohci sunxi-ohci.1: new USB bus registered, assigned bus number 1
[    0.899771] sunxi-ohci sunxi-ohci.1: irq 105, io mem 0xf1c1a400
[    0.964632] hub 1-0:1.0: USB hub found
[    0.968913] hub 1-0:1.0: 1 port detected
[    0.973837] sunxi-ohci sunxi-ohci.1: remove, state 1
[    0.979481] usb usb1: USB disconnect, device number 1
[    0.985853] sunxi-ohci sunxi-ohci.1: USB bus 1 deregistered
[    1.002270] Initializing USB Mass Storage driver...
[    1.007945] usbcore: registered new interface driver usb-storage
[    1.014724] USB Mass Storage support registered.
[    1.020330] file system registered
[    1.025657] android_usb gadget: Mass Storage Function, version: 2009/09/11
[    1.033443] android_usb gadget: Number of LUNs=1
[    1.038767]  lun0: LUN: removable file: (no medium)
[    1.044890] android_usb gadget: android_usb ready
[    1.050461] sunxikbd_script_init: key para not found, used default para. 
[    1.059067] sunxi-rtc sunxi-rtc: rtc core: registered sunxi-rtc as rtc0
[    1.067590] platform reg-20-cs-dcdc2: Driver reg-20-cs-dcdc2 requests probe deferral
[    1.076562] platform reg-20-cs-dcdc3: Driver reg-20-cs-dcdc3 requests probe deferral
[    1.085408] platform reg-20-cs-ldo1: Driver reg-20-cs-ldo1 requests probe deferral
[    1.094112] platform reg-20-cs-ldo2: Driver reg-20-cs-ldo2 requests probe deferral
[    1.102814] platform reg-20-cs-ldo3: Driver reg-20-cs-ldo3 requests probe deferral
[    1.111443] platform reg-20-cs-ldo4: Driver reg-20-cs-ldo4 requests probe deferral
[    1.120159] platform reg-20-cs-ldoio0: Driver reg-20-cs-ldoio0 requests probe deferral
[    1.129132] sunxi_wdt_init_module: sunxi WatchDog Timer Driver v1.0
[    1.136376] sunxi_wdt_probe: devm_ioremap return wdt_reg 0xf1c20ca0, res->start 0x01c20ca0, res->end 0x01c20cbf
[    1.147769] sunxi_wdt_probe: initialized (g_timeout=16s, g_nowayout=0)
[    1.155509] wdt_enable, write reg 0xf1c20cb8 val 0x00000000
[    1.161810] wdt_set_tmout, write 0x000000b0 to mode reg 0xf1c20cb8, actual timeout 16 sec
[    1.176694] sunxi_leds_fetch_sysconfig_para leds is not used in config
[    1.184174] =========script_get_err============
[    1.189495] usbcore: registered new interface driver usbhid
[    1.195827] usbhid: USB HID core driver
[    1.201108] ashmem: initialized
[    1.204810] logger: created 256K log 'log_main'
[    1.210117] logger: created 32K log 'log_events'
[    1.215538] logger: created 32K log 'log_radio'
[    1.220809] logger: created 32K log 'log_system'
[    1.231756] *******************Try sdio*******************
[    1.238315] asoc: sndcodec <-> sunxi-codec mapping ok
[    1.246332] TCP: cubic registered
[    1.250196] *******************Try sd *******************
[    1.256324] NET: Registered protocol family 17
[    1.261604] VFP support v0.3: implementor 41 architecture 2 part 30 variant 7 rev 5
[    1.270342] ThumbEE CPU extension supported.
[    1.276599] Registering SWP/SWPB emulation handler
[    1.283099] platform reg-20-cs-ldoio0: Driver reg-20-cs-ldoio0 requests probe deferral
[    1.294402] platform reg-20-cs-ldo4: Driver reg-20-cs-ldo4 requests probe deferral
[    1.303116] platform reg-20-cs-ldo3: Driver reg-20-cs-ldo3 requests probe deferral
[    1.311690] platform reg-20-cs-ldo2: Driver reg-20-cs-ldo2 requests probe deferral
[    1.320344] platform reg-20-cs-ldo1: Driver reg-20-cs-ldo1 requests probe deferral
[    1.328970] platform reg-20-cs-dcdc3: Driver reg-20-cs-dcdc3 requests probe deferral
[    1.337700] platform reg-20-cs-dcdc2: Driver reg-20-cs-dcdc2 requests probe deferral
[    1.346600] sunxi-rtc sunxi-rtc: setting system clock to 1970-01-01 00:00:04 UTC (4)
[    1.356801] ALSA device list:
[    1.360281]   #0: audiocodec
[    1.363947] Waiting for root device /dev/mmcblk0p2...
[    1.379050] mmc0: new high speed SD card at address e624
[    1.385499] mmcblk0: mmc0:e624 SU02G 1.84 GiB 
[    1.392377]  mmcblk0: p1 p2
[    1.396413] mmcblk mmc0:e624: Card claimed for testing.
[    1.402371] mmc0:e624: SU02G 1.84 GiB 
[    1.406763] platform reg-20-cs-dcdc2: Driver reg-20-cs-dcdc2 requests probe deferral
[    1.415592] *******************sd init ok*******************
[    1.422152] platform reg-20-cs-dcdc3: Driver reg-20-cs-dcdc3 requests probe deferral
[    1.430934] platform reg-20-cs-ldo1: Driver reg-20-cs-ldo1 requests probe deferral
[    1.440858] platform reg-20-cs-ldo2: Driver reg-20-cs-ldo2 requests probe deferral
[    1.449474] platform reg-20-cs-ldo3: Driver reg-20-cs-ldo3 requests probe deferral
[    1.458031] platform reg-20-cs-ldo4: Driver reg-20-cs-ldo4 requests probe deferral
[    1.466855] platform reg-20-cs-ldoio0: Driver reg-20-cs-ldoio0 requests probe deferral
[    1.486178] *******************Try sdio*******************
[    1.495673] *******************Try sd *******************
[    1.505158] *******************Try mmc*******************
[    1.557072] *******************Try sdio*******************
[    1.567720] *******************Try sd *******************
[    1.578171] *******************Try mmc*******************
[    1.592501] EXT4-fs (mmcblk0p2): couldn't mount as ext3 due to feature incompatibilities
[    1.602427] EXT4-fs (mmcblk0p2): couldn't mount as ext2 due to feature incompatibilities
[    1.639198] *******************Try sdio*******************
[    1.652041] *******************Try sd *******************
[    1.664636] *******************Try mmc*******************
[    1.717141] *******************Try sdio*******************
[    1.724299] EXT4-fs (mmcblk0p2): recovery complete
[    1.731380] EXT4-fs (mmcblk0p2): mounted filesystem with ordered data mode. Opts: (null)
[    1.740650] VFS: Mounted root (ext4 filesystem) on device 179:2.
[    1.747541] *******************Try sd *******************
[    1.758036] *******************Try mmc*******************
[    1.764410] devtmpfs: mounted
[    1.767985] Freeing init memory: 140K
[    1.899438] EXT4-fs (mmcblk0p2): re-mounted. Opts: data=ordered
Starting logging: OK
Initializing random number generator... done.
Starting network: OK


Welcome to LicheePi

licheepi login: root
Password: 
# ls /
bin         lib         lost+found  opt         run         tmp
dev         lib32       media       proc        sbin        usr
etc         linuxrc     mnt         root        sys         var
# 

```

