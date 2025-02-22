# *******************************************************************************
# Copyright 2020-2024 Arm Limited and affiliates.
# SPDX-License-Identifier: Apache-2.0
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
# *******************************************************************************


# ========
# Stage 1: Base image including OS and key packages
# ========
ARG njobs
ARG bazel_mem
ARG default_py_version=3.10

FROM ubuntu:22.04 AS tensorflow-base
ARG default_py_version
ENV PY_VERSION="${default_py_version}"

RUN if ! [ "$(arch)" = "aarch64" ] ; then exit 1; fi

# Install core OS packages
RUN apt-get -y update && \
    apt-get -y install software-properties-common && \
    add-apt-repository ppa:ubuntu-toolchain-r/test && \
    apt-get -y install \
      accountsservice \
      apport \
      at \
      autoconf \
      automake \
      bc \
      build-essential \
      cmake \
      cpufrequtils \
      curl \
      ethtool \
      gettext-base \
      gfortran-11 \
      git \
      iproute2 \
      iputils-ping \
      libbz2-dev \
      libc++-dev \
      libcgal-dev \
      libffi-dev \
      libfreetype6-dev \
      libhdf5-dev \
      libjpeg-dev \
      liblzma-dev \
      libncurses5-dev \
      libncursesw5-dev \
      libpng-dev \
      libreadline-dev \
      libssl-dev \
      libsqlite3-dev \
      libtool \
      libxml2-dev \
      libxslt-dev \
      locales \
      lsb-release \
      lvm2 \
      lld \
      moreutils \
      net-tools \
      open-iscsi \
      openjdk-8-jdk \
      openssl \
      pciutils \
      pkg-config \
      policykit-1 \
      python${PY_VERSION} \
      python${PY_VERSION}-dev \
      python${PY_VERSION}-distutils \
      python${PY_VERSION}-venv \
      python3-pip \
      python3-openssl \
      patchelf \
      rsync \
      rsyslog \
      snapd \
      scons \
      ssh \
      sudo \
      swig \
      time \
      udev \
      unzip \
      ufw \
      uuid-runtime \
      vim \
      wget \
      xz-utils \
      zip \
      zlib1g-dev

# Add clang 17
RUN wget -qO- https://apt.llvm.org/llvm-snapshot.gpg.key | tee /etc/apt/trusted.gpg.d/apt.llvm.org.asc
RUN add-apt-repository "deb http://apt.llvm.org/jammy/ llvm-toolchain-jammy-17 main"
RUN apt-get install -y  --no-install-recommends llvm-17 clang-17 clang++-17 python3-dev libomp-17-dev lld-17

# Set default clang, python and pip versions
RUN update-alternatives --install /usr/bin/clang clang /usr/bin/clang-17 1 && \
    update-alternatives --install /usr/bin/clang++ clang++ /usr/bin/clang++-17 1 && \
    update-alternatives --install /usr/bin/gfortran gfortran /usr/bin/gfortran-11 1 && \
    update-alternatives --install /usr/bin/python python /usr/bin/python3 1 && \
    update-alternatives --install /usr/bin/pip pip /usr/bin/pip3 1

# DOCKER_USER for the Docker user
ENV DOCKER_USER=ubuntu

# Setup default user
RUN useradd --create-home -s /bin/bash -m $DOCKER_USER && echo "$DOCKER_USER:Portland" | chpasswd && adduser $DOCKER_USER sudo
RUN echo '%sudo ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers
RUN echo 'debconf debconf/frontend select Noninteractive' | debconf-set-selections

# Import profile for bash
COPY bash_profile /home/$DOCKER_USER/.bash_profile
RUN chown $DOCKER_USER:$DOCKER_USER /home/$DOCKER_USER/.bash_profile
COPY patches/welcome.txt /home/$DOCKER_USER/.
RUN echo '[ ! -z "$TERM" -a -r /home/$DOCKER_USER/welcome.txt ] && cat /home/$DOCKER_USER/welcome.txt' >> /etc/bash.bashrc

# ========
# Stage 2: augment the base image with some essential libs
# ========
FROM tensorflow-base AS tensorflow-libs
ARG njobs
ARG cpu
ARG arch
ARG blas_cpu
ARG blas_ncores

ENV NP_MAKE="${njobs}" \
    CPU="${cpu}" \
    ARCH="${arch}" \
    BLAS_CPU="${blas_cpu}" \
    BLAS_NCORES="${blas_ncores}"

# Key version numbers
ENV YAML_VERSION=master

# Package build parameters
ENV PROD_DIR=/opt \
    PACKAGE_DIR=packages

# Make directories to hold package source & build directories (PACKAGE_DIR)
# and install build directories (PROD_DIR).
RUN mkdir -p $PACKAGE_DIR && \
    mkdir -p $PROD_DIR

# Common compiler settings
ENV CC=clang \
    CXX=clang++ \
    BASE_CFLAGS="-mcpu=${CPU} -march=${ARCH} -O3" \
    BASE_LDFLAGS=""

# Build and install OpenCV from GitHub sources (needed for C++ API examples)
COPY scripts/build-opencv.sh $PACKAGE_DIR/.
RUN $PACKAGE_DIR/build-opencv.sh
ENV LD_LIBRARY_PATH=$PROD_DIR/opencv/install/lib:$LD_LIBRARY_PATH

# Build and install yaml-cpp from Github source (needed for C++ API examples)
COPY scripts/build-yamlcpp.sh $PACKAGE_DIR/.
RUN $PACKAGE_DIR/build-yamlcpp.sh

# ========
# Stage 3: install essential python dependencies into a venv
# ========
FROM tensorflow-libs AS tensorflow-tools
ARG njobs
ARG default_py_version
ARG cpu
ARG arch

ENV PY_VERSION="${default_py_version}" \
    NP_MAKE="${njobs}" \
    CPU="${cpu}" \
    ARCH="${arch}"

# Using venv means this can be done in userspace
WORKDIR /home/$DOCKER_USER
USER $DOCKER_USER
ENV PACKAGE_DIR=/home/$DOCKER_USER/$PACKAGE_DIR
RUN mkdir -p $PACKAGE_DIR

# Setup a Python virtual environment
ENV VIRTUAL_ENV=/home/$DOCKER_USER/python3-venv
RUN python -m venv $VIRTUAL_ENV
ENV PATH="$VIRTUAL_ENV/bin:$PATH"

# Install Rust into user-space, needed for transformers dependencies
RUN curl https://sh.rustup.rs -sSf | sh -s -- -y
ENV PATH="/home/$DOCKER_USER/.cargo/bin:${PATH}"

# Install some basic python packages
RUN pip install --no-cache-dir --upgrade pip
RUN pip install --no-cache-dir "setuptools>=41.0.0" six mock wheel cython==0.29.36 sh

# Install some  basic python packages needed for SciPy
RUN pip install --no-cache-dir pybind11[global] pyangbind pythran

# Install NumPy and SciPy
RUN pip install --no-cache-dir numpy==1.25.2 scipy==1.10.1

# Install some TensorFlow essentials
# enum34 is not compatible with Python 3.6+, and not required
# it is installed as a dependency for an earlier package and needs
# to be removed in order for the OpenCV build to complete.
RUN pip uninstall enum34 -y
RUN HDF5_DIR=/usr/lib/aarch64-linux-gnu/hdf5/serial pip install h5py==3.8.0
RUN pip install --no-cache-dir grpcio tensorflow-io pytest
RUN pip install --no-cache-dir ck==1.55.5 absl-py pycocotools pillow==8.2.0
RUN pip install --no-cache-dir transformers pandas

# Install the backwards-compatible tf-keras package as Keras 3 (requirement
# for tensorflow v2.16) is not yet supported in transformers
RUN pip install --no-cache-dir tf-keras==2.16.0

# Install OpenCV into our venv (needed for MLCommons benchmarks)
RUN pip install --no-cache-dir opencv-python-headless==4.8.0.74

# Pin to 0.2.0 due to incompatibility of newer
# versions with TF 2.14-RC
RUN pip install --no-cache-dir ml_dtypes==0.2.0

# Remove dependency on HDF5 save support in Keras as this does not
# work with TensorFlow 2.11 and Transformers 4.24
COPY scripts/remove-keras-hdf-save-dependency.sh $PACKAGE_DIR/.
RUN $PACKAGE_DIR/remove-keras-hdf-save-dependency.sh

CMD ["bash", "-l"]

# ========
# Stage 4: install Bazel, and build tensorflow
# ========
FROM tensorflow-libs AS tensorflow-dev
ARG njobs
ARG bazel_mem
ARG enable_onednn
ARG onednn_opt
ARG tf_version
ARG default_py_version
ARG tune
ARG arch
ARG eigen_l1_cache
ARG eigen_l2_cache
ARG eigen_l3_cache

ENV NP_MAKE="${njobs}" \
    BZL_RAM="${bazel_mem}" \
    ONEDNN_BUILD="${onednn_opt}" \
    TUNE="${tune}" \
    ARCH="${arch}" \
    EIGEN_L1_CACHE="${eigen_l1_cache}" \
    EIGEN_L2_CACHE="${eigen_l2_cache}" \
    EIGEN_L3_CACHE="${eigen_l3_cache}"
# Key version numbers
ENV PY_VERSION="${default_py_version}" \
    TF_VERSION="${tf_version}"
# Set runtime flag to enable/disable oneDNN backend
ENV TF_ENABLE_ONEDNN_OPTS="${enable_onednn}"
# Set runtime flag to ensure oneDNN primitives are cached for large batch sizes
ENV TF_MKL_OPTIMIZE_PRIMITIVE_MEMUSE=0

# Use a PACKAGE_DIR in userspace
WORKDIR /home/$DOCKER_USER
USER $DOCKER_USER
ENV PACKAGE_DIR=/home/$DOCKER_USER/$PACKAGE_DIR
RUN mkdir -p $PACKAGE_DIR

# Copy in the Python virtual environment
ENV VIRTUAL_ENV=/home/$DOCKER_USER/python3-venv
COPY --chown=$DOCKER_USER:$DOCKER_USER --from=tensorflow-tools $VIRTUAL_ENV /home/$DOCKER_USER/python3-venv
ENV PATH="$VIRTUAL_ENV/bin:$PATH"

# Get Bazel binary for AArch64 using Bazelisk
RUN mkdir -p $PACKAGE_DIR/bazel \
    && wget https://github.com/bazelbuild/bazelisk/releases/download/v1.19.0/bazelisk-linux-arm64 -O $PACKAGE_DIR/bazel/bazel \
    && chmod a+x $PACKAGE_DIR/bazel/bazel
ENV PATH="$PACKAGE_DIR/bazel:$PATH"

# Build TensorFlow
# tensorflow patches
COPY patches/tf_acl.patch $PACKAGE_DIR/.
COPY patches/tf_disable_lnorm_fusions.patch $PACKAGE_DIR/.
# oneDNN patches
COPY patches/onednn_acl_indirect_conv.patch $PACKAGE_DIR/.
COPY patches/onednn_acl_thread_local_scheduler.patch $PACKAGE_DIR/.
COPY patches/onednn_acl_reorder.patch $PACKAGE_DIR/.
COPY patches/onednn_acl_fp32_bf16_reorder.patch $PACKAGE_DIR/.
COPY patches/onednn_acl_dynamic_quantization.patch $PACKAGE_DIR/.
COPY patches/onednn_acl_in_place_sum.patch $PACKAGE_DIR/.
COPY patches/onednn_acl_update_conv2dinfo_constructor.patch $PACKAGE_DIR/.
COPY patches/onednn_acl_stop_linking_to_arm_core_library.patch $PACKAGE_DIR/.
COPY patches/onednn_acl_add_Acdb8a_and_Acdb4a_acl_reorders.patch $PACKAGE_DIR/.
# ACL patches
COPY patches/acl_dynamic_quantization.patch $PACKAGE_DIR/.
COPY patches/acl_in_place_sum.patch $PACKAGE_DIR/.
COPY patches/acl_parallelize_im2col.patch $PACKAGE_DIR/.
COPY patches/acl_thread_local_scheduler.patch $PACKAGE_DIR/.

COPY scripts/build-tensorflow.sh $PACKAGE_DIR/.
RUN $PACKAGE_DIR/build-tensorflow.sh

# build tensorflow-addons
COPY scripts/build-tensorflow-addons.sh $PACKAGE_DIR/.
COPY patches/tf_addons_update.patch $PACKAGE_DIR/.
RUN $PACKAGE_DIR/build-tensorflow-addons.sh

CMD ["bash", "-l"]

# ========
# Stage 5: Install benchmarks and examples
# ========
FROM tensorflow-libs AS tensorflow
ARG enable_onednn

# Set runtime flag to enable/disable oneDNN backend
ENV TF_ENABLE_ONEDNN_OPTS="${enable_onednn}"

WORKDIR /home/$DOCKER_USER
USER $DOCKER_USER

# Copy in the Python virtual environment
ENV VIRTUAL_ENV=/home/$DOCKER_USER/python3-venv
COPY --chown=$DOCKER_USER:$DOCKER_USER --from=tensorflow-dev $VIRTUAL_ENV /home/$DOCKER_USER/python3-venv
ENV PATH="$VIRTUAL_ENV/bin:$PATH"
ENV LD_LIBRARY_PATH=$VIRTUAL_ENV/tensorflow/lib:$LD_LIBRARY_PATH

# Examples, benchmarks, and associated 'helper' scripts will be installed
# in $EXAMPLE_DIR.
ENV EXAMPLE_DIR=/home/$DOCKER_USER/examples
ENV MLCOMMONS_DIR=$EXAMPLE_DIR/MLCommons
RUN mkdir -p $EXAMPLE_DIR
RUN mkdir -p $MLCOMMONS_DIR

# Clone and install MLCommons (MLPerf)
COPY scripts/build-mlcommons.sh $MLCOMMONS_DIR/.
RUN $MLCOMMONS_DIR/build-mlcommons.sh
RUN rm -f $MLCOMMONS_DIR/build-mlcommons.sh

# Install missing Python package dependencies required to run examples
RUN pip install --no-cache-dir pyyaml
RUN pip install --no-cache-dir tqdm
RUN pip install --no-cache-dir protobuf==4.23.4

# MLCommons ServerMode
COPY patches/Makefile.patch $MLCOMMONS_DIR/.
COPY patches/servermode.patch $MLCOMMONS_DIR/.
COPY scripts/build-boost.sh $MLCOMMONS_DIR/.
COPY scripts/build-loadgen-integration.sh $MLCOMMONS_DIR/.

# Copy scripts to download dataset and models
COPY scripts/download-dataset.sh $MLCOMMONS_DIR/.
COPY scripts/download-model.sh $MLCOMMONS_DIR/.
COPY scripts/setup-servermode.sh $MLCOMMONS_DIR/.

# Copy examples
COPY --chown=$DOCKER_USER:$DOCKER_USER examples $EXAMPLE_DIR
COPY patches/welcome_verbose.txt /home/$DOCKER_USER/welcome.txt

CMD ["bash", "-l"]
