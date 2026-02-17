# Containerfile
FROM docker.io/ubuntu:24.04

# 避免交互式安装时的卡顿
ENV DEBIAN_FRONTEND=noninteractive

# 1. 安装基础包 (保留原来的)
RUN apt-get update && apt-get install -y \
    openssh-server \
    sudo \
    curl \
    git \
    vim \
    nano \
    locales \
    udev \
    iputils-ping \
    net-tools \
    && rm -rf /var/lib/apt/lists/*

# 2. 配置 SSH 服务
RUN mkdir /run/sshd

# 【关键修改】允许 Root 登录
# 修改 sshd_config 允许 root 使用密码或密钥登录
RUN sed -i 's/#PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
# 允许密码登录 (调试用，建议后期关掉)
RUN sed -i 's/#PasswordAuthentication.*/PasswordAuthentication yes/' /etc/ssh/sshd_config

# 【关键修改】修复 PAM 限制
# 这一步非常重要！默认 PAM 配置可能会阻止 Root 在容器内登录，或者导致会话立即退出
RUN sed -i 's@session\s*required\s*pam_loginuid.so@session optional pam_loginuid.so@g' /etc/pam.d/sshd

# 3. 设置 Root 密码
# 设置一个默认密码，方便首次登录
RUN echo 'root:rockchip' | chpasswd

# 4. 设置 Locale (解决终端乱码)
RUN locale-gen en_US.UTF-8
ENV LANG en_US.UTF-8
ENV LANGUAGE en_US:en
ENV LC_ALL en_US.UTF-8

# 5. 配置 SSH 密钥目录
# 为 Root 准备 .ssh 目录
RUN mkdir -p /root/.ssh && chmod 700 /root/.ssh
# 【核心修复】将 root 加入 nogroup (GID 65534)
# 这样 SSH 登录时，PAM 读取 /etc/group 就会保留这个组权限
RUN usermod -aG nogroup root
# 也可以顺手把 video/render 加进去，虽然在 rootless 映射下 nogroup 才是起作用的那个
RUN usermod -aG video,render root

# 6. 启动脚本
EXPOSE 22
CMD ["/usr/sbin/sshd", "-D"]