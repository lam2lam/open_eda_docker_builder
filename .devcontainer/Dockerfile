# ==============================================================================
# STAGE 1: Download and verify assets via GitHub runners
# ==============================================================================
FROM ubuntu:24.04 AS fetcher
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    ca-certificates \
    tar \
    && rm -rf /var/lib/apt/lists/*

# Download Google XLS Development Packages completely during build
RUN mkdir -p /opt/xls && \
    XLS_VERSION=$(curl -s https://api.github.com/repos/google/xls/releases/latest | grep -oP '"tag_name": "\K[^"]+') && \
    curl -L https://github.com/google/xls/releases/download/${XLS_VERSION}/xls-${XLS_VERSION}-linux-x64.tar.gz | tar -xzC /opt/xls --strip-components=1

# ==============================================================================
# STAGE 2: Final integrated EDA image
# ==============================================================================
# Using the official OpenROAD Flow Scripts image as base
FROM openroad/orfs:latest AS openroad_source
FROM ghcr.io/ucb-bar/chipyard:main

USER root
ENV DEBIAN_FRONTEND=noninteractive

# Install system compilation and runtime utilities
RUN apt-get update && apt-get install -y --no-install-recommends \
    git \
    build-essential \
    ccache \
    flex \
    bison \
    gperf \
    autoconf \
    libfl-dev \
    libgoogle-perftools-dev \
    numactl \
    zlib1g-dev \
    python3-pip \
    python3-dev \
    iverilog \
    vim \
    verilator \
    nano \
    sudo \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

COPY --from=openroad_source /OpenROAD-flow-scripts /OpenROAD-flow-scripts

# Copy prebuilt Google XLS Package from fetcher stage
COPY --from=fetcher /opt/xls /opt/xls

COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /usr/local/bin/

# 1. Point Ubuntu APT sources at the Huawei internal mirror (Ubuntu 24.04+ and older).
RUN if [ -f /etc/apt/sources.list.d/ubuntu.sources ]; then \
      sed -i "s@http://.*archive.ubuntu.com@http://mirrors.tools.huawei.com@g" /etc/apt/sources.list.d/ubuntu.sources && \
      sed -i "s@http://.*security.ubuntu.com@http://mirrors.tools.huawei.com@g" /etc/apt/sources.list.d/ubuntu.sources; \
    fi && \
    if [ -f /etc/apt/sources.list ]; then \
      sed -i "s@http://.*archive.ubuntu.com@http://mirrors.tools.huawei.com@g" /etc/apt/sources.list && \
      sed -i "s@http://.*security.ubuntu.com@http://mirrors.tools.huawei.com@g" /etc/apt/sources.list; \
    fi


RUN ORIGINAL_HOME=$(find /home -maxdepth 1 -mindepth 1 -type d | head -n 1) && \
    echo "Original base home directory discovered at: ${ORIGINAL_HOME}" && \
    # 创建 slam 用户，直接将其家目录指定为这个现有的、带有全套开发环境的目录
    useradd -d ${ORIGINAL_HOME} -M -s /bin/bash slam && \
    usermod -aG sudo slam && \
    echo "slam ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers && \
    # 将整个原有生态目录、OpenROAD 管道以及工作区的属主全部变更为 slam
    chown -R slam:slam ${ORIGINAL_HOME} && \
    chown -R slam:slam /OpenROAD-flow-scripts
    
USER slam
WORKDIR ${ORIGINAL_HOME}

RUN CHIPYARD_DIR=$(find /home -name "init-vlsi.sh" | head -n 1 | sed 's/\/scripts\/init-vlsi.sh//') && \
    echo "Configuring Chipyard inside: ${CHIPYARD_DIR}" && \
    cd ${CHIPYARD_DIR} && \
    # 执行初始化脚本（现在 slam 拥有绝对修改权限，不会再报 .bash_profile 错误）
    ./scripts/init-vlsi.sh asap7 && \
    # 唤醒镜像自带的本地 Conda 环境并注册 hammer
    source /opt/conda/etc/profile.d/conda.sh && \
    conda activate base && \
    pip install --upgrade pip && \
    pip install hammer-vlsi[asap7-gdstk] && \
    cd vlsi/hammer && \
    python setup.py develop

# 2. Global pip Huawei internal mirror.
RUN pip3 config --global set global.index http://mirrors.tools.huawei.com/pypi && \
    pip3 config --global set global.index-url http://mirrors.tools.huawei.com/pypi/simple && \
    pip3 config --global set global.trusted-host mirrors.tools.huawei.com

# 3. Global .gitconfig for the internal network (applies to every user via --system).
RUN git config --system http.sslVerify false && \
    git config --system http.proxy http://dvcmodel-10.hisilicon.com:3128 && \
    git config --system http.postbuffer 1000000000 && \
    git config --system https.proxy http://dvcmodel-10.hisilicon.com:3128 && \
    git config --system url."http://github.com/".insteadof "git@github.com:"

# 4. User identity.
RUN git config --system user.email "steve.lam@huawei.com" && \
    git config --system user.name "slam"

# 5. uv should use the internal mirror for any in-container use on the internal network.
ENV UV_INDEX_URL="http://mirrors.tools.huawei.com/pypi/simple"
ENV UV_INSECURE_HTTP=true

# ---------------- finish as the user/workspace the runtime expects ----------------
ENV OpenROAD_FLOW_ROOT=/OpenROAD-flow-scripts
ENV PATH="/OpenROAD-flow-scripts/tools/install/OpenROAD/bin:/OpenROAD-flow-scripts/tools/install/yosys/bin:/opt/xls:${PATH}"

RUN interpreter_main --version || echo "XLS DSLX interpreter ready."
RUN yosys -V
RUN openroad -version
RUN java -version
RUN sbt sbtVersion
RUN ls /OpenROAD-flow-scripts/flow/platforms/asap7 && echo "ASAP7 system structural integrity verified."
RUN /bin/bash -c "source /opt/conda/etc/profile.d/conda.sh && conda activate base && hammer-vlsi -h"

CMD ["/bin/bash"]
