# --------------------------------------------------------------
# Build arguments
# --------------------------------------------------------------
ARG GOLANG_VERSION=1.25.5
ARG PDFCPU_VERSION=v0.11.1
ARG GOTENBERG_VERSION=v8.26.0
ARG TIKA_VERSION=3.2.3
ARG NOTO_COLOR_EMOJI_VERSION=main

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
    go mod download && go build -o gotenberg -ldflags "-X github.com/gotenberg/gotenberg/v8/cmd.Version=${GOTENBERG_VERSION}" cmd/gotenberg/main.go

# --------------------------------------------------------------
# Tika fetch stage
# --------------------------------------------------------------
FROM debian:bookworm-slim AS tika-fetch
ARG TIKA_VERSION

# Install minimal tools for fetching and verifying Tika
RUN apt-get update && \
    apt-get install -y --no-install-recommends gnupg curl ca-certificates && \
    curl -Ls https://downloads.apache.org/tika/${TIKA_VERSION}/tika-server-standard-${TIKA_VERSION}.jar -o /tika-server.jar && \
    rm -rf /var/lib/apt/lists/*

# --------------------------------------------------------------
# Final runtime image
# --------------------------------------------------------------
FROM debian:bookworm-slim
ENV DEBIAN_FRONTEND=noninteractive

# Core dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates \
      curl \
      tini \
      default-jre-headless \
      libreoffice \
      unoconv \
      qpdf \
      pdftk-java \
      libimage-exiftool-perl \
      fonts-dejavu \
      fonts-dejavu-extra \
      fonts-liberation \
      fonts-liberation2 \
      fonts-noto-core \
      fonts-noto-ui-core \
      fonts-noto-mono \
      fonts-noto-cjk \
      cabextract \
      gnupg \
      python3 \
      fontconfig \
      && rm -rf /var/lib/apt/lists/*

# Microsoft core fonts
RUN curl -o /tmp/ttf-mscorefonts-installer_3.8.1_all.deb http://httpredir.debian.org/debian/pool/contrib/m/msttcorefonts/ttf-mscorefonts-installer_3.8.1_all.deb && \
    apt-get update && apt-get install -y --no-install-recommends /tmp/ttf-mscorefonts-installer_3.8.1_all.deb && \
    rm -f /tmp/ttf-mscorefonts-installer_3.8.1_all.deb && rm -rf /var/lib/apt/lists/*

# Additional language / emoji fonts
RUN apt-get update && apt-get install -y --no-install-recommends \
      culmus \
      fonts-beng \
      fonts-hosny-amiri \
      fonts-lklug-sinhala \
      fonts-lohit-guru \
      fonts-lohit-knda \
      fonts-samyak-gujr \
      fonts-samyak-mlym \
      fonts-samyak-taml \
      fonts-sarai \
      fonts-sil-abyssinica \
      fonts-sil-padauk \
      fonts-telu \
      fonts-thai-tlwg \
      fonts-wqy-zenhei \
      fonts-arphic-ukai \
      fonts-arphic-uming \
      fonts-ipafont-mincho \
      fonts-ipafont-gothic \
      fonts-unfonts-core \
      fonts-crosextra-caladea \
      fonts-crosextra-carlito \
      fonts-sil-gentium \
      fonts-sil-gentium-basic \
      && rm -rf /var/lib/apt/lists/*

# Noto Color Emoji
RUN curl -Ls "https://github.com/googlefonts/noto-emoji/raw/${NOTO_COLOR_EMOJI_VERSION}/fonts/NotoColorEmoji.ttf" -o /usr/local/share/fonts/NotoColorEmoji.ttf

# Chrome / Chromium
RUN if [ "$(dpkg --print-architecture)" = "amd64" ]; then \
        curl -fsSL https://dl.google.com/linux/linux_signing_key.pub | apt-key add - && \
        echo "deb [arch=amd64] http://dl.google.com/linux/chrome/deb/ stable main" > /etc/apt/sources.list.d/google-chrome.list && \
        apt-get update && apt-get install -y --no-install-recommends google-chrome-stable && \
        mv /usr/bin/google-chrome-stable /usr/bin/chromium; \
    else \
        apt-get update && apt-get install -y --no-install-recommends chromium; \
    fi && rm -rf /var/lib/apt/lists/*

# Python symlink for unoconv
RUN ln -s /usr/bin/python3 /usr/bin/python || true

# Runtime user
RUN useradd -m -u 1000 gotenberg
USER gotenberg

# Copy built binaries
COPY --from=pdfcpu-builder /pdfcpu /usr/local/bin/pdfcpu
COPY --from=gotenberg-builder /build/gotenberg /usr/local/bin/gotenberg
COPY --from=tika-fetch /tika-server.jar /usr/local/bin/tika-server.jar

# Chromium hyphenation data
RUN mkdir -p /tmp/chromium && chown -R 1000:1000 /tmp/chromium

# Environment variables
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
