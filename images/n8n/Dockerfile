# syntax=docker/dockerfile:1

# ---------------------------
# 由 workflow 传入 n8n 版本号（如 2.7.5）
# 给一个默认值，避免 lint 警告（不影响 workflow 传参）
# ---------------------------
ARG N8N_VERSION=latest

# 基于官方 n8n 镜像（不改 entrypoint / CMD）
FROM docker.n8n.io/n8nio/n8n:${N8N_VERSION}

# 官方镜像默认不是 root。安装系统包需要 root
USER root

# ---------------------------
# 安装 git（优先 apt-get；少数情况下若基础镜像是 Alpine，则走 apk）
#
# 你之前尝试“从 Alpine 拷 apk 回填”，失败原因是：
# - 官方 n8n 镜像并不是 Alpine 运行环境（库/符号不兼容）
# 所以正确做法是：使用镜像自身的包管理器。
# ---------------------------
RUN set -eux; \
    if command -v apt-get >/dev/null 2>&1; then \
      echo "检测到 apt-get：使用 Debian/Ubuntu 方式安装 git"; \
      apt-get update; \
      apt-get install -y --no-install-recommends git ca-certificates; \
      rm -rf /var/lib/apt/lists/*; \
    elif command -v apk >/dev/null 2>&1; then \
      echo "检测到 apk：使用 Alpine 方式安装 git"; \
      apk add --no-cache git ca-certificates; \
    else \
      echo "基础镜像没有 apt-get 也没有 apk，无法安装 git。"; \
      echo "这通常意味着上游镜像极度精简（类似 distroless）。需要换更完整的基础镜像变体。"; \
      exit 1; \
    fi

# ---------------------------
# 可选：如果你最终确认 puppeteer 必须依赖“系统 Chromium”，
# 可以在这里安装（注意镜像会变大）
#
# Debian/Ubuntu 常见包名可能是 chromium 或 chromium-browser（不同发行版不同）
# 你可以等 puppeteer 报错后再最小化补齐
# ---------------------------
# RUN set -eux; \
#     if command -v apt-get >/dev/null 2>&1; then \
#       apt-get update; \
#       apt-get install -y --no-install-recommends chromium; \
#       rm -rf /var/lib/apt/lists/*; \
#     elif command -v apk >/dev/null 2>&1; then \
#       apk add --no-cache chromium nss freetype harfbuzz ttf-freefont; \
#     fi

# 切回官方默认的非 root 用户，保持安全模型
USER node