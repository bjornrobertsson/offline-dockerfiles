# Offline Dev Container with code-server, common-utils and docker-in-docker
# Based on analysis of devcontainer features from coder/devcontainer-features and devcontainers/features

FROM ubuntu:22.04

# Prevent interactive prompts during build
ENV DEBIAN_FRONTEND=noninteractive

# Build arguments for versions (can be overridden at build time)
ARG CODE_SERVER_VERSION=4.22.1
ARG DOCKER_VERSION=24.0.7
ARG USERNAME=devcontainer
ARG USER_UID=1000
ARG USER_GID=1000

# Environment variables for features
ENV _REMOTE_USER=${USERNAME}
ENV _REMOTE_USER_HOME=/home/${USERNAME}

# =============================================================================
# COMMON-UTILS FEATURE IMPLEMENTATION
# =============================================================================

# Install essential packages that common-utils provides
RUN apt-get update -y && apt-get install -y --no-install-recommends \
    # Core utilities
    apt-utils \
    bash-completion \
    ca-certificates \
    curl \
    wget \
    rsync \
    unzip \
    bzip2 \
    xz-utils \
    zip \
    # Development tools
    git \
    nano \
    vim-tiny \
    less \
    jq \
    tree \
    htop \
    ncdu \
    man-db \
    manpages \
    manpages-dev \
    strace \
    # Network tools
    openssh-client \
    gnupg2 \
    dirmngr \
    iproute2 \
    procps \
    lsof \
    net-tools \
    psmisc \
    # System libraries
    libc6 \
    libgcc1 \
    libkrb5-3 \
    libgssapi-krb5-2 \
    libicu70 \
    libstdc++6 \
    zlib1g \
    locales \
    sudo \
    # Init system helpers
    init-system-helpers \
    # SSL libraries
    libssl3 \
    # Development dependencies
    build-essential \
    pkg-config \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Set up locale (required by common-utils)
RUN echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen \
    && locale-gen

# Create non-root user with sudo access (common-utils functionality)
RUN groupadd --gid ${USER_GID} ${USERNAME} \
    && useradd --uid ${USER_UID} --gid ${USER_GID} -m -s /bin/bash ${USERNAME} \
    && echo "${USERNAME} ALL=(root) NOPASSWD:ALL" > /etc/sudoers.d/${USERNAME} \
    && chmod 0440 /etc/sudoers.d/${USERNAME}

# Install and configure zsh + Oh My Zsh (common-utils functionality)
RUN apt-get update -y && apt-get install -y --no-install-recommends zsh \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

# Install Oh My Zsh for the user
USER ${USERNAME}
RUN sh -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)" "" --unattended

# Create custom theme directory and set up devcontainer theme
RUN mkdir -p ~/.oh-my-zsh/custom/themes

# Switch back to root for system-level installations
USER root

# Create systemctl shim (common-utils provides this)
RUN echo '#!/bin/sh\nset -e\nif [ -d "/run/systemd/system" ]; then\n    exec /bin/systemctl "$@"\nelse\n    echo "\n\"systemd\" is not running in this container due to its overhead.\nUse the \"service\" command to start services instead. e.g.: \n\nservice --status-all"\nfi' > /usr/local/bin/systemctl \
    && chmod +x /usr/local/bin/systemctl

# Set up PATH restoration (common-utils functionality)
RUN echo "export PATH=${PATH}" > /etc/profile.d/00-restore-env.sh \
    && chmod +x /etc/profile.d/00-restore-env.sh

# =============================================================================
# CODE-SERVER FEATURE IMPLEMENTATION
# =============================================================================

# Download and install specific version of code-server
RUN ARCH=$(uname -m) \
    && case ${ARCH} in \
        x86_64) ARCH="amd64" ;; \
        aarch64) ARCH="arm64" ;; \
        armv7l) ARCH="armv7l" ;; \
        *) echo "Unsupported architecture: ${ARCH}" && exit 1 ;; \
    esac \
    && cd /tmp \
    && wget -q https://github.com/coder/code-server/releases/download/v${CODE_SERVER_VERSION}/code-server-${CODE_SERVER_VERSION}-linux-${ARCH}.tar.gz \
    && tar -xzf code-server-${CODE_SERVER_VERSION}-linux-${ARCH}.tar.gz \
    && mv code-server-${CODE_SERVER_VERSION}-linux-${ARCH} /usr/local/lib/code-server \
    && ln -s /usr/local/lib/code-server/bin/code-server /usr/local/bin/code-server \
    && rm -rf /tmp/code-server-*

# Create code-server entrypoint script
RUN cat > /usr/local/bin/code-server-entrypoint <<EOF
#!/usr/bin/env bash
set -e

if [[ \$(whoami) != "${USERNAME}" ]]; then
    exec su ${USERNAME} -c /usr/local/bin/code-server-entrypoint
fi

# Default flags for code-server
FLAGS=()
FLAGS+=(--auth "\${CODE_SERVER_AUTH:-password}")
FLAGS+=(--bind-addr "\${CODE_SERVER_HOST:-127.0.0.1}:\${CODE_SERVER_PORT:-8080}")
FLAGS+=(--disable-telemetry)
FLAGS+=(--disable-update-check)

# Add password file if specified
if [[ -f "\$CODE_SERVER_PASSWORD_FILE" ]]; then
    export PASSWORD="\$(< "\$CODE_SERVER_PASSWORD_FILE")"
fi

# Add hashed password file if specified
if [[ -f "\$CODE_SERVER_HASHED_PASSWORD_FILE" ]]; then
    export HASHED_PASSWORD="\$(< "\$CODE_SERVER_HASHED_PASSWORD_FILE")"
fi

# Start code-server
exec code-server "\${FLAGS[@]}" "\${CODE_SERVER_WORKSPACE:-/home/${USERNAME}}" > "\${CODE_SERVER_LOG_FILE:-/tmp/code-server.log}" 2>&1
EOF

RUN chmod +x /usr/local/bin/code-server-entrypoint

# Create a simple 'code' command that opens files in code-server
RUN echo '#!/bin/sh\necho "Use code-server in your browser instead of the code command"\necho "Files can be opened directly in the code-server interface"' > /usr/local/bin/code \
    && chmod +rx /usr/local/bin/code

# =============================================================================
# DOCKER-IN-DOCKER FEATURE IMPLEMENTATION
# =============================================================================

# Install Docker dependencies
RUN apt-get update -y && apt-get install -y --no-install-recommends \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release \
    iptables \
    supervisor \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

# Download and install Docker binaries directly (for specific version control)
RUN cd /tmp \
    && wget -q https://download.docker.com/linux/static/stable/x86_64/docker-${DOCKER_VERSION}.tgz \
    && tar -xzf docker-${DOCKER_VERSION}.tgz \
    && mv docker/* /usr/local/bin/ \
    && rm -rf /tmp/docker*

# Add user to docker group
RUN groupadd docker \
    && usermod -aG docker ${USERNAME}

# Download and install Docker Compose v2
RUN COMPOSE_VERSION=$(curl -s https://api.github.com/repos/docker/compose/releases/latest | jq -r .tag_name) \
    && curl -L "https://github.com/docker/compose/releases/download/${COMPOSE_VERSION}/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose \
    && chmod +x /usr/local/bin/docker-compose

# Create docker daemon configuration
RUN mkdir -p /etc/docker \
    && echo '{\n    "storage-driver": "overlay2",\n    "hosts": ["unix:///var/run/docker.sock"],\n    "log-driver": "json-file",\n    "log-opts": {\n        "max-size": "10m",\n        "max-file": "3"\n    }\n}' > /etc/docker/daemon.json

# Create Docker initialization script
RUN cat > /usr/local/share/docker-init.sh <<'EOF'
#!/bin/bash
set -e

# Start Docker daemon in background
if [ ! -S "/var/run/docker.sock" ]; then
    echo "Starting Docker daemon..."
    (
        dockerd \
            --host=unix:///var/run/docker.sock \
            --host=tcp://127.0.0.1:2375 \
            --storage-driver=overlay2 \
            > /tmp/dockerd.log 2>&1
    ) &
    
    # Wait for Docker daemon to be ready
    timeout=60
    while [ $timeout -gt 0 ] && ! docker info >/dev/null 2>&1; do
        sleep 1
        ((timeout--))
    done
    
    if [ $timeout -eq 0 ]; then
        echo "Docker daemon failed to start within 60 seconds"
        exit 1
    fi
    
    echo "Docker daemon started successfully"
else
    echo "Docker daemon is already running"
fi
EOF

RUN chmod +x /usr/local/share/docker-init.sh

# =============================================================================
# FINAL CONFIGURATION
# =============================================================================

# Set up environment for the user
USER ${USERNAME}
WORKDIR /home/${USERNAME}

# Create user directories
RUN mkdir -p ~/.config ~/.local/share ~/.cache

# Create a startup script that initializes Docker and starts code-server
USER root
RUN cat > /usr/local/bin/start-devcontainer <<EOF
#!/bin/bash
set -e

# Initialize Docker daemon
/usr/local/share/docker-init.sh

# Set default environment variables
export CODE_SERVER_AUTH="\${CODE_SERVER_AUTH:-password}"
export CODE_SERVER_HOST="\${CODE_SERVER_HOST:-0.0.0.0}"
export CODE_SERVER_PORT="\${CODE_SERVER_PORT:-8080}"
export CODE_SERVER_WORKSPACE="\${CODE_SERVER_WORKSPACE:-/home/${USERNAME}}"

echo "========================"
echo "Dev Container Started!"
echo "========================"
echo "code-server: http://localhost:\${CODE_SERVER_PORT}"
echo "Docker: Available via docker command"
echo "User: ${USERNAME}"
echo "========================"

# Start code-server (this will run as the specified user)
exec /usr/local/bin/code-server-entrypoint
EOF

RUN chmod +x /usr/local/bin/start-devcontainer

# Set default working directory and user
WORKDIR /home/${USERNAME}
USER ${USERNAME}

# Expose code-server port
EXPOSE 8080

# Default command
CMD ["/usr/local/bin/start-devcontainer"]

# Add labels for documentation
LABEL org.opencontainers.image.title="Offline Dev Container"
LABEL org.opencontainers.image.description="Self-contained dev container with code-server, Docker-in-Docker, and common utilities"
LABEL org.opencontainers.image.version="1.0.0"
LABEL devcontainer.features.code-server="${CODE_SERVER_VERSION}"
LABEL devcontainer.features.docker="${DOCKER_VERSION}"
LABEL devcontainer.features.common-utils="2.5.4"
