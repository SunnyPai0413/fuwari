---
title: MinIO Site Replication 双活部署
published: 2026-06-23
tags:
  - Software
  - Storage
category: Zheteng
draft: false
---

~~开头先说一句：我是鸽子精+大傻子！~~  

MinIO Site Replication 可以用内置IDP！它不严格依赖外部OIDC/LDAP！  

---

到了正文我也不知道说什么了，说点踩过的坑罢：

```minio
MINIO_ROOT_USER=minioadmin
MINIO_ROOT_PASSWORD=minioadmin
MINIO_VOLUMES="/mnt/nvme0n1 /mnt/nvme1n1"
MINIO_OPTS="--address :9000 --console-address :33075"
MINIO_STORAGE_CLASS_STANDARD="EC:0"
```

首先，和某些教程说的不一样，MinIO不支持裸设备 `(raw block device)` ，需要基于文件存储  
最佳实践是XFS  

然后，和某些教程说的不一样，先期单点部署时可以不要冗余机制，即：  
`MINIO_STORAGE_CLASS_STANDARD="EC:0"`

最后放点命令  

```bash
mc alias set sitea https://minio-a.example.com minioadmin minioadmin
mc alias set siteb https://minio-b.example.com minioadmin minioadmin
mc admin replicate add sitea siteb
mc admin replicate info sitea
mc admin replicate status sitea
```

剩下web console点点点去吧，咕咕咕  

***PS: 这里的双活不是强一致性存储解决方案，只提供最终一致性，不能保证双写跨站点立即一致***