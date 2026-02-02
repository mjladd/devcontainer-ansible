# syntax=docker/dockerfile:1
FROM ghcr.io/astral-sh/uv:python3.13-bookworm

# when pipling commands (ex. curl then | bash) the exit code is only from the last command
# so if curl fails but bash exits with 0, Docker things the command succeeded
# pipefail retuns the exit code of the first failing command.
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

# Set environment variables
ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    UV_COMPILE_BYTECODE=1 \
    UV_LINK_MODE=copy \
    UV_PROJECT_ENVIRONMENT=/workspace/.venv \
    VIRTUAL_ENV=/workspace/.venv \
    PATH="/workspace/.venv/bin:$PATH" \
    DEVCONTAINER=true \
    LC_ALL=C.UTF-8 \
    LANG=C.UTF-8 \
    STARSHIP_CONFIG=/usr/local/etc/starship.toml

# Install system dependencies with cache mount
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && \
    apt-get upgrade -y && \
    apt-get install -y --no-install-recommends \
        curl \
        unzip \
        fontconfig \
        ca-certificates \
        gcc \
        g++ \
        git \
        zsh \
        && rm -rf /var/lib/apt/lists/*

# Install Cascadia Code Nerd Font
RUN mkdir -p /usr/local/share/fonts/cascadiacode && \
    curl -fLo "/tmp/CascadiaCode.zip" "https://github.com/ryanoasis/nerd-fonts/releases/download/v3.2.1/CascadiaCode.zip" && \
    unzip "/tmp/CascadiaCode.zip" -d /usr/local/share/fonts/cascadiacode && \
    rm "/tmp/CascadiaCode.zip" && \
    fc-cache -fv

# Copy Starship config
COPY .devcontainer/starship.toml /usr/local/etc/starship.toml

# Set working directory
WORKDIR /workspace

# Copy project files and install dependencies using uv sync
COPY pyproject.toml uv.lock* ./

# Install Python dependencies with uv sync (creates .venv and installs from pyproject.toml)
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --frozen --no-install-project || uv sync --no-install-project

# Use zsh as default shell in the container
ENV SHELL=/bin/zsh

# Install Starship and configure zsh
RUN curl -sS https://starship.rs/install.sh | sh -s -- -y && \
    TARGET_USER="root" && \
    if id -u vscode >/dev/null 2>&1; then TARGET_USER="vscode"; fi && \
    TARGET_HOME="$(getent passwd "$TARGET_USER" | cut -d: -f6)" && \
    sh -c "git clone --depth=1 https://github.com/zsh-users/zsh-syntax-highlighting.git ${TARGET_HOME}/.zsh/zsh-syntax-highlighting" && \
    sh -c "git clone --depth=1 https://github.com/zsh-users/zsh-autosuggestions ${TARGET_HOME}/.zsh/zsh-autosuggestions" && \
    echo 'source $HOME/.zsh/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh' >> "$TARGET_HOME/.zshrc" && \
    echo 'source $HOME/.zsh/zsh-autosuggestions/zsh-autosuggestions.zsh' >> "$TARGET_HOME/.zshrc" && \
    echo 'eval "$(starship init zsh)"' >> "$TARGET_HOME/.zshrc" && \
    chown "$TARGET_USER":"$TARGET_USER" "$TARGET_HOME/.zshrc" "$TARGET_HOME/.zsh" -R && \
    if command -v usermod >/dev/null 2>&1; then usermod -s /bin/zsh "$TARGET_USER" || true; fi

