# ZimaOS Custom App Store

A custom app store for [ZimaOS](https://www.zimaspace.com/) / [CasaOS](https://casaos.io/) with a curated set of self-hosted applications — optimized for the **ZimaBoard 2 (Intel N150)**.

## Add to ZimaOS

In ZimaOS, open the App Store → ⚙️ → Custom App Store and enter:

```
https://github.com/psdl76/zimaos_appstore/releases/latest/download/appstore.zip
```

---

## Apps

### Network

| App | Description | Port |
|---|---|---|
| [AdGuard Home](Apps/AdGuardHome/) | Network-wide ad and tracker blocking DNS server. Protects all devices on your network — no client software needed. | 8088 |
| [Unbound](Apps/Unbound/) | Validating, recursive, caching DNS resolver. Resolves queries directly against root servers — no upstream provider like Cloudflare or Google. Designed as the upstream resolver for AdGuard Home. | 5335 |
| [Netbird](Apps/Netbird/) | Self-hosted WireGuard-based VPN mesh network. Connects your devices securely over the internet with a web dashboard, CrowdSec integration, and a Traefik reverse proxy for exposing services. | 443 |

### Security

| App | Description | Port |
|---|---|---|
| [AliasVault](Apps/AliasVault/) | Password manager and email alias generator. Keeps credentials and identities private with a built-in SMTP server for alias email delivery. | 1443 |

### Cloud

| App | Description | Port |
|---|---|---|
| [Nextcloud AIO](Apps/NextcloudAIO/) | The officially recommended, self-hosted Nextcloud deployment. One mastercontainer manages the full stack: Nextcloud, database, Redis, reverse proxy, and optionally Collabora Online and Talk. Optimized for ZimaBoard 2 with Intel Quick Sync hardware transcoding. | 8080 |

### Photography

| App | Description | Port |
|---|---|---|
| [Immich](Apps/Immich/) | High-performance, self-hosted photo and video backup — a Google Photos alternative with AI smart search, facial recognition, and automatic mobile backup. Optimized for ZimaBoard 2: Intel Quick Sync for video transcoding, OpenVINO for ML inference. | 2283 |

### Smart Home

| App | Description | Port |
|---|---|---|
| [Home Assistant](Apps/HomeAssistant/) | The leading open-source home automation platform. Controls thousands of devices and services locally — no cloud required. Runs as Home Assistant Container (full integrations + automations, no add-ons). | 8123 |
| [Mosquitto](Apps/Mosquitto/) | Lightweight MQTT message broker. The backbone for all IoT and smart home communication — connects Zigbee2MQTT, ESPHome, Tasmota, Shelly, and Home Assistant. | 1883 |
| [Zigbee2MQTT](Apps/Zigbee2MQTT/) | Bridges Zigbee devices to Home Assistant via MQTT — without any proprietary hub or cloud. Supports 3000+ devices. Pre-configured for network-based coordinators (SMLIGHT SLZB series) and USB dongles. | 8091 |
| [Matterbridge](Apps/Matterbridge/) | Matter bridge that connects existing smart home devices to Apple Home, Google Home, Amazon Alexa, and Home Assistant — using plugins for Zigbee2MQTT, Shelly, and more. | 8283 |
| [go2rtc](Apps/go2rtc/) | Universal camera streaming hub. Bridges RTSP, WebRTC, HomeKit, ONVIF, and FFmpeg with near-zero latency. Used as the streaming backend for Home Assistant cameras and Frigate NVR. | 1984 |

---

## Hardware Optimizations (ZimaBoard 2 — Intel N150)

Several apps are pre-configured to use the Intel N150's integrated GPU:

| App | Optimization | Benefit |
|---|---|---|
| Immich | Quick Sync (`/dev/dri`) | Hardware video transcoding — 5–10× faster than CPU |
| Immich | OpenVINO (`release-openvino`) | GPU-accelerated face recognition and CLIP smart search |
| Nextcloud AIO | Quick Sync (`/dev/dri`) | Hardware transcoding for Nextcloud Memories |
| go2rtc | `privileged: true` | Full hardware access including Intel VAAPI/Quick Sync |

> If you run this on non-Intel hardware, remove the `devices: - /dev/dri:/dev/dri` blocks from Immich and Nextcloud AIO before installing.

---

## Stack Overview

```
Devices / Cameras
    │
    ├── Zigbee devices → Zigbee2MQTT → Mosquitto ─────────────────┐
    │                                                              │
    ├── Cameras → go2rtc (WebRTC/RTSP) ────────────────────────── Home Assistant
    │                                                              │
    └── Any device via DNS ──── AdGuard Home → Unbound            │
                                                              Matterbridge
                                                                   │
                                                    Apple Home / Google Home / Alexa
```

---

## Development

Each app lives in its own directory under `Apps/` and follows the [CasaOS AppStore schema](https://github.com/IceWhaleTech/CasaOS-AppStore).

A GitHub Actions workflow automatically builds and publishes a new release on every push to `main` — no manual release step needed.
