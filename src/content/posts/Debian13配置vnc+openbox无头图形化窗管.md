---
title: 简记Debian13配置vnc+openbox无头图形化窗管
published: 2025-08-27
tags:
  - Software
  - OS
category: Zheteng
draft: false
---
1. 更新系统
```bash
sudo apt update && sudo apt upgrade -y
```
2. 安装软件包
```bash
sudo apt install -y openbox tightvncserver xserver-xorg-core
```
3. 添加普通用户
```bash
adduser vncuser # 使用adduser命令，自动创建家目录
```
4. 添加root权限（可选），切换到`vncuser`
```bash
sudo usermod -aG sudo vncuser # 按需添加sudo权限
su - vncuser # 切换到该用户操作
```
5. 设置vnc密码
```bash
vncpasswd # 上限8位
```
6. 配置 VNC 启动脚本
```bash
nano ~/.vnc/xstartup
```
- 编辑配置：
```sh
#!/bin/sh
unset SESSION_MANAGER
unset DBUS_SESSION_BUS_ADDRESS
[ -r $HOME/.Xresources ] && xrdb $HOME/.Xresources
exec openbox-session  # 直接启动openbox，无冗余命令
```
- 添加执行权限：
```bash
chmod +x ~/.vnc/xstartup
```
7. 自启动服务配置
```bash
sudo nano /etc/systemd/system/vncserver@.service
```
- 编辑systemd
```ini
[Unit]
Description=TightVNC Server on :%i
After=network.target

[Service]
Type=forking
User=vncuser
Group=vncuser
ExecStart=/usr/bin/vncserver -depth 24 -geometry 1024x768 :%i
ExecStop=/usr/bin/vncserver -kill :%i

[Install]
WantedBy=multi-user.target
```
- 启用服务（以 display :1 为例）：
```bash
sudo systemctl daemon-reload
sudo systemctl enable vncserver@1.service
sudo systemctl start vncserver@1.service
```
8. 创建缺失的`.Xauthority`文件
```bash
# 手动创建.Xauthority文件（空文件即可，VNC会自动写入内容）
touch ~/.Xauthority

# 设置正确权限（确保属于当前用户）
chmod 600 ~/.Xauthority
```