# syntax=docker/dockerfile:1

# 由 workflow 传入 runners 版本；默认 latest（避免 lint 警告）
ARG RUNNERS_VERSION=latest
FROM n8nio/runners:${RUNNERS_VERSION}

USER root

# ---------------------------
# JS runner：安装常用 JS 包（按需增删）
# ---------------------------
RUN cd /opt/runners/task-runner-javascript && pnpm add moment uuid

# ---------------------------
# Python runner：安装常用 Python 包（按需增删）
# ---------------------------
RUN cd /opt/runners/task-runner-python && uv pip install numpy pandas

# ---------------------------
# allowlist 配置：非常重要
# 没写进 allowlist 的包，即使安装进镜像，Code 节点也不能用
# 这里两个路径都放一份，兼容不同读取路径实现
# ---------------------------
COPY n8n-task-runners.json /etc/n8n-task-runners.json
COPY n8n-task-runners.json /etc/task-runners.json

USER runner