# syntax=docker/dockerfile:1
# ↑ BuildKit 语法声明：允许更好的缓存/多阶段构建能力（建议保留）

# ---------------------------
# 这两个 ARG 用于让 workflow 传入版本号
# ---------------------------
ARG N8N_VERSION

# ---------------------------
# 第一阶段：使用官方 n8n 镜像作为基础（运行环境不动）
# 注意：我们后面会再 FROM 一次同样的镜像，这是正常的多阶段写法
# ---------------------------
FROM docker.n8n.io/n8nio/n8n:${N8N_VERSION} AS base

# ---------------------------
# 第二阶段：apk_donor（捐赠者镜像）
# 用一个“完整正常的 Alpine”把 apk-tools 装好
# 之后把 apk、apk 的配置、钥匙等拷贝到 n8n 镜像里
# ---------------------------
FROM alpine:3.20 AS apk_donor

# 安装 apk-tools 和 ca-certificates
# - apk-tools：提供 apk 命令本体及相关库
# - ca-certificates：TLS 证书链（防止 apk 拉包时报 SSL 问题）
RUN apk add --no-cache apk-tools ca-certificates

# ---------------------------
# 第三阶段：最终镜像（仍然基于官方 n8n 镜像）
# 我们只“补回 apk + 安装 git”，不改变启动逻辑
# ---------------------------
FROM docker.n8n.io/n8nio/n8n:${N8N_VERSION}

# 切到 root 用户（因为要写系统目录、安装包）
USER root

# ---------------------------
# 1) 拷贝 apk 二进制和必要目录
# Alpine 中 apk 常见路径如下：
# - /sbin/apk           apk 命令
# - /lib/apk           apk 运行依赖库
# - /etc/apk           仓库配置/keys 等
# - /usr/share/apk     一些共享数据
# ---------------------------
COPY --from=apk_donor /sbin/apk /sbin/apk
COPY --from=apk_donor /lib/apk /lib/apk
COPY --from=apk_donor /etc/apk /etc/apk
COPY --from=apk_donor /usr/share/apk /usr/share/apk

# ---------------------------
# 2) 拷贝 CA 证书（让 apk/https 能正常工作）
# 有些精简镜像里可能没这个，导致访问 https 源失败
# ---------------------------
COPY --from=apk_donor /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/ca-certificates.crt

# ---------------------------
# 3) 确保 /etc/apk/repositories 存在
# 有些精简镜像可能没有 repositories 配置，apk 会找不到仓库
# 这里我们写入 Alpine 官方源（你可改成镜像站）
# ---------------------------
RUN if [ ! -f /etc/apk/repositories ]; then \
      mkdir -p /etc/apk && \
      printf '%s\n' \
        'https://dl-cdn.alpinelinux.org/alpine/v3.20/main' \
        'https://dl-cdn.alpinelinux.org/alpine/v3.20/community' \
        > /etc/apk/repositories ; \
    fi

# ---------------------------
# 4) 安装 git
# 很多第三方节点如果依赖 github:xxx/yyy 这种方式，会需要 git
#（否则 npm/pnpm 会出现 spawn git ENOENT 之类的错误）
# ---------------------------
RUN apk add --no-cache git

# ---------------------------
# 可选：如果你需要在容器内使用“系统 Chromium”运行 puppeteer，
# 可以打开下面这一行（镜像会变大）
# 什么时候需要？
# - 生产环境无法下载 puppeteer 自带 Chromium
# - 或者你希望统一使用系统 Chromium 并控制版本
# ---------------------------
# RUN apk add --no-cache chromium nss freetype harfbuzz ttf-freefont

# ---------------------------
# 最重要：不要改 entrypoint/CMD
# 官方 n8n 镜像默认使用非 root 的 node 用户运行
# 我们把用户切回去，保持原有安全模型
# ---------------------------
USER node