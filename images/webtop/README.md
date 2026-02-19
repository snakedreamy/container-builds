# webtop（ubuntu-xfce）定制镜像

本目录用于构建一个“薄封装（thin wrapper）”的 Webtop 镜像：  
**上游镜像为 LinuxServer.io 的 webtop:ubuntu-xfce**，我们只做少量定制（例如安装一些 apt 软件、可选安装 Node.js），尽量不触碰上游的初始化与服务管理逻辑。

> 目标：在不破坏上游镜像行为（s6 init、/config 持久化、端口、权限模型、默认用户等）的前提下，为本地/内网桌面开发环境提供“顺手的工具链”。

---

## 为什么选择 ubuntu-xfce

webtop 有多个变体（不同桌面、不同底层发行版）。你选择 `ubuntu-xfce` 的核心原因是：

- 需要安装额外系统软件时，Ubuntu 的 `apt-get install` 最顺手；
- 避免 Alpine `apk` 带来的包名/依赖差异；
- 上游镜像已经是可用成品，我们只要“加工具”，不改它“怎么跑”。

---

## 定制原则（非常重要）

### ✅ 我们会做的事
- 在 Dockerfile 里 **额外安装** 一些 apt 软件（git、curl、调试工具、字体等）。
- 可选：在 Dockerfile 里安装 **Node.js（24 LTS）**，用于在 webtop 里跑 Node 工具/脚本。

### ❌ 我们尽量不做的事（除非你明确需要）
- 不改上游 entrypoint / init（上游通常使用 s6-overlay 体系）。
- 不改上游的目录约定（例如 `/config` 持久化目录）。
- 不改上游端口、服务启动逻辑、权限模型。
- 不往上游复杂逻辑里“塞自定义脚本”，除非确实有必须初始化的东西。

> 简单说：**webtop 是“成品”，我们只做“加装工具”的派生镜像**。

---

## 目录结构建议

```text
images/webtop/
├─ Dockerfile
└─ README.md