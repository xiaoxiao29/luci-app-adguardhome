# Development container for luci-app-adguardhome (OpenWrt SDK build / edit).
#
# 平台强制 linux/amd64：OpenWrt SDK 自带的宿主侧工具（feeds、defconfig、
# luci-base/host 的 po2lmo 等）都是 x86_64 Linux 二进制。本机即使是
# Apple Silicon（arm64），构建/运行此镜像也会自动走 Rosetta/x86 模拟；
# 本包编译时几乎没有重量级编译，模拟开销可忽略。
FROM --platform=linux/amd64 ubuntu:24.04

ENV DEBIAN_FRONTEND=noninteractive \
    LANG=C.UTF-8

# OpenWrt 官方文档推荐的包编译宿主依赖（build-essential … xsltproc），
# 与 .github/workflows/release_all_versions.yml 用到的工具（zstd/rsync…），
# 加上本地静态检查工具与 JetBrains 后端在容器内运行时需要的基础库。
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        build-essential clang flex bison gawk git ca-certificates \
        libncurses-dev libssl-dev libelf-dev swig xsltproc \
        python3 python3-setuptools python3-yaml \
        subversion rsync unzip xz-utils zstd file wget curl time sudo procps \
        lua5.1 lua-check shellcheck \
        libxext6 libxrender1 libxtst6 libxi6 libfreetype6 \
    && rm -rf /var/lib/apt/lists/*

# 非 root 用户（JetBrains/习惯性做法）。UID=501 对应 macOS 首个用户，
# 使 IDEA 以绑定挂载方式引入的宿主机文件对容器用户同样可写。
# 在 Linux 宿主上开发时请改成 `id -u` 的输出。
RUN useradd -m -s /bin/bash -u 501 openwrt \
    && echo "openwrt ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/openwrt \
    # SDK/feeds 缓存目录；同名 docker volume 首次挂载会继承这里的属主
    && mkdir -p /home/openwrt/.openwrt \
    && chown -R openwrt:openwrt /home/openwrt/.openwrt

USER openwrt
WORKDIR /home/openwrt
