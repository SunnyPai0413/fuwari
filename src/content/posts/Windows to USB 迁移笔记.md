---
title: Windows to USB 迁移笔记
published: 2026-05-19
tags:
  - Software
  - OS
category: Zheteng
draft: false
---

准备切 Linux，但又怕一刀切翻车，于是先把原来的 Windows 克隆到外置 NVMe 里，当成过渡系统和应急 fallback。

先说结论：  
**成功从 USB 外置 NVMe 启动，原来的软件、配置、文件基本都还在。**

但是过程并不是克隆完就能跑，中间踩了几个坑：

- 克隆后无法启动：`0xc000000e`
- 修完引导后蓝屏：`0x0000007B`
- 能进系统后，关机/重启时 USB 硬盘盒提前断开
- RTL9210 硬盘盒能用，但不太适合长期 Windows-to-USB

---

## 0. 前情提要

我这次用的是 RTL9210 芯片的 NVMe USB 硬盘盒。

普通当移动硬盘没啥问题，但拿来跑 Windows 系统盘就有点边缘了。

因为 Windows-to-USB 启动时，不是“系统启动后再识别移动硬盘”，而是启动早期就要靠 USB 控制器、UASP/USBSTOR、桥接芯片、NVMe 驱动一路把系统盘读出来。

这条链路里只要有一环加载晚了，就会炸。

所以先叠甲：

> 这不是万能教程，只是一次实战记录。  
> 命令里的盘号和盘符一定要按自己机器实际情况改，别无脑复制。

---

## 1. 目标

我的目标很简单：

1. 把当前内置盘里的 Windows 克隆到外置 NVMe；
2. 保留软件、配置、个人文件；
3. 外置盘可以独立启动；
4. 内置盘腾出来装 Linux；
5. Windows 作为过渡和救命环境。

所以这里不适合重装系统。  
重装当然干净，但软件、注册表、授权、工作环境这些东西搬起来太痛苦。

因此路线就是：

```text
克隆现有系统，而不是重装
````

---

## 2. 第一坑：克隆后无法启动 0xc000000e

克隆完第一次启动，直接报：

```text
0xc000000e
```

大概意思是 Windows Boot Manager 找不到要启动的系统。

一般是：

- BCD 还指着原来的内置盘；
    
- 外置盘 EFI 分区没写好；
    
- 引导文件没生成完整。
    

解决方式：进 WinPE 或 Windows 安装 U 盘。

安装界面按：

```text
Shift + F10
```

打开命令行。

先用 `diskpart` 找外置盘：

```cmd
diskpart
list disk
select disk 1
list vol
```

这里 `disk 1` 只是示例，别照抄。

找到两个分区：

- EFI 分区：FAT32，一般 100MB 到 300MB；
    
- Windows 分区：NTFS，里面有 `Windows`、`Users`、`Program Files`。
    

假设 EFI 是卷 2，Windows 是卷 4：

```cmd
select vol 2
assign letter=S

select vol 4
assign letter=W

exit
```

确认 `W:` 真的是外置 Windows：

```cmd
dir W:\Windows
```

然后重建 UEFI 启动文件：

```cmd
bcdboot W:\Windows /s S: /f UEFI
```

这一步做完，`0xc000000e` 解决。

---

## 3. 第二坑：能引导，但蓝屏 0x7B

修完 BCD 后，Windows Boot Manager 能进了，但系统启动阶段蓝屏：

```text
0x0000007B
INACCESSIBLE_BOOT_DEVICE
```

这个报错就很典型了：

> 引导找到了 Windows，但 Windows 内核启动后找不到自己的系统盘。

原因也很好理解：  
原系统本来是从内置 NVMe/SATA 启动的，现在被我丢到了 USB 外置盘。  
USB / UASP / 存储相关驱动如果没有在启动早期加载，系统自然就找不到自己了。

---

## 4. 离线改注册表

再次进入 WinPE 或安装 U 盘命令行。

假设外置 Windows 分区还是 `W:`。

先备份注册表：

```cmd
copy W:\Windows\System32\Config\SYSTEM W:\SYSTEM.bak
```

加载外置系统的 SYSTEM 注册表：

```cmd
reg load HKLM\OFFSYS W:\Windows\System32\Config\SYSTEM
```

看当前 ControlSet：

```cmd
reg query HKLM\OFFSYS\Select
```

我这里是：

```text
Current        0x1
Default        0x1
```

所以后面改的是：

```text
ControlSet001
```

如果你那里不是 `0x1`，就按实际情况改。

---

## 5. 设置 BootDriverFlags

关键命令：

```cmd
reg add HKLM\OFFSYS\ControlSet001\Control /v BootDriverFlags /t REG_DWORD /d 20 /f
```

这里的 `20` 是十进制，也就是十六进制 `0x14`。

作用就是让 Windows 更早加载 USB 存储相关驱动。  
从内置盘克隆到 USB 后，这一步很关键。

---

## 6. 强制存储驱动早期启动

继续执行：

```cmd
reg add HKLM\OFFSYS\ControlSet001\Services\USBXHCI /v Start /t REG_DWORD /d 0 /f
reg add HKLM\OFFSYS\ControlSet001\Services\USBHUB3 /v Start /t REG_DWORD /d 0 /f
reg add HKLM\OFFSYS\ControlSet001\Services\UASPStor /v Start /t REG_DWORD /d 0 /f
reg add HKLM\OFFSYS\ControlSet001\Services\USBSTOR /v Start /t REG_DWORD /d 0 /f
reg add HKLM\OFFSYS\ControlSet001\Services\disk /v Start /t REG_DWORD /d 0 /f
reg add HKLM\OFFSYS\ControlSet001\Services\partmgr /v Start /t REG_DWORD /d 0 /f
reg add HKLM\OFFSYS\ControlSet001\Services\mountmgr /v Start /t REG_DWORD /d 0 /f
reg add HKLM\OFFSYS\ControlSet001\Services\volmgr /v Start /t REG_DWORD /d 0 /f
reg add HKLM\OFFSYS\ControlSet001\Services\storport /v Start /t REG_DWORD /d 0 /f
reg add HKLM\OFFSYS\ControlSet001\Services\stornvme /v Start /t REG_DWORD /d 0 /f
reg add HKLM\OFFSYS\ControlSet001\Services\storahci /v Start /t REG_DWORD /d 0 /f
```

有些服务项可能不存在，报错也不一定有问题，继续就行。

---

## 7. 处理 StartOverride

有些驱动就算 `Start=0`，也可能被 `StartOverride` 覆盖掉。

所以继续补一刀：

```cmd
reg add HKLM\OFFSYS\ControlSet001\Services\USBXHCI\StartOverride /v 0 /t REG_DWORD /d 0 /f
reg add HKLM\OFFSYS\ControlSet001\Services\USBHUB3\StartOverride /v 0 /t REG_DWORD /d 0 /f
reg add HKLM\OFFSYS\ControlSet001\Services\UASPStor\StartOverride /v 0 /t REG_DWORD /d 0 /f
reg add HKLM\OFFSYS\ControlSet001\Services\USBSTOR\StartOverride /v 0 /t REG_DWORD /d 0 /f
reg add HKLM\OFFSYS\ControlSet001\Services\stornvme\StartOverride /v 0 /t REG_DWORD /d 0 /f
reg add HKLM\OFFSYS\ControlSet001\Services\storahci\StartOverride /v 0 /t REG_DWORD /d 0 /f
```

顺手关掉快速启动：

```cmd
reg add "HKLM\OFFSYS\ControlSet001\Control\Session Manager\Power" /v HiberbootEnabled /t REG_DWORD /d 0 /f
```

最后卸载离线注册表：

```cmd
reg unload HKLM\OFFSYS
```

再重建一次引导：

```cmd
bcdboot W:\Windows /s S: /f UEFI
```

重启。

这次成功进系统。

---

## 8. 进系统后先检查

进系统后别急着开香槟，先确认自己没有启动错盘。

PowerShell：

```powershell
Get-Partition -DriveLetter C | Get-Disk
```

或者打开磁盘管理：

```cmd
diskmgmt.msc
```

确认当前 `C:` 确实在外置 NVMe 上。

然后建议跑一遍：

```cmd
sfc /scannow
DISM /Online /Cleanup-Image /RestoreHealth
chkdsk C: /scan
```

---

## 9. 第三坑：能开机，但关机/重启抽风

系统能进以后，我又遇到一个新问题：

> 开机正常，但关机或重启时，USB 硬盘盒显示已经断开，系统还卡在关机流程里。

对普通移动硬盘来说，这可能只是掉盘。  
但对系统盘来说，这就很刺激了。

Windows 还没关完，根文件系统先没了。  
属于是系统自己把自己脚下的地板拆了。

如果已经卡死，硬盘灯也不闪，可以长按电源强制关机。  
下次开机后先跑：

```cmd
chkdsk C: /scan
```

不放心再跑：

```cmd
sfc /scannow
```

ChatGPT说要关闭快速启动，关闭 USB 选择性暂停、磁盘空闲休眠、PCIe 链路省电，这堆东西我日常就关了，你没关的话出问题了可以考虑一下是不是这些问题

ChatGPT还说要把磁盘驱动器的电源策略改成快速删除，但是这个东西是系统盘，需要重启生效，但是可但是改这个就是为了解决重启失败的问题，闭环了。 ~~遇到困难睡大觉！~~

---


## 10. RTL9210 还值得继续折腾吗？

**遇到困难摆大烂！**

> 如果只是迁 Linux 前的过渡，不值得继续死磕。

现在它已经完成核心任务了：

- 能启动；
    
- 能进原系统；
    
- 软件和文件基本都在；
    
- Linux 迁移期间能当 fallback。
    

剩下的关机/重启问题，更像是硬盘盒固件、USB 控制器、线材、电源管理的兼容性问题。

真要继续查，可能还得折腾：

- 刷 RTL9210 固件；
    
- 换硬盘盒；
    
- 测 UASP / USBSTOR 差异；

但我只是临时用，不想和它过一辈子。 ~~你就说能不能用！~~

---

## 11. 我的最终策略

这个外置 Windows 我最后定位为：

```text
临时工作环境
过渡系统
应急 fallback
不是长期主系统
```

所以后续使用原则也很简单：

1. 不睡眠；
    
2. 不休眠；
    
3. 不开快速启动；
    
4. 不开 BitLocker；
    
5. 不把唯一重要数据放这里；
    
6. 工作文件及时同步到云盘 / Git / NAS；
    
7. 异常关机后跑 `chkdsk C: /scan`；
    
8. 尽快迁完 Linux。
    

---

## 总结

这次 Windows-to-USB 的关键路径：

```text
克隆系统
修 EFI / BCD
解决 0xc000000e
解决 0x7B
关闭快速启动和省电
确认能启动
能用就收手
```

最关键的修复命令是：

```cmd
reg add HKLM\OFFSYS\ControlSet001\Control /v BootDriverFlags /t REG_DWORD /d 20 /f
```

以及把 USB、UASP、NVMe、磁盘相关驱动改成启动早期加载。

Windows-to-USB 不是不能用，但它不是普通克隆工具点一下就能稳定跑的场景。  
它更像是一个“能跑，但要理解启动链路”的偏门玩法。

如果长期用，建议认真选硬盘盒、固件、线材和接口。  
如果只是像我一样迁 Linux 前留个后路，那就别追求完美：

**能启动、能工作、数据有备份，就可以收手了。**

---

## 尾声：脚本

敲那一堆`reg add`我人有点麻，放结尾的目的是让一步步照着做的你也麻一遍， ~~不能只有我一个人难受~~

使用方法：

如果你的外置 Windows 分区是 `W:`，EFI 分区是 `S:`，完整流程就是：

```
fix-usbstor-0x7b.bat W:
bcdboot W:\Windows /s S: /f UEFI
```

*fix-usbstor-0x7b.bat:*

```cmd
@echo off
setlocal EnableExtensions EnableDelayedExpansion

REM ============================================================
REM  fix-usb-boot-driver.bat
REM
REM  Purpose:
REM    Fix Windows cloned-to-USB INACCESSIBLE_BOOT_DEVICE / 0x7B
REM    by enabling USB / UASP / storage drivers for early boot.
REM
REM  Usage:
REM    fix-usb-boot-driver.bat W:
REM
REM  Example:
REM    If your external Windows partition is W:
REM      fix-usb-boot-driver.bat W:
REM
REM  Run from WinPE / Windows Setup command prompt.
REM ============================================================

if "%~1"=="" (
    echo Usage: %~nx0 ^<WindowsDrive^>
    echo Example: %~nx0 W:
    exit /b 1
)

set "WINDRV=%~1"

REM Remove trailing backslash if present
if "%WINDRV:~-1%"=="\" set "WINDRV=%WINDRV:~0,-1%"

set "SYSTEM_HIVE=%WINDRV%\Windows\System32\Config\SYSTEM"

echo.
echo Target Windows drive: %WINDRV%
echo SYSTEM hive: %SYSTEM_HIVE%
echo.

if not exist "%SYSTEM_HIVE%" (
    echo ERROR: Cannot find SYSTEM hive.
    echo Please make sure %WINDRV% is the offline Windows partition.
    exit /b 1
)

echo Backing up SYSTEM hive...
copy "%SYSTEM_HIVE%" "%WINDRV%\SYSTEM.bak" >nul
if errorlevel 1 (
    echo ERROR: Failed to back up SYSTEM hive.
    exit /b 1
)

echo Loading offline SYSTEM hive...
reg load HKLM\OFFSYS "%SYSTEM_HIVE%"
if errorlevel 1 (
    echo ERROR: Failed to load offline SYSTEM hive.
    echo If HKLM\OFFSYS is already loaded, run:
    echo   reg unload HKLM\OFFSYS
    exit /b 1
)

REM ------------------------------------------------------------
REM Detect Current ControlSet
REM ------------------------------------------------------------

set "CURRENT_HEX="

for /f "tokens=3" %%A in ('reg query HKLM\OFFSYS\Select /v Current 2^>nul ^| find /i "Current"') do (
    set "CURRENT_HEX=%%A"
)

if "%CURRENT_HEX%"=="" (
    echo ERROR: Failed to detect Current ControlSet.
    reg unload HKLM\OFFSYS >nul 2>&1
    exit /b 1
)

REM Convert hex-like value, for example 0x1, to decimal
set /a CURRENT_NUM=%CURRENT_HEX%

if "%CURRENT_NUM%"=="1" (
    set "CONTROLSET=ControlSet001"
) else if "%CURRENT_NUM%"=="2" (
    set "CONTROLSET=ControlSet002"
) else if "%CURRENT_NUM%"=="3" (
    set "CONTROLSET=ControlSet003"
) else if "%CURRENT_NUM%"=="4" (
    set "CONTROLSET=ControlSet004"
) else (
    echo ERROR: Unsupported ControlSet number: %CURRENT_NUM%
    reg unload HKLM\OFFSYS >nul 2>&1
    exit /b 1
)

echo Detected Current: %CURRENT_HEX%
echo Using: HKLM\OFFSYS\%CONTROLSET%
echo.

REM ------------------------------------------------------------
REM Enable boot-time USB / storage driver loading
REM ------------------------------------------------------------

echo Setting BootDriverFlags...
reg add HKLM\OFFSYS\%CONTROLSET%\Control /v BootDriverFlags /t REG_DWORD /d 20 /f

echo.
echo Setting storage driver Start values...

reg add HKLM\OFFSYS\%CONTROLSET%\Services\USBXHCI /v Start /t REG_DWORD /d 0 /f
reg add HKLM\OFFSYS\%CONTROLSET%\Services\USBHUB3 /v Start /t REG_DWORD /d 0 /f
reg add HKLM\OFFSYS\%CONTROLSET%\Services\UASPStor /v Start /t REG_DWORD /d 0 /f
reg add HKLM\OFFSYS\%CONTROLSET%\Services\USBSTOR /v Start /t REG_DWORD /d 0 /f
reg add HKLM\OFFSYS\%CONTROLSET%\Services\disk /v Start /t REG_DWORD /d 0 /f
reg add HKLM\OFFSYS\%CONTROLSET%\Services\partmgr /v Start /t REG_DWORD /d 0 /f
reg add HKLM\OFFSYS\%CONTROLSET%\Services\mountmgr /v Start /t REG_DWORD /d 0 /f
reg add HKLM\OFFSYS\%CONTROLSET%\Services\volmgr /v Start /t REG_DWORD /d 0 /f
reg add HKLM\OFFSYS\%CONTROLSET%\Services\storport /v Start /t REG_DWORD /d 0 /f
reg add HKLM\OFFSYS\%CONTROLSET%\Services\stornvme /v Start /t REG_DWORD /d 0 /f
reg add HKLM\OFFSYS\%CONTROLSET%\Services\storahci /v Start /t REG_DWORD /d 0 /f

echo.
echo Setting StartOverride values...

reg add HKLM\OFFSYS\%CONTROLSET%\Services\USBXHCI\StartOverride /v 0 /t REG_DWORD /d 0 /f
reg add HKLM\OFFSYS\%CONTROLSET%\Services\USBHUB3\StartOverride /v 0 /t REG_DWORD /d 0 /f
reg add HKLM\OFFSYS\%CONTROLSET%\Services\UASPStor\StartOverride /v 0 /t REG_DWORD /d 0 /f
reg add HKLM\OFFSYS\%CONTROLSET%\Services\USBSTOR\StartOverride /v 0 /t REG_DWORD /d 0 /f
reg add HKLM\OFFSYS\%CONTROLSET%\Services\stornvme\StartOverride /v 0 /t REG_DWORD /d 0 /f
reg add HKLM\OFFSYS\%CONTROLSET%\Services\storahci\StartOverride /v 0 /t REG_DWORD /d 0 /f

echo.
echo Disabling Fast Startup / Hiberboot...
reg add "HKLM\OFFSYS\%CONTROLSET%\Control\Session Manager\Power" /v HiberbootEnabled /t REG_DWORD /d 0 /f

echo.
echo Unloading offline SYSTEM hive...
reg unload HKLM\OFFSYS
if errorlevel 1 (
    echo WARNING: Failed to unload HKLM\OFFSYS.
    echo You may need to run this manually:
    echo   reg unload HKLM\OFFSYS
    exit /b 1
)

echo.
echo Done.
echo SYSTEM hive backup saved as:
echo   %WINDRV%\SYSTEM.bak
echo.
echo Next recommended command, if your EFI partition is S::
echo   bcdboot %WINDRV%\Windows /s S: /f UEFI
echo.

exit /b 0
```

