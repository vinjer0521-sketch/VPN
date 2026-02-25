#此文档为利用Docker部署VPN后进行管理的指令集，用于容器的日常维护。

Shadowsocks Docker 管理指令集：

本文档基于之前成功部署的环境整理，假设：

自定义镜像名：my-ss-obfs

容器名：ss-server

配置文件路径：~/shadowsocks/config.json

使用的端口：11111（可根据实际替换）

1.镜像构建（仅首次执行）
如果尚未构建包含 obfs 插件的镜像，执行以下步骤：
```
# 创建构建目录并进入
mkdir -p ~/shadowsocks-obfs && cd ~/shadowsocks-obfs

# 创建 Dockerfile（已包含 obfs 插件安装）
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

# 构建镜像
docker build -t my-ss-obfs .
```

2. 创建/编辑配置文件

配置文件位于 ~/shadowsocks/config.json，可用任意编辑器修改：
```
vi ~/shadowsocks/config.json
```
在文本编辑模式下，移动光标进行位置选择，按下i键进入插入模式，进行编辑，移动光标到指定位置进行删除，输入你的内容，完成后按Esc退出编辑模式，输入

:w  保存文件但不退出

:wq  保存文件并退出

:q!  不保存文件强制退出

示例配置（请修改密码和端口）：
```
{
    "server": "0.0.0.0",
    "server_port": 11111,
    "password": "your_password",
    "method": "aes-256-gcm",
    "fast_open": false
}
```
注意：server_port 必须与后续 Docker 端口映射的容器端口一致。

3. 启动容器

首次启动
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

如果已存在同名容器，先删除再运行
```
sudo docker stop ss-server && sudo docker rm ss-server
# 然后执行上述 run 命令
```
4. 查看容器状态
查看运行中的容器：
```
sudo docker ps
```

查看所有容器（包括已停止）
```
sudo docker ps -a
```

查看容器日志（实时）
```
sudo docker logs ss-server
```

查看容器日志（持续跟踪）
```
sudo docker logs -f ss-server
```

查看端口映射
```
sudo docker port ss-server
```

5. 容器生命周期管理
操作命令
```
停止容器	sudo docker stop ss-server
启动容器	sudo docker start ss-server
重启容器	sudo docker restart ss-server
删除容器	sudo docker rm ss-server
强制删除	sudo docker rm -f ss-server
```

6. 修改配置后的生效方法
修改了 config.json 后，需要重启容器使配置生效：
```
sudo docker restart ss-server
```

7. 修改端口或插件选项
如果需要更改端口或混淆参数，建议删除旧容器并重新创建（因为端口映射在创建时固定）。
```
# 停止并删除旧容器
sudo docker stop ss-server && sudo docker rm ss-server

# 修改配置文件中的端口
vi ~/shadowsocks/config.json   # 修改 server_port 为新端口，如 22222

# 重新运行容器（注意 -p 参数使用新端口）
sudo docker run -d --name ss-server \
    -v ~/shadowsocks/config.json:/etc/shadowsocks-libev/config.json \
    -p 22222:22222/tcp \
    -p 22222:22222/udp \
    --restart always \
    my-ss-obfs \
    ss-server -c /etc/shadowsocks-libev/config.json \
    --plugin obfs-server \
    --plugin-opts "obfs=tls;obfs-host=www.bing.com"   # 示例改为 TLS 混淆
```

8. 进入容器内部（调试用）
```
sudo docker exec -it ss-server sh
```
进入后可以查看进程、网络等，例如：
```
ps aux
netstat -tulpn
cat /etc/shadowsocks-libev/config.json
exit
```

9. 镜像管理
列出本地镜像
```
docker images
```

删除镜像
```
docker rmi my-ss-obfs
```
注意：删除前需确保没有容器使用该镜像。

拉取官方镜像（备用）
```
docker pull shadowsocks/shadowsocks-libev
```

10. 防火墙与安全组
无论容器如何配置，务必确保主机防火墙和云平台安全组放行了映射到主机的端口：
```
sudo firewall-cmd --zone=public --add-port=11111/tcp --permanent
sudo firewall-cmd --add-port=11111/udp --permanent
sudo firewall-cmd --reload
```
云平台控制台需添加入站规则允许 TCP 端口（如 11111）。

附：常用组合命令

快速重启并查看日志：
```
sudo docker restart ss-server && sudo docker logs -f ss-server
```
更新配置并重启：
```
vi ~/shadowsocks/config.json
sudo docker restart ss-server
sudo docker logs ss-server
```
完全重建（修改端口或插件）：
```
sudo docker stop ss-server && sudo docker rm ss-server
# 修改配置文件...
# 重新运行 run 命令
```

以上指令集覆盖了 Shadowsocks Docker 部署的日常管理操作。按需执行即可。如有其他需求，可参考具体命令的帮助文档（docker --help）。







