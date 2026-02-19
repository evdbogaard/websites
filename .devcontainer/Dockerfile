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
    && rm -rf /var/lib/apt/lists/*

# Download and install Hugo Extended
RUN curl -L -o hugo.tar.gz "https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_Linux-64bit.tar.gz" \
    && tar -xzf hugo.tar.gz -C /usr/local/bin hugo \
    && rm hugo.tar.gz

# Verify installation
RUN hugo version

# Set default workdir
WORKDIR /workspace
