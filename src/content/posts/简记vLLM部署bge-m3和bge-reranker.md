---
title: 简记vLLM部署bge-m3和bge-reranker
published: 2025-09-30
tags:
  - Software
  - AIGC
category: Zheteng
draft: false
---

### 概述
本配置用于部署一个基于 Docker 的多服务 API 系统，包含 New API 主服务、Redis 缓存、MySQL 数据库以及两个 GPU 加速的文本处理模型（bge-m3 和 bge-reranker-v2-m3）。主要用于提供文本嵌入和重排序功能，支持多语言处理，为开发者提供统一的 Jina AI 格式接口。

---

### 懒狗快捷指南
```yaml
services:
  new-api:
    image: calciumion/new-api:latest
    container_name: new-api
    restart: always
    command: --log-dir /app/logs
    networks:
      - vllm_network
#      - caddy_network
    ports:
      - "3443:3000"
    volumes:
      - ./new-api/data:/data
      - ./new-api/logs:/app/logs
    environment:
      - SQL_DSN=root:123456@tcp(mysql:3306)/new-api  # 指向mysql服务
      - REDIS_CONN_STRING=redis://redis
      - TZ=Asia/Shanghai
    #      - SESSION_SECRET=random_string  # 多机部署时设置，必须修改这个随机字符串！！！！！！！
    #      - NODE_TYPE=slave  # 多机部署的从节点取消注释
    #      - SYNC_FREQUENCY=60  # 如需定期同步数据库，取消注释
    #      - FRONTEND_BASE_URL=https://your-domain.com  # 多机部署带前端URL时取消注释

    depends_on:
      - redis
      - mysql
      - bge-m3
      - bge-reranker
    healthcheck:
      test: ["CMD-SHELL", "wget -q -O - http://localhost:3000/api/status | grep -o '\"success\":\\s*true' | awk -F: '{print $$2}'"]
      interval: 30s
      timeout: 10s
      retries: 3

  redis:
    image: redis:latest
    container_name: redis
    networks:
      - vllm_network
    restart: always

  mysql:
    image: mysql:8.2
    container_name: mysql
    networks:
      - vllm_network
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: 123456  # 确保与SQL_DSN中的密码一致
      MYSQL_DATABASE: new-api
    volumes:
      - ./new-api/mysql_data:/var/lib/mysql
    # ports:
    #   - "3306:3306"  # 如需从Docker外部访问MySQL，取消注释

#volumes:
#  mysql_data:

  # bge-m3 专用容器 - 使用 GPU 0
  bge-m3:
    image: vllm/vllm-openai:latest # 当前版本0.10.1
    container_name: bge-m3
    runtime: nvidia
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
#              count: 1
              capabilities: [gpu]
              device_ids: ['0']  # 明确指定使用 GPU 0
#    ports:
#      - "8000:8000"
    volumes:
      - ./vllm/bge-m3/.cache/huggingface:/root/.cache/huggingface
      - ./vllm/bge-m3/models:/app/models
    networks:
      - vllm_network
    environment:
      - http_proxy=http://localhost:xxxxx # 改成你的代理端口
      - https_proxy=http://localhost:xxxxx # 改成你的代理端口
      - HUGGING_FACE_HUB_TOKEN=hf_xxxxxx # 改成你的token
#      - CUDA_VISIBLE_DEVICES=0  # 明确指定使用 GPU 0
    ipc: host
    command: >
      --model BAAI/bge-m3
      --api-key sk-xxxxxx
      --host 0.0.0.0
      --port 8000
      --tensor-parallel-size 1
      --gpu-memory-utilization 0.8
      --max-model-len 8192
    # 上面command字段不能有注释，api-key随便生成一个就行
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  # bge-reranker-v2-m3 专用容器 - 使用 GPU 1
  bge-reranker:
    image: vllm/vllm-openai:latest # 当前版本0.10.1
    container_name: bge-reranker
    runtime: nvidia
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
#              count: 1
              capabilities: [gpu]
              device_ids: ['1']  # 明确指定使用 GPU 1
#    ports:
#      - "8001:8000"  # 使用不同外部端口
    volumes:
      - ./vllm/bge-reranker/.cache/huggingface:/root/.cache/huggingface
      - ./vllm/bge-reranker/models:/app/models
    networks:
      - vllm_network
    environment:
      - http_proxy=http://localhost:xxxxx # 改成你的代理端口
      - https_proxy=http://localhost:xxxxx # 改成你的代理端口
      - HUGGING_FACE_HUB_TOKEN=hf_xxxxxx # 改成你的token
#      - CUDA_VISIBLE_DEVICES=1  # 明确指定使用 GPU 1
    ipc: host
    command: >
      --model BAAI/bge-reranker-v2-m3
      --api-key sk-xxxxx
      --host 0.0.0.0
      --port 8000
      --tensor-parallel-size 1
      --gpu-memory-utilization 0.8
      --max-model-len 8192
    # 上面command字段不能有注释，api-key随便生成一个就行
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

networks:
  vllm_network:
    external: true

```

---

### Q&A
**1. 多卡部署为什么会报错？**
vLLM 在多 GPU 环境下需要正确配置 tensor-parallel-size 参数。如果配置不当（如 tensor-parallel-size 与实际 GPU 数量不匹配），会导致模型加载失败或性能问题。建议单卡单模型部署，通过 device_ids 明确指定 GPU 设备。
~~（其实你根本不需要多卡跑这种小模型，爱折腾请自便）~~

**2. New API 的重排序接口采用什么格式？**
New API 统一采用 Jina AI 的重排序格式作为标准响应格式。所有其他供应商（Xinference、Cohere 等）的响应都会被转换为 Jina AI 格式，确保开发者获得一致的接口体验。
**所以Rerank模型配置时请使用Jina渠道**

**3. "Model does not support matryoshka representation" 错误是什么意思？**
- **Matryoshka 表示法**：一种允许嵌入模型输出可变维度向量的技术（如 OpenAI 的 text-embedding-3 系列）
- **bge-m3 限制**：BAAI/bge-m3 是固定输出维度模型（1024 维），不支持 dimensions 参数
- **解决方案**：移除**客户端**请求中的 dimensions 参数，使用模型默认的输出维度

**4. 如何测试嵌入功能？**
使用以下 curl 命令测试（替换为实际地址和 API Key）：
```bash
curl https://your-server/v1/embeddings \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d '{
    "input": "The food was delicious and the waiter...",
    "model": "BAAI/bge-m3",
    "encoding_format": "float"
  }'
```


**5. 如何测试重排序功能？**
使用以下 curl 命令测试（替换为实际地址和 API Key）：
```bash
curl https://your-server/v1/rerank \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d '{
    "model": "BAAI/bge-reranker-v2-m3",
    "query": "Organic skincare products",
    "top_n": 3,
    "documents": ["文档1", "文档2", "文档3"]
  }'
```

**6. 健康检查失败怎么办？**
检查服务日志确认模型是否正常加载，确保：
- GPU 驱动和 nvidia-container-runtime 已正确安装
- 模型文件已正确下载到指定目录
- 网络代理配置（如需要）正确无误
```bash
docker logs new-api
docker logs bge-m3
docker logs bge-reranker
```

---

本文内容由AI辅助编写、主要代码由人工完成、已人工测试可用性，部署平台是`Epyc7532` 和 `Nvidia-Tesla-T10 16GB`，内存实际占用`3.8GB`，显存实际占用`2568MB`，请确保资源充足