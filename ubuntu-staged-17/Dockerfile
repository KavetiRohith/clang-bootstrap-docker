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

# Package the LLVM 15 toolchain
WORKDIR /opt
RUN zip -r clang-glibc223.zip $(basename ${INSTALL_STAGE2}) # Uses INSTALL_STAGE2 for the dir name

# ----------------------------------------------------------------
# STAGE 3: Build LLVM/Clang 17.0.6 + Runtimes with Stage 2 compiler (LLVM 15)
# ----------------------------------------------------------------
FROM deps AS stage3

ARG LLVM15_TOOLCHAIN_PATH_FROM_STAGE2=/opt/clang-final # This is INSTALL_STAGE2 from the previous stage
ARG INSTALL_STAGE3=/opt/clang-llvm17-final
ARG LLVM_VERSION_STAGE3=llvmorg-17.0.6

WORKDIR /src
RUN git clone --depth=1 --branch ${LLVM_VERSION_STAGE3} https://github.com/llvm/llvm-project.git llvm-17

# Copy the compiler and its associated runtimes (libc++, etc.) from stage2
COPY --from=stage2 ${LLVM15_TOOLCHAIN_PATH_FROM_STAGE2} /opt/llvm15-bootstrap-toolchain
ENV PATH="/opt/llvm15-bootstrap-toolchain/bin:$PATH"

WORKDIR /build3

RUN cmake -G Ninja /src/llvm-17/llvm \
  -DCMAKE_BUILD_TYPE=Release \
  -DLLVM_ENABLE_PROJECTS="clang;lld" \
  -DLLVM_ENABLE_RUNTIMES="libcxx;libcxxabi;libunwind" \
  -DLLVM_TARGETS_TO_BUILD="X86" \
  -DCMAKE_INSTALL_PREFIX=${INSTALL_STAGE3} \
  -DCMAKE_C_COMPILER=/opt/llvm15-bootstrap-toolchain/bin/clang \
  -DCMAKE_CXX_COMPILER=/opt/llvm15-bootstrap-toolchain/bin/clang++ \
  -DLLVM_TEMPORARILY_ALLOW_OLD_TOOLCHAIN=ON \
  -DLLVM_ENABLE_LIBCXX=ON \
  -DCLANG_DEFAULT_CXX_STDLIB=libc++ \
  # Tell clang to prefer lld from previous toolchain
  -DCLANG_DEFAULT_LINKER=lld \ 
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
  # Ensure libcxx is built with thread support
  -DLIBCXX_ENABLE_THREADS=ON \ 
  # Ensure libcxxabi is built with thread support
  -DLIBCXXABI_ENABLE_THREADS=ON \ 
  # Use lld from bootstrap toolchain path (simpler)
  -DLLVM_USE_LINKER=lld \
  # Add pthread for linking stage3 components and tests
  -DCMAKE_EXE_LINKER_FLAGS="-pthread" \ 
  -DCMAKE_SHARED_LINKER_FLAGS="-pthread" \
  -DCMAKE_MODULE_LINKER_FLAGS="-pthread" \
  # Points to the toolchain used for bootstrapping
  -DCMAKE_PREFIX_PATH=/opt/llvm15-bootstrap-toolchain

RUN ninja install

# Package the LLVM 17 toolchain
WORKDIR /opt
RUN zip -r clang17-glibc223.zip $(basename ${INSTALL_STAGE3})

# ----------------------------------------------------------------
# Final export stages
# ----------------------------------------------------------------

# Export stage for LLVM 15 toolchain (original export target)
FROM scratch AS export-llvm15
COPY --from=stage2 /opt/clang-glibc223.zip /clang15-glibc223.zip

# Export stage for LLVM 17 toolchain
FROM scratch AS export-llvm17
COPY --from=stage3 /opt/clang17-glibc223.zip /clang17-glibc223.zip

# Default export: Makes LLVM 17 the default artifact if no specific target is chosen
FROM export-llvm17 AS export
# The COPY in the alias target (export-llvm17) defines what's in 'export'
# No specific COPY needed here as it inherits from export-llvm17.
# The final image 'export' will contain /clang17-glibc223.zip