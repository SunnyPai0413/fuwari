---
title: OpensuseLeap安装Docker
published: 2025-09-03
tags:
  - Software
  - OS
category: Zheteng
draft: false
---
安装命令：
```bash
zypper install docker docker-compose gnome-keyring libsecret
```
然后可以去`systemd`配置文件改`env`配代理，也可以去`/etc/sysconfig/docker`添加代理`env`的`kv pair`

---

！！！**不是**一些教程里的！！！
```bash
zypper install docker python3-docker-compose
```
会报错：
```bash
localhost:~ # docker version
Client:
 Version:           28.3.3-ce
 API version:       1.51
 Go version:        go1.24.5
 Git commit:        bea959c7b
 Built:             Tue Jul 29 12:00:00 2025
 OS/Arch:           linux/amd64
 Context:           default

Server:
 Engine:
  Version:          28.3.3-ce
  API version:      1.51 (minimum version 1.24)
  Go version:       go1.24.5
  Git commit:       bea959c7b
  Built:            Tue Jul 29 12:00:00 2025
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          v1.7.27
  GitCommit:        05044ec0a9a75232cad458027ca83437aae3f4da
 runc:
  Version:          1.2.6
  GitCommit:        v1.2.6-0-ge89a29929c77
 docker-init:
  Version:          0.2.0_catatonit
  GitCommit:
localhost:~ # docker run --rm hello-world
Unable to find image 'hello-world:latest' locally
docker: error getting credentials - err: exit status 1, out: `GDBus.Error:org.freedesktop.DBus.Error.ServiceUnknown: The name org.freedesktop.secrets was not provided by any .service files`

Run 'docker run --help' for more information
```