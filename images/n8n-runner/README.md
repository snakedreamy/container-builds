# n8n-runner 自定义镜像

基于官方 `n8nio/runners` 扩展：

- JS：moment、uuid
- Python：numpy、pandas
- 同时配置 allowlist，保证 Code 节点可以使用这些包

构建由 GitHub Actions 自动追随上游 `n8nio/runners:latest`（稳定最新版）。