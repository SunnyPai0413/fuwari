---
title: NewAPI大模型网关
published: 2026-01-24
tags:
  - Software
  - AIGC
category: Zheteng
draft: false
---

大模型token好贵，得想点办法了。

前提条件：
- OpenRouter有每天的免费额度
- 注册不需要手机号，邮箱就行（主打匿名隐私，面向web3用户，支持区块链支付）
- 单ip通过多账号高频请求会风控，但是风控似乎“全部”通过Cloudflare实现
- Cloudflare Worker “似乎”是白名单（好人不封自己）
  
理论存在，实践开始！

---

目标1：需要无限的临时邮箱
- 各种临时邮箱api，曾经（2025年初）可以，但目前版本多数已被封禁
- 谷歌、微软、苹果等主流邮箱服务商（测试通过）
- 谷歌无限邮（免费，但反人机机制较强）、微软Outlook（朋友测试过，需要吃Azure的一坨）、iCloud+（随机无限生成，自动转发，但需要iCloud高级订阅）
- 如有其余方案欢迎补充

---

目标2：自动化注册（由于平台相关规定，不建议进行）
- selenium自动操作，imap读取邮件

---

目标3：Cloudflare Worker 提升访问稳定性，而非规避平台规则
- [Calcium-Ion/new-api-worker](https://github.com/Calcium-Ion/new-api-worker)
- 或者自己让AI给你搓一个代理脚本（但是有现成的干嘛还要写）
- 使用量太大可能收费，但cloudflare worker的免费计划不需要绑定银行卡，且cloudflare注册也符合开头的“前提条件”（我是不会告诉你new-api的上级路由也可以是new-api，不仅限于OpenRouter的）

---

术式展开完成。

注：本文所有技术方案仅适用于个人开发者的合法测试与学习，严禁用于批量注册账号、套取平台免费额度进行牟利等违规操作。使用任何平台服务前，请仔细阅读并遵守其用户协议，违规操作可能导致账号封禁等后果。（AI生成水印）