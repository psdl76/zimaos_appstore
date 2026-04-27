# CLAUDE.md — ZimaOS Custom App Store

This file gives a Claude Code session everything it needs to add new apps to this store without prior context.

---

## What this project is

A custom app store for ZimaOS / CasaOS at `https://github.com/psdl76/zimaos_appstore`.

Each app is a directory under `Apps/` containing a `docker-compose.yml` with `x-casaos` annotations. A GitHub Actions workflow (`.github/workflows/release.yml`) automatically builds `appstore.tar.gz` and publishes it as a GitHub Release on every push to `main`. ZimaOS pulls from:

```
https://github.com/psdl76/zimaos_appstore/releases/latest/download/appstore.tar.gz
```

---

## Repository structure

```
Apps/
  AdGuardHome/
    docker-compose.yml
    icon.png
    thumbnail.png
    screenshot-1.png ...
  AliasVault/
  go2rtc/
  HomeAssistant/
  Immich/
  Matterbridge/
  Mosquitto/
  Netbird/
  NextcloudAIO/
  Unbound/
  Zigbee2MQTT/
category-list.json      ← app categories shown in ZimaOS store UI
recommend-list.json     ← featured apps (uses x-casaos name field as appid)
.github/workflows/release.yml
CLAUDE.md
README.md
```

---

## CasaOS app schema — key rules

Reference store for format examples: `https://github.com/IceWhaleTech/CasaOS-AppStore/tree/main/Apps`

### Required structure per app

```yaml
name: <compose-project-name>          # lowercase, hyphens
services:
  <service-name>:
    image: <image>:<tag>
    deploy:
      resources:
        reservations:
          memory: <NM>               # always set a memory reservation
    network_mode: bridge             # or host — see below
    ports: ...
    restart: unless-stopped          # or always for critical services
    volumes: ...
    environment: ...
    x-casaos:                        # per-service annotations
      ports:
        - container: "<port>"
          description:
            en_us: ...
            de_de: ...
      volumes:
        - container: /path/in/container
          description:
            en_us: ...
            de_de: ...
      envs:
        - container: ENV_VAR_NAME
          description:
            en_us: ...
              de_de: ...
    container_name: <name>           # explicit container name always

x-casaos:                            # top-level store metadata
  architectures:
    - amd64
    - arm64
    - arm
  main: <service-name>               # primary service (determines "Open" button)
  author: CasaOS Team
  category: <category>               # must match an entry in category-list.json
  description:
    en_us: |
      ...
    de_de: |
      ...
  developer: <upstream developer name>
  icon: https://cdn.jsdelivr.net/gh/walkxcode/dashboard-icons/png/<name>.png
  screenshot_link:
    - https://cdn.jsdelivr.net/gh/psdl76/zimaos_appstore@main/Apps/<AppDir>/screenshot-1.png
  tagline:
    en_us: ...
    de_de: ...
  thumbnail: https://cdn.jsdelivr.net/gh/walkxcode/dashboard-icons/png/<name>.png
  tips:
    before_install:
      en_us: |
        ...
      de_de: |
        ...
  title:
    en_us: <App Name>
  index: /
  port_map: "<primary-port>"         # port shown in ZimaOS "Open" button
```

### Volume paths

Always use `/DATA/AppData/$AppID/<subdir>` as the source for bind mounts. `$AppID` is resolved by ZimaOS to the app's install name (the compose `name:` field).

```yaml
source: /DATA/AppData/$AppID/config
```

### Image tags

- Use `latest` for most apps
- Use `stable` for Home Assistant (`ghcr.io/home-assistant/home-assistant:stable`)
- Use `release` for Immich server/ML (`ghcr.io/immich-app/immich-server:release`)
- Pin specific tags only when the image is a custom build with required extensions (e.g. Immich's postgres with pgvector)

### Network mode

- Use `bridge` (default) for apps that only need standard TCP/UDP ports
- Use `host` when the app requires mDNS, BLE, dynamic UDP (WebRTC), or direct LAN access:
  - Home Assistant, go2rtc, Matterbridge use `network_mode: host`
  - With `host` mode, omit the `ports:` section — set `port_map:` for the UI button only

### Icons and screenshots

Dashboard icons CDN (check availability first):
```
https://cdn.jsdelivr.net/gh/walkxcode/dashboard-icons/png/<appname>.png
```

Check availability: `curl -sI https://cdn.jsdelivr.net/gh/walkxcode/dashboard-icons/png/<name>.png | grep content-length`

If not available, use the app's official GitHub avatar or logo.

Screenshots go in the app directory as `screenshot-1.png`, `screenshot-2.png`, etc. For initial setup, copy the icon as placeholder: `cp icon.png screenshot-1.png`. Real screenshots should be added manually later.

---

## Target hardware: ZimaBoard 2

- **CPU:** Intel N150 (Alder Lake-N, 4 cores, 6W)
- **GPU:** Intel UHD Graphics (24 EUs, 1000 MHz) — supports Quick Sync + VAAPI + OpenVINO
- **RAM:** 8–16 GB

### Hardware acceleration patterns

**Intel Quick Sync (video transcoding):** Add to the service that does video work:
```yaml
devices:
  - /dev/dri:/dev/dri
```
Then enable in the app's admin UI (e.g. Immich: Administration → Video Transcoding → Quick Sync).

**OpenVINO (ML inference):** Changes the image tag AND adds device rules:
```yaml
image: ghcr.io/immich-app/immich-machine-learning:release-openvino
device_cgroup_rules:
  - 'c 189:* rmw'
devices:
  - /dev/dri:/dev/dri
volumes:
  - type: bind
    source: /dev/bus/usb
    target: /dev/bus/usb
```

**Already applied to:**
- `Immich` — Quick Sync on immich-server, OpenVINO on immich-machine-learning
- `NextcloudAIO` — Quick Sync on mastercontainer (passthrough to child containers)
- `go2rtc` — covered by `privileged: true` (VAAPI/QSV configured in go2rtc.yaml at runtime)

**Always add a note in `before_install` tips** that non-Intel users should remove the `devices:` block.

---

## Patterns used in this store

### Init container (first-run config generation)

Used when an app needs a config file to exist before the main container starts, but the file must be created dynamically. Pattern:

```yaml
services:
  <app>-init:
    image: <same or alpine image>
    restart: "no"
    volumes:
      - type: bind
        source: /DATA/AppData/$AppID/config
        target: /config
    entrypoint: ["/bin/sh", "-c"]
    command:
      - |
        if [ -f /config/app.conf ]; then exit 0; fi   # idempotency check
        cat > /config/app.conf << 'EOF'
        ...default config...
        EOF
    container_name: <app>-init

  <app>:
    depends_on:
      <app>-init:
        condition: service_completed_successfully
```

Used in: Mosquitto (default mosquitto.conf), Netbird (full config generation with secrets).

### Post-start setup container

Used when an app needs API calls or `docker exec` commands after the main containers are healthy. Requires Docker socket. Pattern used in Netbird for token generation.

### Multi-language descriptions

All apps have at minimum `en_us` and `de_de`. Many also have `fr_fr`, `zh_cn`, `ru_ru`. Always provide both EN and DE for `description`, `tagline`, `tips.before_install`.

---

## Workflow for adding a new app

1. **Research** — fetch the official Docker Hub or GitHub page for the app's recommended `docker-compose.yml`
2. **Create directory:** `mkdir -p Apps/<AppName>`
3. **Download icon:**
   ```bash
   curl -sL "https://cdn.jsdelivr.net/gh/walkxcode/dashboard-icons/png/<name>.png" -o Apps/<AppName>/icon.png
   cp Apps/<AppName>/icon.png Apps/<AppName>/thumbnail.png
   for i in 1 2 3; do cp Apps/<AppName>/icon.png Apps/<AppName>/screenshot-$i.png; done
   ```
4. **Write `docker-compose.yml`** following the schema above
5. **Check hardware acceleration** — does this app do video transcoding or ML inference? If yes, add Quick Sync `/dev/dri` pattern
6. **Update `recommend-list.json`** if the app should be featured (use the compose `name:` field as appid)
7. **Commit and push:**
   ```bash
   git add Apps/<AppName>/ recommend-list.json
   git commit -m "Add <AppName>"
   git push
   ```
   GitHub Actions builds and publishes the release automatically.

---

## Git / GitHub

- **Repo:** `git@github.com:psdl76/zimaos_appstore.git`
- **GitHub user:** `psdl76`
- **Default branch:** `main`
- **Auth:** HTTPS via `gh` (token in keyring) — SSH key at `~/.ssh/id_rsa` exists but agent not loaded
- **Push:** `git push` works via HTTPS after `gh auth setup-git`
- **Workflow scope:** token needs `workflow` scope to push changes to `.github/workflows/` — if rejected, user runs: `! gh auth refresh -h github.com -s workflow`
- **CDN URLs** in all compose files use `psdl76` (not `pseidl`) — double-check on any bulk replacements

---

## Categories in use

Defined in `category-list.json`:
- `Cloud` — Nextcloud AIO
- `Network` — AdGuard Home, Unbound, Netbird
- `Photography` — Immich
- `Security` — AliasVault
- `Smart Home` — Home Assistant, Mosquitto, Zigbee2MQTT, Matterbridge, go2rtc

---

## Key reference URLs

| Resource | URL |
|---|---|
| CasaOS AppStore (format reference) | https://github.com/IceWhaleTech/CasaOS-AppStore/tree/main/Apps |
| Dashboard Icons CDN | https://cdn.jsdelivr.net/gh/walkxcode/dashboard-icons/png/ |
| Dashboard Icons search | https://dashboardicons.com |
| ZimaOS Store URL | https://github.com/psdl76/zimaos_appstore/releases/latest/download/appstore.tar.gz |
