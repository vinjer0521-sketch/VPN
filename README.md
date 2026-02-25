#此文档为利用Docker部署VPN后进行管理的指令集，用于容器的日常维护。
Shadowsocks Docker 管理指令集：
本文档基于之前成功部署的环境整理，假设：
自定义镜像名：my-ss-obfs
容器名：ss-server
配置文件路径：~/shadowsocks/config.json
使用的端口：11111（可根据实际替换）

1.镜像构建（仅首次执行）
如果尚未构建包含 obfs 插件的镜像，执行以下步骤：
'''
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
'''




