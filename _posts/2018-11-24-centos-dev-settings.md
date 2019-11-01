---
layout: post
title: "CentOS settings(持续更新中)"
date: 2018-11-24 00:00:00 +0800
author: xiangxiang
categories: env-settings centos
tags: [centos docker dev vmware virtualbox]
---
CentOS/RHEL是最流行的服务器操作系统，所以我选择CentOS作为开发部署环境。
本文主要是配置CentOS至中国国内的源并安装常用的开发依赖。

## CentOS安装
- 安装过程中网络配置hostname并打开网卡
- [替换yum源](https://lug.ustc.edu.cn/wiki/mirrors/help/centos)
{% highlight bash %}
#### centos7
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup
curl https://lug.ustc.edu.cn/wiki/_export/code/mirrors/help/centos?codeblock=3 --output /etc/yum.repos.d/CentOS-Base.repo
yum makecache
{% endhighlight %}

- 安装vmtools和Development Tools
{% highlight bash %}
sudo yum install -y open-vm-tools
reboot

#### 更新系统
sudo yum check-update
sudo yum update

#### 开发环境工具
yum groupinstall 'Development Tools' -y
{% endhighlight %}

## docker
- [centos7的docker安装](https://docs.docker.com/install/linux/docker-ce/centos/)
{% highlight bash %}
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
curl https://download.docker.com/linux/centos/docker-ce.repo --output /etc/yum.repos.d/docker-ce.repo
sudo sed -i 's+download.docker.com+mirrors.ustc.edu.cn/docker-ce+' /etc/yum.repos.d/docker-ce.repo
sudo yum makecache fast
sudo yum install docker-ce docker-ce-cli containerd.io
sudo systemctl enable docker
sudo systemctl start docker
{% endhighlight %}

- 修改docker hub地址 `/etc/docker/daemon.json` 后重启docker服务
{% highlight text %}
{
  "registry-mirrors": ["https://docker.mirrors.ustc.edu.cn/"]
}
{% endhighlight %}

## python
- python3
{% highlight bash %}
sudo yum -y install python-virtualenv
sudo yum -y install python3 python3-pip python3-wheel python3-devel
{% endhighlight %}

- [修改python源](https://mirrors.ustc.edu.cn/help/pypi.html) `$HOME/.config/pip/pip.conf`
```bash
mkdir -p /root/.config/pip
touch $HOME/.config/pip/pip.conf
```

{% highlight text %}
[global]
index-url = https://mirrors.ustc.edu.cn/pypi/web/simple
format = columns
{% endhighlight %}

## disable firewall
{% highlight bash %}
systemctl disable firewalld
{% endhighlight %}

## Clean disk space
{% highlight bash %}
find /var -name "*.log" -exec truncate {} --size 0 \;
yum clean all
rm -rf /var/cache/yum
rm -rf /var/tmp/yum-*
package-cleanup --quiet --leaves --exclude-bin | xargs yum remove -y
package-cleanup --oldkernels --count=1
history -c
{% endhighlight %}

## mount VMware shared folders
{% highlight bash %}
/usr/bin/vmhgfs-fuse .host:/ /mnt/hgfs -o subtype=vmhgfs-fuse,allow_other
{% endhighlight %}