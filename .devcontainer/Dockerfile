# See here for image contents: https://github.com/microsoft/vscode-dev-containers/tree/v0.209.6/containers/ubuntu/.devcontainer/base.Dockerfile

# [Choice] Ubuntu version (use hirsuite or bionic on local arm64/Apple Silicon): hirsute, focal, bionic
ARG VARIANT="hirsute"
FROM mcr.microsoft.com/vscode/devcontainers/base:0-${VARIANT}

# Install useful tools
RUN apt-get update && export DEBIAN_FRONTEND=noninteractive \
    && apt-get -y install --no-install-recommends etcd-client \
    && apt-get -y install --no-install-recommends vim

# Copy utility scripts
COPY scripts /tmp

# Enable useful bash aliases & completion for them
RUN cp tmp/scripts/.bash_aliases /home/vscode \
      cat tmp/scripts/enable_completion.sh >> /home/vscode/.bashrc

