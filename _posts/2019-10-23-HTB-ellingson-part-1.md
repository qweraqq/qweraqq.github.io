---
layout: post
title: "[writeup] HTB Ellingson-Part 1(User Flag)"
date: 2019-10-23 00:00:00 +0800
author: xiangxiang
categories: HTB writeup
tags: [htb hackthebox writup ellingson hashcat werkzeug]
---

 ![](/img/htb-ellingson-info.jpg){:width="512px"}

## 0x00 信息收集:
-  只有ip信息，第一步当然是先扫一下的, NMAP表示服务器只开放了22端口和80端口

{% highlight bash %}
~ # nmap -p- -Pn 10.10.10.139
Nmap scan report for 10.10.10.139
Host is up (0.32s latency).
Not shown: 65533 filtered ports
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
{% endhighlight %}

- 进一步扫描可以确认入口应该就是在web了

{% highlight bash %}
~ # nmap -p 22,80 -sV -sC -Pn 10.10.10.139
Starting Nmap 7.70 ( https://nmap.org )
Nmap scan report for 10.10.10.139
Host is up (0.30s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 49:e8:f1:2a:80:62:de:7e:02:40:a1:f4:30:d2:88:a6 (RSA)
|   256 c8:02:cf:a0:f2:d8:5d:4f:7d:c7:66:0b:4d:5d:0b:df (ECDSA)
|_  256 a5:a9:95:f5:4a:f4:ae:f8:b6:37:92:b8:9a:2a:b4:66 (ED25519)
80/tcp open  http    nginx 1.14.0 (Ubuntu)
|_http-server-header: nginx/1.14.0 (Ubuntu)
| http-title: Ellingson Mineral Corp
|_Requested resource was http://10.10.10.139/index
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
{% endhighlight %}

- 基础的NIKTO扫描及DIRBUSTER扫描并没有带来可用的结果

- 访问网站可以看到这样的页面, Ellingson Mineral Company源自于电影[Hackers(1995)](https://en.wikipedia.org/wiki/Hackers_(film)) ,可能看一下电影会有帮助哦

  ![](/img/htb-ellingson-site.jpg)

- 网站中比较有意思的是三篇article, 访问地址是`http://10.10.10.139/articles/1-3`

{% highlight txt %}
A recent unknown intruder penetrated using a super user account giving him access to our entire system. 
Yesterday the ballest program for a supertanker training model mistakenly thought the vessel was empty and flooded it's tanks.
This caused the vessel to capsize, a virus planted within the Ellingson system claimed responsibility and threatened to capsize more vessels unless five million dollars are transfered to their accounts.
{% endhighlight %}

{% highlight txt %}
Due to the recent security issues we have implemented protections to block brute-force attacks against network services.
As a result if you attempt to log into a service more then 5 times in 1 minute you will have your access blocked for 5 minutes. 
Additional malicious activity may also result in your connection being blocked, 
please keep this in mind and do not request resets if you lock yourself out ... take the 5 minutes and ponder where you went wrong :) 
{% endhighlight %}

{% highlight txt %}
We have recently detected suspicious activity on the network.
Please make sure you change your password regularly and read my carefully prepared memo on the most commonly used passwords. 
Now as I so meticulously pointed out the most common passwords are. Love, Secret, Sex and God -The Plague 
{% endhighlight %}

- 网站index主页上有显示大大的`Site Under Construction`

**简单的分析web上的内容可以想到**:

1. 可能有弱口令
2. 可能有类似fail2ban的机制，实际用hydra去brute force SSH的口令就发现会被ban
3. 建设中的网站可能会开一些DEBUG功能

## 0x01 Werkzeug DEBUGGER RCE
- `http://10.10.10.139/articles/3`看上去像一个动态页面，顺手在后面加一个引号'，
访问`http://10.10.10.139/articles/3'`，就发现了Werkzeug Debugger的页面

![](/img/htb-ellingson-werkzeug1.jpg)

- 查询一下werkzeug的文档，发现了红色醒目的Danger

![](/img/htb-ellingson-werkzeug2.jpg)

- 利用Werkzeug DEBUGGER执行简单的SHELL命令

![](/img/htb-ellingson-werkzeug3.jpg)

- 本地生成一个ssh公私密钥对

{% highlight bash %}
Ellingson # ssh-keygen -f ellingon_rsa
Generating public/private rsa key pair.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in ellingon_rsa.
Your public key has been saved in ellingon_rsa.pub.
The key fingerprint is:
SHA256:5icDgOqsC9XV+he4cQEmJf3LmerxGYMitxKFoK2Abdo root@xxubuntu
The key's randomart image is:
+---[RSA 2048]----+
|      oo+        |
|  ..   =..       |
|.+..... ...      |
|+.+..o.. ...     |
|o=. ..o S.o+     |
|=.E .  = ==.     |
|.o  ..o O.=      |
|o   .o o.B +     |
|o.   .... o      |
+----[SHA256]-----+
Ellingson # cat ellingon_rsa.pub
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC9uREIY6O9nyFU01qievFnCj5t0ln5nOxGFB7MO8kZ4MT/tEwpCLGfwSU/ig1R6zJS8bB0334OKAHbUphSIiRbWhTcLukuV5C/jqOCajNSxc7BQGE5h/uy6nfIjrdO7dvvoVmjib+PuaojJz7XgiNj8IK96faLlDjnZRDbkNyQoRFiqAI/ZD8Th//hGsl3IN+Z+6fhYjBcg16iGEfWS0F8jzF8UAp9Nr9WzGSc7r/u9iBWMLFdpB1oJSzb6IaV28Vpx1hWYX07EXDd/x8qmYUXFsdurT4ZEsCPzKxFzthEXDwAogD1V9TdQ5l4mYjeJIMZuhM44y2EPPiXndBwLwyv root@xxubuntu
{% endhighlight %}

- 写入hal用户的.authorized_keys文件中

![](/img/htb-ellingson-werkzeug4.jpg)
 
- SSH到hal账户,然而并没有user flag

![](/img/htb-ellingson-werkzeug5.jpg)

## 0x02 HASHCAT & USER FLAG
- [LinEnum](https://github.com/rebootuser/LinEnum) 枚举实际没有发现有用的东西

- 人工去翻文件，发现`/var/backups`目录下有一个`shadow.bak`文件并且`hal账户有访问权限`

![](/img/htb-ellingson-werkzeug6.jpg)

- 使用scp拉到本地

![](/img/htb-ellingson-werkzeug7.jpg)

- 祭出[hashcat](https://github.com/hashcat/hashcat) 
    - hashcat相比较于John有可以使用GPU的优势
    - 字典选择了[rockyou](https://github.com/brannondorsey/naive-hashcat/releases/download/data/rockyou.txt)
    - -m 用1800是因为`shadow.bak`中的密码hash是$6$开头

![](/img/htb-ellingson-hashcat1.jpg)
 
- 很快就跑出了第一个密码，但实际并没有作用...

![](/img/htb-ellingson-hashcat2.jpg)
 
- 很久以后跑出了第二个密码

![](/img/htb-ellingson-hashcat3.jpg)
 
- 使用笔记本上W4130这样的破显卡，默认参数大概4个小时不到可以跑完rockyou字典，过程中GPU占用率不到20%，参数上应该可以做优化

 ![](/img/htb-ellingson-hashcat4.jpg)
 
- ssh至margo账户，获得userflag

 ![](/img/htb-ellingson-userflag.jpg)
 
