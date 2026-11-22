# Microsoft's dev container base: Ubuntu + git, curl, zsh, sudo, and a non-root "vscode" user.
FROM mcr.microsoft.com/devcontainers/base:ubuntu-24.04

# tmux for persistent terminal sessions (e.g. Claude in one pane, a server in another).
RUN apt-get update && apt-get install -y --no-install-recommends tmux \
    && rm -rf /var/lib/apt/lists/*

# uv, pinned so rebuilds are reproducible.
COPY --from=ghcr.io/astral-sh/uv:0.13.6 /uv /uvx /usr/local/bin/

# kind: runs a local Kubernetes cluster as Docker containers on WSL's Docker.
ARG KIND_VERSION=v0.33.0
RUN curl -fsSLo /usr/local/bin/kind \
      https://kind.sigs.k8s.io/dl/${KIND_VERSION}/kind-linux-amd64 \
    && chmod +x /usr/local/bin/kind

USER vscode

# Python managed by uv, not apt.
RUN uv python install 3.12

# Claude Code (native installer, ~/.local/bin). Create mount points so they're owned by vscode.
RUN curl -fsSL https://claude.ai/install.sh | bash \
    && mkdir -p /home/vscode/.claude /home/vscode/.kube
ENV PATH="/home/vscode/.local/bin:${PATH}"

RUN echo "set -g mouse on" > /home/vscode/.tmux.conf

# Only uv-managed Pythons; copy instead of hardlink across the WSL bind mount.
ENV UV_PYTHON_PREFERENCE=only-managed
ENV UV_LINK_MODE=copy
