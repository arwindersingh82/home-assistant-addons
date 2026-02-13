# Arwinder Home Assistant Add-ons

This repository contains custom and maintained Home Assistant add-ons used within my personal infrastructure.

The goal of this repository is long-term stability, controlled updates, and independence from upstream availability.

All add-ons included here are either:

- Maintained forks of community projects
- Internal infrastructure services
- Stability-pinned versions of upstream projects

---

## Installation

1. Open **Home Assistant**
2. Navigate to **Settings → Add-ons → Add-on Store**
3. Click the **⋮ menu (top right)**
4. Select **Repositories**
5. Add:

https://github.com/arwindersingh82/tika-gotenberg-home-assistant-addon


6. Click **Add**

The add-ons in this repository will then appear in the add-on store.

---

## Included Add-ons

### Tika + Gotenberg

Provides Apache Tika and Gotenberg services for document processing workflows (e.g. Paperless-ngx).

**Purpose:**

- Reliable document parsing
- PDF processing and conversion
- Long-term availability independent of upstream repository changes

---

## Repository Structure

This repository follows the standard Home Assistant add-on repository layout:



repository.json
addon-name-1/
config.json
Dockerfile
...
addon-name-2/
config.json
Dockerfile
...


Each directory represents a standalone Home Assistant add-on.

---

## Update Strategy

This repository is maintained as a controlled fork environment:

- Upstream repositories are tracked manually
- Updates are merged deliberately
- Docker image versions may be pinned to prevent unexpected breaking changes
- Forgejo is used as the authoritative source with GitHub acting as a distribution mirror

This ensures:

- Stability
- Reproducibility
- Protection against upstream repository removal
- Controlled upgrade paths

---

## Verify Installation / Test Commands

After building or deploying the add-on, you can verify that all binaries and services are correctly installed and functional. Run these commands inside the container terminal:

```bash
# Check binaries exist and display their versions
gotenberg --version
pdfcpu version
java -jar /usr/local/bin/tika-server.jar --version
chromium --version
libreoffice --version
unoconv --version
pdftk --version
qpdf --version
exiftool --version

# Start Tika server in the background and test HTTP endpoint
java -jar /usr/local/bin/tika-server.jar &
curl http://localhost:9998/tika

```

###  These commands confirm:

- Gotenberg and PDF engines are working
- Apache Tika server is available
- Chromium, LibreOffice, unoconv, and other conversion tools are installed
- Metadata and PDF processing engines function as expected

## Philosophy

Infrastructure services such as document ingestion and processing are foundational components.  
They should not depend on external availability or unpinned upstream changes.

This repository exists to:

- Maintain control over core services
- Avoid supply-chain fragility
- Provide long-term operational stability

---

## Disclaimer

These add-ons are maintained for personal use.  
Use at your own discretion and test thoroughly before deploying into production environments.
