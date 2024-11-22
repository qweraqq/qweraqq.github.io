---
layout: post
title: "离线环境配置(持续更新中)"
date: 2024-11-16 00:00:00 +0800
author: xiangxiang
categories: env-settings windows
tags: [win10 env-setings offline]
---
离线状态下配置环境真的头疼

## WSL
- Enable WSL in Windows
- Download WSL-Linux from [https://aka.ms/wslubuntu2204](https://aka.ms/wslubuntu2204) refs [https://learn.microsoft.com/en-us/windows/wsl/install-manual](https://learn.microsoft.com/en-us/windows/wsl/install-manual)
- Download debs from another PC with Internet access, refs [https://unix.stackexchange.com/questions/408346/how-to-download-package-not-install-it-with-apt-get-command](https://unix.stackexchange.com/questions/408346/how-to-download-package-not-install-it-with-apt-get-command)

```bash
apt-get update
apt-get dist-upgrade --download-only
# /var/cache/apt/archives

apt-get dist-upgrade

apt-get install --download-only git libssl-dev libffi-dev build-essential python3-pip python3-venv

# https://devguide.python.org/getting-started/setup-building/#install-dependencies
```


## Windows Terminal
- Download VCLibs from [https://docs.microsoft.com/en-us/troubleshoot/developer/visualstudio/cpp/libraries/c-runtime-packages-desktop-bridge](https://docs.microsoft.com/en-us/troubleshoot/developer/visualstudio/cpp/libraries/c-runtime-packages-desktop-bridge)
- Download Windows Terminal from [https://github.com/microsoft/terminal/releases](https://github.com/microsoft/terminal/releases)
- Install VCLibs && extract Windows-terminal zip && Run

```powershell
# Administrator
Add-AppxPackage -Path .\Microsoft.VCLibs.x64.14.00.Desktop.appx
```

## VSCode
- Download from [https://code.visualstudio.com/docs/?dv=win64user](https://code.visualstudio.com/docs/?dv=win64user)

- For remote server, refs [https://stackoverflow.com/questions/56671520/how-can-i-install-vscode-server-in-linux-offline](https://stackoverflow.com/questions/56671520/how-can-i-install-vscode-server-in-linux-offline)

```bash
commit_id=f06011ac164ae4dc8e753a3fe7f9549844d15e35

# Download url is: https://update.code.visualstudio.com/commit:${commit_id}/server-linux-x64/stable
curl -sSL "https://update.code.visualstudio.com/commit:${commit_id}/server-linux-x64/stable" -o vscode-server-linux-x64.tar.gz

mkdir -p ~/.vscode-server/bin/${commit_id}
tar zxvf vscode-server-linux-x64.tar.gz -C ~/.vscode-server/bin/${commit_id} --strip 1
touch ~/.vscode-server/bin/${commit_id}/0
```

- Extensions
    + [remote-wsl](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-wsl)
    + [python](https://marketplace.visualstudio.com/items?itemName=ms-python.python)
    + [eslint](https://marketplace.visualstudio.com/items?itemName=dbaeumer.vscode-eslint)
    + [prettier](https://marketplace.visualstudio.com/items?itemName=esbenp.prettier-vscode)
    + [beautify](https://marketplace.visualstudio.com/items?itemName=HookyQR.beautify)
    + [vetur](https://marketplace.visualstudio.com/items?itemName=octref.vetur)

## Python
- refs [https://learn.microsoft.com/en-us/azure-data-studio/notebooks/notebooks-python-offline-installation](https://learn.microsoft.com/en-us/azure-data-studio/notebooks/notebooks-python-offline-installation)

```bash
python -m venv some-path

# Internet Access
python -m pip download -r requirements.txt -d wheelhouse

# No Internet Access
python -m pip install -r requirements.txt --no-index --find-links wheelhouse

```

## Node
- Download `nvm-noinstall.zip` from [https://github.com/coreybutler/nvm-windows](https://github.com/coreybutler/nvm-windows)
- Set Windows Env Variable `NVM_HOME` to nvm zip location 
- Set Windows Env Variable `NVM_SYMLINK` to `%NVM_HOME%\nodejs`  **DO NOT create %NVM_HOME%\nodejs folder**
- Add `NVM_HOME` && `NVM_SYMLINK` to PATH
- Download node binary from [https://registry.npmmirror.com/binary.html?path=node/](https://registry.npmmirror.com/binary.html?path=node/)
- Extract to `NVM_HOME` and rename folder name to `vAA.BB.CC` (v22.11.0)

- Create `settings.txt` in `NVM_HOME` with contents

```txt
root: YOUR_NVM_HOME    # D:\nvm
path: YOUR_NVM_SYMLINK # D:\nvm\nodejs
arch: 64
proxy: none
```

- Veriry

```bash
nvm list
nvm use 22.11.0
node -v
```