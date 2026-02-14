# n8n-runner 自定义镜像

基于官方 `n8nio/runners` 扩展，额外打包常用 JS/Python 库，并通过 allowlist 允许 Code 节点使用这些库。

- JS：moment、uuid
- Python：numpy、pandas

提示：即使镜像里安装了包，如果不在 allowlist 中，Code 节点仍然不能使用。