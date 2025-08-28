---
title: Linux 配置 zram 优先 + sdb 次级 swap
published: 2025-08-28
tags:
  - Software
  - OS
category: Zheteng
draft: false
---
## 一、安装并配置 zram（高优先级）

### 1. 安装 zram 工具
```bash
sudo apt install zram-tools
```

### 2. 配置 zram 核心参数
```bash
# 设置压缩算法为 zstd（高效平衡压缩率与速度）
# 设置容量为物理内存的 60%
echo -e "ALGO=zstd\nPERCENT=60" | sudo tee -a /etc/default/zramswap

# 设置 zram 优先级为 100（最高，确保优先使用）
echo "PRIORITY=100" | sudo tee -a /etc/default/zramswap
```

### 3. 重启 zram 服务生效
```bash
sudo systemctl restart zramswap.service
```


## 二、配置 sdb 为次级 swap（低优先级）

### 1. 格式化 sdb 为 swap 分区（注意：会清空数据）
```bash
sudo mkswap /dev/sdb
```

### 2. 获取 sdb 的 UUID（用于开机自动挂载）
```bash
sudo blkid /dev/sdb  # 记录输出中的 UUID 值
```

### 3. 配置 fstab 实现开机自动挂载（优先级 50）
```bash
# 替换下方 "你的sdb的UUID" 为实际 UUID
sudo tee -a /etc/fstab << EOF
UUID=你的sdb的UUID none swap defaults,pri=50 0 0
EOF
```

### 4. 临时启用 sdb swap（立即生效，无需重启）
```bash
sudo swapon -p 50 /dev/sdb
```


## 三、验证配置是否生效

查看当前 swap 设备及优先级（确保 zram 优先级 100 > sdb 优先级 50）：
```bash
sudo swapon -s
```

预期输出示例：
```
Filename                                Type            Size    Used    Priority
/dev/zram0                              partition       8388604 0       100
/dev/sdb                                partition       10485756 0      50
```


## 四、调整系统 swap 策略（更积极使用 zram）

在 `/etc/sysctl.d/` 目录下新建一个专属配置文件（例如 `99-zram-tweaks.conf`，文件名以 `99-` 开头可确保优先级，避免被系统默认配置覆盖），集中管理 `vm.swappiness` 和 `vm.vfs_cache_pressure` 等参数。

1. **在 /etc/sysctl.d/ 下创建新配置文件**  
新建一个专属配置文件（例如 `99-zram-tweaks.conf`），添加所需参数：
```bash
sudo tee /etc/sysctl.d/99-zram-tweaks.conf << EOF
# 更积极将非活跃数据移入 zram
vm.swappiness=60
# 降低缓存回收倾向，保留有用的文件系统缓存
vm.vfs_cache_pressure=40
EOF
```

2. **使配置立即生效**  
无需重启，执行以下命令加载新配置：
```bash
sudo sysctl --system
```

## 五、验证配置是否生效
执行以下命令检查参数是否正确加载：
```bash
# 查看 vm.swappiness 当前值
sysctl vm.swappiness
# 查看 vm.vfs_cache_pressure 当前值
sysctl vm.vfs_cache_pressure
```
若输出结果与你配置的 `60` 和 `40` 一致，说明配置已生效；重启系统后再次检查，值仍不变则表示**永久生效**。

## 配置效果
- 系统优先使用 zram（内存压缩 swap，速度快），当 zram 占满后使用 sdb（磁盘 swap，容量兜底）
- 更积极地将非活跃数据转移到 zram，平衡内存使用与性能