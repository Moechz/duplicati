# duplicati-for-tos

> TerraMaster TOS 7 App Center — **Docker application** package for
> [Duplicati](https://github.com/duplicati/duplicati).

## What this is

A four-file TOS Docker application archive (`duplicatidocker.tar.gz`) that installs
Duplicati on TOS 7 through the App Center:

- `config.ini` — TOS application metadata (`application_type: docker`, id `duplicatidocker`)
- `duplicatidocker.lang` — 23-language superset (TOS requires 14)
- `duplicatidocker.svg` — application icon (552 bytes, 2 SVG elements)
- `docker-compose.yml` — one service, image `duplicati/duplicati:2.4.1.0-stable`

## How it works

| | |
|---|---|
| Container image | `duplicati/duplicati:2.4.1.0-stable` (Docker Hub, used unmodified) |
| Open | `http://<NAS-IP>:8009/` (container port 8200) |
| Runs as | non-root, `user: "1000:1000"` |
| Persistent data | `/data` → `/Volume1/DockerAppData/duplicatidocker/data`, `/backups` → `/Volume1/DockerAppData/duplicatidocker/backups` |
| Logs | `docker logs duplicatidocker` (stdout/stderr) |

`data/Duplicati/` holds `Duplicati-server.sqlite` (jobs, schedules, settings), the job databases and the encryption material; `backups/` is the local destination mounted as `/backups`. Set your backup destination to `/backups/...` inside the container to keep local backups under the application data folder.

## Design highlights

- **Non-root.** `user: "1000:1000"`; verified on real hardware (the container reports uid 1000).
- **Deterministic build**: GNU tar with uid/gid 0 and zeroed timestamps, gzip without a timestamp, plus the review-standards self-check above.
- **Non-root without a wrapper.** The official image's `run-as-user.sh` starts `duplicati-server` with whatever uid the container runs as, so `user: "1000:1000"` is enough; verified on real hardware.
- **Fixed image tag `2.4.1.0-stable`** (never `:latest` / `:stable`), Docker Hub only, no `privileged`, no `network_mode: host`, no `cap_add`, no Docker socket.
- **Two volumes**: `/data` (database and settings) and `/backups` (local backup destination), both below `/Volume1/DockerAppData/duplicatidocker/`.
- **Health check** on the UI (`GET /` on port 8200) with a 60 s start period for the first start.

## Runtime file manifest

Everything the application writes on the NAS lives inside its own data
directory; the container-internal `/tmp` is discarded with the container. The
table lists what is created and how it behaves. Nothing else is written.

| Path | Purpose | Created when | Growth bound | Lifecycle |
|---|---|---|---|---|
| `data/Duplicati/Duplicati-server.sqlite` | server database - jobs, schedules, settings, admin password hash | first start | small, grows with the number of jobs | never overwritten; required to keep your job configuration |
| `data/Duplicati/machineid.txt` | machine key used to encrypt stored credentials | first start | static | never delete; without it saved credentials cannot be decrypted |
| `data/Duplicati/installation.txt` | installation id and first-run marker | first start | static | safe to keep; regenerated if deleted |
| `data/Duplicati/control_dir_v2` | control files used to stop and coordinate the server | first start | small | managed by Duplicati |
| `backups` | local backup destination (mounted as /backups) | first start (empty) | grows with every backup version | yours - keep it, or point jobs at cloud storage instead |

## Build

```bash
scripts/build.sh                  # -> out/duplicatidocker.tar.gz + .sha256
scripts/build.sh 2.4.1.0-2   # packaging iteration bump
scripts/build.sh 2.4.1.0-1 aarch64
```

The build stages the four files, substitutes the version/platform placeholders,
removes CRLF/BOM/AppleDouble, then runs
`tools/verify.py`, a review-standards self-check (archive layout, JSON validity,
required fields, 14+ language coverage, version consistency, Docker-Hub-only
images, fixed tag, reserved/recommended host ports, non-root user, per-service
healthcheck/TZ/restart, data path below the application data root, never the
`/Volume*` wildcard, comment-free sequence blocks, `x-app-meta` position, no
literal secrets and the SVG size/element limits).

## Local verification

```bash
./TOSAppSelfTestingTool -c out/duplicatidocker.tar.gz   # official spec check
bash test/install-and-verify.sh                  # on-device E2E (needs root)
```

> Docker applications cannot be side-loaded through the App Center's manual
> install page (it accepts `.deb` only); the official self-testing tool's `-i`
> mode or `docker compose` are the only ways to install one for testing.

## Store submission checklist

1. Public GitHub repo (code + README only).
2. Release **tag `v2.4.1.0-1`**, assets `duplicatidocker.tar.gz` +
   `duplicatidocker.tar.gz.sha256` (Docker asset naming: `<app_id>.tar.gz`, **no
   platform suffix**).
3. Developer platform → Add Application: ID `duplicatidocker`, package type Docker,
   repository URL, architecture `x86_64`.
4. Version Management → Add Version `2.4.1.0-1` → automated validation → review.

## License / branding

Duplicati is distributed by its upstream authors under **MIT**. The container image is the official image from Docker Hub (`duplicati/duplicati:2.4.1.0-stable`) and is used unmodified; no upstream source code is redistributed here. This packaging is an independent community submission by Moechz and is not affiliated with the upstream project or TerraMaster.

## Limitations

Duplicati stores the encryption passphrase you set for each backup job. If you lose it, the backups cannot be restored - store it outside the NAS as well.
TOS Docker applications cannot mount shared folders: a local backup destination must be `/backups/...` (application data folder). Edit `/Volume1/@apps/DockerEngine/application/duplicatidocker/docker-compose.yml` after installation to add another host path.
The UI is served over plain HTTP on the LAN and its sign-in token is printed in the container log. Do not expose the published port to the internet without a TLS reverse proxy.

---

Packaged for TOS 7 by Moechz. Upstream image used unmodified from Docker Hub.
