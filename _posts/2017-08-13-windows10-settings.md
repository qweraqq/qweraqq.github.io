---
layout: post
title: "Win10环境配置(持续更新中)"
date: 2017-08-13 00:00:00 +0800
author: xiangxiang
categories: env-settings windows
tags: [win10 env-setings chocolatey]
---
记录自己的Windows10装机后的软件配置

## Win10版本选择
- 首选Long-Term Servicing Channel版本的Windows
- 目前使用`Windows 10 Enterprise LTSC 2021`
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

## Win10首次启动
- 不使用在线账号，新建一个本地离线账号
- 不要使用默认的推荐配置，把各种上报信息的功能全部关闭
- 进入windows以后使用`	
slmgr /skms YOUR_KMS_SERVER` 设置kms服务器激活windows
- 在windows设置里面的`隐私`菜单继续关闭上报服务
- 如果不是中文版的win，需要`Control Panel -> Clock and Region -> Region -> (Administrative tab) -> Current language for non-Unicode programs`设置成`Chinese`
- 等Windows update安装更新和驱动
- 如果Windows update有驱动没有安装的，再手动安装驱动

## WSL
### 安装
- [Installation Guide](https://docs.microsoft.com/en-us/windows/wsl/install)
- 下载并安装WSL2依赖的二进制[https://wslstorestorage.blob.core.windows.net/wslblob/wsl_update_x64.msi](https://wslstorestorage.blob.core.windows.net/wslblob/wsl_update_x64.msi)
- 使用管理员打开CMD, 执行
{% highlight console %}
wsl --set-default-version 2
wsl --install -d Ubuntu-20.04
ubuntu2004.exe config --default-user root
{% endhighlight %}

### 环境配置
- .bashrc配置
[The Ultimate Bad Ass .bashrc File](https://gist.github.com/zachbrowne/8bc414c9f30192067831fafebd14255c)

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
sudo sed -i "115c socks5 $windows_host 1080"  /etc/proxychains4.conf
alias px='proxychains4'
{% endhighlight %}

- Python
{% highlight console %}
px apt-get install git libssl-dev libffi-dev build-essential python3-pip python3-venv -y
python3 -m pip install pysocks
px python3 -m pip install pipenv
{% endhighlight %}


## 使用chocolatey安装常用软件
- [Installation Guide](https://chocolatey.org/install)
- 一些软件
{% highlight console %}
choco install -y vcredist-all
choco install -y 7zip.install
choco install -y python3
# choco install -y anaconda3
choco install -y mobaxterm
choco install -y winscp.install
choco install -y googlechrome
choco install -y adobereader
# choco install -y calibre
choco install -y ccleaner
choco install -y curl
choco install -y git.install
choco install -y crystaldiskinfo
choco install -y burp-suite-free-edition
choco install -y ghidra
choco install -y k-litecodecpackfull
choco install -y sysinternals
choco install -y vlc
choco install -y Wget
choco install -y wireshark
choco install -y greenshot
choco install -y vscode
choco install -y obsidian
choco install -y docker-desktop
choco install -y choco-cleaner
choco install -y openssl.light
# choco install -y irfanview
# choco install -y virtualbox
# choco install -y openconnect-gui

# https://docs.microsoft.com/en-us/troubleshoot/developer/visualstudio/cpp/libraries/c-runtime-packages-desktop-bridge
Add-AppxPackage -Path .\Microsoft.VCLibs.x64.14.00.Desktop.appx
choco install microsoft-windows-terminal -y
{% endhighlight %}
