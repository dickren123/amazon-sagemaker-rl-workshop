ARG AWS_REGION
FROM 520713654638.dkr.ecr.${AWS_REGION}.amazonaws.com/sagemaker-rl-tensorflow:coach0.11.0-cpu-py3

##############################################################
# Compatible g++ version for Tensorflow installation
##############################################################

RUN apt update
RUN apt-get install -y gcc-4.9 cmake
RUN update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-4.9 50
RUN apt-get install -y g++-4.9
RUN update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-4.9 50
##############################################################

RUN buildDeps=" \
        wget \
        build-essential \
    " \
    && apt-get update && apt-get install -y --no-install-recommends $buildDeps \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

##############################################################
# MPI & its dependencies
##############################################################

# Install Open MPI
RUN mkdir /tmp/openmpi && \
    cd /tmp/openmpi && \
    wget https://www.open-mpi.org/software/ompi/v3.0/downloads/openmpi-3.0.0.tar.gz && \
    tar zxf openmpi-3.0.0.tar.gz && \
    cd openmpi-3.0.0 && \
    ./configure --enable-orterun-prefix-by-default && \
    make -j $(nproc) all && \
    make install && \
    ldconfig && \
    rm -rf /tmp/openmpi

# Install tensorflow 1.15.0 
RUN pip install --upgrade pip && pip install tensorflow==1.15.0

# Install Horovod, temporarily using CUDA stubs （打开HOROVOD_WITHOUT_GLOO)
RUN ldconfig /usr/local/cuda-9.0/targets/x86_64-linux/lib/stubs && \
    HOROVOD_CPU_ALLREDUCE=MPI HOROVOD_WITH_TENSORFLOW=1 HOROVOD_WITHOUT_GLOO=1 pip install --no-cache-dir horovod && \
    ldconfig

# Create a wrapper for OpenMPI to allow running as root by default
RUN mv /usr/local/bin/mpirun /usr/local/bin/mpirun.real && \
    echo '#!/bin/bash' > /usr/local/bin/mpirun && \
    echo 'mpirun.real --allow-run-as-root "$@"' >> /usr/local/bin/mpirun && \
    chmod a+x /usr/local/bin/mpirun

# Configure OpenMPI to run good defaults:
#   --bind-to none --map-by slot --mca btl_tcp_if_exclude lo,docker0
RUN echo "hwloc_base_binding_policy = none" >> /usr/local/etc/openmpi-mca-params.conf && \
    echo "rmaps_base_mapping_policy = slot" >> /usr/local/etc/openmpi-mca-params.conf

# Install mpi4py

RUN pip install mpi4py

ENV LD_LIBRARY_PATH=/usr/local/openmpi/lib:$LD_LIBRARY_PATH
ENV PATH /usr/local/openmpi/bin/:$PATH
ENV PATH=/usr/local/nvidia/bin:$PATH

ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1

# SSH. Partially taken from https://docs.docker.com/engine/examples/running_ssh_service/
RUN apt-get update && apt-get install -y --no-install-recommends openssh-server && \
    mkdir -p /var/run/sshd

# SSH login fix. Otherwise user is kicked off after login
RUN sed 's@session\s*required\s*pam_loginuid.so@session optional pam_loginuid.so@g' -i /etc/pam.d/sshd

# Create SSH key.
RUN mkdir -p /root/.ssh/ && \
  ssh-keygen -q -t rsa -N '' -f /root/.ssh/id_rsa && \
  cp /root/.ssh/id_rsa.pub /root/.ssh/authorized_keys && \
  printf "Host *\n  StrictHostKeyChecking no\n" >> /root/.ssh/config

# Hostname Fix
COPY resources/changehostname.c /
COPY resources/change-hostname.sh /
COPY resources/change-hostname.sh /usr/local/bin/change-hostname.sh

RUN chmod +x /usr/local/bin/change-hostname.sh
RUN chmod +x /change-hostname.sh

RUN pip install keras
RUN pip install retrying
##############################################################
WORKDIR /opt

############################################
# Roboschool
############################################

RUN apt-get update && apt-get install -y \
      git cmake ffmpeg pkg-config \
      qtbase5-dev libqt5opengl5-dev libassimp-dev \
      libtinyxml-dev \
      libgl1-mesa-dev \
    && cd /opt \
    && apt-get clean && rm -rf /var/cache/apt/archives/* /var/lib/apt/lists/*

RUN apt-get update && \
    apt-get install -y libboost-python-dev

RUN apt-get update \
    && apt-get install -y --no-install-recommends python3.6-dev \
    && ln -s -f /usr/bin/python3.6 /usr/bin/python \
    && apt-get clean \
    && rm -rf /var/cache/apt/archives/* /var/lib/apt/lists/*

RUN curl -fSsL -O https://bootstrap.pypa.io/get-pip.py && \
    python get-pip.py && \
    rm get-pip.py

RUN pip install --upgrade \
    pip \
    setuptools

RUN pip install roboschool==1.0.48

ENV PYTHONUNBUFFERED 1

############################################
# Baselines
############################################
RUN apt-get update && apt-get install -y cmake libopenmpi-dev python3-dev zlib1g-dev wget

ENV BASELINES_PATH /opt/baselines/
RUN git clone https://github.com/openai/baselines.git

RUN pip install -e ${BASELINES_PATH}

############################################
# Stable Baselines
############################################
RUN pip install stable-baselines

############################################
# Test Installation
############################################
# Test to verify if all required dependencies installed successfully or not.
RUN python -c "import gym; import roboschool;"

