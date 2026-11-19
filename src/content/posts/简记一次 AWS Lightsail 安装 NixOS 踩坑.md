---
title: 简记一次 AWS Lightsail 安装 NixOS 踩坑  
published: 2026-11-18  
tags:
  - Software  
  - OS  
  - NixOS  
category: Zheteng  
draft: false 
---

最近准备把手头的云服务器和本地设备统一交给 `OpenTofu + NixOS Flakes` 管理。

不过 AWS Lightsail 没有 NixOS 官方镜像，于是打算用 [nixos-infect](https://github.com/elitak/nixos-infect) 在 Debian 上原地安装。

本以为一条命令就能解决，结果从 Swap、tmpfs 一路折腾到 GRUB，差点把服务器玩成砖。

最后靠 [lightsail-to-nixos](https://github.com/hujimm/lightsail-to-nixos) 成功解决。

不想看折腾过程的，可以直接跳到 [**懒人直达：安装命令 ↓**](#lazy-install)。

## 一、环境

|项目|配置|
|---|---|
|云服务商|AWS Lightsail|
|实例类型|General Purpose（$5/月）|
|原系统|Debian 13 (Trixie)|
|CPU 架构|x86_64|
|内存|512 MB（系统报告约 442 MiB）|
|系统盘|20 GB SSD|
|分区表 / 文件系统|GPT / ext4|
|引导方式|Legacy BIOS (?)|
|目标系统|NixOS 26.05|

512 MB 内存跑个轻量节点倒也够用，不过安装 NixOS 就没那么顺利了。

## 二、踩坑记录

### 1. Swap 把 /tmp 撑爆了

最开始直接使用 `nixos-infect`，设置 `PROVIDER=lightsail`，安装时出现：

```
dd: error writing '/tmp/nixos-infect.xxxxx.swp':
No space left on device
```

第一反应是：20 GB 的磁盘怎么这么快就满了？

检查后发现，`/tmp` 是只有约 222 MiB 的 `tmpfs`。

好家伙，脚本想创建约 1 GiB 的 Swap 解决内存不足，结果把 Swap 文件写进了内存文件系统。

解决方法也简单，下载脚本，把 Swap 文件路径从 `/tmp` 改到磁盘上的 `/var/tmp`。

### 2. Nix 安装器又把 /tmp 撑爆了

Swap 解决后，继续报错：

```
downloading Nix 2.35.6 binary tarball ...
to '/tmp/nix-binary-tarball-unpack.xxxxxxxx'

curl: (23) Failure writing output to destination
```

这回是 Nix 安装器自己还在使用 `/tmp`。

设置 `TMPDIR=/var/tmp` 后解决，终于看到了：

```
Installation finished!
```

本以为大功告成，没想到重启后直接进入了：

```
UEFI Interactive Shell v2.2
EDK II
```

### 3. GRUB 安装成功，但系统没启动

回头翻日志：

```
Installing for i386-pc platform.
Installation finished. No error reported.
```

`i386-pc` 是 BIOS GRUB 的目标平台，而实例却进入了 UEFI Shell。

怀疑是 BIOS/UEFI 引导方式不匹配。不过失败实例没有保留安装前的启动模式检测结果，无法完全确认。

重新创建 Debian 13 实例，检测发现：

```
BOOT MODE: Legacy BIOS
```

磁盘同时存在 BIOS boot 分区和 EFI System Partition，但实际通过 Legacy BIOS 启动。

所以不能仅凭有没有 EFI 分区判断启动方式，得检查固件实际使用的引导模式。

### 4. NixOS 26.05 还有个隐藏坑

排查期间发现，NixOS 26.05 默认启用 `systemd-initrd`，而 `nixos-infect` 首次启动时使用的 `lustrate` 依赖传统 Bash stage-1。

这可能导致内核启动成功，但旧系统迁移失败，最终 SSH 失联。

相关问题：[nixos-infect Issue #276](https://github.com/elitak/nixos-infect/issues/276)

这和前面的 UEFI Shell 是两个独立问题，修好 GRUB 也不代表一定能正常开机。

## 三、最终解决方案

折腾半天，最后发现已经有人专门做了 Lightsail 适配：

[hujimm/lightsail-to-nixos](https://github.com/hujimm/lightsail-to-nixos)

基于 `nixos-infect`，增加了 BIOS/UEFI 检测、磁盘布局识别、磁盘 Swap、传统 initrd 兼容配置以及重启前检查。

而且作者测试的环境刚好也是 Debian 13 + Legacy BIOS + 442 MiB 内存 + 20 GB 磁盘。

~~早知道就不自己折腾了。~~

<a id="lazy-install"></a>

### 懒人直达：安装命令

以下操作会原地替换操作系统，建议使用全新实例，重要数据提前备份。

**1. 下载脚本**

```
curl -fsSL \
  https://raw.githubusercontent.com/hujimm/lightsail-to-nixos/main/lightsail-to-nixos.sh \
  -o lightsail-to-nixos.sh
```

**2. 预检查**

```
bash lightsail-to-nixos.sh --dry-run --lang zh
```

确认磁盘、引导方式和 SSH 配置没有问题后再继续。

**3. 执行安装**

```
sudo TMPDIR=/var/tmp bash lightsail-to-nixos.sh --yes --lang zh
```

如果想自己控制重启时间，可以加 `--no-reboot`。

这次终于成功进入 NixOS 了。

安装完成后，可以检查一下：

```
nixos-version
systemctl --failed
df -h /
ls -ld /old-root
```

然后再测试关机、开机和重复重启，确认引导和网络正常。

没问题后，按照项目说明执行 `--cleanup` 清理旧系统。清理之前最好保留快照，免得又把自己送走了。

## 四、后续

目前只是完成了 NixOS 初始化，后续准备把生成的 `hardware.nix` 纳入 Flakes 仓库。

计划中的管理方式：

- `OpenTofu`：管理云实例、DNS、网络、存储桶等资源。
    
- `NixOS Flakes`：管理系统、软件包、网络和 systemd 服务。
    
- `Git`：统一保存配置，人工审核后再部署。
    

`nixos-infect` 和 `lightsail-to-nixos` 只负责首次初始化，以后正常通过 NixOS 的配置管理和升级机制维护系统。

另外需要注意，`nixos-infect` 的旧版初始化机制与 NixOS 26.11 存在兼容性问题，不能直接照搬 26.05 的注入流程。

但完成初始化和旧系统清理后，仍可以按正常的 NixOS 升级流程评估新版系统。

最后总结一下：

|问题|解决方案|
|---|---|
|Swap 写满 tmpfs|改用磁盘目录|
|Nix 安装器写满 /tmp|`TMPDIR=/var/tmp`|
|GRUB 引导异常|检测实际 BIOS/UEFI 模式|
|systemd-initrd 与 lustrate 不兼容|初始化阶段使用传统 initrd|

这次最大的教训是，**系统配置写错了还能回滚，首次安装把引导搞炸了可就不一定了。**

尤其是 Lightsail 这种缺少方便的底层救援手段的云服务器，切换之前还是先在测试环境跑一遍为妙。

感谢 [nixos-infect](https://github.com/elitak/nixos-infect) 和 [lightsail-to-nixos](https://github.com/hujimm/lightsail-to-nixos) 两个项目的作者。

折腾结束，终于可以继续折腾 Flakes 了。