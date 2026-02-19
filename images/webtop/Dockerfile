FROM lscr.io/linuxserver/webtop:ubuntu-xfce

ENV DEBIAN_FRONTEND=noninteractive

# 先装解压/下载所需的基础工具（xz-utils 是 tar.xz 的关键）
RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates \
    curl \
    tar \
    xz-utils \
    && rm -rf /var/lib/apt/lists/*

# Install Node.js 24.x (LTS) from official "latest-v24.x" directory at build time.
# This will NOT trigger rebuilds by itself; it only updates when you rebuild for other reasons.
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

# npm 全局工具持久化到 /config（volume）
ENV NPM_CONFIG_PREFIX=/config/.npm-global \
    NPM_CONFIG_CACHE=/config/.npm

# 让全局 bin 在任何执行环境下可用（不依赖 .bashrc）
ENV PATH="/config/.npm-global/bin:${PATH}"

# （可选）你要额外装的 apt 软件就放在这里
# RUN apt-get update && apt-get install -y --no-install-recommends \
#     git vim nano \
#     && rm -rf /var/lib/apt/lists/*