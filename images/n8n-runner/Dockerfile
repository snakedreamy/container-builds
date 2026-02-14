# syntax=docker/dockerfile:1

# ---------------------------
# 由 workflow 传入 runners 版本
# ---------------------------
ARG RUNNERS_VERSION=latest
FROM n8nio/runners:${RUNNERS_VERSION}

# ---------------------------
# runner 镜像默认比较精简
# 我们需要 root 来安装 JS/Python 的第三方包
# ---------------------------
USER root

# ---------------------------
# JS runner：在 task-runner-javascript 目录安装常用包
# 这里演示 moment、uuid，你可以换成自己的常用库清单
# ---------------------------
RUN cd /opt/runners/task-runner-javascript && pnpm add moment uuid

# ---------------------------
# Python runner：安装 numpy、pandas
# 官方 runners 使用 uv（比 pip 快/可复现性也更好）
# ---------------------------
RUN cd /opt/runners/task-runner-python && uv pip install numpy pandas

# ---------------------------
# allowlist 配置：非常重要！
# 不允许的包即使安装进镜像，也不能在 Code 节点里 import/require 使用
#
# 文档里常见路径：/etc/n8n-task-runners.json
# 有些运行器实现也会读 /etc/task-runners.json
# 为了兼容，两个路径都放一份（以后上游改了也不怕）
# ---------------------------
COPY n8n-task-runners.json /etc/n8n-task-runners.json
COPY n8n-task-runners.json /etc/task-runners.json

# ---------------------------
# 切回 runner 用户，保持官方默认权限模型
# ---------------------------
USER runner