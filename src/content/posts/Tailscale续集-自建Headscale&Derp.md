---
title: 自建tailscale-derper踩坑日记
published: 2026-05-14
tags:
  - Networking
  - Software
category: Zheteng
draft: false
---

上回折腾了自建 Derper，这回干脆把控制平面也一起迁出来：`Headscale + Headplane + 自建 DERP`。

简单来说：

- `Headscale`：Tailscale 控制平面的开源实现，负责设备注册、密钥交换、ACL、DNS 等。
- `Headplane`：Headscale 的 Web 管理面板，用起来更接近官方 Tailscale 控制台。
- `DERP/Derper`：当 P2P 打洞失败时的中继保底，也可辅助 NAT 穿透。

需要先说明的是，DERP 并不是“明文代理”。Tailscale 的 DERP 只转发已经加密的 WireGuard 数据包，中继节点本身看不到业务流量内容；客户端会从控制平面拿到 DERP Map，再根据延迟等信息选择可用中继。

但是 DERP 也不是普通 Web 服务。`derper` 会在 TLS 内做协议切换，不兼容许多普通 HTTP 反代，所以不要把 DERP 直接丢到 Nginx/Caddy 这种七层 HTTP 反代后面。DERP 的 TCP 端口可以用 Docker 映射，STUN 的 UDP 3478 也必须单独放通。

---

## 0. Pre 检测

先把几个鸡生蛋、蛋生鸡的问题排掉，不然后面会很痛苦。

### 0.1 域名与端口

本文使用两个域名：

```text
derp.example.net       # DERP 中继
tailscale.example.net  # Headscale + Headplane
````

公网侧需要放通：

```text
8443/tcp   # DERP TLS 入口，Docker 映射到容器内 443
3478/udp   # STUN
443/tcp    # Headscale / Headplane，由 Caddy 反代
```

注意：`DERPPort` 可以非标端口，本文用 `8443`；`STUNPort` 是 UDP，不能靠普通 HTTP 反代解决。

### 0.2 创建 Docker 网络

本文里 DERP 和 Headscale 分开网络，Headscale/Headplane 接到 Caddy 所在网络：

```bash
docker network create derper_network || true
docker network create caddy_network || true
```

### 0.3 证书文件准备

`derper -certmode manual` 对证书文件名比较挑剔。建议证书目录里至少准备：

```text
derp.example.net.crt
derp.example.net.key
```

如果用的`acme.sh`管理，用如下命令安装：

```bash
acme.sh --install-cert \  
-d derp.example.net \  
--fullchain-file /path/to/certs/derp.example.net/derp.example.net.crt \  
--key-file /path/to/certs/derp.example.net/derp.example.net.key \  
--reloadcmd "docker restart derp"
# 也可以使用docker compose -f "/path/to/compose.yml" restart
```


如果你的证书管理器导出的是 `fullchain.pem` 和 `privkey.pem`，可以软链接一份：

```bash
mkdir -p ./certs/derp.example.net

ln -sf /path/to/fullchain.pem ./certs/derp.example.net/derp.example.net.crt
ln -sf /path/to/privkey.pem   ./certs/derp.example.net/derp.example.net.key

chmod 600 ./certs/derp.example.net/derp.example.net.key
```

如果你直接挂 1Panel、宝塔、Certd 这类面板的证书目录，也请确认容器内看到的文件名符合这个规则。

### 0.4 文件服务器存 DERP Map

我这里让 Headscale 从文件服务器拉 DERP Map。这样后续加节点、删节点，不需要每次进容器改配置。

这里有一个坑：

- `headscale.derp.urls` 适合放外部 URL，内容是 JSON DERP Map。

- `headscale.derp.paths` 适合放本地文件，内容是 YAML DERP Map。


所以如果你要放到文件服务器，建议使用 JSON：

```json
{
  "Regions": {
    "901": {
      "RegionID": 901,
      "RegionCode": "derp",
      "RegionName": "MY DERP",
      "Nodes": [
        {
          "Name": "901a",
          "RegionID": 901,
          "HostName": "derp.example.net",
          "DERPPort": 8443,
          "STUNPort": 3478,
          "STUNOnly": false
        }
      ]
    }
  }
}
```

保存到文件服务器，比如：

```text
https://assets.example.net/tailscale/derpmap.json
```

先在服务器上测一下能否访问：

```bash
curl -fsSL https://assets.example.net/tailscale/derpmap.json
```

如果这里访问失败，后面 Headscale 可能启动后没有可用 DERP，甚至直接报 `initial DERPMap is empty` 一类错误。

如果你不想依赖文件服务器，也可以用本地 YAML：

```yaml
regions:
  901:
    regionid: 901
    regioncode: derp
    regionname: My DERP
    nodes:
      - name: 901a
        regionid: 901
        hostname: derp.example.net
        derpport: 8443
        stunport: 3478
        stunonly: false
```

然后在 Headscale 配置中使用：

```yaml
derp:
  urls: []
  paths:
    - /etc/headscale/derp.yaml
```

### 0.5 确保能连回服务器

自建控制平面最怕“服务在服务器上，但服务器自己访问不了自己的公网域名”。

后续至少要确认：

```bash
curl -I https://tailscale.example.net
openssl s_client -connect derp.example.net:8443 -servername derp.example.net </dev/null
curl -vk https://derp.example.net:8443/
```

如果服务器自身访问 `tailscale.example.net` 或 `derp.example.net` 会走错线路，建议先解决 DNS、回源、云防火墙、NAT 回环问题。

---

## 1. 启动 DERP

`docker-compose.yml`：

```yaml
services:
  derp:
    image: sparanoid/derp:edge
    container_name: derp
    restart: always
    init: true
    ports:
      - "8443:443"
      - "3478:3478/udp"
      # DERPPort 可以通过 Docker 端口映射改成非标端口
      # STUNPort 是 UDP，需要公网直接放通
    volumes:
      - ./certs/derp.example.net:/app/certs:ro
      # certdir 内建议包含：
      # derp.example.net.crt
      # derp.example.net.key

      - /var/run/tailscale/tailscaled.sock:/var/run/tailscale/tailscaled.sock
      # 后续开启 -verify-clients=true 时使用
      # 需要宿主机运行 tailscaled，并且已经加入对应 tailnet

	command: sh -c "
      derper \
        -hostname derp.example.net \
        -certdir /app/certs \
        -certmode manual \
        - verify-clients false \
        -home blank"

    # 一些可选启动参数：
    # -verify-clients true 启用客户端验证，防止DERP被偷，需要配合Tailscale使用，且需要挂载Tailscale的套接字文件
    # -home blank 访问DERP返回空白界面，简单防止DERP服务被扫。
    # -home https://xxx.com 访问DERP跳转指定URL

    networks:
      - derper_network

networks:
  derper_network:
    external: true
```

启动：

```bash
docker compose up -d
docker logs -f derp
```

测试：

```bash
curl -vk https://derp.example.net:8443/
```

如果 `-home blank` 生效，浏览器访问可能就是空白页，这不是坏事。主要看 TLS 是否正常、容器日志是否正常。

再次强调：本文没有把 DERP 走 Caddy HTTP 反代。DERP 不是普通 WebSocket/HTTP 服务，不建议这么玩。

---

## 2. 启动 Headscale + Headplane

目录结构大概这样：

```text
.
|-- compose.yml
|-- headplane-config
|   `-- config.yaml
|-- headplane-data
|-- headscale-config
|   |-- config.yaml
|   `-- dns_records.json
`-- headscale-data
```

先初始化 `dns_records.json`：

```bash
mkdir -p headscale-config headscale-data headplane-config headplane-data
echo "[]" > headscale-config/dns_records.json
```

后续如果需要添加自定义的dns records，可以在web console配置，但需要重启headscale才能生效，而新增主机的magic dns则不受影响
容器内需要挂载docker socket才能重启，但出于安全考虑，不建议这种做法

### 2.1 Headscale 配置

`./headscale-config/config.yaml`：

```yaml
server_url: https://tailscale.example.net
listen_addr: 0.0.0.0:8080
metrics_listen_addr: 0.0.0.0:9090
grpc_listen_addr: 0.0.0.0:50443
grpc_allow_insecure: false

noise:
  private_key_path: /var/lib/headscale/noise_private.key

prefixes:
  v4: 100.64.0.0/10
  v6: fd7a:115c:a1e0::/48

allocation: sequential

derp:
  server:
    enabled: false
    region_id: 901
    region_code: "derp"
    region_name: "My DERP"
    verify_clients: true
    stun_listen_addr: "0.0.0.0:3478"
    private_key_path: /var/lib/headscale/derp_server_private.key

  urls:
    - https://assets.example.net/tailscale/derpmap.json

  paths: []
  auto_update_enabled: true
  update_frequency: 24h

disable_check_updates: false
ephemeral_node_inactivity_timeout: 30m

database:
  type: sqlite
  debug: false
  gorm:
    prepare_stmt: true
    parameterized_queries: true
    skip_err_record_not_found: true
    slow_threshold: 1000
  sqlite:
    path: /var/lib/headscale/db.sqlite
    write_ahead_log: true
    wal_autocheckpoint: 1000

acme_url: https://acme-v02.api.letsencrypt.org/directory
acme_email: ""
tls_letsencrypt_hostname: ""
tls_letsencrypt_cache_dir: /var/lib/headscale/cache
tls_letsencrypt_challenge_type: HTTP-01
tls_letsencrypt_listen: ":http"
tls_cert_path: ""
tls_key_path: ""

log:
  level: info
  format: text

policy:
  mode: database
  path: ""

dns:
  magic_dns: true
  base_domain: tailnet.example.net
  override_local_dns: true
  nameservers:
    global:
      - 1.1.1.1
      - 1.0.0.1
      - 2606:4700:4700::1111
      - 2606:4700:4700::1001
  split: {}
  search_domains: []
  extra_records: []
  extra_records_path: /etc/headscale/dns_records.json

unix_socket: /var/run/headscale/headscale.sock
unix_socket_permission: "0770"

logtail:
  enabled: false

randomize_client_port: false

taildrop:
  enabled: true
```

这里 `server_url` 必须是客户端能访问到的公网 HTTPS 地址。  
`base_domain` 不要和 `server_url` 完全一样，MagicDNS 用它生成类似 `hostname.tailnet.example.net` 的域名。

### 2.2 Headplane 配置

生成一个 32 字符的 cookie secret：

```bash
openssl rand -hex 16
```

`./headplane-config/config.yaml`：

```yaml
host: "0.0.0.0"
port: 3000

# 不要带 /admin
base_url: "https://tailscale.example.net"

cookie_secret: "替换成 openssl rand -hex 16 生成的值"
cookie_secure: true
cookie_max_age: 86400

data_path: "/var/lib/headplane"

headscale:
  url: "http://headscale:8080"
  public_url: "https://tailscale.example.net"
  config_path: "/etc/headscale/config.yaml"
  config_strict: true
  dns_records_path: "/etc/headscale/dns_records.json"

integration:
  agent:
    enabled: false
    pre_authkey: ""

  docker:
    enabled: true
    container_label: "me.tale.headplane.target=headscale"
    socket: "unix:///var/run/docker.sock"

  kubernetes:
    enabled: false
    validate_manifest: true
    pod_name: "headscale"

  proc:
    enabled: false

# 不配置 OIDC 时，Headplane 登录依赖 Headscale API Key
# oidc:
#   enabled: true
```

Headplane 的 Docker 部署方式要求配置文件和数据目录持久化；如果要让它在 UI 里管理 DNS、ACL 等配置，还需要挂载 Headscale 配置文件和 Docker socket。官方文档也说明了 Docker 模式、`/admin` 访问路径和 API key 登录方式。([Headplane](https://headplane.net/install/docker "Docker | Headplane"))

### 2.3 Docker Compose

`docker-compose.yml`：

```yaml
services:
  headplane:
    image: ghcr.io/tale/headplane:0.6.2
    container_name: headplane
    restart: unless-stopped
    volumes:
      - "./headplane-config/config.yaml:/etc/headplane/config.yaml"
      # This should match headscale.config_path in your config.yaml
      - "./headscale-config/config.yaml:/etc/headscale/config.yaml"

      # If using dns.extra_records in Headscale (recommended), this should
      # match the headscale.dns_records_path in your config.yaml
      - "./headscale-config/dns_records.json:/etc/headscale/dns_records.json"

      # Headplane stores its data in this directory
      - "./headplane-data:/var/lib/headplane"

      # If you are using the Docker integration, mount the Docker socket
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
      - "/etc/localtime:/etc/localtime:ro"
    networks:
      - caddy_network

  headscale:
    image: headscale/headscale:0.28.0
    restart: unless-stopped
    container_name: headscale
    # 这里可以先映射端口，测试访问成功后再配置caddy路由
    # ports:
    #   - "127.0.0.1:8080:8080"
    #   - "127.0.0.1:9090:9090"
    volumes:
      - "./headscale-data:/var/lib/headscale"
      - "./headscale-config:/etc/headscale"
      - "/etc/localtime:/etc/localtime:ro"
    command: serve
    healthcheck:
      test: ["CMD", "headscale", "health"]
      interval: 30s
      timeout: 10s
      retries: 3
    networks:
      - caddy_network

networks:
  caddy_network:
    external: true
```

启动：

```bash
docker compose up -d
docker logs -f headscale
docker logs -f headplane
```

检查健康状态：

```bash
docker exec -it headscale headscale health
```

也可以从公网访问：

```bash
curl -I https://127.0.0.1:8080/health
```

Headscale 官方文档也建议先确认公网 health endpoint 可访问，再继续注册节点。([Headscale](https://headscale.net/stable/usage/getting-started/ "Getting started - Headscale"))

---

## 3. 配置 Caddy 路由

我的 Caddy 是统一的 wildcard 入口，所以只加一个 matcher：

```caddyfile
@tailscale {
  host tailscale.example.net
}

handle @tailscale {
  route {
    # Headplane 默认在 /admin
    @admin path /admin*
    handle @admin {
      reverse_proxy headplane:3000
      header {
        Access-Control-Allow-Origin  *
        Access-Control-Allow-Headers *
        Access-Control-Allow-Methods GET, POST, OPTIONS
      }
    }

    # 其余全部路由到 Headscale
    handle {
      reverse_proxy headscale:8080
    }
  }
}
```

如果你不是 wildcard Caddyfile，也可以单独写：

```caddyfile
tailscale.example.net {
  encode zstd gzip

  @admin path /admin*
  handle @admin {
    reverse_proxy headplane:3000
  }

  handle {
    reverse_proxy headscale:8080
  }
}
```

注意：这里只代理 `tailscale.example.net`。  
`derp.example.net:8443` 不要走这个 Caddy HTTP 反代。

重载 Caddy：

```bash
docker exec -it caddy caddy reload --config /etc/caddy/Caddyfile
```

也可以直接
```bash
docker compose down && docker compose up -d
```

---

## 4. 部署与接入客户端

### 4.1 生成 Headplane 访问密钥

进入 Headscale 容器，生成 API Key：

```bash
docker exec -it headscale headscale apikeys create --expiration 999d
```

然后打开：

```text
https://tailscale.example.net/admin
```

粘贴 API Key 登录。

不配置 OIDC 的情况下，Headplane 不是传统“用户名 + 密码”的永久账户体系，而是靠 Headscale API Key 登录。想要更像正常后台账号，需要后续单独接 OIDC。

### 4.2 创建 Headscale 用户

如果不想在 Web Console 里创建，也可以 CLI 创建：

```bash
docker exec -it headscale headscale users create username
docker exec -it headscale headscale users list
```

Headscale 0.28 以后不要再按老教程写 `namespace` 了，直接用 `users`。

### 4.3 Web Console 配置

进入 Headplane 后，建议先做这些：

1. 配置 Tailnet Name
    
2. 打开 MagicDNS（可选）
    
3. 确认 DNS Base Domain 不和 `tailscale.example.net` 冲突
    
4. 确认 DERP Map 能看到 `My DERP`
    
5. 创建 Pre-auth Key
    

如果你只是个人使用，Pre-auth Key 可以选：

```text
Reusable:   按需开启
Ephemeral:  一般关闭
Expiration: 按需设置
```

### 4.4 命令行生成 Pre-auth Key

也可以 CLI 生成。先看用户 ID：

```bash
docker exec -it headscale headscale users list
```

然后创建：

```bash
docker exec -it headscale headscale preauthkeys create --user <USER_ID>
```

如果需要复用、调整有效期等参数，先看当前版本帮助：

```bash
docker exec -it headscale headscale preauthkeys create --help
```

### 4.5 Linux 客户端加入

```bash
sudo tailscale up \
  --login-server="https://tailscale.example.net" \
  --authkey="你的 preauthkey"
```

如果之前已经登录过官方 Tailscale 或其他 Headscale，建议加 `--reset`：

```bash
sudo tailscale up --reset \
  --login-server="https://tailscale.example.net" \
  --authkey="你的 preauthkey"
```

如果你不想让 Headscale 接管 DNS：

```bash
sudo tailscale up --reset \
  --login-server="https://tailscale.example.net" \
  --authkey="你的 preauthkey" \
  --accept-dns=false
```

如果你打开了 MagicDNS，就不要加 `--accept-dns=false`。

### 4.6 Windows 客户端加入

管理员 PowerShell：

```powershell
tailscale up --login-server="https://tailscale.example.net" --authkey="你的 preauthkey"
```

如果旧状态干扰：

```powershell
tailscale up --reset --login-server="https://tailscale.example.net" --authkey="你的 preauthkey"
```

### 4.7 检查 DERP 是否生效

客户端执行：

```bash
tailscale netcheck
tailscale debug derp-map
```

看输出里是否有类似：

```text
My DERP
derp.example.net
```

再找另一个节点测试：

```bash
tailscale ping <对端节点名或 100.x 地址>
```

如果直连成功，会显示 direct；如果走中继，会显示 DERP。DERP 是保底，不是目标。能直连当然优先直连，不能直连时再走自建中继。

---

## 5. 开启防偷

DERP 不开验证的话，只要别人拿到你的 DERP Map，就可能把你的中继当公共节点用。 ~~小机器带宽本来就贵，不能当赛博慈善家。~~

防偷有两种思路。

### 5.1 方案一：Headscale `/verify`

这是我更推荐给 `Headscale + Derper` 的方式。Headscale 文档里提到，第三方 derper 可以通过 `-verify-client-url` 指向 Headscale 的 `/verify` 端点，由 Headscale 判断客户端是否属于本 Tailnet；`-verify-client-url-fail-open` 控制 Headscale 不可达时是放行还是拒绝，默认行为偏宽松，强迫症可以关掉。([Headscale](https://headscale.net/stable/ref/derp/ "DERP - Headscale"))

~~当然我是懒狗，多derp单headscale，所有derp用一套配置CtrlCV更省事，我用的第二套，~~ **第一套我没测**

修改 DERP 的启动参数：

```yaml
command:
  - derper
  - -hostname
  - derp.example.net
  - -certdir
  - /app/certs
  - -certmode
  - manual
  - -verify-client-url
  - https://tailscale.example.net/verify
  - -verify-client-url-fail-open=false
  - -home
  - blank
```

然后重启：

```bash
docker compose up -d
docker logs -f derp
```

确保 derp 容器能访问：

```bash
docker exec -it derp wget -qO- https://tailscale.example.net/health
```

如果容器访问自己的公网域名不通，先修 DNS/NAT 回环/防火墙，不要急着怪 Headscale。

### 5.2 方案二：`-verify-clients=true` + 本机 tailscaled

如果你想沿用传统方式，也可以让 derper 通过本机 `tailscaled.sock` 判断客户端身份。

前提：

1. DERP 宿主机安装并运行 Tailscale 客户端。
    
2. 这个 Tailscale 客户端已经加入你的 Headscale tailnet。
    
3. derper 容器挂载 `/var/run/tailscale/tailscaled.sock`。
    
4. 启动参数打开 `-verify-clients=true`。
    

修改：

```yaml
volumes:
  - ./certs/derp.example.net:/app/certs:ro
  - /var/run/tailscale/tailscaled.sock:/var/run/tailscale/tailscaled.sock

command: sh -c "
  derper \
	-hostname derp.example.net \
	-certdir /app/certs \
	-certmode manual \
	- verify-clients true \
	-home blank"

# 一些可选启动参数：
# -verify-clients true 启用客户端验证，防止DERP被偷，需要配合Tailscale使用，且需要挂载Tailscale的套接字文件
# -home blank 访问DERP返回空白界面，简单防止DERP服务被扫。
# -home https://xxx.com 访问DERP跳转指定URL
```

Tailscale 官方 derper README 里也提醒了：如果使用 `--verify-clients`，同机需要运行 `tailscaled`，并且客户端需要对 derper 旁边的 tailscaled 可见。([GitHub](https://github.com/tailscale/tailscale/blob/main/cmd/derper/README.md "tailscale/cmd/derper/README.md at main · tailscale/tailscale · GitHub"))

重启后再次测试：

```bash
docker compose up -d
tailscale status
tailscale netcheck
tailscale debug derp-map
tailscale ping <对端节点>
```

---

## 常见坑

### 1. Headscale 启动失败：DERPMap 为空

检查：

```bash
curl -fsSL https://assets.example.net/tailscale/derpmap.json
docker logs headscale
```

如果使用 `urls`，外部 DERP Map 应为 JSON；如果使用 `paths`，本地 DERP Map 应为 YAML。不要混。

### 2. DERP 网页能打开，但客户端不用它

检查 DERP Map 里的端口：

```json
"DERPPort": 8443,
"STUNPort": 3478
```

再检查云防火墙、安全组、本机防火墙是否都放通了 `8443/tcp` 和 `3478/udp`。

### 3. `tailscale netcheck` 过了，但实际还是连不上

`netcheck` 只能说明一部分网络状态正常，不代表所有路径都完全可用。继续看：

```bash
tailscale debug derp-map
tailscale ping <peer>
tailscale status
```

必要时看客户端日志。

### 4. Headplane 登录不了

先重新生成 API Key：

```bash
docker exec -it headscale headscale apikeys create --expiration 999d
```

再确认 Headplane 配置中的：

```yaml
headscale:
  url: "http://headscale:8080"
  public_url: "https://tailscale.example.net"
```

`url` 是容器内访问 Headscale 的地址；`public_url` 是浏览器和用户看到的公网地址。

### 5. MagicDNS 打开后系统 DNS 被改乱

客户端加入时可以临时禁用：

```bash
tailscale up --reset \
  --login-server="https://tailscale.example.net" \
  --authkey="你的 preauthkey" \
  --accept-dns=false
```

等 Headscale 的 DNS 配置确认没问题后再打开。

---

## 总结

至此，控制平面和中继平面都迁到了自己手里：

```text
客户端
  ↓
tailscale.example.net
  ↓
Headscale 控制平面
  ↓
下发 DERP Map / DNS / ACL

客户端 A ←→ 客户端 B
  ↳ 能直连就直连
  ↳ 不能直连就走 derp.example.net:8443
```

这套方案的优点很明显：

- 不依赖 Tailscale 官方控制台
    
- 自己管理用户、设备、DNS、ACL
    
- DERP 节点位置可控，延迟更低
    
- 开启 verify 后，不容易被人白嫖中继
    

缺点也同样明显：

- 证书、端口、DNS、配置都要自己维护
    
- DERP/Headscale 都是关键基础设施，挂了就会影响新连接和中继
    
- 配置文件格式和版本变化比较快，升级前要看 release note
    

不过折腾嘛，本来就是这样：  
先把能跑的跑起来，再把跑起来的修到好用。
