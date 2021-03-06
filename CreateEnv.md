
# 创建两岸三地的工作环境

*<binny@vip.163.com>*，2020-5-13

## 为什么有这个项目

&emsp;&emsp;准备将某国朋友那里的上 T 的大文件分卷传回来，虽然可以通过百度网盘或者其他网盘中转，但是这些网盘要么对国外带宽支持不好，要么对国内支持有限。也可以考虑使用 BitTorrent 协议来对文件进行分发，但是安全性和完整性不一定得到保障。刚好在香港有一台私有的云服务器，于是产生这种想法，本项目需要满足以下要点：

+ &emsp;&emsp;将大文件分卷，逐步传输
+ &emsp;&emsp;传输要很方便，能够无人值守
+ &emsp;&emsp;满足中转的服务器存在硬盘空间小，带宽不足的情况
+ &emsp;&emsp;能自由和方便地在几个不同的国家终端中切换工作界面
+ &emsp;&emsp;对内网的访问畅通，不要使用类似 TeamViewer 等图形化工具导致严重卡顿
+ &emsp;&emsp;可以方便地相互交换文件，手机还能监控进度
+ &emsp;&emsp;安全起见，所有底层要开源，不能有后门

&emsp;&emsp;说白了，就是需要有个私有网盘，并且能做到内网的透明穿透。将内网网盘共享给服务器使用。

## 香港服务器安装网盘服务器

&emsp;&emsp;香港的腾讯服务器，访问各个国家还是很快的，所以我租用了一台只有 50G 硬盘的服务器，安装了 Ubuntu 服务器版用于文件中转。

&emsp;&emsp;选择了开源的 SeaFile 网盘项目，项目的地址为：<https://github.com/haiwen/>

### 搭建网盘服务器

&emsp;&emsp;在 seafile [官方网站](https://www.seafile.com/home/) 上下载 Linux 服务器端的[文件](http://seafile-downloads.oss-cn-shanghai.aliyuncs.com/seafile-server_7.1.3_x86-64.tar.gz)，根据[官方手册](https://cloud.seafile.com/published/seafile-manual-cn/home.md)，顺利安装到香港的服务器中。

&emsp;&emsp;在运行 seahub 服务的时候，出了点小问题，不过很快解决，跟踪了 seahub.sh 启动代码，主要是缺省的 Ubuntu 环境缺少 Python 的支持库原因。大部分的问题都是对 Python 支持库或者 Python 的版本不对引起。

&emsp;&emsp;  ***seahub.sh start*** 启动以后，Web 界面显示： `WEB 访问提示 Internal Server Error`

&emsp;&emsp;通过安装 ***pip3 install django-simple-captcha*** ，解决！

## 安装网盘客户端

### 在国外机器上安装 seafile 客户端

&emsp;&emsp;seafile 的客户端选择很多，各种版本都有。但是 Windows 虚拟盘安装以后，在传输完成并剪切文件的时候，容易出现蓝屏现象，所以，最好不用基于内核驱动的虚拟硬盘方式在 Windows 环境中使用，用界面程序就足够了。

### 在需要共享的所有设备上安装客户端

&emsp;&emsp;手机端也可以支持 seafile，因此，可以很方便地观察文件的传输情况。seafile 虽然比较轻量级，但是该有的功能基本都有了，所以还是很好用的。并且可以选择是否同步到本地端，来控制文件是否需要下载。这个功能很好用，因为当大家都下载的时候，不仅占用带宽，还浪费空间，可以根据需要决定是否下载文件。

## 穿透内网，方便快速同步所有的工作环境

&emsp;&emsp;由于所有的工作主机都位于内网，刚好又有香港具有公网地址的服务器来提供跳转服务，所以，可以在该服务器上搭建用于内网穿透的服务端，这里我选用好用的 frp 工具来穿透内网，做到反向代理的功能。

&emsp;&emsp;frp 项目位于 <https://github.com/fatedier/frp>，功能强大但使用极其简单，只需要在服务器上运行一个 frps 进程，然后在所有位于局域网的电脑上运行 frpc 进程，则实现内网的穿透。而且，可以根据想象搭建出各种复杂的网络环境。我主要工作是用 Putty 和 SecureCRT，属于最简单的应用。用一台电脑快速在各个不同国家和地区的电脑中切换，这些电脑可能是真实的主机，也可能是位于主机内的一台虚拟机。

## 搭建 SS 代理

&emsp;&emsp;如果直接将家里的闲置的无线路由器采用透明代理模式，则在需要去国外的时候，直接连接 WiFi 就行，非常方便。

&emsp;&emsp;首先可以用香港服务器搭建一个 SS 代理，然后在闲置的刷了梅林系统的 R8000 上安装透明代理程序，就完成上述的目的了。服务端的搭建工作可以直接参考 <https://www.24kplus.com/linux/156.html>，写得比较全面，我就不啰嗦了。

&emsp;&emsp;Ubuntu 的初始化安装记录一下，以后免得忘记了：

&emsp;&emsp; ***sudo apt install --no-install-recommends build-essential autoconf libtool libssl-dev gawk debhelper dh-systemd init-system-helpers pkg-config asciidoc xmlto apg libpcre3-dev zlib1g-dev libev-dev libudns-dev libsodium-dev libmbedtls-dev libc-ares-dev automake***

&emsp;&emsp;用 git clone <https://github.com/shadowsocks/shadowsocks-libev.git>，搭建很方便。这里我采用的是 Google 的一种新式加密算法 chacha20-ietf-poly1305，据说性能强大（记得设置防火墙，通过相关的端口）。

## 工作情况

&emsp;&emsp;写了几个简单的脚本用于监控传输的文件是否将香港的硬盘撑满，如果满了则搬运所有同步好的文件到其他目录，然后清空服务器上的文件缓存。

&emsp;&emsp;SeaFile 服务器有些问题，撑满硬盘以后，即便使用官方交代的垃圾清理脚本 seaf-gc.sh 也无法正常清理（难道需要购买专业版？），需要手动删除缓存的文件以后，再启动服务。以后有时间再研究和修复这个问题。

```shell
pushd $PWD #保留工作路径
cd 安装文件的位置
./seafile.sh stop #首先停止服务
rm -rf 文件缓存  #手动清理文件缓存
./seaf-gc.sh  #用官方的工具清理缓存
./seafile.sh start #重新启动服务
popd  #回到工作环境
```

&emsp;&emsp;总体工作顺利，由于用了内网穿透技术，使用类似 SSH 协议工具能够直接在字符界面工作，即便网速很低，也不会出现连接不上和掉线的情况。

<!-- 居中 宽度500 -->
<div align=center>
	<img src="https://github.com/superbinny/wsl2study/blob/master/img/seafile_trans.png" width="500"> 
</div>

## 工作总结

&emsp;&emsp;为了以后不会忘记，所以总结一下步骤：

+ 租用公网服务器 => 安装 Ubuntu => 安装防火墙 firewalld => 安装 mysql 服务器 => 安装 apache2 服务 => 安装 seafile 网盘 => 修改 apache2 的 80 端口配置以支持反向代理 => 安装 frps 服务 => 开放所有需要开放的端口和服务；
+ 提供大文件的客户机安装 seafile 客户端 => 大文件分卷打包 => 设置一个共享文件夹，用于分别传输分卷文件（需要控制总文件的大小，不超过公网服务器的总硬盘空间） => 注册 seafile 客户端 => 运行内网代理软件 frpc => 提供 SSH 服务以供相互互联；
+ 编写一些脚本，用于清理服务器和客户机的文件 => 配置统一的机器列表信息，方便访问；
+ 方便起见，可以将这些启动脚本全部变成服务，用 systemctl 固化在服务的启动列表中，自动开机启动。

&emsp;&emsp;自此，一个比较满意的环境搭建完成，以后去国外搬运大件就很方便了。
