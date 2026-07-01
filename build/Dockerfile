FROM rust:1.92-slim-bookworm AS toolchain

ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y --no-install-recommends \
      curl xz-utils ca-certificates \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

RUN rustup target add wasm32-wasip1

ARG WASI_SDK_VERSION=33
RUN set -eux; \
    case "$(uname -m)" in \
      x86_64)        WS=x86_64-linux ;; \
      aarch64|arm64) WS=arm64-linux ;; \
      *) echo "unsupported arch $(uname -m)"; exit 1 ;; \
    esac; \
    curl -fsSL -o /tmp/wasi-sdk.tar.gz \
      "https://github.com/WebAssembly/wasi-sdk/releases/download/wasi-sdk-${WASI_SDK_VERSION}/wasi-sdk-${WASI_SDK_VERSION}.0-${WS}.tar.gz"; \
    mkdir -p /opt/wasi-sdk; \
    tar -xzf /tmp/wasi-sdk.tar.gz -C /opt/wasi-sdk --strip-components=1; \
    rm /tmp/wasi-sdk.tar.gz; \
    /opt/wasi-sdk/bin/clang --version
ENV WASI_SDK_PATH=/opt/wasi-sdk
ENV CC_wasm32_wasip1=/opt/wasi-sdk/bin/clang \
    AR_wasm32_wasip1=/opt/wasi-sdk/bin/llvm-ar \
    CFLAGS_wasm32_wasip1="--target=wasm32-wasip1 --sysroot=/opt/wasi-sdk/share/wasi-sysroot -mllvm -wasm-enable-sjlj -mllvm -wasm-use-legacy-eh=false"

ARG WASMTIME_VERSION=45.0.0
RUN set -eux; \
    case "$(uname -m)" in \
      x86_64)        WT=x86_64-linux ;; \
      aarch64|arm64) WT=aarch64-linux ;; \
      *) echo "unsupported arch $(uname -m)"; exit 1 ;; \
    esac; \
    curl -fsSL -o /tmp/wasmtime.tar.xz \
      "https://github.com/bytecodealliance/wasmtime/releases/download/v${WASMTIME_VERSION}/wasmtime-v${WASMTIME_VERSION}-${WT}.tar.xz"; \
    tar -xJf /tmp/wasmtime.tar.xz -C /tmp; \
    cp "/tmp/wasmtime-v${WASMTIME_VERSION}-${WT}/wasmtime" /usr/local/bin/; \
    rm -rf /tmp/wasmtime*; \
    wasmtime --version

FROM toolchain AS builder
WORKDIR /src

ARG PNGQUANT_VERSION=3.0.3
ARG PNGQUANT_SHA256=68a12bdd8825f9989f4ee9a6ab0b42727dae57728b939ef63453366697a07232
ENV PNGQUANT_VERSION=${PNGQUANT_VERSION}

RUN set -eux; \
    curl -fsSL -o /tmp/pngquant.crate \
      "https://static.crates.io/crates/pngquant/pngquant-${PNGQUANT_VERSION}.crate"; \
    echo "${PNGQUANT_SHA256}  /tmp/pngquant.crate" | sha256sum -c -; \
    tar -xzf /tmp/pngquant.crate -C /src; \
    rm /tmp/pngquant.crate; \
    test -f "/src/pngquant-${PNGQUANT_VERSION}/rust/bin.rs"

RUN set -eux; \
    f="/src/pngquant-${PNGQUANT_VERSION}/Cargo.toml"; \
    grep -q '^\[dependencies.imagequant-sys\]' "$f"; \
    sed -i 's|^\(\[dependencies.imagequant-sys\]\)|\1\ndefault-features = false|' "$f"; \
    grep -q 'default-features = false' "$f" || { echo "threads patch did not apply" >&2; exit 1; }

COPY Cargo.lock /src/pngquant-${PNGQUANT_VERSION}/Cargo.lock

ENV RUSTFLAGS="-L /opt/wasi-sdk/share/wasi-sysroot/lib/wasm32-wasip1 -l static=setjmp"
RUN set -eux; \
    cd "/src/pngquant-${PNGQUANT_VERSION}"; \
    cargo build --release --locked --target wasm32-wasip1 \
      --no-default-features --features png-static,z-static; \
    cp "target/wasm32-wasip1/release/pngquant.wasm" /tmp/pngquant.raw.wasm; \
    ls -lh /tmp/pngquant.raw.wasm

RUN set -eux; \
    mkdir -p /build; \
    cp /tmp/pngquant.raw.wasm /build/pngquant.wasm; \
    ls -lh /build/pngquant.wasm

RUN set -eux; \
    [ "$(head -c4 /build/pngquant.wasm | od -An -tx1 | tr -d ' \n')" = "0061736d" ]; \
    [ "$(stat -c%s /build/pngquant.wasm)" -gt 262144 ]

COPY test/fixtures/sample.png /smoke/in.png
RUN set -eux; \
    S=/smoke; \
    SIZE_IN="$(stat -c%s "$S/in.png")"; \
    [ "$(head -c8 "$S/in.png" | od -An -tx1 | tr -d ' \n')" = "89504e470d0a1a0a" ] || { echo "fixture is not a PNG" >&2; exit 1; }; \
    wasmtime run -W exceptions=y --dir "$S::/work" /build/pngquant.wasm --force --output /work/out.png 16 /work/in.png; \
    [ -f "$S/out.png" ]; \
    [ "$(head -c8 "$S/out.png" | od -An -tx1 | tr -d ' \n')" = "89504e470d0a1a0a" ] || { echo "output is not a PNG" >&2; exit 1; }; \
    grep -q PLTE "$S/out.png" || { echo "output has no PLTE palette chunk" >&2; exit 1; }; \
    SIZE_OUT="$(stat -c%s "$S/out.png")"; \
    echo "in: ${SIZE_IN} bytes  out: ${SIZE_OUT} bytes"; \
    [ "$SIZE_OUT" -lt "$SIZE_IN" ] || { echo "quantized output not smaller than truecolor input" >&2; exit 1; }; \
    OUT="$(wasmtime run -W exceptions=y /build/pngquant.wasm --version)"; \
    echo "version: [$OUT]"; case "$OUT" in ${PNGQUANT_VERSION}*) : ;; *) echo "bad --version" >&2; exit 1 ;; esac

COPY js ./js
RUN set -eux; \
    mkdir -p /dist; \
    cp /build/pngquant.wasm /dist/pngquant.wasm; \
    cp js/pngquant.mjs      /dist/pngquant.mjs; \
    printf '%s\n' "${PNGQUANT_VERSION}" > /dist/PNGQUANT_VERSION; \
    [ "$(head -c4 /dist/pngquant.wasm | od -An -tx1 | tr -d ' \n')" = "0061736d" ] || { echo "bad magic" >&2; exit 1; }; \
    [ "$(stat -c%s /dist/pngquant.wasm)" -gt 262144 ] || { echo "too small" >&2; exit 1; }; \
    echo "=== dist ==="; ls -lh /dist

FROM scratch AS exportor
COPY --from=builder /dist /dist
