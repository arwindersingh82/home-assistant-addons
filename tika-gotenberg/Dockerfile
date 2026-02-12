# ----------------------------------------------
# Build arguments
# ----------------------------------------------
ARG BUILD_FROM=ghcr.io/hassio-addons/debian-base:9.2.0
ARG GOLANG_VERSION="1.25.5"
ARG PDFCPU_VERSION="v0.9.0"
ARG GOTENBERG_VERSION="v8.26.0"
ARG TIKA_VERSION="3.2.0"

# ----------------------------------------------
# pdfcpu binary build stage
# ----------------------------------------------
FROM golang:${GOLANG_VERSION} AS pdfcpu-binary-stage

ARG PDFCPU_VERSION
ENV CGO_ENABLED=0
ENV PDFCPU_VERSION=${PDFCPU_VERSION}
WORKDIR /home

# Download and build pdfcpu
RUN curl -Ls "https://github.com/pdfcpu/pdfcpu/archive/refs/tags/${PDFCPU_VERSION}.tar.gz" -o pdfcpu.tar.gz && \
    tar --strip-components=1 -xvzf pdfcpu.tar.gz && \
    GOPROXY=direct go mod download && \
    GOPROXY=direct go mod verify && \
    go build -o pdfcpu -ldflags "-s -w -X 'main.version=${PDFCPU_VERSION}' -X 'github.com/pdfcpu/pdfcpu/pkg/pdfcpu.VersionStr=${PDFCPU_VERSION}' -X main.builtBy=gotenberg" ./cmd/pdfcpu && \
    ./pdfcpu version

# ----------------------------------------------
# Gotenberg binary build stage
# ----------------------------------------------
FROM golang:${GOLANG_VERSION} AS gotenberg-binary-stage

ARG GOTENBERG_VERSION
ENV CGO_ENABLED=0
ENV GOTENBERG_VERSION=${GOTENBERG_VERSION}
WORKDIR /home

# Download Gotenberg source
RUN curl -Ls "https://github.com/gotenberg/gotenberg/archive/refs/tags/${GOTENBERG_VERSION}.tar.gz" -o gotenberg.tar.gz && \
    tar --strip-components=1 -xvzf gotenberg.tar.gz

# Pre-fetch dependencies separately to avoid network issues mid-build
RUN GOPROXY=direct go mod download && GOPROXY=direct go mod verify

# Build gotenberg binary
RUN go build -o gotenberg -ldflags "-X 'github.com/gotenberg/gotenberg/v8/cmd.Version=${GOTENBERG_VERSION}'" cmd/gotenberg/main.go

# ----------------------------------------------
# Fetch Apache Tika stage
# ----------------------------------------------
FROM debian:bookworm AS fetch_tika

ARG TIKA_VERSION
ENV NEAREST_TIKA_SERVER_URL="https://dlcdn.apache.org/tika/${TIKA_VERSION}/tika-server-standard-${TIKA_VERSION}.jar" \
    ARCHIVE_TIKA_SERVER_URL="https://archive.apache.org/dist/tika/${TIKA_VERSION}/tika-server-standard-${TIKA_VERSION}.jar" \
    BACKUP_TIKA_SERVER_URL="https://downloads.apache.org/tika/${TIKA_VERSION}/tika-server-standard-${TIKA_VERSION}.jar" \
    DEFAULT_TIKA_SERVER_ASC_URL="https://downloads.apache.org/tika/${TIKA_VERSION}/tika-server-standard-${TIKA_VERSION}.jar.asc" \
    ARCHIVE_TIKA_SERVER_ASC_URL="https://archive.apache.org/dist/tika/${TIKA_VERSION}/tika-server-standard-${TIKA_VERSION}.jar.asc"

SHELL ["/bin/bash", "-o", "pipefail", "-c"]

RUN apt-get update && \
    apt-get install -y --no-install-recommends gnupg wget ca-certificates curl && \
    rm -rf /var/lib/apt/lists/* && \
    curl -sSL https://downloads.apache.org/tika/KEYS | gpg --import && \
    wget -O /tika-server-standard.jar $NEAREST_TIKA_SERVER_URL || \
    wget -O /tika-server-standard.jar $ARCHIVE_TIKA_SERVER_URL || \
    wget -O /tika-server-standard.jar $BACKUP_TIKA_SERVER_URL || exit 1 && \
    wget -O /tika-server-standard.jar.asc $DEFAULT_TIKA_SERVER_ASC_URL || \
    wget -O /tika-server-standard.jar.asc $ARCHIVE_TIKA_SERVER_ASC_URL || exit 1 && \
    gpg --verify /tika-server-standard.jar.asc /tika-server-standard.jar

# ----------------------------------------------
# Final stage
# ----------------------------------------------
FROM ${BUILD_FROM}

ARG GOTENBERG_VERSION
ARG PDFCPU_VERSION
ARG GOTENBERG_USER_GID
ARG GOTENBERG_USER_UID
ARG NOTO_COLOR_EMOJI_VERSION
ARG PDFTK_VERSION="v3.3.3"

# Copy root filesystem
COPY rootfs/ /

# Create gotenberg user
RUN groupadd --gid 1000 gotenberg && \
    useradd --uid 1000 --gid gotenberg --shell /bin/bash --home /home/gotenberg --no-create-home gotenberg && \
    mkdir /home/gotenberg && chown gotenberg: /home/gotenberg

# Install base dependencies
RUN apt-get update -qq && \
    apt-get upgrade -yqq && \
    DEBIAN_FRONTEND=noninteractive apt-get install -y -qq --no-install-recommends \
        curl gnupg tini python3 default-jre-headless wget ca-certificates fonts-dejavu fonts-liberation \
        fonts-noto-core fonts-noto-ui-core fonts-noto-mono && \
    rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

# Install Google Chrome / Chromium
RUN set -e; \
    if [[ "$(dpkg --print-architecture)" == "amd64" ]]; then \
        curl -fsSL https://dl.google.com/linux/linux_signing_key.pub | apt-key add -; \
        echo "deb [arch=amd64] http://dl.google.com/linux/chrome/deb/ stable main" > /etc/apt/sources.list.d/google-chrome.list; \
        apt-get update -qq; \
        DEBIAN_FRONTEND=noninteractive apt-get install -y -qq --no-install-recommends google-chrome-stable; \
        mv /usr/bin/google-chrome-stable /usr/bin/chromium; \
    else \
        apt-get update -qq; \
        DEBIAN_FRONTEND=noninteractive apt-get install -y -qq --no-install-recommends chromium; \
    fi; \
    rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

# Copy binaries from previous stages
COPY --from=pdfcpu-binary-stage /home/pdfcpu /usr/bin/
COPY --from=gotenberg-binary-stage /home/gotenberg /usr/bin/
COPY --from=fetch_tika /tika-server-standard.jar /usr/bin/tika-server-standard.jar

# Environment variables
ENV CHROMIUM_BIN_PATH=/usr/bin/chromium \
    LIBREOFFICE_BIN_PATH=/usr/lib/libreoffice/program/soffice.bin \
    UNOCONVERTER_BIN_PATH=/usr/bin/unoconverter \
    PDFTK_BIN_PATH=/usr/bin/pdftk \
    QPDF_BIN_PATH=/usr/bin/qpdf \
    EXIFTOOL_BIN_PATH=/usr/bin/exiftool \
    PDFCPU_BIN_PATH=/usr/bin/pdfcpu
