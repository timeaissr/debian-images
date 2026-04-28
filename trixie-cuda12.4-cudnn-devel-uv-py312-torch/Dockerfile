# =============================================
# 算力服务器实例镜像 - Debian 13 + CUDA 12.4（Debian 官方源）
# uv 完全管理 Python 3.12，虚拟环境固定在 /opt/venv
# 适合临时训练实例（root 用户）
# =============================================

FROM debian:13

LABEL description="Debian 13 + CUDA 12.4 + uv + Python 3.12 + PyTorch"

# ==================== 环境变量 ====================
ENV DEBIAN_FRONTEND=noninteractive \
    TZ=Asia/Shanghai \
    LANG=zh_CN.UTF-8 \
    LANGUAGE=zh_CN:zh \
    LC_ALL=zh_CN.UTF-8 \
    PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    PATH="/opt/venv/bin:/usr/lib/cuda/bin:/root/.cargo/bin:/usr/local/bin:${PATH}" \
    LD_LIBRARY_PATH=/usr/lib/cuda/lib64:${LD_LIBRARY_PATH} \
    CUDA_HOME=/usr/lib/cuda \
    UV_PYTHON=3.12 \
    UV_PROJECT_ENVIRONMENT=/opt/venv \
    UV_PYTHON_PREFERENCE=only-managed

# ==================== 系统依赖 + 时区 + 语言 ====================
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl wget git vim nano htop tmux ca-certificates gnupg \
    openssh-server openssh-client \
    build-essential cmake ninja-build \
    tzdata locales \
    libssl-dev zlib1g-dev libbz2-dev libreadline-dev libsqlite3-dev \
    libncursesw5-dev xz-utils tk-dev libxml2-dev libffi-dev liblzma-dev \
    ffmpeg libsm6 libxext6 libxkbcommon-x11-0 libdbus-1-3 \
    && rm -rf /var/lib/apt/lists/*

# 配置语言和时区
RUN sed -i 's/# zh_CN.UTF-8 UTF-8/zh_CN.UTF-8 UTF-8/' /etc/locale.gen && \
    sed -i 's/# en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/' /etc/locale.gen && \
    locale-gen && \
    update-locale LANG=zh_CN.UTF-8 LC_ALL=zh_CN.UTF-8 && \
    ln -snf /usr/share/zoneinfo/${TZ} /etc/localtime && \
    echo ${TZ} > /etc/timezone && \
    dpkg-reconfigure -f noninteractive tzdata

# ==================== 安装 CUDA 12.4（Debian 官方源）====================
# trixie 里的 CUDA 相关包位于 non-free / contrib，cuDNN 对应 Debian 包名为 nvidia-cudnn
RUN sed -i 's/Components: main/Components: main contrib non-free non-free-firmware/' /etc/apt/sources.list.d/debian.sources && \
    apt-get update && \
    apt-get install -y --no-install-recommends \
        nvidia-cuda-toolkit \
        nvidia-cuda-dev \
        nvidia-cudnn \
    && rm -rf /var/lib/apt/lists/* && \
    ldconfig

# ==================== 安装 uv ====================
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /usr/local/bin/

# ==================== 创建虚拟环境 ====================
RUN mkdir -p /opt/venv && \
    uv python install 3.12 && \
    uv venv /opt/venv --python 3.12

# ==================== uv 安装依赖 ====================
# 使用 uv add 安装依赖（全部安装到 /opt/venv）
RUN uv add \
        torch torchvision torchaudio \
        --index-url https://download.pytorch.org/whl/cu124 \
    && uv add \
        transformers accelerate datasets peft deepspeed bitsandbytes \
        numpy pandas scipy scikit-learn matplotlib seaborn plotly \
        jupyterlab jupyter notebook tensorboard ipykernel ipython \
        requests tqdm pyyaml omegaconf hydra-core


# ==================== 配置 SSH ====================
RUN mkdir -p /var/run/sshd && \
    mkdir -p /root/.ssh && \
    ssh-keygen -A && \
    sed -ri 's/^PermitRootLogin\s+.*/PermitRootLogin yes/' /etc/ssh/sshd_config && \
    sed -i '/StrictHostKeyChecking/d' /etc/ssh/ssh_config && \
    echo "    StrictHostKeyChecking no" >> /etc/ssh/ssh_config

RUN cat <<'EOF' >> /root/.bashrc
echo "========================================"
echo "Debian 13 算力实例已启动"
echo "时区: $(cat /etc/timezone)    语言: $LANG"
echo "SSH 端口: 22"
echo "JupyterLab: http://$(hostname -I | awk '{print $1}'):8888"
echo "虚拟环境位置: /opt/venv"
echo "运行方式:   uv run python xxx.py"
echo "添加包:     uv add <package_name>"
echo "========================================"
EOF

EXPOSE 22 8888 6006 8000

WORKDIR /workspace

CMD ["/usr/sbin/sshd", "-D", "-e"]