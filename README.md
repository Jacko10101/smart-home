# smart-home

Single-Pi smart-home cluster. GitOps via ArgoCD. Built and operated solo.

The goal is a resilient, fully-local home automation system — no cloud dependencies in the critical path. See `CLAUDE.md` for the North Star and operational details.

## Hardware

- Raspberry Pi 5 (8GB)
- 1TB NVMe SSD
- UPS
- SONOFF Zigbee 3.0 USB coordinator
- WiFi (eth0 currently disconnected)

## What's running

- **Home Assistant** — automation hub, Lovelace in YAML mode
- **Zigbee2MQTT** — Zigbee coordinator bridge
- **Mosquitto** — MQTT broker
- **ArgoCD** — GitOps, app-of-apps pattern
- **kube-prometheus-stack** — Prometheus + Alertmanager (Grafana disabled)
- **alertmanager-ntfy** — Alertmanager → ntfy.sh phone notifications
- **Backup CronJob** — daily tar of HA + Z2M PVCs

## Devices

- 6× Philips Hue White and Color Ambiance E14 (living room)
- 2× SONOFF SNZB-02D temperature/humidity sensors (living room, hallway)
- 1× SONOFF SNZB-03 motion sensor (living room)
- 3× Innr SP 242 smart plugs with power monitoring (hallway lamp, kitchen heater, dining room TV)

## Access

LAN only via NodePorts (see `CLAUDE.md` for URLs). No remote access currently configured.

## Repo layout

- `argocd-apps/` — ArgoCD `Application` objects; the app-of-apps watches this directory
- `applications/` — actual manifests for each service

## Operating it

See `CLAUDE.md` for orientation, gotchas, and common commands.
