
# 让WSL2中的Kali使用无线网卡

## 为什么有这个项目

&emsp;&emsp;最近在研究破解WiFi密码，由于有了新的WPA3安全标准，因此，也产生了新的破解思路，即利用WiFi基站漫游功能中用于身份认证的PMKID，来直接破解密码，而PMKID的出现，甚至都不需要像以往那样，针对每个ESSID做破解字典了。

&emsp;&emsp;看看以下破解公式，就知道用PMKID破解无线密码，变得轻松很多。具体原理就不在这里深入探讨了，以后准备找个时间专门讨论这个问题：

&emsp;&emsp;***PMKID = HMAC-SHA1-128(PMK, "PMK Name" | MAC_AP | MAC_STA)***

&emsp;&emsp;以往是在虚拟机中做破解，既笨重又没有效率，现在微软推出WSL2，刚好用来做WiFi密码破解的实验环境。但是，目前放出的WSL2没有想象中的完美，因此，我决定改造WSL2，来达到和VMware虚拟机一样的效果。我喜爱WSL2主要原因在于，WSL2启动太快了，只要一秒钟，而与Windows的完美结合，还占用最少的资源，加上可以直接与Linux通信的VSCode，使得编程的研究环境大大友好，感觉Windows和Linux的混合编程从来没有这么美好过。（PS. 通过迁移工具，将WSL2迁移到独立的虚拟文件硬盘中，走到哪里都可以不用重新安装和配置Kali环境了）

&emsp;&emsp;还有个想法，Mark一下。如果用这个技术编译进安卓模拟器中，就可以用任何无线网卡，通过WiFi万能密钥功能获取密码了。并且还让普通手机透明对接任何大功率无线网卡。

## 改造Kali，安装超级工具集

&emsp;&emsp;从微软商店下载安装最新的Kali2020，经过各种必要的准备，便可以开始进行内核编译和升级了。

&emsp;&emsp;升级的目的是为了打造一个强有力的Kali，因此，无线网卡的加载是非常重要的。以下是在WSL中安装的Kali界面和工具集。具体过程就省略了，和本次技术实践的目的不一样。

![完整的WSL2 Kali环境](https://github.com/superbinny/wsl2study/blob/master/img/kali_full.png)

## WSL2的改造思路
&emsp;&emsp;微软在新的WSL2中（安装请参考[WSL2安装](https://docs.microsoft.com/zh-cn/windows/wsl/wsl2-install)），使用了Hyper-V技术支持独立的Linux应用，但遗憾的是，无论是微软的Hyper-V技术还是这个Windows 10 2004版上新的WSL2，都不直接支持无线网卡等USB设备。最近在研究WSL的过程中，发现利用微软开源的WSL内核代码，通过手动编译内核，加入一个名为USBIP的项目，通过在宿主机中运行USBIP的服务程序，来将主机的USB网卡通过USBIP转发给WSL内核中，从而实现主机USB设备透明地提供给WSL使用。

&emsp;&emsp;在Windows中安装USBIP，可以通过下载GIT代码来编译实现：git clone <https://github.com/cezuni/usbip-win.git> 。USBIP的功能就是将任何主机端的USB口，通过以太报文传送到客户端，让客户端虚拟出这个USB设备，实现USB协议的透传效果。

&emsp;&emsp;编译的过程就不写了，可以直接使用编译好的结果：[USBIP的执行文件](https://github.com/cezuni/usbip-win/releases)。关于如何在Windows中安装非合法签名的驱动，请自行百度。有个小问题需要注意，如果BIOS设置为安全启动，则无法在WIndows中修改成测试模式的。

&emsp;&emsp;在每台设备上，内核代码都是和操作系统绑定，因此，没有可能编译一次就到处使用的。必须在该主机上编译完成以后，再替换掉WSL2的启动镜像。而最后形成的Kali镜像文件，是可以安全覆盖现有的镜像文件，达到不断升级，不断完善调试环境的目的。

&emsp;&emsp;WSL的运行由两部分组成，操作系统的启动部分以及某个操作系统运行部分。

&emsp;&emsp;操作系统启动部分，所有的WSL都是公用的，因此，所谓内核编译，就是替换掉这个公用的部分。无论你是运行Kali还是Ubuntu还是其他的系统，此部分都是共同的。所以，在Kali中编译了内核代码，并且替换现有的内核以后，实际上，每个镜像都具备了统一种内核功能。比如，我们这次完成的USBIP支持功能。

## 编译内核代码并替换内核

&emsp;&emsp;微软开放了WSL2的内核，直接可以通过GitHub下载：git clone <https://github.com/microsoft/WSL2-Linux-Kernel>。

&emsp;&emsp;编译时间很快，可能是我机器强大的原因，也可能是另外的原因，系统在编译时候，自动加了-j12选项，太智能了。估计是系统检测到环境中CPU内核等数量，自动加上的优化选项。总之，自从微软加入开源的Linux阵营后，秒杀所有其他努力活着的程序猿（比如有了VSCode，我几乎不再使用其他的各种IDE环境了）。不过为了使用网卡驱动，必须有几个选项在menuconfig选择的时候，勾选的。具体的我忘记了，主要是在编译网卡驱动的时候，提示找不到各种头文件的时候或者找不到某些变量的时候，重新来勾选内核的某些支持功能。因为我们的无线网卡主要运行在嗅探模式，并不是所有网络都支持。我选择的是瑞昱Realtek-RTL8187无线网卡，该网卡功率可以调节，并且网上有源码支持内核的编译。

![Realtek-RTL8187无线网卡](https://github.com/superbinny/wsl2study/blob/master/img/Realtek-RTL8187.jpg)

&emsp;&emsp;最好先***make distclean***清除所有的垃圾文件，然后重新编译。最近发现WSL2-Linux-Kernel的变化挺大的，所以，最好.config备份工作以后，最好还是经常做一下git reset --hard以及git pull来更新最新内核文件。为了简单起见，最好从当前的操作系统中拷贝定制的.config文件来正确编译新的内核：

&emsp;&emsp;***cp /proc/config.gz . & gzip -d ./config.gz & mv config .config & make menuconfig***

&emsp;&emsp;编译结束以后，可以***make modules_install & make heards_install & make install***来安装内核文件和支持库到 /lib/modules 中。

&emsp;&emsp;这里我有个技巧：可以随便在某个硬盘x:上建立一个Source目录，然后软连接到该目录，以便于内外交换各种文件和在外部宿主机上编译代码。我的所有源码文件都存放在这个x:\Source中。

&emsp;&emsp;***mkdir ~/source & ln -s /mnt/x/Source ~/source***

&emsp;&emsp;![编译后产生的vmlinux文件](https://github.com/superbinny/wsl2study/blob/master/img/vmlinux.png)

&emsp;&emsp;编译后，在本级目录中找到vmlinux文件，拷贝到~/source中，然后停止WSL虚拟机（PS C:\WINDOWS\system32>***wsl --shutdown***）,再然后，备份好***C:\Windows\System32\lxss\tools\kernel***，将vmlinux改名为kernel以后，重新启动Kali。

&emsp;&emsp;![编译后产生的vmlinux文件](https://github.com/superbinny/wsl2study/blob/master/img/uname_r.png)

&emsp;&emsp;至此，已经改造了新的WSL2内核，用来加载我们的USBIP驱动了。

## 编译USBIP并加入到内核中

