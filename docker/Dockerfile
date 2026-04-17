FROM nvidia/cudagl:11.3.0-devel-ubuntu20.04
ENV NVIDIA_DRIVER_CAPABILITIES=all
ENV OPENCV_IO_ENABLE_OPENEXR=1
# Dockerビルド中はGPU不在のためCUDAアーキテクチャを明示指定（未設定だとarch_list空でIndexError）
ENV TORCH_CUDA_ARCH_LIST="6.0 6.1 7.0 7.5 8.0 8.6"

ENV TZ=Asia/Tokyo
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

# Canonicalサーバー(91.189.x.x, 185.125.x.x)がブロックされているためJAISTミラーを使用
RUN sed -i 's|http://archive.ubuntu.com|http://ftp.jaist.ac.jp/pub/Linux/ubuntu|g' /etc/apt/sources.list && \
    sed -i 's|http://security.ubuntu.com|http://ftp.jaist.ac.jp/pub/Linux/ubuntu|g' /etc/apt/sources.list

RUN apt-get update --fix-missing && \
    DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends --fix-missing \
    bash-completion \
    build-essential \
    bzip2 \
    ca-certificates \
    checkinstall \
    cmake \
    cmake-curses-gui \
    curl \
    doxygen \
    freeglut3-dev \
    g++ \
    gcc \
    gfortran \
    git \
    htop \
    libatlas-base-dev \
    libavcodec-dev \
    libavformat-dev \
    libboost-all-dev \
    libdc1394-22-dev \
    libegl1 \
    libfaac-dev \
    libflann-dev \
    libgflags-dev \
    libgoogle-glog-dev \
    libgphoto2-dev \
    libgtk2.0-dev \
    libhdf5-dev \
    libjpeg-dev \
    libjpeg8-dev \
    libmp3lame-dev \
    libopencore-amrnb-dev \
    libopencore-amrwb-dev \
    libpng-dev \
    libproj-dev \
    libprotobuf-dev \
    libswscale-dev \
    libtbb-dev \
    libtheora-dev \
    libtiff5-dev \
    libv4l-dev \
    libvorbis-dev \
    libvulkan1 \
    libxext6 \
    libxine2-dev \
    libxvidcore-dev \
    libyaml-cpp-dev \
    libzmq3-dev \
    pkg-config \
    proj-data \
    protobuf-compiler \
    qt5-default \
    rsync \
    tmux \
    unzip \
    v4l-utils \
    vim \
    vulkan-utils \
    wget \
    x264 \
    xvfb \
    yasm \
    && rm -rf /var/lib/apt/lists/*

RUN cd / && git clone https://github.com/pybind/pybind11 && \
    cd pybind11 && git checkout v2.10.0 && \
    mkdir build && cd build && cmake .. -DCMAKE_BUILD_TYPE=Release -DPYBIND11_INSTALL=ON -DPYBIND11_TEST=OFF && \
    make -j6 && make install

RUN cd / && wget https://gitlab.com/libeigen/eigen/-/archive/3.4.0/eigen-3.4.0.tar.gz && \
    tar xvzf ./eigen-3.4.0.tar.gz && \
    cd eigen-3.4.0 && \
    mkdir build && \
    cd build && \
    cmake .. && \
    make install

# uvのインストールとPATH設定
RUN curl -LsSf https://astral.sh/uv/install.sh | sh
ENV PATH="/root/.local/bin:$PATH"

# /workspace/kitsunai-projectを作成して移動
WORKDIR /workspace/kitsunai-project
COPY pyproject.toml uv.lock .python-version ./
RUN uv sync --frozen

# .venvをPATHに追加してactivate相当の動作を実現
ENV PATH="/workspace/kitsunai-project/.venv/bin:$PATH"

# ------------------
# maniskillのセットアップ
COPY docker/nvidia_icd.json /usr/share/vulkan/icd.d/nvidia_icd.json
COPY docker/nvidia_layers.json /etc/vulkan/implicit_layer.d/nvidia_layers.json

# ------------------
# foundationposeのセットアップ
# ビルドに必要なあれこれとgdown
RUN uv pip install pip cython "setuptools<70.0.0" wheel gdown

# pytorch3d
RUN uv pip install "git+https://github.com/facebookresearch/pytorch3d.git@stable" --no-build-isolation

# kaolin
RUN cd / && git clone --recursive --branch v0.14.0 https://github.com/NVIDIAGameWorks/kaolin && \
    cd kaolin && \
    FORCE_CUDA=1 uv pip install . --no-build-isolation

# nvdiffrast
RUN cd / && git clone https://github.com/NVlabs/nvdiffrast && \
    cd nvdiffrast && \
    uv pip install . --no-build-isolation

# foundationpose
RUN cd /workspace/kitsunai-project && \
    git clone https://github.com/NVlabs/FoundationPose.git && \
    cd FoundationPose/mycpp && \
    mkdir -p build && cd build && cmake .. -DPYTHON_EXECUTABLE=$(which python) && make -j11

RUN cd /workspace/kitsunai-project/FoundationPose/bundlesdf/mycuda && \
    uv pip install --no-build-isolation .

# モデルとdemo_dataをgdownで配置
# FoundationPoseの下にweights/を作成し、その中で以下コマンド
RUN mkdir -p /workspace/kitsunai-project/FoundationPose/weights && \
    cd /workspace/kitsunai-project/FoundationPose/weights && \
    gdown --folder https://drive.google.com/drive/folders/1BEQLZH69UO5EOfah-K9bfI3JyP9Hf7wC?usp=drive_link && \
    gdown --folder https://drive.google.com/drive/folders/12Te_3TELLes5cim1d7F7EBTwUSe7iRBj?usp=drive_link

# FoundationPoseの下にdemo_data/を作成し、その中で以下のコマンド
RUN mkdir -p /workspace/kitsunai-project/FoundationPose/demo_data && \
    cd /workspace/kitsunai-project/FoundationPose/demo_data && \
    gdown https://drive.google.com/file/d/1LjshMkgy2l_SvBwe4nK2IqP38X2ad0Py/view?usp=drive_link && \
    gdown https://drive.google.com/file/d/1AwV9sESDKMgXGUu2n1o0Pc4x2JGYdVB3/view?usp=drive_link && \
    unzip "*.zip"
