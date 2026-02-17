FROM lscr.io/linuxserver/baseimage-ubuntu:noble

LABEL org.opencontainers.image.title="vscode-ssh-dev"
LABEL org.opencontainers.image.description="Ubuntu (LSIO baseimage) + OpenSSH for VS Code Remote-SSH dev server"
LABEL org.opencontainers.image.licenses="MIT"

ENV DEBIAN_FRONTEND=noninteractive

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
    sqlite3 \
    libsqlite3-dev \
    build-essential \
    gdb \
    cmake \
    python3 \
    python3-pip \
    python3-venv \
    ca-certificates \
    tar \
    && rm -rf /var/lib/apt/lists/*

# 将 abc 用户家目录指派给 /config
RUN usermod -d /config abc && \
    usermod -s /bin/bash abc

RUN mkdir -p /var/run/sshd && chmod 0755 /var/run/sshd

COPY root /

RUN chmod +x /etc/cont-init.d/40-configure-ssh && \
    chmod +x /etc/services.d/openssh/run

EXPOSE 2222