# Stage 1: Build Namecoin Core from source
FROM debian:bookworm-slim AS builder

ARG NAMECOIN_VERSION=nc30.2
ARG TARGETARCH

RUN apt-get update && apt-get install -y \
    autoconf \
    build-essential \
    ca-certificates \
    cmake \
    curl \
    git \
    libevent-dev \
    libboost-dev \
    libdb5.3++-dev \
    libsqlite3-dev \
    libzmq3-dev \
    pkg-config \
    pkgconf \
    python3 \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /build

# Clone and checkout the specific release tag
RUN git clone --depth 1 --branch ${NAMECOIN_VERSION} https://github.com/namecoin/namecoin-core.git

WORKDIR /build/namecoin-core

# Build Namecoin Core using CMake (nc30.x dropped autotools)
RUN cmake -B build \
    -DBUILD_GUI=OFF \
    -DBUILD_TESTS=OFF \
    -DBUILD_BENCH=OFF \
    -DENABLE_IPC=OFF \
    -DWITH_ZMQ=ON \
    -DCMAKE_BUILD_TYPE=Release

RUN cmake --build build -j$(nproc)

# Copy binaries manually (cmake --install fails due to upstream man page bug #565)
RUN mkdir -p /output/usr/local/bin && \
    find build/src -maxdepth 1 -type f \( -name 'namecoind' -o -name 'namecoin-cli' -o -name 'namecoin-tx' -o -name 'namecoin-wallet' -o -name 'bitcoin' \) -executable -exec cp {} /output/usr/local/bin/ \; && \
    ls -la /output/usr/local/bin/

# Stage 2: Runtime image
FROM debian:bookworm-slim

# Install runtime dependencies
RUN apt-get update && apt-get install -y \
    bash \
    ca-certificates \
    curl \
    libevent-2.1-7 \
    libdb5.3++ \
    libsqlite3-0 \
    libzmq5 \
    tini \
    jq \
    && rm -rf /var/lib/apt/lists/*

# Copy built binaries
COPY --from=builder /output/usr/local/bin/ /usr/local/bin/

# Ensure namecoin-prefixed binaries exist (upstream bug #565: 'bitcoin' not rebranded)
RUN if [ ! -f /usr/local/bin/namecoind ] && [ -f /usr/local/bin/bitcoin ]; then \
      ln -sf bitcoin /usr/local/bin/namecoind; \
    fi && \
    if [ ! -f /usr/local/bin/namecoin-cli ] && [ -f /usr/local/bin/bitcoin-cli ]; then \
      ln -sf bitcoin-cli /usr/local/bin/namecoin-cli; \
    fi

# Copy wrapper scripts
COPY docker_entrypoint.sh /usr/local/bin/
COPY check-rpc.sh /usr/local/bin/
COPY properties.sh /usr/local/bin/
COPY migrations.sh /usr/local/bin/
COPY reindex.sh /usr/local/bin/
COPY assets/compat/config_spec.yaml /mnt/assets/
COPY assets/compat/config_rules.yaml /mnt/assets/
COPY assets/compat/namecoin.conf.template /mnt/assets/

RUN chmod +x /usr/local/bin/docker_entrypoint.sh \
    /usr/local/bin/check-rpc.sh \
    /usr/local/bin/properties.sh \
    /usr/local/bin/migrations.sh \
    /usr/local/bin/reindex.sh

# Create data directory
RUN mkdir -p /root/.namecoin

EXPOSE 8334 8336

ENTRYPOINT ["tini", "--"]
CMD ["docker_entrypoint.sh"]
