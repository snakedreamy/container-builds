# syntax=docker/dockerfile:1

ARG N8N_VERSION=latest

# 1) 字体准备阶段：用标准 alpine 安装字体（这里肯定有 apk）
FROM alpine:3.22 AS fonts
RUN set -eux; \
    apk add --no-cache \
      ttf-dejavu \
      font-noto-cjk \
      fontconfig; \
    fc-cache -f

# 2) 最终镜像：仍然基于官方 n8n 镜像（哪怕它是 hardened、没有 apk）
FROM ghcr.io/n8n-io/n8n:${N8N_VERSION}

USER root

# 只拷字体文件（最通用、风险最低）
COPY --from=fonts /usr/share/fonts /usr/share/fonts

# 如果底座本身带 fontconfig，则刷新缓存；没有也不报错
RUN set -eux; \
    if command -v fc-cache >/dev/null 2>&1; then \
      fc-cache -f; \
    else \
      echo "fc-cache not found in base image; skipped cache refresh"; \
    fi

USER node