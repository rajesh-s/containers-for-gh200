ARG CUDA_VERSION=12.4.1
ARG IMAGE_DISTRO=ubuntu22.04
ARG PYTHON_VERSION=3.12

# ---------- Builder Base ----------
FROM nvcr.io/nvidia/cuda:${CUDA_VERSION}-devel-${IMAGE_DISTRO} AS base

# Set arch lists for all targets
# 'a' suffix is not forward compatible but enables all optimizations
ARG TORCH_CUDA_ARCH_LIST="9.0a"
ENV TORCH_CUDA_ARCH_LIST=${TORCH_CUDA_ARCH_LIST}
ARG VLLM_FA_CMAKE_GPU_ARCHES="90a-real"
ENV VLLM_FA_CMAKE_GPU_ARCHES=${VLLM_FA_CMAKE_GPU_ARCHES}

# Update apt packages and install dependencies
ENV DEBIAN_FRONTEND=noninteractive
RUN apt update
RUN apt upgrade -y
RUN apt install -y --no-install-recommends \
        curl \
        git \
        libibverbs-dev \
        zlib1g-dev

# Clean apt cache
RUN apt clean
RUN rm -rf /var/lib/apt/lists/*
RUN rm -rf /var/cache/apt/archives

# Set compiler paths
ENV CC=/usr/bin/gcc
ENV CXX=/usr/bin/g++

# Install uv
RUN curl -LsSf https://astral.sh/uv/install.sh | env UV_INSTALL_DIR=/usr/local/bin sh

# Setup build workspace
WORKDIR /workspace

# Prep build venv
ARG PYTHON_VERSION
RUN uv venv -p ${PYTHON_VERSION} --seed --python-preference only-managed
ENV VIRTUAL_ENV=/workspace/.venv
ENV PATH=${VIRTUAL_ENV}/bin:${PATH}
ENV CUDA_HOME=/usr/local/cuda
ENV LD_LIBRARY_PATH=${CUDA_HOME}/lib64:${LD_LIBRARY_PATH}

RUN uv pip install numpy==2.0.0
# Install pytorch nightly
RUN uv pip install torch torchvision torchaudio --extra-index-url https://download.pytorch.org/whl/cu126

FROM base AS build-base
RUN mkdir /wheels

# Install build deps that aren't in project requirements files
# Make sure to upgrade setuptools to avoid triton build bug
RUN uv pip install -U build cmake ninja pybind11 setuptools wheel

FROM build-base AS build-triton
ARG TRITON_REF=release/3.2.x
ARG TRITON_BUILD_SUFFIX=+cu126
ENV TRITON_WHEEL_VERSION_SUFFIX=${TRITON_BUILD_SUFFIX:-}
RUN git clone https://github.com/triton-lang/triton.git
RUN cd triton && \
    git checkout ${TRITON_REF} && \
    git submodule sync && \
    git submodule update --init --recursive -j 8 && \
    uv build python --wheel --no-build-isolation -o /wheels

FROM build-base AS build-xformers
ARG XFORMERS_REF=v0.0.29.post2
ARG XFORMERS_BUILD_VERSION=0.0.29.post2+cu126
ENV BUILD_VERSION=${XFORMERS_BUILD_VERSION:-${XFORMERS_REF#v}}
RUN git clone  https://github.com/facebookresearch/xformers.git
RUN cd xformers && \
    git checkout ${XFORMERS_REF} && \
    git submodule sync && \
    git submodule update --init --recursive -j 8 && \
    uv build --wheel --no-build-isolation -o /wheels

FROM build-base AS build-flashinfer
ARG FLASHINFER_ENABLE_AOT=1
ARG FLASHINFER_REF=v0.2.2.post1
ARG FLASHINFER_BUILD_SUFFIX=cu126
ENV FLASHINFER_LOCAL_VERSION=${FLASHINFER_BUILD_SUFFIX:-}
RUN git clone https://github.com/flashinfer-ai/flashinfer.git
RUN cd flashinfer && \
    git checkout ${FLASHINFER_REF} && \
    git submodule sync && \
    git submodule update --init --recursive -j 8 && \
    uv build --wheel --no-build-isolation -o /wheels

FROM build-base AS build-vllm
ARG VLLM_REF=v0.8.4
RUN git clone https://github.com/vllm-project/vllm.git
RUN cd vllm && \
    git checkout ${VLLM_REF} && \
    git submodule sync && \
    git submodule update --init --recursive -j 8 && \
    uv pip install -r requirements/build.txt && \
    uv build --wheel --no-build-isolation -o /wheels


FROM base AS vllm-openai
COPY --from=build-flashinfer /wheels/* wheels/
COPY --from=build-triton /wheels/* wheels/
COPY --from=build-vllm /wheels/* wheels/
COPY --from=build-xformers /wheels/* wheels/

# Install and cleanup wheels
RUN uv pip install wheels/*
RUN rm -r wheels

# Install pynvml
RUN uv pip install pynvml pandas

# Add additional packages for vLLM OpenAI
RUN uv pip install accelerate hf_transfer modelscope bitsandbytes timm boto3 runai-model-streamer runai-model-streamer[s3] tensorizer

# Clean uv cache
RUN uv clean

# python3-config https://github.com/astral-sh/uv/issues/10263
RUN export PATH="$(dirname $(realpath .venv/bin/python)):$PATH"

# InfiniStore dependencies -> not needed with patched LMCache below
# RUN apt-get update && apt-get install -y --no-install-recommends \
#     libuv1-dev \
#     libflatbuffers-dev \
#     libspdlog-dev \
#     libfmt-dev \
#     ibverbs-utils \
#     libibverbs-dev \
#     libboost-dev \
#     libboost-stacktrace-dev \
#     git \
#     curl \
#     build-essential

# RUN git clone https://github.com/bytedance/InfiniStore.git && \
#     cd InfiniStore && git checkout 0.2.33 && \
#     pip install -e .
# RUN apt update
# RUN apt install -y cmake libuv1-dev libspdlog-dev libboost-dev libboost-all-dev # pybind11-dev python3-dev
# RUN uv pip install pybind11
# RUN git clone -b v1.12.0 https://github.com/google/flatbuffers.git && \
#   cd flatbuffers && \
#   cmake -B build -DFLATBUFFERS_BUILD_TESTS=OFF && \
#   cmake --build build -j && \
#   cmake --install build && cd ..
# RUN git clone https://github.com/bytedance/InfiniStore
# RUN sed -i 's|^INCLUDES = -I/usr/local/.*|INCLUDES = -I/root/.local/share/uv/python/cpython-3.12.10-linux-aarch64-gnu/include/python3.12|' InfiniStore/src/Makefile
# RUN cd InfiniStore/src && make
# RUN cd InfiniStore && pip install --no-deps --no-build-isolation -e .

# # LMCache dependencies
# RUN uv pip install -U aiofiles pyyaml redis nvtx safetensors transformers psutil aiohttp sortedcontainers prometheus_client msgspec
# RUN git clone https://github.com/LMCache/torchac_cuda.git && \
#     cd torchac_cuda && \
#     python setup.py install

RUN git clone https://github.com/rajesh-s/LMCache.git && \
    cd LMCache && \
    sed -i 's/2\.5\.1/2.6.0/g' pyproject.toml setup.py && \
    sed 's#numpy==1\.26\.4#numpy#g' pyproject.toml setup.py requirements.txt && \
    python setup.py install

# Enable hf-transfer
ENV HF_HUB_ENABLE_HF_TRANSFER=1
RUN uv pip install numpy==2.0.0 datasets aiohttp

# Install nsys for profiling
ARG NSYS_URL=https://developer.nvidia.com/downloads/assets/tools/secure/nsight-systems/2025_2/
ARG NSYS_PKG=nsight-systems-cli-2025.2.1_2025.2.1.130-1_arm64.deb

RUN apt-get update && apt install -y wget libglib2.0-0
RUN wget ${NSYS_URL}${NSYS_PKG} && dpkg -i $NSYS_PKG && rm $NSYS_PKG
RUN apt install -y --no-install-recommends tmux cmake

# API server entrypoint
ENTRYPOINT ["vllm", "serve"]
# CMD ["/bin/bash"]