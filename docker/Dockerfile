# Run from the root of the project

# Use the rust image as the base image for the build stage
FROM rust:latest AS base
# buildkit will provide this automatically, no need to pass it in
ARG TARGETARCH

FROM base AS builder-amd64
ENV TARGET_RUST_ARCH="x86_64-unknown-linux-musl"

FROM base AS builder-arm64
ENV TARGET_RUST_ARCH="aarch64-unknown-linux-musl"
RUN apt-get update \
    && apt-get install clang llvm -y --no-install-recommends \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*
ENV CC_aarch64_unknown_linux_musl=clang
ENV AR_aarch64_unknown_linux_musl=llvm-ar
ENV CARGO_TARGET_AARCH64_UNKNOWN_LINUX_MUSL_RUSTFLAGS="-Clink-self-contained=yes -Clinker=rust-lld"

FROM builder-${TARGETARCH} AS builder
RUN echo "Building for architecture: ${TARGET_RUST_ARCH}"
# Copy the source code
COPY . /sim-ln

# Install the dependencies required for building sim-ln
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
       protobuf-compiler musl-tools build-essential perl \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Set to 1 to instruct openssl-sys to link OpenSSL statically, ensuring a fully static binary
ENV OPENSSL_STATIC=1
# Set to 1 to enable vendoring, making openssl-sys build OpenSSL from source instead of using system libraries
ENV OPENSSL_VENDORED=1

RUN rustup target add ${TARGET_RUST_ARCH}
RUN rustup component add rustfmt
RUN cd /sim-ln && RUSTFLAGS='-C target-feature=+crt-static' cargo build --locked --release --target=${TARGET_RUST_ARCH}
RUN mv /sim-ln/target/${TARGET_RUST_ARCH}/release/sim-cli /sim-ln/sim-cli

# Use a new stage with a smaller base image to reduce image size
FROM alpine:latest

RUN apk update && apk upgrade

# Copy the sim-cli binaries from the build stage to the new stage
COPY --from=builder /sim-ln/sim-cli /usr/local/bin/

# Copy the entrypoint script to the container
COPY docker/entrypoint.sh /entrypoint.sh

RUN chmod +x entrypoint.sh

ENTRYPOINT [ "/entrypoint.sh" ]