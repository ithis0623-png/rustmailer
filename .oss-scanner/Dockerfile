FROM rust:1-bookworm
ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y --no-install-recommends \
    clang \
    cmake \
    nasm \
    pkg-config \
    protobuf-compiler \
    libssl-dev \
    nodejs \
    npm \
    python3 \
    unzip \
    curl \
    git \
    && rm -rf /var/lib/apt/lists/*

RUN npm install -g pnpm

COPY . /src
WORKDIR /src

# Pre-fetch all crate dependencies
RUN cargo fetch

# Build web frontend if needed
RUN if [ -d "web" ] && [ -f "web/package.json" ]; then cd web && pnpm install --frozen-lockfile && pnpm run build && cd ..; fi

# Pre-compile the Rust project and tests with debug symbols
RUN cargo build --bin rustmailer
RUN cargo test --no-run

# Run smoke tests
RUN (timeout 300 cargo test || echo "WARNING: baseline tests reported issues")

# Enforce offline mode for the audit phase
ENV CARGO_NET_OFFLINE=true
