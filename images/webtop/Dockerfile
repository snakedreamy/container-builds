# ============================================================
# Webtop 开发镜像（Ubuntu + XFCE）定制版
#
# 目标：
# 1) 系统 Python（python3 + python-is-python3）+ uv（/usr/local/bin/uv）
# 2) 项目级 .venv（每个项目目录一个 .venv，可删可重建）
# 3) 缓存持久化到 /config（UV_CACHE_DIR / PIP_CACHE_DIR / npm cache）
# 4) Node.js 24.x（从官方 release 的 latest-v24.x 拉取并校验 sha256）
#
# 说明：
# - LinuxServer 的 webtop 镜像通常把 HOME 指到 /config（运行时用户多为 abc）
# - /config 一般会挂载为持久化卷，适合放缓存/项目
# ============================================================

FROM lscr.io/linuxserver/webtop:ubuntu-xfce

# 避免 apt 在构建时弹交互提示
ENV DEBIAN_FRONTEND=noninteractive

# ============================================================
# 1) 安装基础工具 & Python 相关（系统 Python）
#    - python3 / python3-venv / python3-pip / python-is-python3
#    - xz-utils：tar.xz 解压必备（Node 官方包为 .tar.xz）
#    - build-essential / cmake / gdb：编译/调试常用
# ============================================================
RUN apt-get update && apt-get install -y --no-install-recommends \
    sudo \
    git \
    curl \
    wget \
    vim \
    nano \
    htop \
    iproute2 \
    sqlite3 \
    libsqlite3-dev \
    build-essential \
    gdb \
    cmake \
    python3 \
    python3-venv \
    python3-pip \
    python-is-python3 \
    ca-certificates \
    tar \
    xz-utils \
  && rm -rf /var/lib/apt/lists/*

# ============================================================
# 2) 安装 Node.js 24.x（支持 amd64/arm64）
#    - 从 nodejs.org 的 release 目录下载（latest-v24.x）
#    - 拉取 SHASUMS256.txt 并校验，避免包被篡改
#
# 注意：
# - ARG 的值不会“自动触发重建”，只有你重新 build 时才会取到最新的 patch
# ============================================================
ARG NODE_DIST=latest-v24.x
RUN set -eux; \
    arch="$(dpkg --print-architecture)"; \
    case "${arch}" in \
      amd64) node_arch="x64" ;; \
      arm64) node_arch="arm64" ;; \
      *) echo "Unsupported arch: ${arch}" >&2; exit 1 ;; \
    esac; \
    base="https://nodejs.org/download/release/${NODE_DIST}"; \
    cd /tmp; \
    curl -fsSLO "${base}/SHASUMS256.txt"; \
    file="$(awk "/node-v[0-9]+\\.[0-9]+\\.[0-9]+-linux-${node_arch}\\.tar\\.xz\$/ {print \$2; exit}" SHASUMS256.txt)"; \
    test -n "${file}"; \
    curl -fsSLO "${base}/${file}"; \
    grep " ${file}\$" SHASUMS256.txt | sha256sum -c -; \
    tar -xJf "${file}" -C /usr/local --strip-components=1; \
    rm -f "/tmp/${file}" /tmp/SHASUMS256.txt; \
    node --version; \
    npm --version

# ============================================================
# 3) 安装 uv（强烈推荐容器用法：直接安装到 /usr/local/bin）
#
# 你之前报错的核心原因：
# - uv 安装脚本会默认装到 $HOME/.local/bin
# - webtop 中 HOME 常为 /config，所以实际落点是 /config/.local/bin
# - 但你写死去 /root/.local/bin 复制，导致找不到
#
# 这里使用 UV_UNMANAGED_INSTALL，直接装到 /usr/local/bin：
# - 路径固定、不依赖 HOME
# - 更适合 Docker 构建（不会修改 shell rc）
# ============================================================
RUN set -eux; \
    curl -LsSf https://astral.sh/uv/install.sh | env UV_UNMANAGED_INSTALL="/usr/local/bin" sh; \
    uv --version; \
    if command -v uvx >/dev/null 2>&1; then uvx --version; fi

# ============================================================
# 4) 缓存与全局工具的“持久化友好”默认路径（写到 /config）
#    - uv/pip 缓存：加速依赖下载与构建
#    - npm 全局安装：避免写 /usr/local 造成权限问题（尤其运行时是 abc 用户）
#
# 说明：
# - 你的“项目级 .venv”是可再生产物，不建议跨系统/跨小版本长期迁移
#   换系统/换 Python 小版本时更推荐删 .venv 后用 uv.lock 直接重建
# ============================================================
ENV UV_CACHE_DIR=/config/.cache/uv \
    PIP_CACHE_DIR=/config/.cache/pip \
    NPM_CONFIG_PREFIX=/config/.npm-global \
    NPM_CONFIG_CACHE=/config/.npm

# 让 npm 全局 bin 在任何执行环境都可用（不依赖 .bashrc）
ENV PATH="/config/.npm-global/bin:${PATH}"

# ============================================================
# 5) 预创建持久化目录 & 处理权限（尽量给 abc 用户）
#    - /config/projects：建议作为你的项目根目录（webtop 常把 /config 当 HOME）
# ============================================================
RUN set -eux; \
    mkdir -p \
      /config/projects \
      /config/.cache/uv \
      /config/.cache/pip \
      /config/.npm-global \
      /config/.npm; \
    if id abc >/dev/null 2>&1; then \
      chown -R abc:abc /config/projects /config/.cache /config/.npm-global /config/.npm || true; \
    fi

# ============================================================
# 6) 使用方式（写在注释里，方便你以后复制）
#
# 推荐：FastAPI 新项目（全部用 uv + 项目级 .venv）
#
#   mkdir -p ~/projects/fastapi-demo
#   cd ~/projects/fastapi-demo
#
#   uv init
#   uv add fastapi "uvicorn[standard]"
#
#   mkdir -p app
#   cat > app/main.py <<'PY'
#   from fastapi import FastAPI
#
#   app = FastAPI()
#
#   @app.get("/")
#   def read_root():
#       return {"hello": "world"}
#
#   @app.get("/health")
#   def health():
#       return {"status": "ok"}
#   PY
#
#   uv sync
#   uv run uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
#
# 验证：
#   curl -s http://127.0.0.1:8000/ | python -m json.tool
#   curl -s http://127.0.0.1:8000/health | python -m json.tool
# ============================================================