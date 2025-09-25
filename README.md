# Offline Dev Container with Code-Server

A self-contained development container that combines code-server (VS Code in the browser), Docker-in-Docker, and essential development utilities without relying on external devcontainer features.

## 🎯 Purpose

This Dockerfile creates a fully offline-capable development environment that includes:

- **Code-Server**: VS Code running in a web browser for remote development
- **Docker-in-Docker**: Full Docker daemon and CLI tools for containerized development
- **Essential Development Tools**: Common utilities typically provided by devcontainer features
- **Zero External Dependencies**: All functionality implemented directly in the Dockerfile

## 🏗️ Architecture

### Self-Contained Design

Instead of relying on external devcontainer features, this container implements all functionality directly:

```
┌─────────────────────────────────────────────────────────────┐
│                    Ubuntu 22.04 Base                       │
├─────────────────────────────────────────────────────────────┤
│  Common-Utils Implementation (Self-Contained)              │
│  • Essential packages (git, curl, wget, etc.)              │
│  • Non-root user with sudo access                          │
│  • Zsh + Oh My Zsh configuration                           │
│  • Locale and PATH configuration                           │
├─────────────────────────────────────────────────────────────┤
│  Code-Server Implementation (Self-Contained)               │
│  • Architecture-aware binary download                      │
│  • Complete directory structure preservation               │
│  • Custom entrypoint scripts                               │
├─────────────────────────────────────────────────────────────┤
│  Docker-in-Docker Implementation (Self-Contained)          │
│  • Docker daemon and CLI tools                             │
│  • Docker Compose v2                                       │
│  • Initialization and management scripts                   │
└─────────────────────────────────────────────────────────────┘
```

## 🚀 Features

### Code-Server
- **Version**: 4.22.1 (configurable via build args)
- **Architecture Support**: AMD64, ARM64, ARMv7l
- **Web Access**: Available on port 8080
- **Authentication**: Password-based (configurable)
- **Extensions**: Full VS Code extension support via Open VSX

### Docker-in-Docker
- **Docker Version**: 24.0.7 (configurable via build args)
- **Docker Compose**: Latest v2 release
- **Storage Driver**: Overlay2 for optimal performance
- **Daemon Management**: Automatic startup and health checking

### Development Environment
- **User**: `devcontainer` (UID/GID 1000)
- **Shell**: Bash and Zsh with Oh My Zsh
- **Sudo Access**: Passwordless sudo for the devcontainer user
- **Essential Tools**: git, curl, wget, jq, tree, htop, and more

## 🔧 Build Configuration

### Build Arguments

| Argument | Default | Description |
|----------|---------|-------------|
| `CODE_SERVER_VERSION` | `4.22.1` | Code-server version to install |
| `DOCKER_VERSION` | `24.0.7` | Docker version to install |
| `USERNAME` | `devcontainer` | Non-root user name |
| `USER_UID` | `1000` | User ID |
| `USER_GID` | `1000` | Group ID |

### Build Command

```bash
# Default build
docker build -t ubuntu-offline:22.04 offline

# Custom build with different versions
docker build \
  --build-arg CODE_SERVER_VERSION=4.23.0 \
  --build-arg DOCKER_VERSION=24.0.8 \
  --build-arg USERNAME=developer \
  -t ubuntu-offline:custom \
  offline
```

## 🚀 Usage

### Quick Start

```bash
# Run the container
docker run -d \
  --name dev-container \
  --privileged \
  -p 8080:8080 \
  -v "$(pwd):/home/devcontainer/workspace" \
  ubuntu-offline:22.04

# Access code-server in your browser
open http://localhost:8080
```

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `CODE_SERVER_AUTH` | `password` | Authentication method (password/none) |
| `CODE_SERVER_HOST` | `0.0.0.0` | Bind address |
| `CODE_SERVER_PORT` | `8080` | Port number |
| `CODE_SERVER_WORKSPACE` | `/home/devcontainer` | Default workspace |
| `CODE_SERVER_PASSWORD_FILE` | - | Path to password file |
| `CODE_SERVER_LOG_FILE` | `/tmp/code-server.log` | Log file location |

### Advanced Usage

```bash
# Run with custom configuration
docker run -d \
  --name dev-container \
  --privileged \
  -p 8080:8080 \
  -e CODE_SERVER_AUTH=none \
  -e CODE_SERVER_WORKSPACE=/workspace \
  -v "$(pwd):/workspace" \
  -v "/var/run/docker.sock:/var/run/docker.sock" \
  ubuntu-offline:22.04

# Interactive shell access
docker exec -it dev-container bash

# View logs
docker logs dev-container
```

## 🔒 Security Considerations

### Privileged Mode
This container requires `--privileged` mode for Docker-in-Docker functionality. Consider these alternatives for enhanced security:

```bash
# Option 1: Mount Docker socket (less isolated)
docker run -d \
  -v /var/run/docker.sock:/var/run/docker.sock \
  ubuntu-offline:22.04

# Option 2: Use sysbox runtime (requires sysbox installation)
docker run -d \
  --runtime=sysbox-runc \
  ubuntu-offline:22.04
```

### Network Security
- Code-server binds to `0.0.0.0` by default for container access
- Use `CODE_SERVER_HOST=127.0.0.1` for localhost-only access
- Consider using reverse proxy with SSL termination for production

## 🛠️ Customization

### Adding Development Tools

```dockerfile
# Add to the common-utils section
RUN apt-get update && apt-get install -y \
    python3 \
    python3-pip \
    nodejs \
    npm \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*
```

### Custom Code-Server Extensions

```bash
# Install extensions via command line
docker exec dev-container code-server \
  --install-extension ms-python.python \
  --install-extension ms-vscode.vscode-typescript-next
```

### Persistent Configuration

```bash
# Mount configuration directories
docker run -d \
  -v code-server-config:/home/devcontainer/.config \
  -v code-server-data:/home/devcontainer/.local/share \
  ubuntu-offline:22.04
```

## 🔍 Troubleshooting

### Common Issues

#### Code-Server Not Starting
```bash
# Check logs
docker logs dev-container

# Check code-server process
docker exec dev-container ps aux | grep code-server

# Manual start for debugging
docker exec -it dev-container /usr/local/bin/code-server-entrypoint
```

#### Docker Daemon Issues
```bash
# Check Docker daemon status
docker exec dev-container docker info

# Restart Docker daemon
docker exec dev-container /usr/local/share/docker-init.sh

# Check Docker logs
docker exec dev-container cat /tmp/dockerd.log
```

#### Permission Issues
```bash
# Check user context
docker exec dev-container whoami
docker exec dev-container id

# Fix ownership if needed
docker exec dev-container sudo chown -R devcontainer:devcontainer /home/devcontainer
```

### Performance Optimization

```bash
# Increase memory limit
docker run -d \
  --memory=4g \
  --cpus=2 \
  ubuntu-offline:22.04

# Use tmpfs for temporary files
docker run -d \
  --tmpfs /tmp:rw,noexec,nosuid,size=1g \
  ubuntu-offline:22.04
```

## 📁 Directory Structure

```
/usr/local/
├── bin/
│   ├── code-server              # Symlink to code-server binary
│   ├── code-server-entrypoint   # Code-server startup script
│   ├── start-devcontainer       # Main container entrypoint
│   ├── systemctl                # Systemctl shim
│   └── docker*                  # Docker CLI tools
├── lib/
│   └── code-server/             # Complete code-server installation
│       ├── bin/code-server      # Actual code-server binary
│       ├── lib/node             # Node.js runtime
│       └── lib/vscode/          # VS Code libraries
└── share/
    └── docker-init.sh           # Docker daemon initialization
```

## 🤝 Contributing

### Development Setup

```bash
# Clone the repository
git clone <repository-url>
cd Prebuilt-Image

# Build and test
docker build -t ubuntu-offline:dev offline
docker run --rm -it ubuntu-offline:dev bash

# Run tests
docker run --rm ubuntu-offline:dev code-server --version
```

### Making Changes

1. **Modify the Dockerfile** in the `offline/` directory
2. **Test the build** with different architectures if possible
3. **Update documentation** including this README
4. **Test functionality** including code-server and Docker-in-Docker

## 📋 Comparison with Devcontainer Features

| Feature | This Implementation | External Features |
|---------|-------------------|-------------------|
| **Common-Utils** | ✅ Built-in | ❌ External dependency |
| **Code-Server** | ✅ Built-in | ❌ External dependency |
| **Docker-in-Docker** | ✅ Built-in | ❌ External dependency |
| **Offline Capability** | ✅ Fully offline | ❌ Requires internet |
| **Customization** | ✅ Full control | ⚠️ Limited |
| **Maintenance** | ⚠️ Manual updates | ✅ Automatic |
| **Size** | ⚠️ Larger | ✅ Smaller |

## 📄 License

This project is provided as-is for educational and development purposes. Please ensure compliance with the licenses of included software:

- **Code-Server**: MIT License
- **Docker**: Apache License 2.0
- **Ubuntu**: Various open-source licenses
- **Oh My Zsh**: MIT License

## 🔗 References

- [Code-Server Documentation](https://coder.com/docs/code-server)
- [Docker-in-Docker Best Practices](https://docs.docker.com/engine/security/userns-remap/)
- [Devcontainer Features](https://containers.dev/features)
- [Ubuntu 22.04 Documentation](https://releases.ubuntu.com/22.04/)