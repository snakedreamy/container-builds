# syntax=docker/dockerfile:1

ARG RUNNERS_VERSION=latest
# 使用 GHCR 上的官方 runners 镜像
FROM ghcr.io/n8n-io/runners:${RUNNERS_VERSION}

USER root

# --- JavaScript Runner 扩展 ---
# 安装常用库，方便 Function 节点调用
RUN cd /opt/runners/task-runner-javascript && \
    pnpm add moment uuid lodash

# --- Python Runner 扩展 ---
# 安装数据处理常用库
RUN cd /opt/runners/task-runner-python && \
    uv pip install numpy pandas requests

# --- 安全配置 ---
# ✅ 只覆盖 /etc/n8n-task-runners.json（按官方方式）
COPY n8n-task-runners.json /etc/n8n-task-runners.json
RUN chmod 0644 /etc/n8n-task-runners.json

USER runner