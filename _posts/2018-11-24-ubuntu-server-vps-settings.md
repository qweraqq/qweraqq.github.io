---
layout: post
title: "Ubuntu server settings(持续更新中)"
date: 2017-08-13 00:00:00 +0800
author: xiangxiang
categories: env-settings vps
tags: [ubuntu vps ufw]
---
记录自己的vps基础配置


## SSH settings
- Add user
{% highlight console %}
adduser shen
usermod -aG sudo shen # add shen to root group 
{% endhighlight %}

-  Add new SSH key to non-root account
{% highlight console %}
su shen
cd ~
mkdir ~/.ssh
chmod 700 ~/.ssh
touch ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
{% endhighlight %}

- 本地generate SSH key

- Add ssh public key to `~/.ssh/authorized_keys`

- Change SSHD settings: `vim /etc/ssh/sshd_config`
{% highlight text %}
PasswordAuthentication no
PermitRootLogin no
Port 22221
{% endhighlight %}

- Restart SSH service
{% highlight console %}
sudo service ssh start
{% endhighlight %}

## bbr
- Enable bbr
{% highlight console %}
sudo echo "net.core.default_qdisc=fq" >> /etc/sysctl.conf
sudo echo "net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.conf
sudo sysctl -p
{% endhighlight %}

- Check bbr status
{% highlight console %}
sudo sysctl net.ipv4.tcp_available_congestion_control
# net.ipv4.tcp_available_congestion_control = bbr cubic reno
sudo lsmod | grep bbr
{% endhighlight %}

## Firewall
- Install ufw
{% highlight console %}
sudo apt-get install ufw
{% endhighlight %}

- Rules for ufw
{% highlight console %}
sudo ufw allow 22221/tcp # SSH
sudo ufw allow 25387 # SS
sudo ufw allow 28888 # ResilioSync
sudo ufw default deny incoming
sudo ufw default allow outgoing
{% endhighlight %}

- Enable ufw
{% highlight console %}
sudo ufw enable
{% endhighlight %}

## Shadowsocks
- Install
{% highlight console %}
sudo apt update
sudo apt install shadowsocks-libev
{% endhighlight %}

- Shadowsocks settings `/etc/shadowsocks-libev/config.json`

{% highlight text %}
{
    "server":"0.0.0.0",
    "server_port":25387,
    "local_address":"127.0.0.1",
    "local_port":1080,
    "password":"xxxxxxxxxxxxxxx",
    "timeout":600,
    "method":"aes-256-gcm"
}
{% endhighlight %}

## ResilioSync
- Install
{% highlight console %}
sudo echo "deb http://linux-packages.resilio.com/resilio-sync/deb resilio-sync non-free" | sudo tee /etc/apt/sources.list.d/resilio-sync.list
sudo apt-get install curl -y
sudo apt-get install gnupg -y
curl -LO http://linux-packages.resilio.com/resilio-sync/key.asc && sudo apt-key add ./key.asc
sudo apt-get update
sudo apt-get install resilio-sync
{% endhighlight %}

- Settings `/etc/resilio-sync/config.json`
{% highlight text %}
{
    "storage_path" : "/var/lib/resilio-sync/",
    "pid_file" : "/var/run/resilio-sync/sync.pid",

    "webui" :
    {
        "listen" : "0.0.0.0:28888"
    }
}
{% endhighlight %}

- Log on the ip:28888/gui
- Add new key and select storage at `/home/rslsync/xxx`
- Change /etc/resilio-sync/config.json webui listen settings ```0.0.0.0 -> 127.0.0.1```

## 开机自启动任务
- 教程[Other systemd deamon service](https://www.digitalocean.com/community/tutorials/how-to-use-systemctl-to-manage-systemd-services-and-units)

- `lib/systemd/system/intellij-activication.service`
{% highlight text %}
[Unit]
Description=Intellij activication service
After=network.target


[Service]
User=nobody
Group=nogroup
Type=simple
ExecStart=/usr/local/bin/IntelliJIDEALicenseServer_linux_amd64 -p 18413 -u shenxx  2>&1> /dev/null

[Install]
WantedBy=multi-user.target
{% endhighlight %}

- `lib/systemd/system/vlmscd.service`
{% highlight text %}
[Unit]
Description=KMS Server By vlmcsd
After=network.target

[Service]
Type=forking
PIDFile=/var/run/vlmcsd.pid
ExecStart=/usr/local/bin/vlmcsd-x64-glibc -p /var/run/vlmcsd.pid
ExecStop=/bin/kill -HUP $MAINPID
PrivateTmp=true
 
[Install]
WantedBy=multi-user.target
{% endhighlight %}
