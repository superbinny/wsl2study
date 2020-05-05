
# 让 WSL2 中的 Kali 使用无线网卡

*<binny@vip.163.com>*，2020-5-2

## 为什么有这个项目

&emsp;&emsp;最近在研究 WiFi 密码的最新破解方法，由于有了新的 WPA3 安全标准，因此，也产生了新的破解思路，即利用 WiFi 基站漫游功能中用于身份认证的 PMKID 加密算法，来直接破解密码，而 PMKID 的加密机制，甚至都不需要像以往那样，针对每个 ESSID 做破解字典了。看看以下破解公式，就知道用 PMKID 破解无线密码，变得轻松很多。具体原理就不在这里深入探讨了，以后准备找个时间专门讨论这个问题：

&emsp;&emsp; ***PMKID = HMAC-SHA1-128(PMK, "PMK Name" | MAC_AP | MAC_STA)***

&emsp;&emsp;以往是在虚拟机中做破解，既笨重又没有效率，现在微软推出 WSL2，刚好用来做 WiFi 密码破解的实验环境。但是，目前放出的 WSL2 没有想象中的完美，因此我才决定改造 WSL2，来达到和真机或者 VMware 虚拟机一样的效果。我喜爱 WSL2 主要原因在于，WSL2 启动太快了，只要一秒钟，可以与 Windows 完美结合，还占用最少的资源，加上可以直接与 Linux 通信和调试的 VSCode，使得编程的研究环境大大友好，感觉 Windows 和 Linux 的混合编程从来没有这么美好过。（PS. 通过迁移工具，将 WSL2 迁移到独立的虚拟文件硬盘中，走到哪里都可以不用重新安装和配置 Kali 环境了）

&emsp;&emsp;还有个想法 Mark 一下。如果用这个技术编译进安卓模拟器中，就可以用任何无线网卡，通过 WiFi 万能密钥功能获取密码了。并且还让普通手机透明对接任何大功率无线网卡。USB Over Internet 很多年前就在使用，解决了 Hyper-V 虚拟机不直接支持 USB 设备的烦恼，这次用在 WSL 上，也算是新瓶装旧水。

## 改造 Kali，安装超级工具集

&emsp;&emsp;从微软商店下载安装最新的 Kali，经过各种必要的准备，便可以开始进行内核编译和升级了。

&emsp;&emsp;升级的目的是为了打造一个强有力的 Kali，因此，无线网卡的加载是非常重要的。以下是在 WSL 中安装的 Kali 界面和工具集。具体过程就省略了，和本次技术实践的目的不一样。

![完整的 WSL2 Kali 环境](https://github.com/superbinny/wsl2study/raw/master/img/kali_full.jpg)

## WSL2 的改造思路

&emsp;&emsp;通过对 Windows 的升级，来最终获取 WSL2 的使用体验。不过也可以在 VMware 中安装最新的体验版来直接使用 WSL2 的所有特性。我注册了微软的体验者用户，然后从[Windows10 体验站](https://www.microsoft.com/en-us/software-download/windowsinsiderpreviewiso?wa=wsignin1.0)中下载最新的体验包 **Windows10_InsiderPreview_EnterpriseVL_x64_zh-cn_19041.iso**，直接安装这个体验包，可以省去很多升级的麻烦。

&emsp;&emsp;微软在新的 WSL2 中（安装请参考 [WSL2 安装](https://docs.microsoft.com/zh-cn/windows/wsl/wsl2-install)），使用了 Hyper-V 技术支持独立的 Linux 应用（换句话说，Hyper-V 主要解决 Windows 不完美支持大小写以及软硬链接等Linux中的各种问题，但是，可能会产生安全方面的问题，比如这次内核的改造刚才中，发现了一些安全方面存在的隐患，Windows 和 Linux 的结合，最头疼的实际上是用户权限分配问题，还有个隐藏的问题就是，Windows 如果强行搞一个大小写敏感的文件系统，会给黑客带来很多遐想的空间）。

&emsp;&emsp;遗憾的是，无论是微软的 Hyper-V 技术还是这个 Windows 10 2004 版上新的 WSL2，都不直接支持无线网卡等 USB 设备。最近在研究 WSL 的过程中，发现利用微软开源的 WSL 内核代码，通过手动编译内核，加入一个名为 USB/IP 的项目，通过在宿主机中运行 USB/IP 的服务程序，来将主机的USB网卡通过 USB/IP 转发给 WSL 内核中，从而实现主机 USB 设备透明地提供给 WSL 使用。

&emsp;&emsp;在 Windows 中安装 USB/IP，可以通过下载 GIT 代码来编译实现：git clone <https://github.com/cezuni/usbip-win.git> 。USB/IP 的功能就是将任何主机端的 USB 口，通过以太报文传送到客户端，让客户端虚拟出这个 USB 设备，实现 USB 协议的透传效果。

&emsp;&emsp;编译的过程就不写了，可以直接使用编译好的结果：[USB/IP 的执行文件](https://github.com/cezuni/usbip-win/releases)。关于如何在 Windows 中安装非合法签名的驱动，请自行百度。有个小问题需要注意，如果 BIOS 设置为安全启动，则无法在 Windows 中修改成测试模式的。

&emsp;&emsp;在每台设备上，内核代码都是和操作系统绑定的，因此，没有可能编译一次就到处使用。必须在该主机上编译完成以后，再替换掉 WSL2 的启动镜像。而最后形成的 Kali 镜像文件，是可以安全覆盖现有的镜像文件，达到不断升级，不断完善调试环境的目的。

&emsp;&emsp;WSL 的运行由两部分组成，操作系统的启动部分以及某个操作系统运行部分。

&emsp;&emsp;操作系统启动部分，对于所有的 WSL 都是公用的，所谓内核编译就是替换掉这个公用的部分。无论你是运行 Kali 还是 Ubuntu 还是其他的系统，此部分都是共享的。所以，在 Kali 中编译了内核代码，并且替换现有的内核以后，每个镜像都具备了统一种新内核功能。比如我们这次完成对 USB/IP 的支持功能。

## 编译内核代码并替换内核

&emsp;&emsp;微软开放了 WSL2 的内核，直接可以通过 GitHub 下载：git clone <https://github.com/microsoft/WSL2-Linux-Kernel>。

&emsp;&emsp;编译前，最好先 uname -r 来查询对应的内核版本，然后 checkout 相同版本出来进行编译。编译的时间很快，可能是我机器强大的原因，也可能是另外的原因，系统在编译时候，自动加了 -j12 选项，太智能了。估计是系统检测到环境中 CPU 内核等数量，自动加上的优化选项。总之，自从微软加入开源的 Linux 阵营后，秒杀所有其他努力活着的程序猿（有了 VSCode 和 Windows Terminal，我几乎不再使用其他的各种 IDE 环境了）。不过为了使用网卡驱动，必须有几个选项在 menuconfig 选择的时候勾选的。具体的我忘记了，主要是在编译网卡驱动的时候，提示找不到各种头文件或者找不到某些变量或者出现某些奇怪问题的时候，重新来勾选内核的某些支持功能。由于我们的无线网卡主要运行在嗅探模式，而并不是所有网卡都支持 master 模式的。我选择的是瑞昱 Realtek-RTL8187L 无线网卡，该网卡功率可以调节，并且网上有源码支持驱动在内核上的编译。

![Realtek-RTL8187无线网卡](https://github.com/superbinny/wsl2study/blob/master/img/Realtek-RTL8187.jpg)

&emsp;&emsp;最好先 ***make distclean*** 清除所有的垃圾文件，然后重新编译。最近发现 WSL2-Linux-Kernel 的变化挺大的，所以可以将 .config 备份以后，经常做一下 ***git reset --hard*** 以及 ***git pull*** 来更新最新内核文件。为了简单起见，编译前可以先从当前的操作系统中拷贝定制的 ***.config*** 文件来正确编译新的内核。

&emsp;&emsp; ***cp /proc/config.gz . & gzip -d ./config.gz & mv config .config & make menuconfig***

&emsp;&emsp;由于新内核支持 USB/IP，所以在 menuconfig 中需要选择：

+ **Device Drivers->USB support->USB/IP support[M]**
+ **Device Drivers->USB support->Number of USB/IP virtual host controllers(1)**
+ **Device Drivers->Network device support->USB Network Adapters[M]**
+ 以及各种对USB网卡和你需要的USB设备驱动支持......

&emsp;&emsp;编译结束以后，可以 ***make modules_install & make heards_install & make install*** 来安装内核文件和相应的头文件到  **/lib/modules**  中。

&emsp;&emsp;这里我有个技巧：随便在某个硬盘 x:\ 上建立一个 Source 目录，然后软软链接到该目录，以便于内外交换各种文件和在外部宿主机上编译代码。我的所有源码文件都存放在这个  x:\Source  中以节约 WSL 空间，另外由于 Windows 部分支持大小写敏感，最好这个共享的目录设置成大小写敏感，以支持某些 Linux 的应用。

&emsp;&emsp;&emsp;&emsp; ***mkdir ~/source & ln -s /mnt/x/Source ~/source***

&emsp;&emsp;![编译后产生的 vmlinux 文件](https://github.com/superbinny/wsl2study/blob/master/img/vmlinux.png)

&emsp;&emsp;编译后，在本级目录中找到 vmlinux 文件，拷贝到 ~/source 中，然后关闭 WSL 虚拟机（PS C:\WINDOWS\system32>***wsl --shutdown***），再然后备份好 ***C:\Windows\System32\lxss\tools\kernel*** ，将 vmlinux 改名为 kernel，便可以重新启动 Kali。
/
&emsp;&emsp;![新内核版本](https://github.com/superbinny/wsl2study/blob/master/img/uname_r.png)

&emsp;&emsp;至此，已经改造了新的 WSL2 内核来加载我们的 USB/IP 驱动。

## 编译 USB/IP 工具

&emsp;&emsp;单独编译 USB/IP 的加载工具，以便于后面和主机的通讯以及模拟仿真。

&emsp;&emsp;进入内核代码的 tools，开始编译：

&emsp;&emsp; ***cd tools/usb/usbip & ./autogen.sh & ./configure & make install***

## 在 Kali 中挂载无线网卡

### 在 Windows 中提供 USB/IP 服务

&emsp;&emsp;见证奇迹的时候即将开始。首先我们要将 Windows 设置成测试模式，以便于加载 USB/IP 的驱动，并提供 USB/IP 服务。这个过程可以在百度上搜索。如果实在搞不定，有时间我再写一篇 USB/IP 在 Windows 下的编译和使用。但没技术含量的事情实在没兴趣做。

&emsp;&emsp;以下是我用 usbip 列出 Windows 上的设备并且绑定该设备提供服务的画面，注意，其中 1-4 为 RTL8187L 网卡。此时，**注意将 Windows 防火墙设置中加入服务提供程序 usbipd.exe**，不然一会Kali中找不到：

&emsp;&emsp; ![Windows加载USB/IP](https://github.com/superbinny/wsl2study/blob/master/img/usbip_win.png)

### 在 Kali 中启动 USB/IP，以接入 Windows 的 USB 设备

&emsp;&emsp;回到我们的 Kali ，此时此刻，如果使用命令 lsusb 是无法列出任何一个设备的。我们可以在命令行上先查询 Windows 提供的服务是否存在，如果存在，则列出 Windows 上绑定的 USB 设备，然后在 Linux 中进行远程加载。加载以后，该设备就成为 Linux 中可以进一步使用的设备了。

&emsp;&emsp;首先我们要知道 Windows 中绑定虚拟 WSL 网卡的那个 IP 地址是什么，以便于远程绑定，为此可以运行：

&emsp;&emsp;&emsp;&emsp; ***export wsl_ip=$(cat /etc/resolv.conf | grep nameserver | awk '{print $2}')***

&emsp;&emsp;将该地址找到并且保存在环境变量 wsl_ip 中以备使用。我这里是 172.20.80.1（大家可以想象一下，该地址可以是任何一种设备，比如树莓派）。此时，Windows 中应该有调试信息显示，如果没有，很不幸地，要么前面没做好，要么防火墙没通过。

&emsp;&emsp;&emsp;&emsp; ***usbip list -r $wsl_ip***

&emsp;&emsp;在挂载前，我们还需要加载前面内核编译产生的几个新模块。它们分别是 ***usbcore.ko、usb-common.ko、usbip-core.ko 和 vhci-hcd.ko***，此外，还有其他的模块，根据你的需要加载。通过 modprobe 命令加载进系统：

+ &emsp;&emsp;modprobe usbcore
+ &emsp;&emsp;modprobe usb-common
+ &emsp;&emsp;modprobe usbip-core
+ &emsp;&emsp;modprobe vhci-hcd

&emsp;&emsp;我们可以先用 lsusb 测试一下，是否可以列出新的普通 USB 设备（例如 Usb hub），然后我们可以开始挂载该 IP 的 USB 设备了：***usbip attach -r $wsl_ip -b 1-4***。用 lsusb 测试一下，Kali 中是否多出一个新的 USB 设备：

&emsp;&emsp;![Kali 加载 USB/IP](https://github.com/superbinny/wsl2study/blob/master/img/usbip_kali.jpg)

&emsp;&emsp;剩下的最后一步就是添加无线网卡驱动。如果加载了驱动，本 Kali 就可以像真机一样，享用无线网卡带来的各种福利了。

### 加载无线网卡驱动，用 wifite 测试网卡破解功能

&emsp;&emsp;加载驱动有两种方式，一种是下载驱动源码在这个配置好的 kali 中编译和安装。第二种，我发现有个开源项目可以完美解决我手头的 8187L 驱动安装问题：git clone <https://github.com/matrix/backports-rtl8187>，于是就偷懒直接用了，下载以后，运行脚本程序，如果有问题可以简单修改一下该脚本，比如，该脚本必须用普通用户身份运行，我就不信邪修改成超级用户。

&emsp;&emsp;以下就是见证奇迹的时候。新版的 wifite2 一改旧版本编程风格（没事的时候，大家可以看看旧的 wifite 一代，编码风格丑得一米），并且支持 PMKID、Pyrit 和 pcxtools 等新特性，可用于进一步改造成 WiFi 破解神器。

&emsp;&emsp;![wifite 运行界面](https://github.com/superbinny/wsl2study/blob/master/img/wifite.png)

### 备份 Kali 虚拟机，直接在其他机器上使用

&emsp;&emsp;在我们的 Kali 中，好不容易安装了各种编译工具以及辛苦搭建的环境，我们希望能在多个地方重复使用，就像使用 VMware 的镜像一样。方便起见，可以使用迁移工具，将 Kali 镜像文件备份下来，然后在其他机器上随便安装一个新的 Kali，再用这个镜像覆盖掉该镜像，便完成了整个定制 Kali 环境的迁移、部署和备份，做到不断打磨破解利剑的目的。

&emsp;&emsp;使用 LxRunOffline 工具，在 Kali 关机的情况下，运行：

&emsp;&emsp; ***.\LxRunOffline.exe m -n kali-linux -d x:\some_where***

&emsp;&emsp;来迁移到新的位置 x:\some_where，然后将 ext4.vhdx 备份下来即可（如果找不到 ext4.vhdx 文件，而是显示一堆 Linux 目录，那就是你的系统可能没有完美的 Hyper-V 支持，需要开启这个模式才行），这个镜像文件可以随身携带以免丢失。用这个虚拟机镜像文件去覆盖其他版本的 WSL 即可。该文件也可以 mount 到其他操作系统上，做数据盘使用。

## 后记

&emsp;&emsp;有幸赶上微软收购 Github，重磅进入 Linux ，微软推出的一系列工具和措施极大提高生产效率，降低编程的枯燥性，带来了编写和调试代码的舒适度。基本上是怀着分享的心态来做这个教程，与大家共享有关 Linxu 内核和 Windows 结合带来的生产力革命性的进步。

&emsp;&emsp;编写这个教程花了我整天的时间，利用了这个假期，加深了对  VSCode 的认识，也让我下决心抛弃了很多食之无味弃之可惜的工具。

&emsp;&emsp;此外，跨平台的研究，实际上不仅仅局限在此，物联网时代，需要我们的设备也可以方便地相互访问，利用计算机强大的计算能力和物联网设备的低成本和灵活部署性，可以应用于很多新的领域。就破解的事情来说，我们完全可以将树莓派等设备上安装无线网卡，利用 Windows 直观操作的便捷性，来轻松地操作分布式设备，达到运筹帷幄的效果。

## 下一步工作

&emsp;&emsp;用安卓模拟器，结合 Kali，打造强大的 APP 研究环境和 WiFi 密钥破解工具，该工具将结合“X能密钥”等APP，直接从服务器获取所有的已知密码，形成独特的破解基础库。
