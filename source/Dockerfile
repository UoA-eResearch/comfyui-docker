
# Defines build arguments for the versions of PyTorch, CUDA, and cuDNN to use
# https://hub.docker.com/r/pytorch/pytorch/tags
ARG PYTORCH_VERSION=2.10.0
ARG CUDA_VERSION=13.0
ARG CUDNN_VERSION=9

# This image is based on the latest official PyTorch image, because it already contains CUDA, CuDNN, and PyTorch
FROM pytorch/pytorch:${PYTORCH_VERSION}-cuda${CUDA_VERSION}-cudnn${CUDNN_VERSION}-runtime

# Installs Git, because ComfyUI and the ComfyUI Manager are installed by cloning their respective Git repositories
RUN apt-get update --assume-yes && \
    apt-get install --assume-yes \
        git \
        sudo \
        libgl1 \
        libglib2.0-0 \
        python3-dev \
        ninja-build \
        build-essential && \
    rm -rf /var/cache/apt/archives /var/lib/apt/lists/*

# Defines build arguments for the versions of ComfyUI and ComfyUI Manager to use
# https://github.com/Comfy-Org/ComfyUI/releases
ARG COMFYUI_VERSION=0.17.2
# https://github.com/Comfy-Org/ComfyUI-Manager/tags
ARG COMFYUI_MANAGER_VERSION=main

# Clones the ComfyUI repository and checks out the latest release
RUN git clone https://github.com/Comfy-Org/ComfyUI.git /opt/comfyui && \
    cd /opt/comfyui && \
    git checkout "v${COMFYUI_VERSION}"

ARG COMFYUI_PATH=/opt/comfyui
WORKDIR ${COMFYUI_PATH}

# Clones the ComfyUI Manager repository and checks out the latest release; ComfyUI Manager is an extension for ComfyUI that enables users to install
# custom nodes and download models directly from the ComfyUI interface; instead of installing it to "/opt/comfyui/custom_nodes/ComfyUI-Manager", which
# is the directory it is meant to be installed in, it is installed to its own directory; the entrypoint will symlink the directory to the correct
# location upon startup; the reason for this is that the ComfyUI Manager must be installed in the same directory that it installs custom nodes to, but
# this directory is mounted as a volume, so that the custom nodes are not installed inside of the container and are not lost when the container is
# removed; this way, the custom nodes are installed on the host machine
RUN git clone https://github.com/Comfy-Org/ComfyUI-Manager.git ${COMFYUI_PATH}/custom_nodes/ComfyUI-Manager && \
    cd ${COMFYUI_PATH}/custom_nodes/ComfyUI-Manager && \
    git checkout ${COMFYUI_MANAGER_VERSION}

# Installs the required Python packages for both ComfyUI and the ComfyUI Manager
ENV UV_SYSTEM_PYTHON=1
RUN rm /usr/lib/python3*/EXTERNALLY-MANAGED

COPY config.ini ${COMFYUI_PATH}/custom_nodes/ComfyUI-Manager/config.ini

RUN uv pip install \
    --requirement ${COMFYUI_PATH}/requirements.txt \
    --requirement ${COMFYUI_PATH}/custom_nodes/ComfyUI-Manager/requirements.txt

COPY custom_nodes_list ./custom_nodes_list
# comfyui_manager/cm_cli/__main__.py
RUN python ${COMFYUI_PATH}/custom_nodes/ComfyUI-Manager/cm-cli.py install --exit-on-fail $(tr '\n' ' ' < ./custom_nodes_list | sed 's/ $//')
RUN git clone https://github.com/Gourieff/ComfyUI-ReActor ${COMFYUI_PATH}/custom_nodes/ComfyUI-ReActor && \
    cd ${COMFYUI_PATH}/custom_nodes/ComfyUI-ReActor && \
    pip install -r requirements.txt && \
    python install.py
RUN python ${COMFYUI_PATH}/custom_nodes/ComfyUI-Manager/cm-cli.py show installed
RUN python ${COMFYUI_PATH}/custom_nodes/ComfyUI-Manager/cm-cli.py fix all

ADD workflows ${COMFYUI_PATH}/user/default/workflows/
ADD input ${COMFYUI_PATH}/input/

# Exposes the default port of ComfyUI (this is not actually exposing the port to the host machine, but it is good practice to include it as metadata,
# so that the user knows which port to publish)
EXPOSE 8188

# Adds the startup script to the container; the startup script will create all necessary directories in the models and custom nodes volumes that were
# mounted to the container and symlink the ComfyUI Manager to the correct directory; it will also create a user with the same UID and GID as the user
# that started the container, so that the files created by the container are owned by the user that started the container and not the root user
ADD entrypoint.sh /entrypoint.sh
ENTRYPOINT ["/bin/bash", "/entrypoint.sh"]
