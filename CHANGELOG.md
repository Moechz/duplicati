# Changelog

All notable user-visible changes to the Duplicati TOS application package.

## 2.4.1.0-1 — 2026-11-08

### Added
- First release. Duplicati (`duplicati/duplicati:2.4.1.0-stable`, Docker Hub official image, unmodified)
  packaged as a TOS 7 Docker application, published on port 8009 and running
  as a non-root user (1000:1000).
- Persistent data is kept under `/Volume1/DockerAppData/duplicatidocker/`.
