# n8n 自定义镜像（用于支持社区节点安装）

这个镜像基于官方 `docker.n8n.io/n8nio/n8n`，只做最小增强：

- 安装 `git`（解决部分社区节点依赖 `github:` 拉取时没有 git 的问题）
- 不修改官方 entrypoint / CMD，保持官方启动逻辑不变

证书信任（自签 CA）：不要写进镜像，按官方方式挂载到 `/opt/custom-certificates`。