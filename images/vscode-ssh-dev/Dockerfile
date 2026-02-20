FROM lscr.io/linuxserver/baseimage-ubuntu:noble

LABEL org.opencontainers.image.title="vscode-ssh-dev"
LABEL org.opencontainers.image.description="Ubuntu (LSIO baseimage) + OpenSSH for VS Code Remote-SSH dev server"
LABEL org.opencontainers.image.licenses="MIT"

ENV DEBIAN_FRONTEND=noninteractive

# Build-time knobs
ARG NODE_DIST=latest-v24.x

RUN apt-get update && apt-get install -y --no-install-recommends \
    openssh-server \
    openssh-client \
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

# Install Node.js 24.x (LTS) from official "latest-v24.x" directory at build time.
# This will NOT trigger rebuilds by itself; it only updates when you rebuild for other reasons.
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

# Install uv (latest) without modifying shell rc files; place into /usr/local/bin
RUN set -eux; \
    curl -LsSf https://astral.sh/uv/install.sh | env UV_NO_MODIFY_PATH=1 sh; \
    install -m 0755 /root/.local/bin/uv /usr/local/bin/uv; \
    if [ -f /root/.local/bin/uvx ]; then install -m 0755 /root/.local/bin/uvx /usr/local/bin/uvx; fi; \
    rm -f /root/.local/bin/uv /root/.local/bin/uvx; \
    uv --version

# Persist-friendly defaults (no .bashrc required)
ENV NPM_CONFIG_PREFIX=/config/.npm-global \
    NPM_CONFIG_CACHE=/config/.npm \
    UV_CACHE_DIR=/config/.cache/uv \
    PIP_CACHE_DIR=/config/.cache/pip \
    AUTO_UPDATE=true \
    NODE_TOOLS_UPDATE_INTERVAL_DAYS=7

ENV PATH="/config/.npm-global/bin:/config/.local/bin:${PATH}"

# Assign abc user's home to /config (LSIO baseimage uses abc by default)
RUN usermod -d /config abc && \
    usermod -s /bin/bash abc

RUN mkdir -p /var/run/sshd && chmod 0755 /var/run/sshd

# Your manifests live here (e.g., manifests/node-global-tools.txt)
COPY manifests /opt/vscode-ssh-dev/manifests

# Your existing root/ overlay (cont-init, services, etc.)
COPY root /

RUN chmod +x /etc/cont-init.d/40-configure-ssh && \
    chmod +x /etc/cont-init.d/50-bootstrap-node-tools && \
    chmod +x /etc/services.d/openssh/run

EXPOSE 2222