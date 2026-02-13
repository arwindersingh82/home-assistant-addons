# ----------------------------------------------
# Build arguments
# ----------------------------------------------
ARG BUILD_FROM=ghcr.io/hassio-addons/debian-base:9.2.0
ARG GOLANG_VERSION="1.25.5"
ARG PDFCPU_VERSION="v0.9.0"
ARG GOTENBERG_VERSION="v8.26.0"
ARG TIKA_VERSION="3.2.0"
ARG NOTO_COLOR_EMOJI_VERSION="main"
ARG PDFTK_VERSION="v3.3.3"

# ----------------------------------------------
# pdfcpu build stage
# ----------------------------------------------
FROM golang:${GOLANG_VERSION} AS pdfcpu-binary-stage
ARG PDFCPU_VERSION
ENV CGO_ENABLED=0
WORKDIR /home

RUN curl -Ls "https://github.com/pdfcpu/pdfcpu/archive/refs/tags/${PDFCPU_VERSION}.tar.gz" -o pdfcpu.tar.gz && \
    tar --strip-components=1 -xvzf pdfcpu.tar.gz && \
    GOPROXY=direct go mod download && \
    GOPROXY=direct go mod verify && \
    go build -o pdfcpu \
      -ldflags "-s -w -X 'main.version=${PDFCPU_VERSION}' \
      -X 'github.com/pdfcpu/pdfcpu/pkg/pdfcpu.VersionStr=${PDFCPU_VERSION}' \
      -X main.builtBy=gotenberg" \
      ./cmd/pdfcpu

# ----------------------------------------------
# Gotenberg build stage
# ----------------------------------------------
FROM golang:${GOLANG_VERSION} AS gotenberg-binary-stage
ARG GOTENBERG_VERSION
ENV CGO_ENABLED=0
WORKDIR /home

RUN curl -Ls "https://github.com/gotenberg/gotenberg/archive/refs/tags/${GOTENBERG_VERSION}.tar.gz" -o gotenberg.tar.gz && \
    tar --strip-components=1 -xvzf gotenberg.tar.gz && \
    GOPROXY=direct go mod download && \
    GOPROXY=direct go mod verify && \
    go build -o gotenberg \
      -ldflags "-X 'github.com/gotenberg/gotenberg/v8/cmd.Version=${GOTENBERG_VERSION}'" \
      cmd/gotenberg/main.go

# ----------------------------------------------
# Fetch Apache Tika
# ----------------------------------------------
FROM debian:bookworm AS fetch_tika
ARG TIKA_VERSION
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

RUN apt-get update && \
    apt-get install -y --no-install-recommends gnupg wget ca-certificates curl && \
    rm -rf /var/lib/apt/lists/* && \
    curl -sSL https://downloads.apache.org/tika/KEYS | gpg --import && \
    wget -O /tika-server-standard.jar https://dlcdn.apache.org/tika/${TIKA_VERSION}/tika-server-standard-${TIKA_VERSION}.jar || \
    wget -O /tika-server-standard.jar https://archive.apache.org/dist/tika/${TIKA_VERSION}/tika-server-standard-${TIKA_VERSION}.jar && \
    wget -O /tika-server-standard.jar.asc https://downloads.apache.org/tika/${TIKA_VERSION}/tika-server-standard-${TIKA_VERSION}.jar.asc || \
    wget -O /tika-server-standard.jar.asc https://archive.apache.org/dist/tika/${TIKA_VERSION}/tika-server-standard-${TIKA_VERSION}.jar.asc && \
    gpg --verify /tika-server-standard.jar.asc /tika-server-standard.jar

# ----------------------------------------------
# Final stage
# ----------------------------------------------
FROM ${BUILD_FROM}

# Copy root filesystem
COPY rootfs/ /

# Create user
RUN groupadd --gid 1000 gotenberg && \
    useradd --uid 1000 --gid gotenberg --shell /bin/bash \
    --home /home/gotenberg --no-create-home gotenberg && \
    mkdir /home/gotenberg && \
    chown gotenberg: /home/gotenberg \


# ----------------------------------------------
# Install system dependencies (FULL original set)
# ----------------------------------------------
RUN apt-get update -qq && \
    apt-get upgrade -yqq && \
    DEBIAN_FRONTEND=noninteractive apt-get install -y -qq --no-install-recommends \
      curl gnupg tini python3 default-jre-headless \
      wget ca-certificates cabextract xfonts-utils \
      qpdf libimage-exiftool-perl \
      fonts-dejavu fonts-dejavu-extra \
      fonts-liberation fonts-liberation2 \
      fonts-noto-core fonts-noto-ui-core fonts-noto-mono fonts-noto-cjk \
      fonts-crosextra-caladea fonts-crosextra-carlito \
      fonts-linuxlibertine fonts-sil-gentium fonts-sil-gentium-basic \
      fonts-arphic-ukai fonts-arphic-uming \
      fonts-ipafont-mincho fonts-ipafont-gothic \
      fonts-unfonts-core fonts-wqy-zenhei \
      fonts-thai-tlwg fonts-telu fonts-beng fonts-lohit-guru \
      fonts-lohit-knda fonts-samyak-gujr fonts-samyak-mlym fonts-samyak-taml \
      fonts-sarai fonts-sil-abyssinica fonts-sil-padauk culmus \
      libreoffice unoconv && \
    ln -s /usr/bin/unoconv /usr/bin/unoconverter && \
    rm -rf /var/lib/apt/lists/*

# ----------------------------------------------
# Install MS Core Fonts properly (original method)
# ----------------------------------------------
RUN apt-get update -qq && \
    DEBIAN_FRONTEND=noninteractive apt-get install -y -qq ttf-mscorefonts-installer && \
    rm -rf /var/lib/apt/lists/*

# ----------------------------------------------
# Install Noto Emoji
# ----------------------------------------------
RUN curl -Ls "https://github.com/googlefonts/noto-emoji/raw/${NOTO_COLOR_EMOJI_VERSION}/fonts/NotoColorEmoji.ttf" \
    -o /usr/local/share/fonts/NotoColorEmoji.ttf && \
    fc-cache -fv

# ----------------------------------------------
# Install Chrome / Chromium
# ----------------------------------------------
RUN set -e; \
    if [[ "$(dpkg --print-architecture)" == "amd64" ]]; then \
        curl -fsSL https://dl.google.com/linux/linux_signing_key.pub | apt-key add -; \
        echo "deb [arch=amd64] http://dl.google.com/linux/chrome/deb/ stable main" > /etc/apt/sources.list.d/google-chrome.list; \
        apt-get update -qq; \
        DEBIAN_FRONTEND=noninteractive apt-get install -y -qq google-chrome-stable; \
        mv /usr/bin/google-chrome-stable /usr/bin/chromium; \
    else \
        apt-get update -qq; \
        DEBIAN_FRONTEND=noninteractive apt-get install -y -qq chromium; \
    fi; \
    rm -rf /var/lib/apt/lists/*

# ----------------------------------------------
# Install PDFTK
# ----------------------------------------------
RUN curl -o /usr/bin/pdftk-all.jar \
  "https://gitlab.com/api/v4/projects/5024297/packages/generic/pdftk-java/${PDFTK_VERSION}/pdftk-all.jar" && \
    echo '#!/bin/sh\nexec java -jar /usr/bin/pdftk-all.jar "$@"' > /usr/bin/pdftk && \
    chmod +x /usr/bin/pdftk /usr/bin/pdftk-all.jar

# ----------------------------------------------
# Copy built binaries
# ----------------------------------------------
COPY --from=pdfcpu-binary-stage /home/pdfcpu /usr/bin/
COPY --from=gotenberg-binary-stage /home/gotenberg /usr/bin/
COPY --from=fetch_tika /tika-server-standard.jar /usr/bin/tika-server-standard.jar

# ----------------------------------------------
# Environment variables
# ----------------------------------------------
ENV CHROMIUM_BIN_PATH=/usr/bin/chromium \
    CHROMIUM_HYPHEN_DATA_DIR_PATH=/tmp/chromium \
    CHROMIUM_DISABLE_SANDBOX=true \
    LIBREOFFICE_BIN_PATH=/usr/lib/libreoffice/program/soffice.bin \
    UNOCONVERTER_BIN_PATH=/usr/bin/unoconverter \
    PDFTK_BIN_PATH=/usr/bin/pdftk \
    QPDF_BIN_PATH=/usr/bin/qpdf \
    EXIFTOOL_BIN_PATH=/usr/bin/exiftool \
    PDFCPU_BIN_PATH=/usr/bin/pdfcpu

# ----------------------------------------------
# Expose + entry
# ----------------------------------------------
EXPOSE 3000
ENTRYPOINT ["/usr/bin/tini", "--"]
CMD ["gotenberg"]
