# syntax=docker/dockerfile:1

FROM debian:bookworm-slim

# Set Hugo version (change as needed)
ARG HUGO_VERSION=0.155.3

# Install dependencies
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
    ca-certificates \
    curl \
    git \
    unzip \
    # Install PowerShell dependencies
    && apt-get install -y --no-install-recommends wget apt-transport-https software-properties-common \
    && rm -rf /var/lib/apt/lists/*

# Install PowerShell
RUN wget -q "https://packages.microsoft.com/config/debian/12/packages-microsoft-prod.deb" -O packages-microsoft-prod.deb \
    && dpkg -i packages-microsoft-prod.deb \
    && rm packages-microsoft-prod.deb \
    && apt-get update \
    && apt-get install -y --no-install-recommends powershell \
    && rm -rf /var/lib/apt/lists/*

# Download and install Hugo Extended
RUN curl -L -o hugo.tar.gz "https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_Linux-64bit.tar.gz" \
    && tar -xzf hugo.tar.gz -C /usr/local/bin hugo \
    && rm hugo.tar.gz

# Verify installation
RUN hugo version

# Create a non-root user and set permissions
RUN useradd -m -s /usr/bin/pwsh vscode \
    && mkdir -p /workspace \
    && chown -R vscode:vscode /workspace

# Set default workdir
WORKDIR /workspace

# Switch to non-root user
USER vscode