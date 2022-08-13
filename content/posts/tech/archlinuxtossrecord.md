---
layout: "posts"
title: R9000P 2021版的ArchLinux踩坑记录（持续更新）
subtitle:   ""
description: ""
author: ChaosAttractor
date: 2022-08-07
publishdate: 2022-08-07
lastmod: 2022-08-08
image: "img/posts/archlinuxtossrecord/gnome-desktop.avif"
categories: [ "Tech" ]
tags:
    - 技术
    - 折腾记录
    - ArchLinux
URL: "/tech/archlinuxtossrecord/"
---

**本文使用R9000P 2021作为硬件环境，若同为此机器或同为AMD集显+NVIDIA独显或同为联想笔记本可能才会比较有参考价值，请自行判断内容是否适用**

**本文最后修改于2022年8月8日，由于arch是滚动更新的且上游软件包同步极快，请自行判断内容是否过时**

## 折腾的开始

由于Win11的各种摆烂以及高的吓人的资源占用，而且在pd供电的情况下卡的吓人，于是总算下定决心重新用回Arch

那既然要用Linux桌面，第一个问题肯定就是选什么桌面环境，我曾经用过KDE Plasma、Gnome 40，i3等的桌面环境

我并不是非常喜欢平铺式的窗口管理器，而且我是一个重度触控板用户，苹果的magic trackpad已经成为了日常标配，目前在Linux下能有优秀的触控板体验的只有Wayland下的Gnome 40+，以及加上KDE对Wayland的支持一塌糊涂和GTK4+libadwita真的非常好看，于是就选择了Gnome作为我的桌面环境

## 显卡驱动

由于我使用的R9000P是一款AMD核显+NVIDIA独显且可以开启独显直连的笔记本，再加上网上Wayland相关的文献较少，于是显卡驱动便成了一个很头疼的问题

### 仅启用NVIDIA独显（独显直连） 

首先自然是安装NVIDIA的驱动，NVIDIA的闭源驱动分为三个包，分别是[nvidia](https://archlinux.org/packages/?name=nvidia)（或如需使用其他内核请安装[nvidia-dkms](https://archlinux.org/packages/?name=nvidia-dkms)）、[nvidia-utils](https://archlinux.org/packages/?name=nvidia-utils)、[nvidia-settings](https://archlinux.org/packages/?name=nvidia-settings)（图形化配置界面，可选，推荐安装）和用于32位程序的[lib32-nvidia-utils](https://archlinux.org/packages/?name=lib32-nvidia-utils)（可选）

首先是在仅使用NV独显的情况下使用Wayland，刚刚装完NVIDIA驱动并启动GDM你会发现登录选项里只有"Gnome"和"Gnome经典模式"，没有"Gnome on Xorg"和"Gnome on Xorg经典模式"，因为GDM的规则文件理论上应该默认启用Wayland，这说明有哪里配置出了问题使得GDM禁用了Wayland

查阅/usr/lib/udev/rules.d/61-gdm.rules后发现，里面有这样一段内容
```sh
# Check if suspend/resume services necessary for working wayland support is available
TEST{0711}!="/usr/bin/nvidia-sleep.sh", GOTO="gdm_disable_wayland"
TEST{0711}!="/usr/lib/systemd/system-sleep/nvidia", GOTO="gdm_disable_wayland"
IMPORT{program}="/bin/sh -c \"sed -e 's/: /=/g' -e 's/\([^[:upper:]]\)\([[:upper:]]\)/\1_\2/g' -e 's/[[:lower:]]/\U&/g' -e 's/^/NVIDIA_/' /proc/driver/nvidia/params\""
ENV{NVIDIA_PRESERVE_VIDEO_MEMORY_ALLOCATIONS}!="1", GOTO="gdm_disable_wayland"
IMPORT{program}="/bin/sh -c 'echo NVIDIA_HIBERNATE=`systemctl is-enabled nvidia-hibernate`'"
ENV{NVIDIA_HIBERNATE}!="enabled", GOTO="gdm_disable_wayland"
IMPORT{program}="/bin/sh -c 'echo NVIDIA_RESUME=`systemctl is-enabled nvidia-resume`'"
ENV{NVIDIA_RESUME}!="enabled", GOTO="gdm_disable_wayland"
IMPORT{program}="/bin/sh -c 'echo NVIDIA_SUSPEND=`systemctl is-enabled nvidia-suspend`'"
ENV{NVIDIA_SUSPEND}!="enabled", GOTO="gdm_disable_wayland"
LABEL="gdm_nvidia_end"
```
这下破案了，GDM检测到NVIDIA的睡眠、休眠和唤醒服务没有默认打开，于是就自动禁用Wayland了

首先使用systemctl打开这些服务
```sh
$ systemctl enable nvidia-hibernate
$ systemctl enable nvidia-resume
$ systemctl enable nvidia-suspend
```
然后在给nvidia的内核模块添加一个参数，在/etc/modprobe.d/nvidia.conf中加入：
```sh
# /etc/modprobe.d/nvidia.conf
options nvidia "NVreg_PreserveVideoMemoryAllocations=1"
```
重启，然后你就会发现GDM的登录选项里有"Gnome on Xorg"的选项了，说明Wayland启用成功了

NVIDIA的最新专有驱动已经提供了较为完善的Wayland支持，包括XWayland等

### 使用AMD核显和NVIDIA独显（即混合模式/Hybrid）

首先自然是安装AMD的驱动，AMD的驱动只需要安装[mesa](https://archlinux.org/packages/?name=mesa)，以及如果需要32位支持还需安装[lib32-mesa](https://archlinux.org/packages/?name=lib32-mesa)

然后就是GDM的默认规则不知道为何在混合模式+NV专有驱动下会莫名其妙的默认禁用Wayland，在/usr/lib/udev/rules.d/61-gdm.rules里有这样一段
```sh
# If this is a hybrid graphics laptop with vendor nvidia driver, disable wayland
LABEL="gdm_hybrid_nvidia_laptop_check"
TEST!="/run/udev/gdm-machine-is-laptop", GOTO="gdm_hybrid_nvidia_laptop_check_end"
TEST!="/run/udev/gdm-machine-has-hybrid-graphics", GOTO="gdm_hybrid_nvidia_laptop_check_end"
TEST!="/run/udev/gdm-machine-has-vendor-nvidia-driver", GOTO="gdm_hybrid_nvidia_laptop_check_end"
GOTO="gdm_disable_wayland"
LABEL="gdm_hybrid_nvidia_laptop_check_end"
```
这里我把「GOTO="gdm_disable_wayland"」给注释掉了（前面加个#号）来禁止GDM在混合模式自动关闭Wayland

然后重启一下你就会发现可以正常进入Wayland了

#### 在混合模式下无法准确获取EDID文件（R9000P）

使用Wayland进入桌面后，首先就会发现：“诶？我这165hz的屏咋变成60hz了？”，然后打开设置一看，显示屏幕只支持60hz  

在很久之前我初次尝试Wayland的时候我很自然的觉得这是驱动问题，直到我用混合模式+Xorg进了一次桌面，发现也还是这样  

这说明，在混合模式下，R9000P的显示屏没有返回给系统正确的EDID信息，然后我很自然的就去网上搜索了“Xorg怎么强制修改显示器刷新率”，结果网上的方式改完，直接屏幕闪成鬼（

这时我注意到，如果在独显直连模式下，即系统使用正确的EDID文件的情况下，设置里显示的刷新率是165.002HZ，说明这块屏幕不是标准165hz的行刷新率

在Xorg下解决这个问题很简单，先使用独显直连模式进入系统，这时EDID是正确的，然后使用xrandr --verbose获取显示器的详细信息，并在里面找到显示器所使用的Modeline，例如
```sh
'Modeline "2560x1600"  777.41  2560 2608 2640 2720  1600 1603 1609 1732 -hsync -vsync'
```
然后根据[ArchWiki](https://wiki.archlinux.org/title/Xrandr_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)#%E6%B7%BB%E5%8A%A0%E6%9C%AA%E8%A2%AB%E6%A3%80%E6%B5%8B%E5%88%B0%E7%9A%84%E6%9C%89%E6%95%88%E5%88%86%E8%BE%A8%E7%8E%87)提供的方法设置就行了

但是在Wayland下这个问题就没这么简单了，因为Wayland下只能去使用提供EDID文件，并不能直接去设置Modeline

起初我根据ArchWiki指路的方法使用[edid-generator](https://github.com/akatrevorjay/edid-generator)尝试根据之前的Modeline自己生成一个EDID文件然后使用，但是指定使用后log里一直莫名其妙的报错

解决方法也很简单，直接重启到独显直连模式下然后把使用的EDID文件给截取出来然后在指定显示器用就好了，首先安装好[read-edid](https://archlinux.org/packages/?name=read-edid)这个包，然后使用如下命令提取EDID文件
```sh
# -m是指定显示器编号
# 把edid_filename.bin替换成你edid文件的文件名
$ sudo get-edid -m 0 > 把edid_filename.bin
```
然后在/usr/lib/firmware下创建一个名为edid的文件夹然后放进去
```sh
$ sudo mkdir /usr/lib/firmware/edid/
# 把edid_filename.bin替换成你edid文件的文件名
$ sudo cp ./edid_filename.bin /usr/lib/firmware/edid/
```
最后在你的内核参数中添加
```sh
# 把edid_filename.bin替换成你edid文件的文件名，eDP-1替换为你的显示器接口
drm.edid_firmware=eDP-1:edid/把edid_filename.bin
```
然后重启，在去设置里把显示器刷新率调到165hz就好啦

#### 自动断电（PCI-Express Runtime D3 (RTD3) Power Management）

Hybrid的很重要的一点是要能够自动让独显能够自动在空载时断电，这样可以节省非常多的显卡核外功耗，目前的主流解决方案对Wayland的支持并不友好，如BBSwitch等，而且Optimus Manager对Wayland也没有支持，但是NVIDIA在20系以上的显卡+Intel Coffee Lake以上的芯片组的平台上支持了一个新技术，可以非常方便的实现此功能，部分AMD的处理器（如R900P的5800H）也可以支持

**需要注意的是，此功能和TLP似乎并不兼容，一起使用时会出现无法理解的莫名其妙的问题导致无法启用，也无法正常给显卡断电**

要查看是否支持以及是否开启可以使用这条命令获取电源管理信息来查看
```sh
$ cat /proc/driver/nvidia/gpus/0000:01:00.0/power
```
然后输出的内容中如果输出为「Disabled (not support)」即为不支持，如果成功打开会这样显示
```sh
Runtime D3 status:          Enabled (fine-grained)
```
「fine-grained」是模式，代表精细控制，具体区别我也不是很了解，但是官方文档演示用的就是这个模式

这个功能在20系显卡上默认是关闭的，但到了30系显卡默认开启了，要手动开启这个功能需要给nvidia内核模块添加参数，也非常简单，只需在/etc/modprobe.d/nvidia.conf中加入
```sh
# /etc/modprobe.d/nvidia.conf
options nvidia "NVreg_DynamicPowerManagement=0x02"
```
这里0x02是启用精细控制，0x00代表关闭，0x01代表启用粗率控制，0x03代表自动（默认值），即在20系显卡上为关闭，30系为打开  

启用此功能后，还需将显卡的电源控制策略从on切换到auto，由于NV的GPU还会集成声卡等多个设备，在曾经较老的内核上需要禁用除了显卡本身之外的其他设备，不然会出问题，在现在驱动逐渐完善后已经可以全部设置为auto了

在笔记本上一般只有GPU本身和一个集成音频控制器，它们的电源策略模式可以在这里用cat或者echo命令直接获取/更改：
```sh
/sys/bus/pci/devices/0000\:01\:00.0/power/control
/sys/bus/pci/devices/0000\:01\:00.1/power/control
```

要自动设置的话，只需添加一个udev规则即可，比如/lib/udev/rules.d/80-nvidia-pm.rules中加入
```sh
# /lib/udev/rules.d/80-nvidia-pm.rules

# Enable runtime PM for NVIDIA VGA/3D controller devices on driver bind
ACTION=="bind", SUBSYSTEM=="pci", ATTR{vendor}=="0x10de", ATTR{class}=="0x030000", TEST=="power/control", ATTR{power/control}="auto"
ACTION=="bind", SUBSYSTEM=="pci", ATTR{vendor}=="0x10de", ATTR{class}=="0x030200", TEST=="power/control", ATTR{power/control}="auto"
ACTION=="bind", SUBSYSTEM=="pci", ATTR{vendor}=="0x10de", ATTR{class}=="0x040300", TEST=="power/control", ATTR{power/control}="auto"

# Disable runtime PM for NVIDIA VGA/3D controller devices on driver unbind
ACTION=="unbind", SUBSYSTEM=="pci", ATTR{vendor}=="0x10de", ATTR{class}=="0x030000", TEST=="power/control", ATTR{power/control}="on"
ACTION=="unbind", SUBSYSTEM=="pci", ATTR{vendor}=="0x10de", ATTR{class}=="0x030200", TEST=="power/control", ATTR{power/control}="on"
ACTION=="unbind", SUBSYSTEM=="pci", ATTR{vendor}=="0x10de", ATTR{class}=="0x040300", TEST=="power/control", ATTR{power/control}="on"
```
这样在nvidia设备/驱动加载的时候就应该会自动更改了

重启之后，用这两条命令检测刚刚设置的udev规则是否正常生效
```sh
$ cat /sys/bus/pci/devices/0000\:01\:00.0/power/control
$ cat /sys/bus/pci/devices/0000\:01\:00.1/power/control
```
如果输出为auto的话，就说明规则生效了、**如果使用了TLP，会导致刚刚的udev规则莫名其妙的无法生效，即使在关闭所有PCI相关功能后也是如此**

规则生效后，还是之前获取获取电源管理信息的命令，如果成功的话应该可以看到
```sh
# cat /proc/driver/nvidia/gpus/0000:01:00.0/power
Runtime D3 status:          Enabled (fine-grained)
Video Memory:               Off

GPU Hardware Support:
 Video Memory Self Refresh: Supported
 Video Memory Off:          Supported

Power Limits:
 Default:                   N/A milliwatts
 GPU Boost:                 N/A milliwatts
```
如果Video Memory是Off的话，就说明成功了，可以看到此时显卡的功耗墙也变为了N/A milliwatts

如果不是，可以看下nvidia-smi
```sh
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 515.65.01    Driver Version: 515.65.01    CUDA Version: 11.7     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  NVIDIA GeForce ...  Off  | 00000000:01:00.0 Off |                  N/A |
| N/A   37C    P0    N/A /  N/A |      4MiB /  8192MiB |      0%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+
                                                                               
+-----------------------------------------------------------------------------+
| Processes:                                                                  |
|  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
|        ID   ID                                                   Usage      |
|=============================================================================|
|    0   N/A  N/A     42832      G   /usr/bin/gnome-shell                2MiB |
+-----------------------------------------------------------------------------+
```
如果有除了gnome-shell之外的其他进程的话，把进程关掉后稍微等个几秒重新运行前面的命令查看状态，应该就能成功了

设置成功可以使用如下命令获取所有显卡的状态
```sh
$ cat /sys/class/drm/card*/device/power_state
```
输出设备可能有多个（核显和独显），如果中有一个为D3cold即说明显卡成功断电

<!-- TODO:全局环境变量全部使用NVIDIA渲染来实现NVIDIA Only -->
#### 在混合模式下只使用NVIDIA显卡
> ⚠️ 注意  
> optimus-manager 官方文档中不推荐将此环境变量添加到全局环境。  
> > It is not recommended to set those variables system-wide (like in /etc/environment), since it would defeat the point of hybrid mode, and can also break desktop compositing on some environment (resulting in a black screen).
>
> 也就是说会遇到这些问题：  
> - Kwin / 显示特效混合器（混成器）奔溃（毛玻璃效果消失，原本是透明或毛玻璃效果的背景变黑）  
> - 黑屏

可以通过指定全局变量的方法来实现，不需要时删除即可
```sh
# /etc/environment
__NV_PRIME_RENDER_OFFLOAD=1
__GLX_VENDOR_LIBRARY_NAME="nvidia"
__VK_LAYER_NV_optimus="NVIDIA_only"
```

至此，终于可以在Wayland下愉快的使用了

<!-- TODO:硬件解码（VAAPI） --> 

## 开机无法使用蓝牙

不知道为什么，我开机后电脑的蓝牙一定是默认关闭的，bluetooth.service看了下也是正常enable的，即使参照[ArchWiki](https://wiki.archlinux.org/title/Bluetooth_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)#%E5%BC%80%E6%9C%BA%E5%90%8E%E8%87%AA%E5%8A%A8%E5%90%AF%E5%8A%A8)开启了自动打开，也仍然无法开机自动打开蓝牙，非常的奇怪

直到有一天我开机后看了一眼rfkill
```sh
# rfkill list
0: ideapad_wlan: Wireless LAN
	Soft blocked: no
	Hard blocked: no
1: phy0: Wireless LAN
	Soft blocked: no
	Hard blocked: no
2: ideapad_bluetooth: Bluetooth
	Soft blocked: yes
	Hard blocked: no
3: hci0: Bluetooth
	Soft blocked: yes
	Hard blocked: no
```
然后我就懵了，我用电脑的飞行模式键(Fn+F8)打开了飞行模式又关掉后，又重新运行了一遍rfkill：
```sh
# rfkill list
0: ideapad_wlan: Wireless LAN
	Soft blocked: no
	Hard blocked: no
1: phy0: Wireless LAN
	Soft blocked: no
	Hard blocked: no
2: ideapad_bluetooth: Bluetooth
	Soft blocked: no
	Hard blocked: no
3: hci0: Bluetooth
	Soft blocked: no
	Hard blocked: no
```
蛤？这说明了两件事，首先这个飞行模式键肯定有专门的驱动来适配并且因为我没装过所以肯定并入内核了，然后这个驱动莫名其妙的在开机时默认block了蓝牙

根据ideapad_xxx这个前缀，我一番查询后找到了[ideapad_laptop](https://github.com/torvalds/linux/blob/master/drivers/platform/x86/ideapad-laptop.c)这个驱动模块，查询了源码后发现了这样两段代码
```c
static bool no_bt_rfkill;
module_param(no_bt_rfkill, bool, 0444);
MODULE_PARM_DESC(no_bt_rfkill, "No rfkill for bluetooth.");
```
```c
static int ideapad_register_rfkill(struct ideapad_private *priv, int dev)
{
	unsigned long rf_enabled;
	int err;

	if (no_bt_rfkill && ideapad_rfk_data[dev].type == RFKILL_TYPE_BLUETOOTH) {
		/* Force to enable bluetooth when no_bt_rfkill=1 */
		write_ec_cmd(priv->adev->handle, ideapad_rfk_data[dev].opcode, 1);
		return 0;
	}

    ...
}
```
说明这个内核模块有一个参数，可以让这个模块不去管蓝牙，于是解决这个问题就很简单了，只需要在/etc/modprobe.d/ideapad_laptop.conf加入
```sh
# /etc/modprobe.d/ideapad_laptop.conf
options ideapad_laptop "no_bt_rfkill=1"
```
重启，再使用rfkill查看状态
```sh
# rfkill list
0: ideapad_wlan: Wireless LAN
	Soft blocked: no
	Hard blocked: no
2: phy0: Wireless LAN
	Soft blocked: no
	Hard blocked: no
3: hci0: Bluetooth
	Soft blocked: no
	Hard blocked: no
```
然后就会发现蓝牙正常了