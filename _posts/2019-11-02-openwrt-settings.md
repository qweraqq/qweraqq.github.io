---
layout: post
title: "[网络折腾]Openwrt settings(持续更新中)"
date: 2019-11-02 00:00:00 +0800
author: xiangxiang
categories: env-settings openwrt
tags: [openwrt chinadns dnscrypt-proxy dnsmasq]
---

上网究竟有多难，看完本文才知道

## 0x00 首次设置
- 首次设置建议使用有限网络
- openwrt的WAN/LAN口默认是DHCP，WAN口接可以接入互联网的路由器(比如运营商的光猫)，LAN口接电脑
- 将opkg源设置为国内源
{% highlight bash %}
# SSH至openwrt
# 使用sed替换opkg源地址
sed -i 's_downloads\.openwrt\.org_mirrors.ustc.edu.cn/lede_' /etc/opkg/distfeeds.conf

# 验证更换opkg源地址成功
opkg update
{% endhighlight %}

- 安装luci界面并启动web服务
{% highlight bash %}
opkg install luci
/etc/init.d/uhttpd enable
/etc/init.d/uhttpd start
{% endhighlight %}

- 浏览器访问openwrt地址
- 接下来在openwrt上安装软件并且配置，WAN口的配置放到最后

## 0x01 科学上网
- install
{% highlight bash %}
opkg install shadowsocks-libev-ss-local shadowsocks-libev-ss-redir shadowsocks-libev-ss-rules shadowsocks-libev-config shadowsocks-libev-ss-tunnel shadowsocks-libev-ss-server
opkg install luci-app-shadowsocks-libev
{% endhighlight %}

- [update list settings](https://github.com/shadowsocks/luci-app-shadowsocks/wiki/use-crontab-to-update-the-ignore.list)
{% highlight text %}
touch /root/update_ignore_list
chmod +x /root/update_ignore_list
{% endhighlight %}

使用如下的脚本并加入定时任务
{% highlight text %}
#!/bin/sh

set -e -o pipefail

wget -O- 'http://ftp.apnic.net/apnic/stats/apnic/delegated-apnic-latest' | awk -F\| '/CN\|ipv4/ { printf("%s/%d\n", $4, 32-log($5)/log(2)) }' >  /tmp/ignore.list

mv /tmp/ignore.list /etc/

if pidof ss-redir>/dev/null; then
    /etc/init.d/shadowsocks-libev restart
fi
{% endhighlight %}

- ss_redir设置(大陆白名单)
{% highlight text %}
Source settings -> Scr default设置为checkdst
Destination settings -> Dst ip/net bypass file设置为/etc/ignore.list
                     -> Dst default设置为forward
{% endhighlight %}

## 0x02 DNS
#### dnscrypt-proxy
- [dnscrypt-proxy的官方安装步骤](https://github.com/DNSCrypt/dnscrypt-proxy/wiki/Installation-on-OpenWRT)

- 不要使用openwrt opkg中的dnscrypt-proxy(版本老, 功能支持较少)

- 直接在openwrt上从[github](https://github.com/DNSCrypt/dnscrypt-proxy/releases)下载最新的release
{% highlight bash %}
# 安装依赖的CA证书
opkg install ca-bundle
cd /tmp
# 根据实际情况选择版本
wget https://github.com/DNSCrypt/dnscrypt-proxy/releases/download/2.0.45/dnscrypt-proxy-linux_arm64-2.0.45.tar.gz -O dnscrypt-proxy.tar.gz 
tar -zxvf dnscrypt-proxy.tar.gz 
cp linux-arm64/dnscrypt-proxy /usr/sbin/dnscrypt-proxy # 注意路径
wget https://raw.githubusercontent.com/etam/DNS-over-HTTPS-for-OpenWRT/master/dnscrypt-proxy -O /etc/init.d/dnscrypt-proxy
chmod +x /usr/sbin/dnscrypt-proxy
chmod +x /etc/init.d/dnscrypt-proxy
cp linux-arm64/example-dnscrypt-proxy.toml /etc/config/dnscrypt-proxy.toml
{% endhighlight %}

- 配置`/etc/config/dnscrypt-proxy.toml`
{% highlight text %}
server_names = ['google', 'yandex', 'cloudflare']     # 取消对于server_names的注释
listen_addresses = ['127.0.0.1:55553', '[::1]:55553'] # 端口不要和其它服务冲突
force_tcp = false                                     # 国内网络环境下可以全部走tcp
proxy = 'socks5://127.0.0.1:1080'                     # 建议使用代理
fallback_resolver = ['223.5.5.5:53', '1.1.1.1:53']    # fallback的DNS也可以设置为国内DNS
ignore_system_dns = true                              # 不使用ISP提供的DNS
netprobe_address = '114.114.114.114:53'               # 用于检测网络的地址，也建议更换为国内的DNS
cache = false                                         # ChinaDNS的机制要求上游DNS服务器禁用缓存
{% endhighlight %}

- 验证`dnscrypt-proxy`
{% highlight bash %}
# 验证配置文件语法
dnscrypt-proxy -config /etc/config/dnscrypt-proxy.toml -check
dnscrypt-proxy -config /etc/config/dnscrypt-proxy.toml -resolve www.google.com
{% endhighlight %}

- 启动`dnscrypt-proxy`服务
{% highlight bash %}
/etc/init.d/dnscrypt-proxy enable
/etc/init.d/dnscrypt-proxy start
{% endhighlight %}

#### [china-dns](https://github.com/aa65535/openwrt-chinadns)
- 为opkg加入一个[新的源](http://openwrt-dist.sourceforge.net/)
{% highlight bash %}
wget http://openwrt-dist.sourceforge.net/openwrt-dist.pub
opkg-key add openwrt-dist.pub

# use the following command to get architecture:
opkg print-architecture | awk '{print $2}'
# 比如树莓派是aarch64_cortex-a53

# 加入源，注意替换aarch64_cortex-a53
echo "src/gz openwrt_dist http://openwrt-dist.sourceforge.net/packages/base/aarch64_cortex-a53" >>/etc/opkg/customfeeds.conf
echo "src/gz openwrt_dist_luci http://openwrt-dist.sourceforge.net/packages/luci" >>/etc/opkg/customfeeds.conf
{% endhighlight %}

- 安装china-dns
{% highlight bash %}
opkg update
opkg install ChinaDNS
opkg install luci-app-chinadns
{% endhighlight %}

- 配置china-dns
1. 默认 DNS 服务器端口为 `5353`, 可使用`LuCI`进行配置  
2. 可搭配路由器自带的Dnsmasq使用 借助其 DNS 缓存提升查询速度  
   > LuCI 中定位至「网络 - DHCP/DNS」  
   >「基本设置」 **本地服务器** 填写 `127.0.0.1#5353`  
   >「HOSTS和解析文件」勾选 **忽略解析文件**  
3. 更新 `/etc/chinadns_chnroute.txt`
{% highlight bash %}
wget -O- 'http://ftp.apnic.net/apnic/stats/apnic/delegated-apnic-latest' | awk -F\| '/CN\|ipv4/ { printf("%s/%d\n", $4, 32-log($5)/log(2)) }' > /etc/chinadns_chnroute.txt
{% endhighlight %}

#### 修改`/etc/dnsmasq.conf`
{% highlight text %}
# No forward list:
server=/lan/
server=/internal/
server=/intranet/
server=/private/
server=/workgroup/
server=/10.in-addr.arpa/
server=/16.172.in-addr.arpa/
server=/168.192.in-addr.arpa/
server=/254.169.in-addr.arpa/
server=/d.f.ip6.arpa/
{% endhighlight %}

## 0x03 上海电信桥接及IPTV设置
- PPPoE设置中`Advanced Settings`中取消`Use DNS servers advertised by peer`
{% highlight text %}
这个比较迷默认情况下会导致DNS行为和预期的不一致
root@OpenWrt:/etc# cat /etc/resolv.conf
# Interface wan
nameserver 116.228.111.118
nameserver 180.168.255.18
{% endhighlight %}

- IPTV VLAN设置，只需要参考VLAN51和85(如果路由器有switch芯片)
{% highlight text %}
config switch_vlan
        option device 'switch0'
        option vlan '4'
        option ports '2t 1t 4t'
        option vid '51'

config switch_vlan
        option device 'switch0'
        option vlan '6'
        option ports '2t 1t 4t'
        option vid '85'
{% endhighlight %}

![](/img/openwrt-vlan.JPG)

- IPTV VLAN设置，只需要参考VLAN51和85(如果路由器没有switch芯片,比如软路由)
{% highlight text %}
config interface 'IPTV_VLAN51'
        option proto 'none'
        option ifname 'eth0.51 eth3.51'
        option type 'bridge'
        option delegate '0'

config interface 'IPTV_VLAN85'
        option proto 'none'
        option ifname 'eth0.85 eth3.85'
        option type 'bridge'
        option delegate '0'
{% endhighlight %}

![](/img/openwrt-vlan-x86_1.JPG)
![](/img/openwrt-vlan-x86_2.JPG)

- IPTV DHCP-option 在`/etc/dnsmasq.conf`加入

{% highlight text %}
dhcp-option-force=125,00:00:00:00:1a:02:06:48:47:57:2d:43:54:03:04:5a:58:48:4e:0a:02:20:00:0b:02:00:55:0d:02:00:2e:3c:1e:00:00:01:00:02:03:43:50:45:03:0e:45:38:20:45:50:4f:4e:20:52:4f:55:54:45:52:04:03:31:2e:30
dhcp-option=15
dhcp-option=28
dhcp-option=60,00:00:01:06:68:75:61:71:69:6E:02:0A:48:47:55:34:32:31:4E:20:76:33:03:0A:48:47:55:34:32:31:4E:20:76:33:04:10:32:30:30:2E:55:59:59:2E:30:2E:41:2E:30:2E:53:48:05:04:00:01:00:50
{% endhighlight %}


## 0x04 多wan口设置
- [openwrt的官方文档](https://openwrt.org/docs/guide-user/network/wan/multiwan/mwan3)

- 安装mwan3
{% highlight bash %}
opkg update
opkg install mwan3
opkg install luci-app-mwan3
{% endhighlight %}

- vlan设置(有交换芯片)
1. step 1: 从现有的LAN VLAN中选择选择一个作为第二个WAN口，在`Network`->`Switch中`由`untagged`更改为`off`

![](/img/openwrt-mwan-vlan1.JPG)

2. step 2: 利用刚刚空余出来的网卡新建一个VLAN，也就是图中VLAN3

![](/img/openwrt-mwan-vlan2.JPG)

3. step 3: 重启路由器

4. step 4: 在`Network`->`Interface`中新建一个`interface`，取一个名字如`wanb`

5. step 5: 在该`interface`的`Firewall settings`中设置为`wan`的firewall zone

6. step 6: 设置由路由器自身发起的包默认从哪里发起

7. step 7: 在`Network`->`Interface`中两个WAN口的`Advanced Settings`中`Use gateway metric`必须不同，可以分别是10和20

- vlan设置(软路由)
直接设置对应的物理网卡即可

![](/img/openwrt-mwan-vlan3.JPG)


## 0x05 端口聚合设置
- 安装[proto-bonding](https://openwrt.org/packages/pkgdata/proto-bonding)
{% highlight bash %}
opkg install proto-bonding
{% endhighlight %}

- 使用命令行配置,如果需要重启后仍然生效,需要添加到`/etc/rc.local`
- 默认会带一个bond0, 模式为balance-rr, 且貌似不能修改
{% highlight bash %}
# 查看链路聚合状态
cat /sys/class/net/bond0/bonding/mode
cat /proc/net/bonding/bond1
{% endhighlight %}
- 以下的配置设置了两个bonding, 一个为默认的balance-rr模式, 一个为lacp模式
{% highlight text %}
ip link add bond1 type bond
ip link set eth0 master bond1
ip link set eth1 master bond1
ip link add bond2 type bond mode 802.3ad
ip link set eth2 master bond2
ip link set eth3 master bond2
exit 0
{% endhighlight %}
- bonding生效后, 直接把bondx当作普通网卡设置即可

## 0x06 网络速度测试
- 安装[iperf3](https://openwrt.org/packages/pkgdata/iperf3)
{% highlight bash %}
opkg install iperf3
{% endhighlight %}
- 测速方式
{% highlight bash %}
# 服务端
iperf3 -s

# client端
iperf3 -c server_ip
{% endhighlight %}
