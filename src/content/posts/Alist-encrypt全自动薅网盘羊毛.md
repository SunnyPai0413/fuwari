---
title: Alist-encrypt全自动薅网盘羊毛
published: 2024-09-30
tags:
  - Software
category: Zheteng
draft: false
---

近期本地服务器架构经常变动，引发了我对个人数据安全性的担忧。因此一番寻找后，综合考量价格和性能，选择了189网盘作为云备份提供商。

### 加密方案

由于有一些隐私文件，因此不便于直接上传到网盘。在仔细阅读 [alist文档](https://alist.nn.ci/) 后，找到了一个为alist的webdav提供加密的项目：alist-encrypt

这个项目不仅限于可以为alist提供加密，也可以为标准webdav接口提供加密。直接上传到encrypt目录就可以自动完成加密

![](https://assets.blog.edge.sunnypai.top/Alist-encrypt全自动薅网盘羊毛01.png)

### 存储配置

alist部署和添加存储的教程这里就不再提供。由于服务商接口变动较为频繁，请以官方文档为准。同时该方案无高可用性保证、分享时请遵循服务商EULA，避免不合理的挤占带宽或服务器资源。

[[安装教程] https://alist.nn.ci/guide/install/script.html](https://alist.nn.ci/guide/install/script.html)  
[[添加存储教程] https://alist.nn.ci/guide/drivers/](https://alist.nn.ci/guide/drivers/)  
[[配置加密教程] https://github.com/traceless/alist-encrypt](https://github.com/traceless/alist-encrypt)

### 加密配置

alist-encrypt 项目提供了两种部署方式：node.js 和 docker，并给出了docker compose文件。

```yaml
version: '3' #新版本docker compose可以省略version选项
services:
  alist-encrypt:
    image: prophet310/alist-encrypt:beta #beta标签不提供arm版本，arm请使用:beta-1.1
    restart: unless-stopped
    hostname: alist-encrypt
    container_name: alist-encrypt
    volumes:
      - ./alist-encrypt:/node-proxy/conf
    environment:
      TZ: Asia/Shanghai
      ALIST_HOST: 192.168.31.254:5254        # 建议加个设置项，修改为自己的ip和端口
    ports:
      - 5344:5344
    network_mode: bridge
```

`docker compose up -d` 启动后，输入<ip:5344/public/index.html>进入web管理面板，默认账号admin密码123456

路径填写alist网盘的绝对路径，endpoint的/dav不需要写；多个路径使用英文逗号隔  
具体配置见文章开头的图片

### 自动化配置

首先安装所需使用的包：

```bash
sudo apt-get -y install inotify-tools rclone
sudo -v ; curl https://rclone.org/install.sh | sudo bash
```

---

将alist-encrypt服务的dav endpoint保存到rclone：

```bash
rclone config #用本地目录的拥有者的身份配置
```

rclone配置文件储存在当前用户的家目录中，因此切换用户后无法读取对应的配置

---

为了便于本地NAS向网盘推送备份，我写了一个脚本以便本地目录出现变动后向网盘自动同步：

```bash
#!/bin/bash
inotifywait -m -r -e modify,create,delete /AppData/alist/local-drive |
while read path action file; do
    rclone sync /path/to/local/dir alist-encrypt:/remote/dir/encrypt --verbose --progress --delete-excluded
done
```

注：仅限于Linux和部分Unix系统，inotify基于inode实现文件系统级监听服务  
另：若目录过于庞大，例如超过524288个子目录，则inotify无法完成监听，需要额外配置以增大监听目录数量限制（我更建议使用其他方式实现）

```bash
find /path/to/local/dir -type d | wc -l #查看需要监听的目录的子目录数量
cat /proc/sys/fs/inotify/max_user_watches #查看inotify的监听目录限制
```

大多数情况下，默认限制足够使用

```bash
find /AppData/alist/local-drive/ -type d | wc -l
cat /proc/sys/fs/inotify/max_user_watches

16371
751311
---
du -sh /AppData/alist/local-drive/
3.2T    /AppData/alist/local-drive/
```

这是我的个人存储，3.2TB只有16371个子目录，距离我的默认限制751311还有数十倍的差距，因此在PB级以下基本无需担心

# 当然要是本地几PB的容量，也不需要薅网盘羊毛了吧（碎碎念）

---

最后则是自动化运行，使用systemd守护进程实现

```
[Unit]
Description=Alist Encrypted Drive Sync Service
After=network.target

[Service]
Type=simple
ExecStart=/path/to/inotify-sync.sh
StandardOutput=append:/var/log/alist-sync.log
StandardError=append:/var/log/alist-sync.log
Restart=always
RestartSec=5
User=root
# 设置环境变量
Environment=PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
# 如果需要更多变量，可以使用多个 Environment= 行
# Environment=VAR_NAME=value

[Install]
WantedBy=multi-user.target
```

在目录发生变动后，rclone就会自动开始同步

---

#### 更新：关于优先级和资源限制的配置 (Nov. 30th, 2024)

由于该systemd进程默认没有资源限制，在新增巨量小文件时会占用大量内存和CPU资源，导致系统宕机

因此需要使用 `Nice` 和 `MemoryLimit` 来限制该服务的资源使用

```
[Unit]
Description=Alist Encrypted Drive Sync Service
After=network.target

[Service]
Type=simple
Nice=10 # -20 至 19 之间，默认是0，数字越大优先级越低
MemoryLimit=5G # 可以更换K M G T等单位

ExecStart=/path/to/inotify-sync.sh
StandardOutput=append:/var/log/alist-sync.log
StandardError=append:/var/log/alist-sync.log
Restart=always
RestartSec=5
User=root
# 设置环境变量
Environment=PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
# 如果需要更多变量，可以使用多个 Environment= 行
# Environment=VAR_NAME=value

[Install]
WantedBy=multi-user.target
```