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
    XLS_VERSION=$(curl -s https://github.com | grep -oP '"tag_name": "\K[^"]+') && \
    curl -L https://github.com{XLS_VERSION}/xls-${XLS_VERSION}-linux-x86_64.tar.gz | tar -xzC /opt/xls --strip-components=1

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
    && apt-get clean && rm -rf /var/lib/apt/lists/*

# Copy prebuilt Google XLS Package from fetcher stage
COPY --from=fetcher /opt/xls /opt/xls

# Build Verilator from source
RUN git clone https://github.com /tmp/verilator && \
    cd /tmp/verilator && \
    git checkout v5.028 && \
    autoconf && \
    ./configure --prefix=/usr/local && \
    make -j$(nproc) && \
    make install && \
    rm -rf /tmp/verilator

ENV PATH="/opt/xls/bin:${PATH}"
WORKDIR /workspace

