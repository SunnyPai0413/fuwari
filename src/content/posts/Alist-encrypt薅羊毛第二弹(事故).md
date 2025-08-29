---
title: Alist-encrypt薅羊毛第二弹(事故)
published: 2025-04-18
tags:
  - Software
category: Zheteng
draft: false
---
在Alist Encrypt使用过程中又遇到了新的问题，在这里先引用先前的一篇部署文章：  
[Alist-encrypt全自动薅网盘羊毛 - 向阳哌小站](https://www.sunnypai.top/posts/alist-encrypt%E5%85%A8%E8%87%AA%E5%8A%A8%E8%96%85%E7%BD%91%E7%9B%98%E7%BE%8A%E6%AF%9B/)

---

新的问题是这样的：  
增量备份的测试中并无异常，但全量备份会直接报错终止，只能加密并上传少数几个文件，多数文件依旧无法上传。

究其原因，主要是因为Alist挂载网盘存储后端，会有非官方客户端的限制。这一点可以通过本地加密后使用官方客户端推送到云端解决。

但在使用Alist Encrypt的本地加解密功能时遇到了新的问题：我的文件总数高达39万个，而Alist Encrypt开发者限制了单次处理10000个文件。通过沟通得知10000这一数字是他随意设置，并未经过精心设计，于是我萌生了修改代码的念头。

![](https://assets.blog.edge.sunnypai.top/Alist-encrypt薅羊毛第二弹(事故)01.png)

我假设了500MB/s的IO（来自HDD阵列或SSD平均读写），又选择了10MB/文件的平均值（来自我NAS的个人文件统计），附加90%的SLA预设得到的2h24min日停机时长，按每次抢修45分钟的中小事故预设值得出平均每天停机4次，从而得出5.4h的单次uptime，于是取整设置了100万的文件数量，兴致满满准备提交PR，结果本以为一次能成功的测试出现了大问题，运行中途随机时间点报错找不到文件：

```bash
@@finish filePath /home/dev/projects/alist-encrypt/test/0626332.txt /home/dev/projects/testout/.temp/0626332.txt
Error: ENOENT: no such file or directory, rename '/home/dev/projects/testout/.temp/0626332.txt' -> '/home/dev/projects/testout/0626332.txt'
    at Object.renameSync (node:fs:1021:11)
    at ReadStream.<anonymous> (/home/dev/projects/alist-encrypt/node-proxy/src/utils/convertFile.ts:91:12)
    at ReadStream.emit (node:events:530:35)
    at endReadableNT (node:internal/streams/readable:1698:12)
    at processTicksAndRejections (node:internal/process/task_queues:90:21)
@@finish filePath /home/dev/projects/alist-encrypt/test/0626333.txt /home/dev/projects/testout/.temp/0626333.txt
@@finish filePath /home/dev/projects/alist-encrypt/test/0626334.txt /home/dev/projects/testout/.temp/0626334.txt
[ERROR] 04:39:49 Error: ENOENT: no such file or directory, rename '/home/dev/projects/testout/.temp/0626332.txt' -> '/home/dev/projects/testout/0626332.txt'
@@finish filePath /home/dev/projects/alist-encrypt/test/0626335.txt /home/dev/projects/testout/.temp/0626335.txt
```

更换平台和算法（AES-CTR和RC4）测得如下结果：

|   |   |   |   |   |   |
|---|---|---|---|---|---|
|**测试环境**|**文件系统**|**加密算法**|**完成数1**|**完成数2**|**性能对比**|
|12900 台式机（USSD）|EXT4|AES-CTR|26万|62万|延迟分配缓解竞争|
|||RC4|37万|100万|轻量算法减少I/O|
|EPYC7282 服务器（RAIDZ0）|ZFS|AES-CTR|5万|7万|COW 增加竞争概率|
|||RC4|2万|4万|低延迟暴露问题|

一开始，“性能越好出错越早结果越糟糕”的奇怪现象让我进入了思维惯性，一番思索也没想清楚具体原因

询问他人并借助人工智能工具分析后才得出初步结论：

**根本原因**：Node.js 的异步 I/O 模型与文件系统特性共同作用，导致高并发文件操作时出现 **竞态条件（Race Condition）**，表现为 "文件存在但报错 `ENOENT`"。具体表现为：

1. **文件状态不一致**：异步操作未正确同步，导致文件创建、写入、重命名等步骤的执行顺序不可控。
    
2. **文件系统行为差异**：EXT4 的延迟分配与 ZFS 的写时复制（COW）特性，放大了不同硬件环境下的竞态窗口。
    

关掉兴冲冲写好准备提交的PR，深深叹了一口气，薅羊毛的路任重而道远啊！

**现象解读：**

1. **EXT4 表现更好**：
    
    - 延迟分配机制将数据暂存内存，**缩短了文件操作的可见性延迟**。
        
    - USB的高延迟意外成为 "天然并发限制器"，变相降低了竞态概率。
        
2. **ZFS 表现更差**：
    
    - 写时复制（COW）和事务组提交机制导致 **元数据更新延迟更高**。
        
    - NVMe 的低延迟特性反而让竞态窗口更易被触发（操作速度 > 同步速度）。