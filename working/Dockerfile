FROM ubuntu:16.04 AS deps

ENV DEBIAN_FRONTEND=noninteractive

# Install dependencies
RUN apt-get update && apt-get install -y \
    software-properties-common \
    build-essential \
    git \
    cmake \
    ninja-build \
    curl \
    zlib1g-dev \
    libxml2-dev \
    zip \
    gnupg \
    libssl-dev \
    libbz2-dev \
    libreadline-dev \
    libsqlite3-dev \
    libffi-dev \
    libncurses5-dev \
    libncursesw5-dev \
    xz-utils \
    tk-dev \
    uuid-dev \
    wget && \
    rm -rf /var/lib/apt/lists/*

# Build Python 3.6.15 from source
WORKDIR /tmp
RUN wget https://www.python.org/ftp/python/3.6.15/Python-3.6.15.tgz && \
    tar -xf Python-3.6.15.tgz && \
    cd Python-3.6.15 && \
    ./configure --prefix=/usr/local/python3.6 && \
    make -j$(nproc) && make install && \
    ln -sf /usr/local/python3.6/bin/python3.6 /usr/local/bin/python3 && \
    ln -sf /usr/local/python3.6/bin/pip3 /usr/local/bin/pip3 && \
    cd .. && rm -rf Python-3.6.15*

# Install CMake 3.21.4
RUN curl -LO https://github.com/Kitware/CMake/releases/download/v3.21.4/cmake-3.21.4-linux-x86_64.sh && \
    chmod +x cmake-3.21.4-linux-x86_64.sh && \
    ./cmake-3.21.4-linux-x86_64.sh --skip-license --prefix=/usr/local && \
    rm cmake-3.21.4-linux-x86_64.sh

# ----------------------------------------------------------------

FROM deps AS stage1

ARG INSTALL_STAGE1=/opt/clang-stage1

WORKDIR /src
RUN git clone --depth=1 --branch llvmorg-15.0.7 https://github.com/llvm/llvm-project.git llvm-bootstrap

WORKDIR /build1

RUN apt-get update && apt-get install -y clang-8 lld-8 && rm -rf /var/lib/apt/lists/*

RUN cmake -G Ninja /src/llvm-bootstrap/llvm \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_C_COMPILER=clang-8 \
  -DCMAKE_CXX_COMPILER=clang++-8 \
  -DLLVM_TEMPORARILY_ALLOW_OLD_TOOLCHAIN=ON \
  -DLLVM_ENABLE_PROJECTS="clang;lld" \
  -DLLVM_TARGETS_TO_BUILD="X86" \
  -DCMAKE_INSTALL_PREFIX=${INSTALL_STAGE1}

RUN ninja install

# ----------------------------------------------------------------

FROM deps AS stage2

ARG INSTALL_STAGE2=/opt/clang-final

WORKDIR /src
RUN git clone --depth=1 --branch llvmorg-15.0.7 https://github.com/llvm/llvm-project.git llvm-main

COPY --from=stage1 /opt/clang-stage1 /opt/clang-stage1
ENV PATH="/opt/clang-stage1/bin:$PATH"

WORKDIR /build2

RUN cmake -G Ninja /src/llvm-main/llvm \
  -DCMAKE_BUILD_TYPE=Release \
  -DLLVM_ENABLE_PROJECTS="clang;lld" \
  -DLLVM_ENABLE_RUNTIMES="libcxx;libcxxabi;libunwind" \
  -DLLVM_TARGETS_TO_BUILD="X86" \
  -DCMAKE_INSTALL_PREFIX=${INSTALL_STAGE2} \
  -DCMAKE_C_COMPILER=/opt/clang-stage1/bin/clang \
  -DCMAKE_CXX_COMPILER=/opt/clang-stage1/bin/clang++ \
  -DLLVM_TEMPORARILY_ALLOW_OLD_TOOLCHAIN=ON \
  -DLLVM_ENABLE_TERMINFO=OFF \
  -DLIBCXX_ENABLE_STATIC=ON \
  -DLIBCXX_ENABLE_SHARED=OFF \
  -DLIBCXXABI_ENABLE_STATIC=ON \
  -DLIBCXXABI_ENABLE_SHARED=OFF \
  -DLIBUNWIND_ENABLE_STATIC=ON \
  -DLIBUNWIND_ENABLE_SHARED=OFF \
  -DLIBCXX_ENABLE_STATIC_ABI_LIBRARY=ON \
  -DLIBCXX_USE_COMPILER_RT=OFF \
  -DLIBCXX_CXX_ABI=libcxxabi \
  -DLLVM_USE_LINKER=lld \
  -DCMAKE_PREFIX_PATH=/opt/clang-stage1

RUN ninja install

# Package the toolchain
WORKDIR /opt
RUN zip -r clang-glibc223.zip clang-final

# Final export stage
FROM scratch AS export
COPY --from=stage2 /opt/clang-glibc223.zip /
