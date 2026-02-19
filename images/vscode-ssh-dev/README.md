# vscode-ssh-dev

一个用于 **VS Code Remote-SSH** 的开发服务器容器镜像：基于 **LinuxServer.io baseimage-ubuntu（noble）**，内置 OpenSSH（端口 `2222`），并预装常用开发工具链（Python/C/C++、Node.js LTS、uv 等）。镜像设计目标是：

- **不依赖 `.bashrc`**：即使是“非交互式执行”（例如扩展直接 exec 命令）也能稳定找到 `python/node/npm/uv`。
- **可控更新策略**：镜像重建只在你 **手动改动文件** 或 **上游底座镜像更新** 时发生；Node 的更新发生在“重建时”，而不是“Node 发布新版本就自动触发重建”。
- **数据持久化**：把需要持久化的内容放进 `/config`（volume），例如 SSH 主机密钥、npm 全局工具、缓存等。

---

## 目录结构

```text
images/vscode-ssh-dev/
├─ Dockerfile
├─ manifests/
│  └─ node-global-tools.txt
└─ root/
   ├─ etc/
   │  ├─ cont-init.d/
   │  │  ├─ 40-configure-ssh
   │  │  └─ 50-bootstrap-node-tools
   │  └─ services.d/
   │     └─ openssh/
   │        └─ run
   └─ ...
````

> 说明：`root/` 会被复制到镜像根目录，用于 s6-overlay（LSIO 风格）的初始化和服务管理。

---

## 核心设计

### 1) SSH 服务（端口 2222）

* 容器内运行 OpenSSH server，监听 `2222`。
* 默认更安全：建议使用 **公钥登录**；密码登录可通过环境变量开关控制。
* `40-configure-ssh` 会创建/更新 `/etc/ssh/sshd_config.d/99-vscode-ssh-dev.conf`，避免对主配置反复 `sed`。

#### Host keys 持久化

* Host keys 存在 `/config/ssh_host_keys`（持久化）。
* 启动时会生成/复用，然后软链接到 `/etc/ssh/ssh_host_*` 位置供 sshd 使用。

---

### 2) Python：系统 Python + uv + 项目级 `.venv`

镜像内安装：

* `python3`
* `python3-venv`
* `python3-pip`
* `python-is-python3`（确保 `python` 指向 `python3`）
* `uv`（安装到 `/usr/local/bin/uv`）

推荐使用方式：

* **每个项目目录放一个 `.venv/`**
* 用 `uv` 管理依赖（`pyproject.toml + uv.lock`），从而在系统升级/换底座后可以 **删 `.venv` 直接重建**。

缓存持久化：

* `UV_CACHE_DIR=/config/.cache/uv`
* `PIP_CACHE_DIR=/config/.cache/pip`

> 注意：`.venv` 作为可再生产物，不建议做“跨系统 Python 小版本永久迁移”。换系统/换 Python 小版本时更推荐删除并用 lock 重建。

---

我们就按 **FastAPI 新项目**来走一遍（从 0 到能跑起来），全部用 `uv`，并且用“项目级 `.venv`”。


#### 创建项目目录并初始化

```bash
mkdir -p ~/projects/fastapi-demo
cd ~/projects/fastapi-demo

uv init
```

执行完会生成 `pyproject.toml`（uv 识别项目的关键）。

---

#### 添加依赖（FastAPI + Uvicorn）

```bash
uv add fastapi "uvicorn[standard]"
```

这一步会：

* 写入 `pyproject.toml` 的 dependencies
* 生成/更新 `uv.lock`（锁定具体版本）

---

#### 写最小 FastAPI 应用

创建 `app/main.py`：

```bash
mkdir -p app
cat > app/main.py <<'PY'
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def read_root():
    return {"hello": "world"}

@app.get("/health")
def health():
    return {"status": "ok"}
PY
```

---

#### 同步环境（创建 `.venv` 并安装依赖）

```bash
uv sync
```

现在项目里会出现 `.venv/`。

---

#### 运行服务（推荐用 uv run）

```bash
uv run uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
```

然后在另一个终端里验证：

```bash
curl -s http://127.0.0.1:8000/ | python -m json.tool
curl -s http://127.0.0.1:8000/health | python -m json.tool
```

---

####（可选但强烈建议）加开发依赖：ruff + pytest

ruff 可以同时当 linter+formatter（非常省事）：

```bash
uv add --dev ruff pytest httpx
```

* `pytest`：测试框架
* `httpx`：测试 FastAPI 常用（TestClient/请求）

跑格式化/检查：

```bash
uv run ruff format .
uv run ruff check .
```

---

#### 写一个最小测试（可选）

```bash
cat > test_app.py <<'PY'
from fastapi.testclient import TestClient
from app.main import app

client = TestClient(app)

def test_root():
    r = client.get("/")
    assert r.status_code == 200
    assert r.json() == {"hello": "world"}
PY
```

运行测试：

```bash
uv run pytest -q
```

---

#### 升级/换底座后怎么处理？

如果未来 Python 小版本变了、或你重建容器导致 `.venv` 不可用：

```bash
rm -rf .venv
uv sync
```

因为依赖被 `uv.lock` 锁住了，重建会装出同一套环境。

---

#### 项目结构（最终）

```text
fastapi-demo/
├─ app/
│  └─ main.py
├─ pyproject.toml
├─ uv.lock
├─ .venv/
└─ test_app.py   (可选)
```


---

### 3) Node.js：官方 Node 24 LTS（构建时追 latest-v24.x）

镜像在构建时从 Node 官方的 `latest-v24.x` 目录获取当前最新 Node 24.x tarball，并校验 `SHASUMS256.txt` 后安装。

* **不会**因为 Node 发布新 patch 就自动触发 CI 重建
* **会**在你触发重建时（手动修改文件/底座镜像更新）自动安装“当时最新的 Node 24.x”

相关 Dockerfile 关键点：

* `ARG NODE_DIST=latest-v24.x`
* 从 `https://nodejs.org/download/release/${NODE_DIST}/SHASUMS256.txt` 解析对应架构 tarball，然后安装

---

### 4) npm 全局工具持久化 + 自动更新（Codex 追新功能）

npm 全局前缀固定在 `/config/.npm-global`，并将其加入 `PATH`：

* `NPM_CONFIG_PREFIX=/config/.npm-global`
* `PATH=/config/.npm-global/bin:...`

因此：

* `npm i -g ...` 安装的全局工具会持久化在 `/config`，重建镜像不会丢。
* `manifests/node-global-tools.txt` 定义了“全局工具基线”（source of truth）。

#### 工具清单（node-global-tools.txt）

* 每行一个包名
* 可混合：锁版本 / 不锁版本

  * 锁版本：`name@x.y.z` 或 `@scope/name@x.y.z`
  * 不锁版本：`name` 或 `@scope/name`

当前默认只包含 Codex CLI（不锁版本）：

* `@openai/codex`

> 注：npm 包名是 `@openai/codex`，安装后命令是 `codex`。

#### Bootstrap 行为（50-bootstrap-node-tools）

启动时会执行一段幂等逻辑：

**强制同步（install/sync）触发条件：**

* 第一次启动（没有 state）
* 清单文件变更（hash 改变）
* Node major 变化（例如未来从 24 升到 26）

同步动作：

* 锁版本项：确保安装到指定版本（不对则重装/降级）
* 不锁版本项：确保已安装（缺了则安装）

**自动更新（update）策略：**

* 默认 `AUTO_UPDATE=true`
* 默认节流周期 `NODE_TOOLS_UPDATE_INTERVAL_DAYS=7`
* 只更新“不锁版本项”（例如 Codex），执行 `@latest`

**状态文件：**

* `/config/.bootstrap/state.json` 记录上次成功 sync/update 时间等信息，用于节流与变更检测。

**失败策略：**

* npm 安装/更新失败不会阻塞容器启动（网络抖动时仍保证 SSH 可用）

---

## 运行参数与环境变量

### 端口

* SSH：`2222/tcp`

### SSH 相关（由 40-configure-ssh 处理）

* `USER_NAME`：默认 `abc`
* `PASSWORD_ACCESS`：`true/false`（默认 `false`）
* `USER_PASSWORD` / `USER_PASSWORD_FILE`：设置用户密码（仅在 `PASSWORD_ACCESS=true` 时有意义）
* `SUDO_ACCESS`：`true/false`（默认 `false`）
* 公钥注入（支持多种来源）：

  * `PUBLIC_KEY`
  * `PUBLIC_KEY_FILE`
  * `PUBLIC_KEY_DIR`
  * `PUBLIC_KEY_URL`（例如 GitHub `https://github.com/<user>.keys`）

### Node 工具 bootstrap

* `AUTO_UPDATE`：默认 `true`
* `NODE_TOOLS_UPDATE_INTERVAL_DAYS`：默认 `7`
* `NPM_CONFIG_PREFIX`：默认 `/config/.npm-global`
* `NPM_CONFIG_CACHE`：默认 `/config/.npm`

### Python/uv 缓存

* `UV_CACHE_DIR`：默认 `/config/.cache/uv`
* `PIP_CACHE_DIR`：默认 `/config/.cache/pip`

---

## 推荐的 docker run 示例

> 请确保把 `/config` 挂载为 volume，以持久化 host keys、npm 全局工具和缓存。

```bash
docker run -d \
  --name vscode-ssh-dev \
  -p 2222:2222 \
  -v vscode-ssh-dev-config:/config \
  -e PUBLIC_KEY_URL="https://github.com/<your_github_username>.keys" \
  -e SUDO_ACCESS=true \
  ghcr.io/<owner>/vscode-ssh-dev:latest
```

VS Code 侧连接示例（~/.ssh/config）：

```sshconfig
Host my-dev
  HostName <server-ip>
  Port 2222
  User abc
  IdentityFile ~/.ssh/id_ed25519
```

---

## 更新策略总结（重要）

* **镜像重建触发：**

  * 你修改 `images/vscode-ssh-dev/**`（如 Dockerfile、脚本、清单）
  * 或上游底座镜像（`lscr.io/linuxserver/baseimage-ubuntu:noble`）更新导致 CI 触发重建

* **Node 更新：**

  * 不会因为 Node 发布新版本自动触发 CI
  * 但每次你触发重建时会自动安装“当时最新的 Node 24.x（latest-v24.x）”

* **Codex 更新：**

  * 由运行时 bootstrap 控制（不锁版本项）
  * 默认每 7 天自动更新一次（可通过环境变量关闭或修改周期）

---

## 常见问题（FAQ）

### Q: 为什么不用 `.bashrc` 自动激活 conda/nvm？

A: 某些工具/扩展会直接 exec 命令，不启动交互式 shell，因此 `.bashrc` 不一定会被加载。本镜像通过 **容器级 ENV + PATH** 保证 `python/node/npm/uv` 在任何执行场景下可用。

### Q: `/config/.npm-global` 会不会因为 Node 版本变化而坏掉？

A: 通常纯 JS 工具没问题；极少数带 native addon 的工具跨 Node major 可能需要重装。此镜像在 Node major 变化时会触发强制同步（你升级 major 时也应视为迁移事件）。

### Q: 项目 Python 依赖如何长期稳定？

A: 推荐 `uv` + `uv.lock`。换底座/换 Python 版本时，删 `.venv` 并用 lock 重建即可；缓存放在 `/config/.cache/uv` 和 `/config/.cache/pip` 加速重建。

---

## 维护指南（你未来怎么改）

* 想增加/调整全局 Node 工具：

  * 编辑 `manifests/node-global-tools.txt`
  * 重启容器或等待下次启动 bootstrap 自动应用
* 想更新 Node major（例如未来升级到 26 LTS）：

  * 修改 Dockerfile 中 `NODE_DIST=latest-v24.x` → `latest-v26.x`
  * 触发重建并发布镜像
* 想关闭 Codex 自动更新：

  * 运行容器时设置：`-e AUTO_UPDATE=false`
  * 或把 `NODE_TOOLS_UPDATE_INTERVAL_DAYS` 调大


