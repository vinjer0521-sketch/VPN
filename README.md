# Shadowsocks + simple-obfs 部署指南（Docker）

> 在境外 VPS 上快速搭建带流量混淆的 Shadowsocks 代理服务。
> 全部步骤已在 **CentOS Stream 9 + Docker** 上实测通过。
>
> 最后更新：2026-10-09

---

## ⚠️ 阅读须知（请先看这 4 条）

1. **这是「代理」不是「VPN」。** Shadowsocks 只接管应用的 HTTP/SOCKS 流量，不封装 IP 层、不自动接管全局路由。叫它 VPN 只是口语习惯；理解这一点，后面排查问题会容易很多（例如「为什么某些软件不走代理」）。
2. **合规提醒。** 本文仅供技术学习与访问公开的境外资料使用，请遵守你所在国家及服务器所在地的法律法规。
3. **本文含推广链接。** 通过文中的 Vultr 推广链接注册，作者会获得返利；你拿到的赠送额度不变。不想用推广链接，直接去 `vultr.com` 官网即可。
4. **截图为早期版本，界面可能已变化**，请以官网当前界面为准。

## 目录

- [一、为什么自己搭](#一为什么自己搭)
- [二、购买云服务器](#二购买云服务器)
- [三、连接到服务器](#三连接到服务器)
- [四、部署 Shadowsocks + obfs](#四部署-shadowsocks--obfs)
- [五、验收：确认服务真的可用](#五验收确认服务真的可用)
- [六、客户端配置](#六客户端配置)
- [七、故障排查](#七故障排查)
- [八、安全加固（建议做完再用）](#八安全加固建议做完再用)
- [九、进阶：这套方案的局限与升级路径](#九进阶这套方案的局限与升级路径)
- [十、FAQ](#十faq)
- [附录 A：Debian/Ubuntu 对照命令](#附录-adebianubuntu-对照命令)
- [附录 B：运维速查](#附录-b运维速查)

---

## 一、为什么自己搭

查第一手资料经常需要访问 Google、GitHub、官方文档和文献库。市面上的现成工具通常有两个问题：**不稳定**（共享线路、随时失效）和**不安全**（你并不知道中间人是谁）。自己搭一台服务器，线路和数据都掌握在自己手里。

## 二、购买云服务器

任何境外 KVM 云服务器都可以，本文以 [Vultr](https://www.vultr.com/?ref=9871882) 为例（相当于境内的阿里云）。

> 选购要点（比选哪家更重要）：
> - **地区**：日本（东京/大阪）、新加坡、美西，国内延迟较低
> - **配置**：1 核 1G 足够跑代理；按流量计费的注意月流量上限
> - **不要选 IPv6 ONLY** 的机型，否则国内很多网络连不上
> - **系统**：CentOS Stream 9（本文步骤）/ Debian 12 / Ubuntu 22.04+ 均可（见附录 A）

注册后进入控制台，左侧菜单 **Products → Deploy New Server**：

1. 选择 **Cloud Compute / Shared CPU**（性价比更高）
2. 选择机房位置（亚洲地区延迟更低）

   ![选择服务器位置](https://github.com/user-attachments/assets/5a42d70e-6cf6-4a83-bdd6-67c24804597e)
3. 选择规格（1G 内存够用，带宽按自己的月用量选）

   ![选择服务器规格](https://github.com/user-attachments/assets/4e3f00fc-7a35-488e-9751-0769834346ca)
4. 确认订单（自动备份每月额外约 1.2 美元，可不选；注意别选 IPv6 ONLY）

   ![确认订单](https://github.com/user-attachments/assets/da7394bf-a88a-4b2f-afc4-30d323f78671)
5. 点 **Configure Software** 选择操作系统，本文选 **CentOS 9 Stream x64**

   ![选择操作系统](https://github.com/user-attachments/assets/f8d2f838-9d6f-44bb-b686-3ffdebe8ff94)

部署完成后，点 **Cloud Instance** 查看服务器 IP 和 root 密码：

![服务器信息](https://github.com/user-attachments/assets/f2c060c2-3720-4336-aed2-91e26d53137a)

## 三、连接到服务器

### Windows：PuTTY

下载 [PuTTY](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html) 并打开：

1. **Session** 里填服务器 IP，Port 默认 `22`，Connection type 选 `SSH`，点 **Open**
2. 首次连接会提示指纹，点 Accept
3. 输入 root 密码（输入时不显示是正常的）

![PuTTY 连接](https://user-images.githubusercontent.com/84239400/119023900-037fc100-b992-11eb-85ea-525a61a69657.png)

**建议**：在 `Window → Selection` 里把鼠标模式设为 `Windows`，这样可以直接右键粘贴命令，省很多事。

![PuTTY 粘贴设置](https://github.com/user-attachments/assets/cd89ebed-e5be-4098-b32d-8fde647941af)

### Linux / macOS：终端

```bash
ssh root@你的服务器IP
```

首次连接输入 `yes` 回车，然后输入密码即可。

![终端连接](https://user-images.githubusercontent.com/84239400/119024049-32963280-b992-11eb-8c05-db6df1f7fbfa.png)

## 四、部署 Shadowsocks + obfs

### 4.1 安装 Docker

```bash
sudo dnf install -y dnf-plugins-core          # 提供 dnf config-manager 子命令，缺了会报错
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io
sudo systemctl enable --now docker
sudo docker version                            # 验证安装
```

![Docker 安装](https://github.com/user-attachments/assets/c222eac3-1707-414a-a416-4c935030a712)

![验证安装](https://github.com/user-attachments/assets/78e8248b-36b4-46e5-a458-d08ff83a73fa)

### 4.2 写 Shadowsocks 配置

> **密码不要用示例值。** 先生成一个强密码：
> ```bash
> openssl rand -base64 24
> ```

```bash
mkdir -p ~/shadowsocks
cat > ~/shadowsocks/config.json <<'EOF'
{
    "server": "0.0.0.0",
    "server_port": 11111,
    "password": "把你生成的强密码填这里",
    "method": "aes-256-gcm",
    "mode": "tcp_and_udp",
    "timeout": 300,
    "fast_open": false
}
EOF
```

> ⚠️ **两个容易踩的点**：
> - `"mode": "tcp_and_udp"` 必须写上，否则 ss-server **只监听 TCP**，下面映射的 UDP 端口是白搭（默认是 `tcp_only`）。不需要 UDP 流量（游戏、QUIC、部分语音）的话，也可以把 mode 去掉、同时删掉 UDP 端口映射，保持一致。
> - `server_port` 示例用的 `11111`，建议改成自己的端口，避免和别人的特征重合。

![配置文件](https://github.com/user-attachments/assets/0e4126c0-b72c-45c0-9f8c-202a78055eb7)

### 4.3 构建带 obfs 插件的镜像

官方镜像里没有 `obfs-server`，需要自己构建一个：

```bash
mkdir -p ~/shadowsocks-obfs && cd ~/shadowsocks-obfs

cat > Dockerfile <<'EOF'
FROM shadowsocks/shadowsocks-libev

USER root

RUN set -ex \
    && apk add --no-cache --virtual .build-deps \
        autoconf \
        automake \
        build-base \
        libtool \
        linux-headers \
        gettext \
        asciidoc \
        xmlto \
        libsodium-dev \
        libev-dev \
        c-ares-dev \
        mbedtls-dev \
        pcre2-dev \
        git \
    && git clone https://github.com/shadowsocks/simple-obfs.git /tmp/simple-obfs \
    && cd /tmp/simple-obfs \
    && git submodule update --init --recursive \
    && ./autogen.sh \
    && ./configure --disable-documentation \
    && make \
    && make install \
    && apk del .build-deps \
    && rm -rf /tmp/simple-obfs

ENV PATH="/usr/local/bin:${PATH}"
EOF

docker build -t my-ss-obfs .          # 约几分钟
```

> 说明：`simple-obfs` 项目已进入只读维护状态，官方 `shadowsocks/shadowsocks-libev` 镜像也很久没更新。**能跑，但不建议长期依赖**，升级路径见 [第九节](#九进阶这套方案的局限与升级路径)。

### 4.4 启动容器

```bash
sudo docker run -d --name ss-server \
    --restart always \
    --log-opt max-size=10m --log-opt max-file=3 \
    -v ~/shadowsocks/config.json:/etc/shadowsocks-libev/config.json:ro \
    -p 11111:11111/tcp \
    -p 11111:11111/udp \
    my-ss-obfs \
    ss-server -c /etc/shadowsocks-libev/config.json \
    --plugin obfs-server \
    --plugin-opts "obfs=http;obfs-host=www.bing.com"
```

**关于端口的正确理解（原文档这里容易看反）：**

```
公网 ──► 11111 (obfs-server，对外监听，伪装成访问 www.bing.com 的 HTTP 流量)
              │
              └──► 容器内部随机本地端口 (真正的 ss-server，例如 36757)
```

- 对外只需要放行 **11111**；
- 日志里那个随机端口（如 36757）是 **ss-server 在容器内部的监听端口**，不对外、也不需要放行。

```bash
sudo docker logs ss-server          # 确认服务正常，日志里会打印出端口信息
```

![启动日志](https://github.com/user-attachments/assets/93f2c81e-0cab-4408-ab5d-df4ec2965bed)

### 4.5 防火墙放行端口

```bash
sudo firewall-cmd --zone=public --add-port=11111/tcp --permanent
sudo firewall-cmd --zone=public --add-port=11111/udp --permanent
sudo firewall-cmd --reload
sudo firewall-cmd --zone=public --list-ports     # 确认已放行
```

![防火墙放行](https://github.com/user-attachments/assets/e701be39-9249-4f11-922f-3b0fa8012d6c)

![确认端口](https://github.com/user-attachments/assets/26910422-7c5f-4fd5-9565-b158fde0a8ba)

> ⚠️ **别忘了云厂商那一层防火墙。** Vultr / 阿里云 / 腾讯云控制台里通常还有一层 Cloud Firewall（安全组），如果启用了，**必须在那里也放行 11111**，否则系统防火墙放行了也连不上。这是最高频的「部署完连不上」原因。

### 4.6（可选）开启 BBR 加速

```bash
sysctl net.ipv4.tcp_congestion_control          # 先看当前值，可能已经是 bbr

if ! grep -q "^net.ipv4.tcp_congestion_control=bbr" /etc/sysctl.conf; then
    echo "net.core.default_qdisc=fq" | sudo tee -a /etc/sysctl.conf
    echo "net.ipv4.tcp_congestion_control=bbr" | sudo tee -a /etc/sysctl.conf
fi
sudo sysctl -p

sysctl net.ipv4.tcp_congestion_control          # 应输出 bbr
lsmod | grep bbr                                # 应有 tcp_bbr 模块
```

> 原写法直接 `tee -a` 会**重复追加**，多执行几次会在 `sysctl.conf` 里堆一堆重复行；上面的写法先判断再写。

![BBR](https://github.com/user-attachments/assets/3225bce6-0e6c-44d8-a8a4-e46e45aaeb2e)

## 五、验收：确认服务真的可用

部署完先别急着配客户端，在服务器上跑这三条：

```bash
sudo docker ps --filter name=ss-server          # 状态应为 Up
sudo docker logs ss-server --tail 20            # 不应有 error
sudo ss -tlnp | grep 11111                      # 应看到 obfs-server 在监听
```

然后在**你自己的电脑上**做一次真实穿透测试（Windows 用 `curl.exe`，注意 PowerShell 里 `curl` 是别名，必须写 `curl.exe`）：

```powershell
curl.exe -x socks5h://127.0.0.1:本地代理端口 https://api.openai.com/v1/models
```

- 返回 **`401` + `Missing bearer authentication`** → **链路完全正常**（只是没带 API key 而已）
- 返回 `curl: (28) 超时` / `(000)` → 回到[第七节](#七故障排查)排查

## 六、客户端配置

### 6.1 先搞清楚：哪些客户端支持这种「SS + obfs」配置

| 客户端 / 内核 | obfs（SIP003）支持 | 说明 |
|---|---|---|
| **mihomo（Clash.Meta）** | ✅ **内置** | 推荐。`plugin: obfs` 开箱即用 |
| v2rayN（xray 内核） | ❌ | xray 不支持 SIP003，面板里也没有「插件」字段 |
| v2rayN（sing-box 内核） | ⚠️ | 支持，但要另放插件可执行文件 |
| Shadowsocks-windows 4.4.0.185 | ✅ | 需要手动放 `obfs-local.exe`（见 6.3） |
| v2rayNG（Android，xray） | ❌ | |
| Clash Meta for Android / Shadowrocket | ✅ | |

> **这是很多人卡住的地方**：用 v2rayN（默认 xray 内核）添加这台服务器，填完地址端口密码加密方式，**怎么都连不上** —— 因为缺了 obfs 这一步，而面板里根本没有「插件」选项。解法：换 mihomo 内核，或用下面的完整配置。

### 6.2 推荐：mihomo（v2rayN / Clash Verge 等）

新建配置并填入（<u>把 IP、端口、密码换成你自己的</u>）：

```yaml
mixed-port: 7890            # 本机 HTTP + SOCKS5 混合入口
allow-lan: false
bind-address: 127.0.0.1
mode: rule
log-level: info
ipv6: false
unified-delay: true
tcp-concurrent: true
find-process-mode: off

dns:
  enable: true
  ipv6: false
  enhanced-mode: normal
  nameserver: [223.5.5.5, 119.29.29.29]     # 国内 DNS
  fallback: [1.1.1.1, 8.8.8.8]              # 国外 DNS

proxies:
  - name: "我的VPS"
    type: ss
    server: 你的服务器IP
    port: 11111
    cipher: aes-256-gcm
    password: "你的密码"
    udp: true
    plugin: obfs                            # ★ 关键：mihomo 内置 simple-obfs
    plugin-opts:
      mode: http
      host: www.bing.com                    # 与服务端 --plugin-opts 保持一致

proxy-groups:
  - name: PROXY
    type: select
    proxies: [我的VPS, DIRECT]

rules:
  - DOMAIN-SUFFIX,openai.com,PROXY
  - DOMAIN-SUFFIX,chatgpt.com,PROXY
  - GEOIP,lan,DIRECT
  - GEOIP,CN,DIRECT                          # 国内直连，不走代理
  - MATCH,PROXY                              # 其余全部走代理
```

> 用 v2rayN 的话：**设置 → 完整配置模板 → Core 类型选 `mihomo` → 粘贴上面的 YAML → 勾选启用**。
> 注意 `mixed-port` 要和 v2rayN 的「本地端口」一致，冲突时改其中一个即可。

**手机端 / 不想装 GUI**：也可以用任意支持 `ss + obfs` 的客户端（Shadowrocket、Clash Meta for Android、mihomo 命令行版）。

### 6.3 备选：Shadowsocks-windows 4.4.0.185

> ⚠️ 该客户端 **2021 年后已停止维护**，仅作过渡方案。已知缺陷：
> - PAC/规则库更新需要直连 GitHub，在国内**必然更新失败**，规则会一直停留在旧版本
> - 系统代理开关状态不稳定（实测会出现「已开启但注册表里又变回直连」）
> - 内置 privoxy 的 HTTP 代理端口**每次启动随机变化**，写死端口的软件重启后即失效
> - 不支持 shadowsocks-2022

**正确顺序（顺序反了会连不上）：**

1. 下载 [Shadowsocks-4.4.0.185.zip](https://github.com/shadowsocks/shadowsocks-windows/releases/download/4.4.0.0/Shadowsocks-4.4.0.185.zip) 并解压
2. 下载 [simple-obfs v0.0.5](https://github.com/shadowsocks/simple-obfs/releases/tag/v0.0.5)，解压后**把其中的 `obfs-local.exe` 复制到 Shadowsocks 程序目录**（与 `Shadowsocks.exe` 同一层）
3. 打开客户端，添加服务器，按下表填写

| 字段 | 填写内容 |
|---|---|
| 服务器地址 | 你的服务器公网 IP |
| 服务器端口 | `11111`（你改过就填你改的） |
| 密码 | 与 `config.json` 一致 |
| 加密 | `aes-256-gcm` |
| **插件程序** | **`obfs-local`**（不是 `simple-obfs`，这也是常见错误点） |
| **插件选项** | **`obfs=http;obfs-host=www.bing.com`** |
| 备注 | 随意 |

4. 保存 → 托盘菜单选中该服务器 → 选择「系统代理模式」→「PAC 模式」或「全局模式」

![客户端配置](https://github.com/user-attachments/assets/03efeb80-f298-49e0-8366-29400b6332d4)

![插件配置](https://github.com/user-attachments/assets/a36fd22f-3d75-40d6-b930-1a7b39b64b5f)

### 6.4 分享 / 导入：ss:// 链接格式

```
ss://<base64(加密方式:密码)>@服务器IP:端口/?plugin=obfs-local%3Bobfs%3Dhttp%3Bobfs-host%3Dwww.bing.com#备注
```

生成 base64 的方法（Windows PowerShell）：

```powershell
[Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes("aes-256-gcm:你的密码"))
```

> ⚠️ `ss://` 链接里含明文密码，**不要发到公开的群或仓库里**。

## 七、故障排查

### 7.1 分层排查（自上而下，哪层断了一目了然）

| 层级 | 命令（Windows 本地） | 正常表现 |
|---|---|---|
| ① 服务器 TCP 可达 | `Test-NetConnection 你的IP -Port 11111` | `TcpTestSucceeded : True` |
| ② 本地代理端口 | `Test-NetConnection 127.0.0.1 -Port 本地端口` | True（客户端已在运行） |
| ③ 端到端穿透 | `curl.exe -x socks5h://127.0.0.1:本地端口 https://api.openai.com/v1/models` | `401` |

①不通 → 防火墙/云安全组/端口被墙 ②不通 → 客户端没启动或端口填错 ③不通但①②正常 → 插件参数不一致（`obfs`/`obfs-host` 写错）、密码或加密方式不匹配

### 7.2 curl 返回码判读

| 现象 | 含义 | 处理 |
|---|---|---|
| `HTTP 401` | **链路完全正常**（只是没带 key） | ✅ 不用管 |
| `curl: (6) Could not resolve host` | DNS 问题 | 换 DNS（`223.5.5.5` / `1.1.1.1`） |
| `curl: (28) Connection timed out` / `code=000` | 连接被丢弃 | 查防火墙 + 云安全组；确认端口开放 |
| `curl: (35) SSL connect error` | TLS 握手被重置 | 换端口；或升级到 VLESS+Reality |
| `HTTP 403` + 响应头 `cf-mitigated: challenge` | **Cloudflare 人机质询，不是封 IP** | 用浏览器访问即可通过；不要用 curl 判断网站是否可访问 |
| 服务端本机 `curl https://chatgpt.com` 返回 403 | 机房 IP 被 Cloudflare 挑战，**属正常** | 不影响客户端使用，别以为服务器被封了 |
| 国内网站变慢 / 打不开国内银行 | 走了全局代理 | 客户端切「规则/PAC 模式」而不是「全局模式」 |

### 7.3 服务端排查

```bash
sudo docker logs ss-server --tail 50              # 看启动/报错
sudo docker restart ss-server                     # 重启容器
sudo ss -tlnp | grep 11111                        # 确认 obfs-server 在监听
sudo firewall-cmd --zone=public --list-ports      # 确认端口已放行
curl -sS https://ipinfo.io/json                   # 确认服务器出口 IP/地区（应为服务器所在国）
```

### 7.4 客户端连不上：先核对这四项

1. **端口**是否和 `server_port` 一致（容器映射是否也是同一个）
2. **密码**是否和 `config.json` 完全一致（注意首尾空格）
3. **加密方式**是否 `aes-256-gcm`
4. **插件**：`obfs-local` + `obfs=http;obfs-host=www.bing.com`，且 `obfs-local.exe` 确实已放进客户端目录

## 八、安全加固（建议做完再用）

```bash
# 1) SSH 用密钥登录，并禁用密码登录
ssh-keygen -t ed25519                       # 本地生成密钥
ssh-copy-id root@你的服务器IP                # 上传公钥
# 确认用密钥能登录后，在服务器上：
sudo sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo systemctl restart sshd

# 2) 装 fail2ban 防爆破
sudo dnf install -y epel-release && sudo dnf install -y fail2ban
sudo systemctl enable --now fail2ban

# 3) 只放行必要端口，其余关闭
sudo firewall-cmd --zone=public --list-ports

# 4) 及时打安全补丁
sudo dnf upgrade -y --security
```

其他要点：

- **别用示例端口 11111**，改成自己的
- **密码用 `openssl rand -base64 24` 生成**，不要用生日/常用词
- **不要公开分享 `ss://` 链接**（含明文密码）；多人使用建议一人一端口一密码，便于停用
- **限制 Docker 日志**（4.4 节已加 `--log-opt`），否则日志会把 VPS 磁盘写满 —— 这是 VPS「跑着跑着就挂了」的常见原因

## 九、进阶：这套方案的局限与升级路径

必须承认的事实：**`simple-obfs` 属于老一代混淆，抗主动探测能力已经较弱**（HTTP 头特征固定、没有真实 TLS），近年被识别的情况明显增多。它能用，但不应作为长期方案。

| 方案 | 抗封锁 | 客户端支持 | 说明 |
|---|---|---|---|
| **SS + simple-obfs**（本文） | 弱 | 广 | 最简单，适合过渡 |
| SS-2022 | 中 | mihomo / sing-box | 加密升级为 `2022-blake3-aes-256-gcm`，带重放保护 |
| **VLESS + Reality**（推荐） | **强** | v2rayN / sing-box / mihomo | 直接借用真实站点 TLS 指纹，无需域名和证书 |

升级到 VLESS + Reality 大约需要：服务端换成带 Reality 入站的 xray，客户端在 v2rayN 里导入一条 `vless://` 链接。**具体步骤可另开一篇文档**（欢迎提 issue 催更）。

## 十、FAQ

**Q：可以多台设备同时用吗？**
可以，同一份配置可以共用；但更推荐一人一个端口/密码，方便单独停用和统计。

**Q：会被封吗？**
IP 或端口被阻断是可能的。被墙的典型表现是「服务器能 ping 通，但客户端连不上」；此时换 IP / 换端口，或升级到第九节的方案。

**Q：为什么访问国内网站变慢了？**
客户端选了「全局模式」，国内流量也绕道境外了。切「规则/PAC 模式」，或在 mihomo 配置里保留 `GEOIP,CN,DIRECT`。

**Q：怎么换端口？**
改 `~/shadowsocks/config.json` 里的 `server_port` → 删掉旧容器重建（`docker rm -f ss-server` 后重跑 4.4 的命令，注意两处端口都要改）→ 防火墙放行新端口 → 客户端同步修改。

**Q：怎么更新？**
```bash
cd ~/shadowsocks-obfs
docker build --no-cache -t my-ss-obfs .
docker rm -f ss-server
# 然后重新执行 4.4 节的 docker run 命令
```

**Q：流量用超了怎么办？**
在云厂商控制台看用量并升级套餐；也可以在容器上加流量统计（`--network` 或 vnstat）。

---

## 附录 A：Debian/Ubuntu 对照命令

```bash
# 安装 Docker（官方脚本最省事）
curl -fsSL https://get.docker.com | sudo sh
sudo systemctl enable --now docker

# 防火墙改放行（Debian 默认用 nftables/ufw）
sudo ufw allow 11111/tcp
sudo ufw allow 11111/udp

# 其余步骤（config.json / Dockerfile / docker run）完全一致
```

`simple-obfs` 的 Dockerfile 基于 Alpine，**与宿主系统无关**，可以直接复用。

## 附录 B：运维速查

```bash
sudo docker ps -a --filter name=ss-server     # 状态
sudo docker logs -f ss-server                 # 实时日志
sudo docker restart ss-server                 # 重启
sudo docker rm -f ss-server                   # 删除（先备份 config.json）
sudo firewall-cmd --zone=public --list-ports  # 已放行端口
sysctl net.ipv4.tcp_congestion_control        # BBR 状态
cat ~/shadowsocks/config.json                 # 查看（含密码，注意别截图外发）
```

---

## 打完收工

喜欢就点个 star 或 fork 一下吧 ❤️

更多资料：可自行搜索「shadowsocks 搭建教程」「vless reality 配置」。如需转载，请注明出处。
