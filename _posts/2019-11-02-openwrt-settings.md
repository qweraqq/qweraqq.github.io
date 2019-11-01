---
layout: post
title: "Openwrt settings(持续更新中)"
date: 2019-11-02 00:00:00 +0800
author: xiangxiang
categories: env-settings openwrt
tags: [openwrt chinadns dnscrypt-proxy dnsmasq]
---

上网究竟有多难，看完本文才知道

## 首次设置
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

## 科学上网
- install
{% highlight bash %}
opkg install shadowsocks-libev
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


## DNS
#### dnscrypt-proxy
- [dnscrypt-proxy的官方安装步骤](https://github.com/DNSCrypt/dnscrypt-proxy/wiki/Installation-on-OpenWRT)

- 不要使用openwrt opkg中的dnscrypt-proxy(版本老, 功能支持较少)

- 直接在openwrt上从[github](https://github.com/DNSCrypt/dnscrypt-proxy/releases)下载最新的release
{% highlight bash %}
# 安装依赖的CA证书
opkg install ca-bundle
cd /tmp
# 根据实际情况选择版本
wget https://github.com/DNSCrypt/dnscrypt-proxy/releases/download/2.0.29/dnscrypt-proxy-linux_arm64-2.0.29.tar.gz -O dnscrypt-proxy.tar.gz 
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
force_tcp = true                                      # 国内网络环境下建议全部走tcp
proxy = 'socks5://127.0.0.1:1080'                     # 建议使用代理
fallback_resolver = '119.29.29.29:53'                 # fallback的DNS建议设置为国内DNS
netprobe_address = '223.5.5.5:53'                     # 用于检测网络的地址，也建议更换为国内的DNS
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
