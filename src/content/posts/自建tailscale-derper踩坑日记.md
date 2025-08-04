---
title: 自建tailscale-derper踩坑日记
published: 2024-12-21
tags:
  - Networking
  - Software
category: Zheteng
draft: false
---

这是一篇踩坑记录，关于tailscale derper的。

[**懒狗快捷指南：**](http://localhost:4321/posts/%E8%87%AA%E5%BB%BAtailscale-derper%E8%B8%A9%E5%9D%91%E6%97%A5%E8%AE%B0/#%E9%83%A8%E7%BD%B2%E6%B5%81%E7%A8%8B)

##### 引言（和主题关系不大，不想看直接跳过）：

众所周知，中国大陆的网络环境，ISP几乎不提供公网ipv4地址，而云服务器的流量价格是海外同类产品的数倍乃至十数倍：以阿里云为例，流量价格为500CNY/TB（ECS共享流量包），而AWS的价格则是5USD/TB（lightsail实例），虽然大陆也有一些“大带宽”云服务器，例如雨云和物语云，但SLA保证远低于头部厂商。

（你也不想在需要编译软件时，打开vscode remote-ssh时，提示个连接超时吧？）

### 选择 tailscale 原因：

废话不多说，有以下几点：

- 使用P2P打洞，流量不流经中转服务器节点
    
- 相较于同样P2P的zerotier，部署简便开箱即用
    
- 跨平台（Windows、Linux、iOS、Android、MacOS）乃至对集群环境（Kubernetes、AWS等）兼容性好
    
- 打洞失败有官方中继可用，无需额外支出
    

但是随着设备越来越多、流量越来越大，官方中继已经难以满足使用需求，具体问题有延迟过高(~300ms)、带宽较小(~1Mbps)。经查询后了解到具有自建中转的做法，可以极大的提升中国大陆内和跨境数据传输的体验。

### 软件架构：

tailscale分为控制平面和数据平面，是“半个零信任”的架构。控制平面负责身份认证，数据平面负责打洞和中继。这两者都有官方开源的自建方案：Headscale和Derper。

Headscale私有化了控制平面，在具有较高的安全性需求的环境可以考虑使用，而Derper则是私有化的数据平面节点，具体节点则由tailscale客户端选择。由于我是个人使用，因此只做了derper的自建。

### 部署踩坑：

derper是使用golang（go语言）编写的。为了操作简便，我使用docker部署。

- The DERP protocol does a protocol switch inside TLS from HTTP to a custom bidirectional binary protocol. It is thus incompatible with many HTTP proxies. Do not put `derper` behind another HTTP proxy. [*tailscale/cmd/derper](https://github.com/tailscale/tailscale/tree/main/cmd/derper#guide-to-running-cmdderper)
    

#### 这是derper的第一个坑点：

你会在网上找到很多教程，其中写着通过各种代理：Nginx、Caddy……  
注意：**Derper是Tailscale自研的协议，虽然一样使用HTTP/TLS，端口也是80/443，但通过WEB代理会拒绝连接。**

#### 然后就到了第二个坑点：

Derper有官方的域名部署，还有通过ip derper部署的方式，即使用 `"InsecureForTests": true` 这一测试环境使用的ACL配置项，跳过TLS验证和域名验证。

例如这个项目：[yangchuansheng/ip_derper](https://github.com/yangchuansheng/ip_derper) ，是ip derper常用的docker项目。我尝试使用非标端口进行部署，一开始测试通过，但在使用大约2-3小时后，自建derper在客户端使用`tailscale netcheck`命令测试通过，但无论如何都无法实现连接，已经建立的连接也被强制中断，包括打洞和中继。报错在这里我没有保存，**我个人也不推荐使用ip derper搭建**。

~~我在这里卡了两天时间，查了几十篇教程和文档，你要是愿意搞不舍得买个域名的钱请自行折腾。~~

另一种则是tailscale官方的使用域名搭建。你可以选择开放80&443端口给它，内置的acme会获取letsencrypt证书。但由于我的80/443被其余web服务占用，我使用了manual方式。

#### 接下来来到了第三个坑点：

derper程序不会识别 `cert.pemcert.key` 这样的证书文件，所以直接绑定到通配符证书目录不可用，而是需要**重命名为**`example.com.crt`**和**`example.com.key`，再进行使用。以及如果docker或容器内使用的不是root用户，还需要重新**配置证书权限**。

#### 最后一个坑点是我踩了最长时间才爬出来的：

我选择的是 [fredliang44/derper-docker](https://github.com/fredliang44/derper-docker) 这个项目，按照README使用 `DERP_ADDR=':8443'` 配置项进行自定义端口，但启动时会出现神秘报错：

```bash
Attaching to derper
derper  | 2024/12/21 04:41:12 no config path specified; using /var/lib/derper/derper.key
derper  | 2024/12/21 04:41:12 derper: serving on ":8443" with TLS
derper  | 2024/12/21 04:41:12 derper: listen tcp: lookup tcp/8443": unknown port
derper exited with code 0
```

不管使用default bridge、custom bridge 还是 host 网络，均无法访问。macvlan模式由于tailscale官方文档要求禁止部署在防火墙后，而我又没有自己的公网ip池，因此无法使用。

一开始怀疑是镜像问题，后面尝试重新构建docker image，甚至连带golang依赖也重新构建了，均无法运行。换了其余好几个版本的镜像也是一样的报错，才意识到可能是第三方教程和官方README的问题：他们不约而同的选择了 `ENV` 定义非标端口，而这一配置项会被翻译成官方二进制文件的 `-a` 参数。而在容器内测试使用这一参数时，无法识别这一命令……

所以，即使想使用非标端口，也请删除这一ENV，然后**使用** `-p <custom_port>:443` **这样的形式，在bridge network实现部署**。

### 部署流程：

好吧，今天又说了太多废话。下面直接放配置文件：

```yaml
services:
  derper:
    container_name: derper
    environment:
#      - DERP_ADDR=":8443" # 这里不要加，会启动失败
      - DERP_DOMAIN=derp.sunnypai.top
      - DERP_CERT_MODE=manual
      - DERP_CERT_DIR=/app/certs
      - DERP_HTTP_PORT=-1
# 这里如果不使用非标端口，直接删掉ports部分配置，使用host
#    network_mode: host
    ports:
      - 8443:443
      - 3478:3478/udp
    volumes:
      - /root/AppData/derper/certs/derp.sunnypai.top:/app/certs
      - /var/run/tailscale/tailscaled.sock:/var/run/tailscale/tailscaled.sock
    image: fredliang/derper:latest
    restart: unless-stopped
    networks:
      - derper_network

networks:
  derper_network:
    external: true
```

如果不使用非标端口，也可以使用letsencrypt配置证书，这边建议看其余教程，_~~我懒得测试~~_

然后在控制平面的Access Controls文件中添加这些内容：（请注意缩进）

```
	// 增加下方这部分
	"derpMap": {
		"Regions": {
			"912": { // 这个节点里面的912随便填，900以上即可
				"RegionID":   912,
				// 这里可以改名，code和name不要求一致
				"RegionCode": "derper_self",
				"RegionName": "Derper Self",
				"Nodes": [
					{
						"Name":     "derper_self",
						"RegionID": 912,
						"DERPPort": 8443, // 更换为自己的端口号
						"STUNPort": 3478,

						"HostName": "example.com", // 此 HostName 参数填写自己的域名

						"InsecureForTests": false, // 测试时可以先写成true
					},
				],
			},
		},
	},
```

最后重启全部客户机节点，即可实现自建derper部署。

半天后追加：

高峰期2%~3%丢包率，平均延迟75ms，最长279ms，使用酒店wifi，到路由延迟5-10ms，效果很棒！  
欢迎搭建成功的朋友们评论留言使用体验！

---

感谢来自 [Kiyoi](https://blog.kiyoi.xyz/) 的支持。