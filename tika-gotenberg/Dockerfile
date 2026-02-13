# --------------------------------------------------------------
# Build arguments
# --------------------------------------------------------------
ARG GOLANG_VERSION=1.25.5
ARG PDFCPU_VERSION=v0.11.1
ARG GOTENBERG_VERSION=v8.26.0
ARG TIKA_VERSION=3.2.3
ARG NOTO_COLOR_EMOJI_VERSION=main
#ARG S6_OVERLAY_VERSION=v3.2.2.0

# --------------------------------------------------------------
# PDFCPU build stage
# --------------------------------------------------------------
FROM golang:${GOLANG_VERSION} AS pdfcpu-builder
ARG PDFCPU_VERSION
ENV CGO_ENABLED=0
WORKDIR /build

RUN curl -fSL https://github.com/pdfcpu/pdfcpu/archive/refs/tags/${PDFCPU_VERSION}.tar.gz -o pdfcpu.tar.gz && \
    tar -xzf pdfcpu.tar.gz --strip-components=1 && rm pdfcpu.tar.gz && \
    cd cmd/pdfcpu && \
    go mod download && \
    go build -o /pdfcpu \
      -ldflags "-s -w -X main.version=${PDFCPU_VERSION} -X github.com/pdfcpu/pdfcpu/pkg/pdfcpu.VersionStr=${PDFCPU_VERSION} -X main.builtBy=gotenberg"

# --------------------------------------------------------------
# Gotenberg build stage
# --------------------------------------------------------------
FROM golang:${GOLANG_VERSION} AS gotenberg-builder
ARG GOTENBERG_VERSION
ENV CGO_ENABLED=0
WORKDIR /build

RUN curl -fSL https://github.com/gotenberg/gotenberg/archive/refs/tags/${GOTENBERG_VERSION}.tar.gz -o gotenberg.tar.gz && \
    tar -xzf gotenberg.tar.gz --strip-components=1 && rm gotenberg.tar.gz && \
    go mod download && \
    go build -o gotenberg -ldflags "-X github.com/gotenberg/gotenberg/v8/cmd.Version=${GOTENBERG_VERSION}" cmd/gotenberg/main.go

# --------------------------------------------------------------
# Tika fetch stage
# --------------------------------------------------------------
FROM debian:bookworm-slim AS tika-fetch
ARG TIKA_VERSION

RUN apt-get update && \
    apt-get install -y --no-install-recommends curl ca-certificates && \
    curl -Ls https://downloads.apache.org/tika/${TIKA_VERSION}/tika-server-standard-${TIKA_VERSION}.jar -o /tika-server.jar && \
    rm -rf /var/lib/apt/lists/*

# --------------------------------------------------------------
# Final minimal runtime
# --------------------------------------------------------------
FROM debian:bookworm-slim AS runtime
ENV DEBIAN_FRONTEND=noninteractive

# Install only runtime dependencies
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
      ca-certificates curl default-jre-headless libreoffice unoconv qpdf \
      pdftk-java libimage-exiftool-perl fonts-dejavu fonts-dejavu-extra \
      fonts-liberation fonts-liberation2 fonts-noto-core fonts-noto-ui-core \
      fonts-noto-mono fonts-noto-cjk cabextract gnupg python3 fontconfig \
      chromium xz-utils && \
    rm -rf /var/lib/apt/lists/*

# Microsoft fonts
RUN curl -o /tmp/ttf-mscorefonts-installer_3.8.1_all.deb \
      http://httpredir.debian.org/debian/pool/contrib/m/msttcorefonts/ttf-mscorefonts-installer_3.8.1_all.deb && \
    apt-get update && apt-get install -y --no-install-recommends /tmp/ttf-mscorefonts-installer_3.8.1_all.deb && \
    rm -f /tmp/ttf-mscorefonts-installer_3.8.1_all.deb && rm -rf /var/lib/apt/lists/*

# Noto Color Emoji
RUN curl -Ls "https://github.com/googlefonts/noto-emoji/raw/${NOTO_COLOR_EMOJI_VERSION}/fonts/NotoColorEmoji.ttf" -o /usr/local/share/fonts/NotoColorEmoji.ttf

# Symlink python for unoconv
RUN ln -s /usr/bin/python3 /usr/bin/python || true

ARG S6_OVERLAY_VERSION=v3.2.2.0

# --------------------------------------------------------------
# S6 Overlay
# --------------------------------------------------------------
RUN curl -L -o /tmp/s6-overlay-x86_64.tar.xz \
    https://github.com/just-containers/s6-overlay/releases/download/${S6_OVERLAY_VERSION}/s6-overlay-x86_64.tar.xz && \
    tar -xJf /tmp/s6-overlay-x86_64.tar.xz -C / --strip-components=1 && \
    rm /tmp/s6-overlay-x86_64.tar.xz

# Runtime user
RUN useradd -m -u 1000 gotenberg

# Wrapper scripts
RUN echo '#!/bin/bash\nexec java -jar /usr/local/bin/tika-server-standard.jar "$@"' > /usr/local/bin/tika && \
    chmod +x /usr/local/bin/tika && \
    echo '#!/bin/bash\nexec java -jar /usr/local/bin/pdftk-all.jar "$@"' > /usr/local/bin/pdftk && \
    chmod +x /usr/local/bin/pdftk

# Switch to non-root
USER gotenberg

# Copy built binaries only
COPY --from=pdfcpu-builder /pdfcpu /usr/local/bin/pdfcpu
COPY --from=gotenberg-builder /build/gotenberg /usr/local/bin/gotenberg
COPY --from=tika-fetch /tika-server.jar /usr/local/bin/tika-server-standard.jar

# Optional: your HA rootfs
COPY rootfs/ /

# Environment
ENV CHROMIUM_BIN_PATH=/usr/bin/chromium \
    LIBREOFFICE_BIN_PATH=/usr/lib/libreoffice/program/soffice.bin \
    UNOCONVERTER_BIN_PATH=/usr/bin/unoconv \
    PDFTK_BIN_PATH=/usr/bin/pdftk \
    QPDF_BIN_PATH=/usr/bin/qpdf \
    EXIFTOOL_BIN_PATH=/usr/bin/exiftool \
    PDFCPU_BIN_PATH=/usr/local/bin/pdfcpu \
    CHROMIUM_DISABLE_SANDBOX=true \
    CHROMIUM_HYPHEN_DATA_DIR_PATH=/tmp/chromium

EXPOSE 3000 9998

ENTRYPOINT ["/command/s6-rc-init"]
CMD []
