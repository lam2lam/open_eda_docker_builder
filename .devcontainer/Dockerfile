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
FROM openroad/orfs:latest

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
    && apt-get clean && rm -rf /var/lib/apt/lists/*

# Copy prebuilt Google XLS Package from fetcher stage
COPY --from=fetcher /opt/xls /opt/xls

COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /usr/local/bin/

# Build Verilator from source
RUN git clone https://github.com/verilator/verilator /tmp/verilator && \
    cd /tmp/verilator && \
    git checkout v5.028 && \
    autoconf && \
    ./configure --prefix=/usr/local && \
    make -j$(nproc) && \
    make install && \
    rm -rf /tmp/verilator

# 1. Point Ubuntu APT sources at the Huawei internal mirror (Ubuntu 24.04+ and older).
RUN if [ -f /etc/apt/sources.list.d/ubuntu.sources ]; then \
      sed -i "s@http://.*archive.ubuntu.com@http://mirrors.tools.huawei.com@g" /etc/apt/sources.list.d/ubuntu.sources && \
      sed -i "s@http://.*security.ubuntu.com@http://mirrors.tools.huawei.com@g" /etc/apt/sources.list.d/ubuntu.sources; \
    fi && \
    if [ -f /etc/apt/sources.list ]; then \
      sed -i "s@http://.*archive.ubuntu.com@http://mirrors.tools.huawei.com@g" /etc/apt/sources.list && \
      sed -i "s@http://.*security.ubuntu.com@http://mirrors.tools.huawei.com@g" /etc/apt/sources.list; \
    fi

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
ENV PATH="/opt/xls/bin:${PATH}"
WORKDIR /workspace

