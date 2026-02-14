# n8n 自定义镜像

这个镜像基于官方 `docker.n8n.io/n8nio/n8n`，只做两件事：

1. 恢复 `apk`（避免官方镜像精简导致 `apk: not found`）
2. 安装 `git`（解决某些社区节点依赖 `github:` 拉取时没有 git 的问题）

注意：不修改官方 entrypoint / CMD，尽量保持与官方镜像一致。

证书信任：使用官方方式挂载 `/opt/custom-certificates`（不要把私密证书写进镜像）。