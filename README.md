Shadowsocks + obfs 一键部署指南 (Docker)
本文档基于 CentOS 9 编写，适用于在 VPS 上快速搭建带混淆的 Shadowsocks 服务。所有步骤均已测试通过，请按顺序执行。

## 搭建自己的 VPN

很多时候我们需要使用 Google 查询第一手资料，比如官方的文档网站，或者一些文献，虽然市面上有一些 VPN 工具，但是它们不是很稳定，并且不是很安全，比较好的方式就是自己搭建一个属于自己的 VPN 服务，自己掌控。

## 优惠购买云服务器vultr

在搭建之前需要一台境外的云服务器，而 [vultr]([https://www.vultr.com/?ref=9871882]) 服务商比较稳定，安全，相当于境内的阿里云。

值得说的一点是， [vultr](https://www.vultr.com/?ref=9871882) 给新用户的福利相当给力，登录即可赠送100-300美金免费额度，你可以点击 [vultr 专属赠送新用户 100 美元](https://www.vultr.com/?ref=9871882) 进去抢先注册。

右上角有注册按钮，你也可以切换成中文界面：

<img width="1446" alt="vpn搭建教程" src="https://user-images.githubusercontent.com/84239400/119019442-be0cc500-b98c-11eb-895b-0d6300dce93d.png">

![](https://user-images.githubusercontent.com/84239400/119019760-0d52f580-b98d-11eb-9837-aba0990f1dfb.png)

接着使用邮箱和密码就可以注册了。

![](https://user-images.githubusercontent.com/84239400/119020042-4ee3a080-b98d-11eb-8341-bfc30f4b103c.png)

进去之后你可以看到这个页面，说明你已经通过 [vultr 专属优惠链接](https://www.vultr.com/?ref=9871882) 获得了 100 美元赠送的资格：

![](https://user-images.githubusercontent.com/84239400/119020173-733f7d00-b98d-11eb-8ecc-a1afeb556fe6.png)

现在你只要选择支付宝充值 10 美元以上，就可以获得额外赠送的 100 美元。

![](https://user-images.githubusercontent.com/84239400/119020280-966a2c80-b98d-11eb-9113-8f5f0b75c5ea.png)

接下来就可以在这个平台选购云服务器了。

点击页面左边菜单栏的 「Products」进入服务器选购页面。

### Choose Server 选择服务器

选择 Dedicated cpu/Shared cpu均可（后者性价比更高） 即可：

![](<img width="423" height="432" alt="image" src="https://github.com/user-attachments/assets/ca31bc55-b042-489a-be2b-f7d53472af82" />
)

### Server Location

服务器的位置，可以选择美国地区，比如纽约；也可以选择亚洲的服务器，如日本的东京、大阪：

<img width="1026" height="609" alt="image" src="https://github.com/user-attachments/assets/5a42d70e-6cf6-4a83-bdd6-67c24804597e" />


### Server Type

这里我们选择内存和带宽，1G内存用于部署VPN完全够用，带宽则取决于你每个月的使用情况自己决定

<<img width="1583" height="870" alt="image" src="https://github.com/user-attachments/assets/4e3f00fc-7a35-488e-9751-0769834346ca" />
>

### Server Size

这里可以看到要部署的服务器的情况，如果需要选择自动备份，每月需要多1.2刀，注意不要选择IPv6 ONLY的，要不然无法搭建使用。

<img width="1044" height="1335" alt="image" src="https://github.com/user-attachments/assets/da7394bf-a88a-4b2f-afc4-30d323f78671" />


选择完了之后，我们点击下面的Configure Software选择需要使用的操作系统，选择CentOS（9 Stream X64），

<img width="1056" height="1206" alt="image" src="https://github.com/user-attachments/assets/f8d2f838-9d6f-44bb-b686-3ffdebe8ff94" />


点击Configure，需要一点时间用于部署。这时候你就拥有了自己的一台云服务器了：

<img width="1959" height="192" alt="image" src="https://github.com/user-attachments/assets/f2c060c2-3720-4336-aed2-91e26d53137a" />

点击 Cloud Instance ，可以看到你服务器的 IP 地址和密码：

<img width="1860" height="807" alt="image" src="https://github.com/user-attachments/assets/55c59cb9-3beb-43cb-adea-3ac2deb3c941" />


## 连接到你的服务器

### windows 系统连接到 vultr 服务器

windows 可以下载[PuTTY](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html)工具。

打开之后选择 session，将你在 vultr 网上得到的服务器 ip 复制过来，输入到这里：

![](https://user-images.githubusercontent.com/84239400/119023900-037fc100-b992-11eb-85ea-525a61a69657.png)

Port 默认 22，Connection Type 选择 SSH，接着点击 Open，连接进去之后输入你云服务器的密码。

为了便于我们粘贴，在Window-Selection中选择Windows，这样我们可以直接使用鼠标右键进行复制和粘贴

<img width="675" height="684" alt="image" src="https://github.com/user-attachments/assets/cd89ebed-e5be-4098-b32d-8fde647941af" />


### Linux 和 MacOS 连接到 vultr 服务器

Linux 和 MacOS 可以打开终端，使用如下命令连接：

```
ssh root@xx.xx.xxx.xx
```

`xx.xx.xxx.xx`就是你的服务器上的 IP 地址。

连接进去之后输入 yes 后按回车，接着输入你云服务器的密码，即可使用云服务器。

<img width="772" alt="vpn搭建教程" src="https://user-images.githubusercontent.com/84239400/119024049-32963280-b992-11eb-8c05-db6df1f7fbfa.png">



## 快速搭建 VPN

### 安装Docker

```
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io
sudo systemctl start docker
sudo systemctl enable docker
```

<img width="1010" height="584" alt="image" src="https://github.com/user-attachments/assets/c222eac3-1707-414a-a416-4c935030a712" />


### 验证安装:

```
sudo docker version
```
<img width="771" height="618" alt="image" src="https://github.com/user-attachments/assets/78e8248b-36b4-46e5-a458-d08ff83a73fa" />


### 准备 Shadowsocks 配置文件，创建配置目录和文件，请将 your_password 替换为强密码，端口可自行修改,加密方式为aes-256-gcm

```
mkdir -p ~/shadowsocks
cat > ~/shadowsocks/config.json <<EOF
{
    "server": "0.0.0.0",
    "server_port": 11111,
    "password": "your_password",
    "method": "aes-256-gcm",
    "fast_open": false
}
EOF
```

<img width="480" height="222" alt="image" src="https://github.com/user-attachments/assets/0e4126c0-b72c-45c0-9f8c-202a78055eb7" />

###构建包含 obfs 插件的 Docker 镜像，官方镜像缺少 obfs-server，我们需要手动构建包含该插件的镜像。

```
mkdir -p ~/shadowsocks-obfs && cd ~/shadowsocks-obfs
```
创建Dockerfile：

```
cat > Dockerfile <<EOF
FROM shadowsocks/shadowsocks-libev

USER root

RUN set -ex \\
    && apk add --no-cache --virtual .build-deps \\
        autoconf \\
        automake \\
        build-base \\
        libtool \\
        linux-headers \\
        gettext \\
        asciidoc \\
        xmlto \\
        libsodium-dev \\
        libev-dev \\
        c-ares-dev \\
        mbedtls-dev \\
        pcre2-dev \\
        git \\
    && git clone https://github.com/shadowsocks/simple-obfs.git /tmp/simple-obfs \\
    && cd /tmp/simple-obfs \\
    && git submodule update --init --recursive \\
    && ./autogen.sh \\
    && ./configure --disable-documentation \\
    && make \\
    && make install \\
    && apk del .build-deps \\
    && rm -rf /tmp/simple-obfs

ENV PATH="/usr/local/bin:\${PATH}"
EOF
```

构建镜像（耗时约几分钟）：

```
docker build -t my-ss-obfs .
```

###运行 Shadowsocks 容器：
使用刚构建的镜像启动容器，映射端口并启用 HTTP 混淆

```
sudo docker run -d --name ss-server \
    -v ~/shadowsocks/config.json:/etc/shadowsocks-libev/config.json \
    -p 11111:11111/tcp \
    -p 11111:11111/udp \
    --restart always \
    my-ss-obfs \
    ss-server -c /etc/shadowsocks-libev/config.json \
    --plugin obfs-server \
    --plugin-opts "obfs=http;obfs-host=www.bing.com"
```
<img width="951" height="284" alt="image" src="https://github.com/user-attachments/assets/20b7e8bf-3c87-48fd-bd38-e7addee2add1" />


查看启动日志，确认服务正常：

```
sudo docker logs ss-server
```
预期输出为：
<img width="918" height="222" alt="image" src="https://github.com/user-attachments/assets/93f2c81e-0cab-4408-ab5d-df4ec2965bed" />
可以看到11111为我的端口，随机混淆端口为36757

###防火墙放行端口
```
sudo firewall-cmd --zone=public --add-port=11111/tcp --permanent
sudo firewall-cmd --add-port=11111/udp --permanent
sudo firewall-cmd --reload
```
<img width="951" height="156" alt="image" src="https://github.com/user-attachments/assets/e701be39-9249-4f11-922f-3b0fa8012d6c" />
查看已放行端口：
```
sudo firewall-cmd --zone=public --list-ports
```
<img width="927" height="48" alt="image" src="https://github.com/user-attachments/assets/26910422-7c5f-4fd5-9565-b158fde0a8ba" />
可以看到11111的端口已被放行

###（可选）开启 BBR 加速
```
echo "net.core.default_qdisc=fq" | sudo tee -a /etc/sysctl.conf
echo "net.ipv4.tcp_congestion_control=bbr" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```
<img width="954" height="153" alt="image" src="https://github.com/user-attachments/assets/a8d43acb-0dec-4656-8f5a-5104022b80cf" />

验证 BBR 已启用：
```
sysctl net.ipv4.tcp_congestion_control   # 应输出 bbr
lsmod | grep bbr                          # 应有 tcp_bbr 模块
```
<img width="864" height="90" alt="image" src="https://github.com/user-attachments/assets/3225bce6-0e6c-44d8-a8a4-e46e45aaeb2e" />



## 开始使用 VPN

### PC 端使用 VPN

点击下载[shadowsocks](https://github.com/shadowsocks/shadowsocks-windows/releases/download/4.4.0.0/Shadowsocks-4.4.0.185.zip)，可参考：[Shadowsocks Windows客户端快速使用指南](https://www.howru.cc/articles/414.html)

 ### 手机使用 VPN

去应用市场下载 shadowsocks，配置好你自己云服务器得到的 IP 地址（你的yultr服务器的IP），端口号，密码，加密方式，开启代理即可访问。


### 客户端配置
下载simple-obfhttps://github.com/shadowsocks/simple-obfs/releases/tag/v0.0.5，解压文件，将其中的obfs-local.exe文件放到Shadowsocks文件夹中
<img width="1062" height="612" alt="image" src="https://github.com/user-attachments/assets/a36fd22f-3d75-40d6-b930-1a7b39b64b5f" />

在 Shadowsocks 客户端中配置以下参数（以 Windows为例）：
服务器地址：你的服务器公网 IP
端口：11111
密码：your_password（与配置文件一致）
加密方式：aes-256-gcm
插件：选择 simple-obfs 或 obfs-local
插件选项：obfs=http;obfs-host=www.bing.com
<img width="773" height="699" alt="image" src="https://github.com/user-attachments/assets/03efeb80-f298-49e0-8366-29400b6332d4" />





## 访问 Google

速度可以：

![](https://user-images.githubusercontent.com/84239400/119024332-7b4deb80-b992-11eb-8616-d63fd5aca694.png)

## 更多VPN搭建教程
- [vpn搭建教程](https://www.google.com/search?q=vpn%E6%90%AD%E5%BB%BA%E6%95%99%E7%A8%8B&oq=vpn%E6%90%AD%E5%BB%BA%E6%95%99%E7%A8%8B&aqs=chrome.0.69i59j35i39j0l3j69i60j69i65j69i60.2589j0j7&sourceid=chrome&ie=UTF-8)

## 赞赏
喜欢就给我个star或者fork一下吧❤️，谢谢！


