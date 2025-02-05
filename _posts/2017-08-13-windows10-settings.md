---
layout: post
title: "Win11环境配置(持续更新中)"
date: 2017-08-13 00:00:00 +0800
author: xiangxiang
categories: env-settings windows
tags: [win10 env-setings chocolatey]
---
记录自己的Windows10装机后的软件配置

## Win10版本选择
- 首选Long-Term Servicing Channel版本的Windows
- 目前使用`Windows 11 Enterprise LTSC 2024`
- 下载iso后可以使用免费的[rufus](https://rufus.ie/)制作启动盘

## Win10安装注意事项
- 在安装选择`Custom: install Windows only (advnaced)`
 ![](/img/windows-10-custom-install.jpg)

- 在选择安装位置的页面，使用`shift`+`F10`的快捷键调出命令行
{% highlight console %}
> diskpark              # 使用diskpart手动分区

DISKPART> list disk
DISKPART> select disk x # x为需要格式化的安装磁盘
DISKPART> detail disk   # 查看完整的磁盘信息确认没有选择错误
DISKPART> clean         # 清空磁盘上现有的分区信息
DISKPART> convert gpt   # 使用GPT分区
DISKPART> exit          # 退出diskpart
{% endhighlight %}

- 然后在图像化的选择页面刷新一下，选择刚刚清楚过分区的磁盘，一直下一步即可

## Win11首次启动
- 不使用在线账号，新建一个本地离线账号
- 不要使用默认的推荐配置，把各种上报信息的功能全部关闭
- 进入windows以后使用KMS激活

{% highlight console %}
slmgr /ipk M7XTQ-FN8P6-TTKYV-9D4CC-J462D
slmgr /skms kms.03k.org
{% endhighlight %}

- 在windows设置里面的`隐私`菜单继续关闭上报服务
- 如果不是中文版的win，需要`Control Panel -> Clock and Region -> Region -> (Administrative tab) -> Current language for non-Unicode programs`设置成`Chinese`
- 等Windows update安装更新和驱动
- 如果Windows update有驱动没有安装的，再手动安装驱动

## Windows Features

`Control Panel -> Programs -> Turn Windows features on or off`

Enable
- .NET Framework 3.5
- Telnet Client
- TFTP Client
- Windows Sandbox
- Windows Subsystem for Linux
- Virtual Machine Platform

## Windows Store
使用管理员打开CMD, 执行

{% highlight console %}
wsreset -i
restart PC
{% endhighlight %}

## Office LTSC
- [https://learn.microsoft.com/en-us/office/ltsc/2024/deploy](https://learn.microsoft.com/en-us/office/ltsc/2024/deploy)
- Download [Office Deploy Tool](https://www.microsoft.com/en-us/download/details.aspx?id=49117)
- Create the [configuration.xml](https://config.office.com/deploymentsettings)
- Install (CMD)

{% highlight console %}
setup.exe /configure office-2024-ltsc.xml
{% endhighlight %}

## Explorer menu - Show More as Default

{% highlight console %}
reg add HKCU\Software\Classes\CLSID\{86ca1aa0-34aa-4e8b-a509-50c905bae2a2}\InprocServer32 /ve /d "" /f
{% endhighlight %}

## Java
- Download JDK from [https://adoptium.net/](https://adoptium.net/)
- Download Gradle from [https://gradle.org/install/#manually](https://gradle.org/install/#manually)
- SET Windows ENV
    + `JAVA_HOME`
    + `GRADLE_HOME`
    + `GRADLE_USER_HOME`
- Add to PATH
    + `%JAVA_HOME%\bin\`
    + `%GRADLE_HOME%\bin\`

## Android
- SET Windows ENV: `ANDROID_HOME`

## VSCode Community
- Desktop Development C++ MSVC

## MSTSC force TCP
- [refs](https://answers.microsoft.com/en-us/windows/forum/all/advance-troubleshooting-for-remote-desktop/bd334727-f449-477d-af88-73910593dac8)

```
Follow the path mentioned below.
HKEY_LOCAL_MACHINE\SOFTWARE\Policies\Microsoft\Windows NT\Terminal Services\Client
Now, right-click the Client folder and choose New > DWORD (32-bit) Value.
Type fClientDisableUDP as the name of the new item.
Finally, double-click it, set its Value data to 1 and click the OK button.


Turn off UDP in Group Policy
Press the Windows key + R, type gpedit.msc, and click OK.
Navigate to the path below.
Computer Configuration > Administration Templates > Windows Components > Remote Desktop Services > Remote Desktop Connection Client
Now, double-click Turn Off UDP On Client in the right pane.
Then, tick the Enabled radio button and click OK.
```

## WSL
### 安装
- [Installation Guide](https://docs.microsoft.com/en-us/windows/wsl/install)
- 使用管理员打开CMD, 执行
{% highlight console %}
wsl --update
wsl --set-default-version 2
wsl --install -d Ubuntu-24.04
{% endhighlight %}

### 环境配置
- 代理
{% highlight console %}
sudo apt-get update
sudo apt-get install proxychains-ng -y
{% endhighlight %}

在.bashrc最后增加
{% highlight console %}
export windows_host=`ip route | grep default | awk '{print $3}'`
# export ALL_PROXY=socks5://$windows_host:1080
# export HTTP_PROXY=$ALL_PROXY
# export http_proxy=$ALL_PROXY
# export HTTPS_PROXY=$ALL_PROXY
# export https_proxy=$ALL_PROXY
sudo sed -i "161c socks5 $windows_host 1080"  /etc/proxychains4.conf
alias px='proxychains4'
{% endhighlight %}

- 常用依赖
{% highlight console %}
apt-get update
apt-get dist-upgrade
apt-get install git libssl-dev libffi-dev build-essential python3-pip python3-venv -y

# https://devguide.python.org/getting-started/setup-building/#build-dependencies
apt-get build-dep python3
apt-get install pkg-config
apt-get install build-essential gdb lcov pkg-config \
      libbz2-dev libffi-dev libgdbm-dev libgdbm-compat-dev liblzma-dev \
      libncurses5-dev libreadline6-dev libsqlite3-dev libssl-dev \
      lzma lzma-dev tk-dev uuid-dev zlib1g-dev


mkdir Projects
px git clone git@github.com:python/cpython.git
cd cpython
git checkout v3.13.1
./configure --enable-optimizations --with-lto
make
make test
sudo make altinstall


apt-get build-dep linux

apt-get install -y bc bison build-essential ccache curl flex g++-multilib gcc-multilib git git-lfs gnupg gperf imagemagick lib32readline-dev lib32z1-dev libelf-dev liblz4-tool libncurses-dev libsdl1.2-dev libssl-dev libxml2 libxml2-utils lzop pngcrush rsync schedtool squashfs-tools xsltproc zip zlib1g-dev p7zip-full p7zip-rar dwarves cmake libdwarf-dev libdw-dev pkgconf linux-tools-generic linux-tools-common bpfcc-tools libbpfcc libbpfcc-dev linux-generic libbpf-dev python-is-python3

{% endhighlight %}

- Misc
    + [https://learn.microsoft.com/en-us/windows/wsl/tutorials/gui-apps](https://learn.microsoft.com/en-us/windows/wsl/tutorials/gui-apps)
    + git
{% highlight console %}
git config --global core.editor "vim"
git config --global user.name "YOUR_NAME"
git config --global user.email "YOUR_EMAIL"
git config --global core.eol lf
git config --global core.autocrlf false
{% endhighlight %}

## 使用chocolatey安装常用软件
- [Installation Guide](https://chocolatey.org/install)
- 一些软件
{% highlight console %}
choco install -y vcredist-all
choco install -y 7zip.install
choco install -y python312

# choco install -y anaconda3
choco install -y mobaxterm
choco install -y winscp.install
choco install -y googlechrome
choco install -y adobereader
# choco install -y calibre
choco install -y curl
choco install -y git.install
choco install -y crystaldiskinfo
# choco install -y burp-suite-free-edition
# -i 用于不安装依赖
choco install -i -y ghidra 
choco install -y k-litecodecpackfull
choco install -y sysinternals
choco install -y vlc
choco install -y Wget
choco install -y wireshark
choco install -y greenshot
choco install -y vscode
choco install -y obsidian
# choco install -y docker-desktop
choco install -y choco-cleaner
choco install -y openssl.light
# choco install -y irfanview
# choco install -y virtualbox
# choco install -y openconnect-gui

{% endhighlight %}
