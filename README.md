# 让WSL2中的Kali使用无线网卡
&emsp;&emsp;最近在研究破解WiFi密码，由于有了新的WPA3安全标准，因此，也产生了新的破解思路，即利用等同于4次握手的PMKID，来直接破解密码，而PMKID的出现，甚至都不需要针对每个ESSID做字典了。

&emsp;&emsp;看看以下破解公式，就知道用PMKID破解无线密码，变得轻松很多。具体原理就不在这里深入探讨了，以后准备找个时间专门讨论这个问题：

&emsp;&emsp;*PMKID = HMAC-SHA1-128(PMK, "PMK Name" | MAC_AP | MAC_STA)*

&emsp;&emsp;以往是在虚拟机中做破解，既笨重又没有效率，现在微软退出WSL2，刚好用来做破解的环境。但是，WSL2没有想象中的完美，因此，我决定改造WSL2，来达到虚拟机一样的效果。主要在于，WSL2启动太快了，只要一秒钟。而与Windows的完美结合，结合直接可以与Linux通信的VSCode，使得编程的研究环境大大友好，感觉从来没有这么美好过。（PS.通过迁移工具，将WSL2迁移到独立的虚拟文件硬盘中，走到哪里都可以不用重新安装和配置环境了）

&emsp;&emsp;还有个想法，MARK一下。如果用这个技术编译进安卓模拟器中，就可以用任何无线网卡，通过WiiI万能密钥功能获取密码了。并且还让普通手机透明对接任何大功率无线网卡。

&emsp;&emsp;微软在新的WSL中，使用了HyperV技术支持独立的Linux应用，但遗憾的是，无论是微软的Hyper-V技术还是这个新的WSL2，都不直接支持无线网卡等USB设备。最近在研究WSL的过程中，发现利用微软开源的WSL内核代码，通过手动编译内核，加入一个名为USBIP的项目，通过在宿主机中运行USBIP的服务程序，来将主机的USB网卡通过USBIP转发给WSL内核中，从而实现主机USB设备透明地提供给WSL使用。

&emsp;&emsp;在Windows中安装USBIP，可以通过下载GIT代码来编译实现：git clone https://github.com/cezuni/usbip-win.git 。USBIP的功能就是将任何主机端的USB口，通过以太报文传送到客户端，让客户端虚拟出这个USB设备，实现USB协议的透传效果。

&emsp;&emsp;编译的过程就不写了，可以直接使用编译好的结果：[USBIP的执行文件](https://github.com/cezuni/usbip-win/releases)

&emsp;&emsp;关于如何在Windows中安装非合法签名的驱动，请自行百度。
