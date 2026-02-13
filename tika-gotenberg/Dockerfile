# ----------------------------------------------
# Build arguments
# ----------------------------------------------
ARG GOLANG_VERSION=1.25.5
ARG PDFCPU_VERSION=v0.11.1
ARG GOTENBERG_VERSION=v8.26.0
ARG TIKA_VERSION=3.2.3
ARG PDFTK_VERSION=v4.1.9

# ----------------------------------------------
# pdfcpu build stage
# ----------------------------------------------
FROM golang:${GOLANG_VERSION} AS pdfcpu-builder
ARG PDFCPU_VERSION
ENV CGO_ENABLED=0
WORKDIR /build

RUN curl -Ls https://github.com/pdfcpu/pdfcpu/archive/refs/tags/${PDFCPU_VERSION}.tar.gz \
    | tar -xz --strip-components=1 && \
    cd cmd/pdfcpu && \
    go mod download && \
    go build -o /pdfcpu \
      -ldflags "-s -w \
      -X main.version=${PDFCPU_VERSION} \
      -X github.com/pdfcpu/pdfcpu/pkg/pdfcpu.VersionStr=${PDFCPU_VERSION}"


# ----------------------------------------------
# Gotenberg build stage
# ----------------------------------------------
FROM golang:${GOLANG_VERSION} AS gotenberg-builder
ARG GOTENBERG_VERSION
ENV CGO_ENABLED=0
WORKDIR /build

RUN curl -Ls https://github.com/gotenberg/gotenberg/archive/refs/tags/${GOTENBERG_VERSION}.tar.gz \
    | tar -xz --strip-components=1 && \
    go mod download && \
    go build -o gotenberg \
      -ldflags "-X github.com/gotenberg/gotenberg/v8/cmd.Version=${GOTENBERG_VERSION}" \
      cmd/gotenberg/main.go

# ----------------------------------------------
# Fetch Apache Tika
# ----------------------------------------------
FROM debian:bookworm-slim AS tika-fetch
ARG TIKA_VERSION
RUN apt-get update && \
    apt-get install -y --no-install-recommends wget ca-certificates && \
    wget -O /tika-server.jar \
      https://dlcdn.apache.org/tika/${TIKA_VERSION}/tika-server-standard-${TIKA_VERSION}.jar && \
    rm -rf /var/lib/apt/lists/*

# ----------------------------------------------
# Final runtime image
# ----------------------------------------------
FROM debian:bookworm-slim

ENV DEBIAN_FRONTEND=noninteractive

# Install core dependencies
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
      ca-certificates \
      pdftk-java \
      curl \
      wget \
      tini \
      default-jre-headless \
      libreoffice \
      unoconv \
      chromium \
      qpdf \
      libimage-exiftool-perl \
      fonts-dejavu \
      fonts-liberation \
      fonts-noto-core \
      fonts-noto-cjk \
      fonts-noto-mono \
      fonts-noto-ui-core \
      cabextract \
    && rm -rf /var/lib/apt/lists/*

## Install MS core fonts
#RUN apt-get update && \
#    apt-get install -y --no-install-recommends ttf-mscorefonts-installer && \
#    rm -rf /var/lib/apt/lists/*

## Install PDFTK (java version)
#ARG PDFTK_VERSION
#RUN wget -O /usr/local/bin/pdftk-all.jar \
#    https://gitlab.com/api/v4/projects/5024297/packages/generic/pdftk-java/${PDFTK_VERSION}/pdftk-all.jar && \
#    echo '#!/bin/sh\nexec java -jar /usr/local/bin/pdftk-all.jar "$@"' > /usr/local/bin/pdftk && \
#    chmod +x /usr/local/bin/pdftk

# Copy built binaries
COPY --from=pdfcpu-builder /pdfcpu /usr/local/bin/pdfcpu
COPY --from=gotenberg-builder /build/gotenberg /usr/local/bin/gotenberg
COPY --from=tika-fetch /tika-server.jar /usr/local/bin/tika-server.jar

RUN mkdir -p /tmp/chromium && \
    chown -R 1000:1000 /tmp/chromium

# Create runtime user
RUN useradd -m -u 1000 gotenberg
USER gotenberg


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

EXPOSE 3000

ENTRYPOINT ["/usr/bin/tini", "--"]
CMD ["gotenberg"]
