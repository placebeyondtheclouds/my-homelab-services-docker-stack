ARG CUDA_VERSION=12.6.2
ARG DISTRO_VERSION=24.04
ARG OS_DISTRO=ubuntu
ARG CUDNN_TAG=cudnn

FROM nvidia/cuda:${CUDA_VERSION}-${CUDNN_TAG}-devel-${OS_DISTRO}${DISTRO_VERSION}

RUN DEBIAN_FRONTEND=noninteractive apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    cmake \
    libnvidia-ml-dev \
    python3 \
    python3-pip \
    python3-venv \
    git \
    ninja-build && apt-get clean && rm -rf /var/lib/apt/lists/*

RUN git clone https://github.com/kevmo314/scuda.git

WORKDIR /scuda

# COPY . .

RUN cd codegen && python3 -m venv venv && venv/bin/pip install -r requirements.txt && venv/bin/python3 ./codegen.py

RUN cmake .
RUN cmake --build .

CMD ["./local.sh", "server"]
