# syntax=docker/dockerfile:1

ARG RUNNERS_VERSION=latest
# 使用 GHCR 上的官方 runners 镜像
FROM ghcr.io/n8n-io/runners:${RUNNERS_VERSION}

USER root

# --- JavaScript Runner 扩展 ---
# 安装常用库，方便 Function 节点调用
RUN cd /opt/runners/task-runner-javascript && \
    pnpm add moment uuid lodash chrome-remote-interface

# --- Python Runner 扩展 ---
# 安装数据处理常用库
RUN cd /opt/runners/task-runner-python && \
    uv pip install numpy pandas requests

# 不建议在这里覆盖 allowlist 配置，应该下载官方的配置后修改参数，再通过持久化文件覆盖

USER runner