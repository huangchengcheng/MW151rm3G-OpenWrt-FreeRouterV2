
# 在MW151rm3G上部署OpenWrt + FreeRouterV2

## 项目背景

打造一枚低成本便携WIFI路由器，提供多设备零配置无感[翻墙](http://zh.wikipedia.org/wiki/%E7%AA%81%E7%A0%B4%E7%BD%91%E7%BB%9C%E5%AE%A1%E6%9F%A5)。

基于OpenWrt + PPTP VPN的[FreeRouterV2](https://github.com/lifetyper/FreeRouter_V2)是目前最完美（精确，易维护）的翻墙方案。

## 硬件选择

MW151rm3G是一款支持把3G或者普通有线网络转换成WIFI网络的低价且小巧的路由器。它自带4M闪存32M内存，基本上是支持OpenWrt的最低配置了。如果不需要3G功能且有能力拆机换大闪存和内存，建议入手MW151rm，价格更低。但是，市面零售闪存颗粒质量普遍比厂商自带的低很多。基于这点加上拆机影响美观上的考虑，我就不折腾了。

其实MW151rm3G是大名鼎鼎的wr703n的马甲版本，硬件电路完全一样，但价格更低。除非你喜欢TP的logo，不然实在没必要买wr703n。另外，MW151rm3G因为出货量不多，目前市面上的货基本上硬件还是v1初版。而wr703n已经改版好几次，改版一般都意味着降成本，而且OpenWrt可能会不兼容。

## 软件准备

### 变身wr703n

虽然硬件上MW151rm3G同wr703n完全一样，但是它们自带的软件是不同的。据说，不能通过水星的webUI直接刷OpenWrt，后果不详。所以我们需要这个[MW151rm3G_to_wr703nv1](http://pan.baidu.com/wap/link?uk=53469291&shareid=343179&third=0)先把它彻底变成wn703n的马甲。然后就可以在TP的webUI刷OpenWrt。

### 如何选择OpenWrt固件

好了，现在我们手上的已经是"wr703n"。重点来了，现在我们需要在wr703n上搞定支持根据FreeRouterV2的OpenWrt环境。根据FreeRouterV2的作者推荐的方式，是先直接安装官方编译的OpenWrt固件，然后再装FreeRouterV2项目必需的packages。但是，在我的实际操作过程中发现，不管是attitude_adjustment，barrier_breaker还是trunk分支版本，刷完之后所剩ROM空间均不到1M，无法容纳必要的packages。

所以，我们只能自己动手，利用OpenWrt提供的[Image Generator](http://wiki.openwrt.org/doc/howto/obtain.firmware.generate)自己生成bin文件，把下列这些必要packages包含进去：

> + ipset
> + ip
> + kmod-ipt-ipopt
> + iptables-mod-filter
> + kmod-ipt-filter
> + iptables-mod-u32
> + kmod-ipt-u32
> + ppp-mod-pptp
> + dnsmasq-full

其实还少一个luci，但是wr703n实在是ROM太小，加了它bin就build不出来了。庆幸的是luci还真不是必要的，它只是一个OpenWrt的webUI，没有它我们依然可以在命令行环境下完成OpenWrt的配置。

### 定制OpenWrt固件

#### 下载Image Generator

这里我用的是[barrier_breaker RC2版本的Image Generator](http://downloads.openwrt.org/barrier_breaker/14.07-rc2/ar71xx/generic/OpenWrt-ImageBuilder-ar71xx_generic-for-linux-x86_64.tar.bz2)。相比trunk分支，RC版本相对更稳定些，而attitude_adjustment分支压根就没有提供dnsmasq-full这个package，当时的dnsmasq只到2.62版本，而ipset功能在2.66版本才加入。

#### 打包定制固件

找一台Linux的机器，注意不要用太老版本的系统，否则打包的时候可能会报kernel太老的错误。

先把压缩包解压出来：

	tar -xjvf openWrt-ImageBuilder-ar71xx_generic-for-linux-x86_64.tar.ba2

进到包的根目录下make，把必要的packages一起打包了：

	make image PROFILE="TLWR703" PACKAGES="ipset ip kmod-ipt-ipopt iptables-mod-filter kmod-ipt-filter iptables-mod-u32 kmod-ipt-u32 ppp-mod-pptp -dnsmasq dnsmasq-full"

注意下“-dnsmasq”意思是先去掉默认自带的dnsmasq版本。

没有Linux环境的同学，可以直接下载我[打包好的固件](http://pan.baidu.com/s/1dD3pJqT)。

## 刷OpenWrt固件

不出意外的话，打包完成后可以在/bin/ar71xx目录下找到openwrt-ar71xx-generic-tl-wr703n-v1-squashfs-factory.bin和openwrt-ar71xx-generic-tl-wr703n-v1-squashfs-sysupgrade.bin。前者适用于从TP自带的webUI升级，后者适用于从另一个OpenWrt版本中升级，可以在luci中升级，也可以在命令行下用[sysupgrade命令升级](http://wiki.openwrt.org/doc/howto/generic.sysupgrade)。

##配置OpenWrt

### 设置root密码开启SSH

在这个项目中，我们永远也用不上luci了，所以所有的配置都需要在命令行下完成。

首先用一根网线连到MW151rm3G的LAN口，电脑上设置静态ip为192.168.1.2。然后telnet到192.168.1.1，你将看到OpenWrt的欢迎界面。

接下来马上设置root密码：

	passwd root

然后，就可以退出telnet用我们熟悉的SSH方式管理OpenWrt

### 把网口配置成WAN，并且打开无线

MW151rm3G是LAN/WAN共用一个网口，OpenWrt初始配置成LAN口，为了第一次telnet可以连接。而且，初始配置下，无线端口是关闭的。

用vi打开/etc/config/network文件：

	vi /etc/config/network

注释掉此行，目的是释放eth0给后面WAN口使用：

	# option ifname 'eth0'

然后增加WAN口，把eth0配置成WAN：

	config interface 'wan'
    		option ifname 'eth0'
    		option proto 'dhcp'

接下来我们通过修改/etc/config/wireless文件：

	vi /etc/config/wireless

注释掉此行，就可以打开无线端口：

	# option disabled 1

最后，用reboot命令重启下路由器，让配置生效。如果reboot不好用，就断电重启吧。
这时，基本的AP功能应该配置好了，插上网线连上OpenWrt这个SSID你的设备应该就可以上网了。而且，这时候SSH走的也是无线接口了。
当然，上面的配置万一你哪里一不小心搞错了（比如，把LAN转成WAN了，但是忘了打开无线），可能SSH就再也连不上。这时候你就需要reset OpenWrt了：

> 1. 断电重启MW151rm, 等LED闪烁时，戳一下Reset孔，然后过一会儿如果你看到LED狂闪，就说明已经进入OpenWrt的safe mode了；
> 2. 这时候，你又可以从电脑插根网线就能telnet上去了，然后在命令下用firstboot命令把之前的配置抹去；
> 3. reboot一下，你又回到刚刷完OpenWrt的状态了。

### 增加PPTP VPN接口

还是在/etc/config/network中，我们来增加一个VPN接口：

	config 'interface' 'vpn' 
	        option 'ifname'    		'pptp-vpn'  
	        option 'proto'     		'pptp'
	        option 'username'  		''
	        option 'password'  		''
	        option 'server'    		'' 
	        option 'buffering' 		'1'
	        option 'defaultroute'	'0'

注意，最后一条是告诉系统不要创建到VPN接口的default路由，不然所有流量都会走VPN。
留空的参数，去问你的VPN供应商吧。

然后，我们还要把VPN接口配置到WAN侧。
打开/etc/config/firewall文件：

	vi /etc/config/firewall

修改WAN的zone为:

	config zone
		option name			wan
		list   network		'wan'
		list   network		'wan6'
		list   network		'vpn'
		option input		REJECT
		option output		ACCEPT
		option forward		REJECT
		option masq			1
		option mtu_fix		1

对比默认的，你会发现，其实就增加了这一行：

	list   network		'vpn'

再重启下路由器，如果一切顺利，用ifconfig命令可以看到多了一个pptp-vpn的接口，并且流量不为0。

## 部署FreeRouterV2

这个参考[FreeRouterV2的项目手册](https://github.com/lifetyper/FreeRouter_V2/blob/master/FreeRouterV2_HandBook.pdf)进行就可以了。而且手册里不仅仅只有部署，把原理都讲得很透了了，据作者说是按文科生也能看懂的标准写的。

## TODO
增加3G接口配置，实现3G to WIFI自动翻墙。
