# Licheepi Zero  LCD驱动指南（BSP)

编译Licheepi zero的BSP（板级支持包）的启动镜像分步：
1：修改uboot参数。
2：配置fex文件。
3：设置多控制台调试。
4：Badapple！
5：移植littlelvgl。

# UBOOT配置

通过配置uboot选项，借助uboot自带的设备树，可以快速测试屏幕。

在使用uboot配置之前，需要先配置好uboot环境，[相关参考](https://wiki.sipeed.com/soft/Lichee/zh/Zero-Doc/System_Development/uboot_build.html)

uboot详细配置可以参考 [Uboot 配置](https://wiki.sipeed.com/soft/Lichee/zh/Zero-Doc/System_Development/uboot_conf.html)，在这里我们只关心LCD相关配置

可以手动修改如下参数：

```
ARM architecture  --->
	[*] Enable graphical uboot console on HDMI, LCD or VGA   这个就是在显示设备上使能串口控制 
    [ ] VGA via LCD controller support# 使能支持VGA通过LCD的控制器，就是LCD和VAG转换需要的控制器       
(x:800,y:480,depth:18,pclk_khz:33000,le:87,ri:40,up:31,lo:13,hs:1,vs:1,sync:3,vmode:0) LCD pane #该选项就是配置LCD的分辨率的配置选项可以看到x是800 y是480 等等一些关于LCD的配置内容，点击回车进去可以对其进行修改。                 
	(1)   LCD panel display clock phase #这个是LCD的显示时钟相位
    ()    LCD panel power enable pin #LCD的电源使能引脚
    ()    LCD panel reset pin #LCD的复位引脚
    (PB4) LCD panel backlight pwm pin#背光PWN引脚 这个应该是调节亮度的引脚PB4
    [*]   LCD panel backlight pwm is inverted#反转PWN背光引脚
    [ ]   LCD panel needs to be configured via i2c # LCD panel support (Generic parallel interface LCD panel)  --->     #这个选择支持的LCDpanel
    (X) Generic parallel interface LCD panel#这里选择支持通用的并行的LCD接口
    ( ) Generic lvds interface LCD panel#这个是LVDS接口
    ( ) MIPI 4-lane, 513Mbps LCD panel via SSD2828 bridge chip
    ( ) eDP 4-lane, 1.62G LCD panel via ANX9804 bridge chip
    ( ) Hitachi tx18d42vm LCD panel
    ( ) tl059wv5c0 LCD panel    
```

也可以直接`make LicheePi_Zero_800x480LCD_defconfig`使用预先配置好的参数

配置完毕后，若屏幕无问题，上电后LCD会显示出经典的企鹅图像~~不是QQ企鹅~~![](https://raw.githubusercontent.com/USTHzhanglu/picture/main/img/image-20210728193746646.png)

如果想换成自己的小企鹅，可以参考这篇文章： [开机logo替换 - Sipeed Wiki](https://wiki.sipeed.com/soft/Lichee/zh/Zero-Doc/System_Development/uboot_logo.html)。

# FEX文件配置

对于BSP开发，系统配置为fex文件配置，所以驱动LCD，也要对该文件进行修改。

## 1.复制FEX文件

如果之前有配置fex文件，则继续使用那份文件，否则：

复制一份 *lichee/tools/pack/chips/sun8iw8p1/configs/tiger-cdr/sys_config.fex*

```
cp /root/lichee/tools/pack/chips/sun8iw8p1/configs/tiger-cdr/sys_config.fex /root/u-boot
vim /root/u-boot/sys_config.fex
```

## 2.DISP INIT

找到disp init configuration [403,1],

对于大部分参数，我们不需要修改，原文件已经大致配置好了，我们只需要关心以下几个参数

```
;disp init configuration
;
;disp_mode             (0:screen0<screen0,fb0>)
;screenx_output_type   (0:none; 1:lcd; 3:hdmi;)
;screenx_output_type：屏幕输出类型，这里选择lcd；
;fbx format            (4:RGB655 5:RGB565 6:RGB556 7:ARGB1555 8:RGBA5551 9:RGB888 10:ARGB8888 12:ARGB4444)
;fbx format fb格式，一般情况下使用的都是ARGB8888；
;fbx pixel sequence    (0:ARGB 1:BGRA 2:ABGR 3:RGBA)
;fbx pixel sequence fb像素，和fb格式有关，选择ARGB;

```

剩下的一些屏幕亮度背光亮度等参数可以根据应用条件灵活调节。

配置完成后参数如下：

```
disp_init_enable         = 1
disp_mode                = 0

screen0_output_type      = 1
screen0_output_mode      = 4

fb0_format               = 10
fb0_pixel_sequence       = 0
fb0_scaler_mode_enable   = 0
fb0_width                = 0
fb0_height               = 0

lcd0_backlight           = 102

lcd0_bright              = 50
lcd0_contrast            = 50
lcd0_saturation          = 57
lcd0_hue                 = 50
```

## 3.lcd0 configuration

对于不同的LCD，接口，分辨率和频率很多参数都不一样，因此要针对屏幕单独配置，这里以4.3寸和5.0寸lcd屏幕为例，说明下如何配置这些参数。

```
;lcd_if:               0:hv(sync+de); 1:8080; 2:ttl; 3:lvds; 4:dsi; 5:edp; 6:extend dsi,LCD
；lcd接口方式，使用板载hv接口就选择0.
;lcd_x:                lcd horizontal resolution
;lcd水平分辨率，800/480：5/4.3寸
;lcd_y:                lcd vertical resolution
;lcd垂直分辨率，480/272：5/4.3寸
;lcd_width:            width of lcd in mm
;lcd_height:           height of lcd in mm
;fb分辨率，设置为0时等同于屏幕分辨率

```

除了以上参数，还有一些需要查datasheet的参数，如果找不到手册，亦可尝试使用相同尺寸的其他屏幕的参数。下面介绍下4.3寸和5寸的参数：

<center>5寸LCD相关参数

![image-20210729105913005](https://raw.githubusercontent.com/USTHzhanglu/picture/main/img/image-20210729105913005.png)

<center>4.3寸LCD相关参数

![image-20210729110747858](https://raw.githubusercontent.com/USTHzhanglu/picture/main/img/image-20210729110747858.png)

<center>fex参数
</center>

```
;lcd_hbp:              hsync back porch
;行同步信号后肩，符号为Thb，单位为1VCLK的时间，也称HBPD。
;lcd_ht:               hsync total cycle
;行同步信号总周期，符号为Th，单位为1VCLK的时间。
;lcd_vbp:              vsync back porch
;帧同步信号后肩，符号为Tvb，单位为1行（Line）的时间，也称VBPD
;lcd_vt:               vysnc total cycle
;帧同步信号总周期，符号为Tv，单位为1行（Line）的时间。
;lcd_hspw:             hsync plus width
;行同步信号脉宽，符号为Thw，也称HSPW。
;lcd_vspw:             vysnc plus width
;帧同步信号脉宽，符号为Tvw，也称VSPW。
```

并且有：

```
Th = Thdisp + Thbp + Thfp + Thw

Tv = Tvdisp + Tvbp + Tvfp + Tvw
```

其中Thdisp和Tvdisp是固定的（屏幕分辨率），其他的只要在min与max之间即可，因此对于一些datasheet或者fex中没有的参数，我们只需要确保Th和Tv不超过最大值即可（Th和Tv变化会导致clk变化，所以一般选择典型值）；

例如5寸屏：

```
1060 = 800 + 256 + 3 + 1（其中Thfp和Thw datasheet并没有写）
540 = 480 +45 + 5 +10（其中Tvfp和Tvw datasheet并没有写）
```

设置完成后参数如下(5寸）：

```
[lcd0_para]
lcd_used            = 1
lcd_driver_name     ="default_lcd"
lcd_if              = 0
lcd_x               = 800
lcd_y               = 480
lcd_width           = 0
lcd_height          = 0
lcd_dclk_freq       = 33
lcd_pwm_used        = 1
lcd_pwm_ch          = 0
lcd_pwm_freq        = 50000
lcd_pwm_pol         = 1
lcd_pwm_max_limit   = 255
lcd_hbp             = 256
lcd_ht              = 1060
lcd_hspw            = 1
lcd_vbp             = 45
lcd_vt              = 540
lcd_vspw            = 10
lcd_frm             = 0
lcd_hv_clk_phase    = 0
lcd_hv_sync_polarity = 0
lcd_gamma_en        = 0
lcd_bright_curve_en = 1
lcd_cmap_en         = 0
```

## 4.生成固件

生成script.bin

`fex2bin sys_config.fex script.bin`

然后放入第一分区。

此时开机后会有提示：

```
Setting up a 800x480 lcd console (overscan 0x0)
dotclock: 33000kHz = 33000kHz: (1 * 3MHz * 66) / 6
```

并且输入`cat /dev/urandom >/dev/fb0`

屏幕会被雪花填充

![IMG_20210730_164919](https://raw.githubusercontent.com/USTHzhanglu/picture/main/img/IMG_20210730_164919.jpg)

# 设置多控制台调试

有了屏幕，就自然会希望控制台出现在屏幕上，实现串口,LCD双控制。

## 1.修改u-boot.cmd文件

在console=ttyS0前加入console=tty0：

```
setenv bootargs console=tty0 console=ttyS0,115200 panic=5 rootwait root=/dev/mmcblk0p2 earlyprintk rw
setenv bootm_boot_mode sec
setenv machid 1029
load mmc 0:1 0x41000000 uImage
load mmc 0:1 0x41d00000 script.bin
bootm 0x41000000
```

`mkimage -C none -A arm -T script -d boot.cmd boot.scr`后将scr放入第一分区。

## 2.重新编译kernel

打开Framebuffer Console support后，开机将会把信息打印到屏幕上。

make meconfig

```
CONFIG_FRAMEBUFFER_CONSOLE
CONFIG_FRAMEBUFFER_CONSOLE_DETECT_PRIMARY
Location: 
     -> Device Drivers  
          -> Graphics support  
            -> Console display driver support
                <*> Framebuffer Console support  
                [*]   Map the console to the primary display device
```

## 3.修改/etc/inittab

将原有的console注释掉，改为ttyS0和tty0，ttyS0是USB键盘输入，tty0就是正常输出输入

 ```
 #console::respawn:/sbin/getty -L  console 0 vt100
 ttyS0::respawn:/sbin/getty -L ttyS0 115200 vt100
 tty0::respawn:-bin/sh   
 ```

然后重启就可以双端控制了

![QQ图片20210730172750](https://raw.githubusercontent.com/USTHzhanglu/picture/main/img/QQ%E5%9B%BE%E7%89%8720210730172750.jpg)

# Badapple!

~~有屏幕的地方就要有badapple！~~

## 1.安装播放器

使用buildroot构建，选用mplayer(17年以后的buildroot不在包含此软件)

`make meconfig`

在配置视频前，需要先配置音频。

找到alsa_utils，选中，里面的也全部选中；

 ```
 Prompt: alsa-utils 
    Location:
      -> Target packages 
  (1)   -> Audio and video applications 
  			[*] alsa-utils  --->
 ```

找到alsa_libs，选中；

```
Prompt: alsa-lib                                         
  Location:                                             
     -> Target packages                                   
       -> Libraries
 (1)     -> Audio/Sound 
 			-*- alsa-lib  --->
```

配置视频：

找到mplayer，选中；

```
Prompt: alsa-utils 
   Location:
     -> Target packages 
 (1)   -> Audio and video applications 
            [*] mplayer
            [*] Build and install mplayer
            [*] Build and install mencoder  
```

顺便可以把fbv也选中（图片）

```
Prompt: GIF support                                          	Location:
	 -> Target packages
		-> Graphic libraries and applications
        	[*] fbv
        	[*]   PNG support
        	[*]   JPEG support
        	[*]   GIF support                     
```

`make`。

然后把生成的rootfs.gz解压到rootfs分区。

## 2.上机测试

下载badapple丢到文件系统中

然后使用

```
mplayer vo=fbdev badapple.flv
```

播放视频

效果如下：

![badapple](https://raw.githubusercontent.com/USTHzhanglu/picture/main/img/badapple.gif)

# 移植littlelvgl

获取源码：

```
git clone https://github.com/lvgl/lv_port_linux_frame_buffer.git

git submodule update --init --recursive
```

修改makefile，指定CC为 arm-linux-gnueabihf-gcc等。

```
CC = arm-linux-gnueabihf-gcc -std=c99
CFLAGS = -O3 -g0 -I$(LVGL_DIR)/ -Wall -Wshadow -Wundef -Wmaybe-uninitialized
```

修改lv_conf.h,设置色彩深度16位。

```
/*====================
   COLOR SETTINGS
 *====================*/

/*Color depth: 1 (1 byte per pixel), 8 (RGB332), 16 (RGB565), 32 (ARGB8888)*/
#define LV_COLOR_DEPTH     16

```



`make`

生成的demo文件即为可执行文件，放入文件系统，使用./demo 执行

![IMG_20210802_1800](https://raw.githubusercontent.com/USTHzhanglu/picture/main/img/IMG_20210802_1800.jpg)

