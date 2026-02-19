FROM ghcr.io/linuxserver/webtop:ubuntu-xfce

ENV DEBIAN_FRONTEND=noninteractive

# 1) 只加你需要的 apt 软件（尽量少动）
RUN apt-get update && apt-get install -y --no-install-recommends \
    git curl wget vim nano \
    # 你需要的其它包...
    && rm -rf /var/lib/apt/lists/*

# 2) 可选：安装 Node 24 LTS（构建时追 latest-v24.x）
#    （这不会改变 webtop 的 init/服务，只是多了 node/npm）
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
    node --version; npm --version