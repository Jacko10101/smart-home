# Smart Home — Claude Context

Personal smart-home automation running on a single Raspberry Pi 5 with k3s. Built and operated solo. The code in this repo is the source of truth for everything that runs on the cluster.

## Quick orientation

- **Pi access**: `ssh pi` (alias for `raspi` / `raspberry`) → user `admin`, host `192.168.1.235`. `kubectl` requires `sudo` because k3s config is root-readable only.
- **What's running**: Home Assistant, Zigbee2MQTT, Mosquitto (MQTT broker), ArgoCD (GitOps), kube-prometheus-stack (Prometheus + Grafana + Alertmanager), nightly backup CronJob.
- **Service URLs on the LAN** (these are all NodePort, NOT the in-cluster ports the upstream docs suggest):
  | Service | URL |
  |---|---|
  | Home Assistant | `http://192.168.1.235:31123` |
  | Zigbee2MQTT | `http://192.168.1.235:31678` |
  | ArgoCD | `http://192.168.1.235:30113` |
  | Grafana | `http://192.168.1.235:31300` |
  | Prometheus | `http://192.168.1.235:31090` |
  | Alertmanager | `http://192.168.1.235:30723` |
- **mDNS `homeassistant.local` does NOT broadcast** on this network — always use the IP.

## GitOps flow (and its quirks)

1. The `argocd` app (`argocd-apps/argocd.yaml`) is the "app of apps" — it watches the `argocd-apps/` directory and reconciles all Application objects.
2. **`argocd` app has no auto-sync** — new commits to `main` are detected (OutOfSync) but require manual sync (`kubectl patch application -n argocd argocd --type merge -p '{"operation":{"sync":{}}}'` or via UI).
3. Once the parent syncs, individual app sync behavior varies:
   - `kube-prometheus-stack`: auto-sync + self-heal ON. Inline Helm values in `argocd-apps/kube-prometheus-stack.yaml`.
   - All others (`home-assistant`, `zigbee2mqtt`, `mosquitto`, `backup`): no syncPolicy — manual sync only.
4. **`applications/kube-prometheus-stack/prometheus-rules/smart-home-alerts.yaml` is NOT in ArgoCD.** It was applied manually 142+ days ago and any changes must be applied by hand: `cat <file> | ssh pi 'sudo kubectl apply -f -'`. Fix is queued (task #16).
5. **`applications/kube-prometheus-stack/values.yaml` is orphaned** — nothing references it. The real values live inline in `argocd-apps/kube-prometheus-stack.yaml`. Don't edit the orphaned file.

## Verified gotchas

- **Zigbee2MQTT CrashLoopBackOff after pi reboot.** The `fix-usb-permissions` init container chowns `/dev/ttyUSB0` to `root:18`, but only runs on pod creation. After a host reboot, udev recreates `/dev/ttyUSB0` with default `root:dialout` (GID 20) and k8s only restarts the failed main container, so the chown never re-runs. **Fix:** `ssh pi 'sudo kubectl rollout restart deployment/zigbee2mqtt -n zigbee'`. Permanent fix queued (task #3 — udev rule on host).
- **HA password reset requires a pod restart.** `hass --script auth change_password` writes to `/config/.storage/auth_provider.homeassistant` on disk, but the running HA process holds the credential cache in memory. Login keeps returning `invalid_auth` until the pod is restarted. After any password reset: `ssh pi 'sudo kubectl rollout restart deployment/home-assistant -n home-assistant'`.
- **HA configmap uses `subPath` mounts** for `configuration.yaml`, `scenes.yaml`, `automations.yaml`, `ui-lovelace.yaml`. ConfigMap updates do NOT propagate to the running pod with subPath mounts. After editing `applications/home-assistant/configmap.yaml`, restart the deployment.
- **Lovelace is in YAML mode**, not UI mode. All dashboard changes happen in `applications/home-assistant/configmap.yaml` under `ui-lovelace.yaml`, not from the HA UI.
- **HA entity IDs are wrong.** The 6 living-room Hue bulbs are `light.bedroom_left_1` through `light.bedroom_right_2` because they were originally paired in the bedroom. Scenes, dashboards, automations all carry the wrong name. Rename queued (task #5) — easier now while there are 6 than later when there are 30.

## Architecture notes

- Single-node k3s on Pi 5 (8GB), 1TB NVMe SSD, UPS, SONOFF Zigbee 3.0 USB coordinator.
- `local-path-provisioner` backs all PVCs at `/var/lib/rancher/k3s/storage/`.
- Tailscale operator exposes Home Assistant (`tailscale.com/expose: "true"` on the service).
- Backup CronJob (3am daily) tars HA + z2m PVC contents to `/backups` **on the same NVMe** — not off-pi. Task #2 to fix.
- HA recorder is sqlite (`home-assistant_v2.db` ~11MB currently). Will need MariaDB/Postgres before ~50 entities (task #6).

## Currently fragile, in priority order

1. **Pi is on WiFi**, not ethernet. `eth0` is NO-CARRIER (cable physically disconnected). All traffic goes via brcmfmac driver, which spammed errors before every recorded crash. Strongly suspected as root cause of the recurring hangs (Jan/Feb/Mar 2026, latest March 12 → 2-month outage until May 15). Task #9.
2. **Backups not off-pi.** Same disk dies → backups die. Two-month silent backup gap in 2026 went undetected (no alert wired up). Task #2.
3. **HA long-lived access token committed in plaintext** at `argocd-apps/kube-prometheus-stack.yaml:65` (10-year expiry). Assume compromised, must rotate. Task #13.
4. **Grafana admin password is the default** (`prom-operator`). Task #14.
5. **systemd watchdog is on** (`RuntimeWatchdogSec=30s` via `/etc/systemd/system.conf.d/watchdog.conf`) — hard hangs now auto-reboot within 30 seconds. Big win, recently enabled.

## Common operational commands

```bash
# All pods status
ssh pi 'sudo kubectl get pods -A'

# Tail Home Assistant logs
ssh pi 'sudo kubectl logs -n home-assistant -l app=home-assistant -f'

# Tail Zigbee2MQTT logs
ssh pi 'sudo kubectl logs -n zigbee -l app=zigbee2mqtt -f'

# Restart HA after config change
ssh pi 'sudo kubectl rollout restart deployment/home-assistant -n home-assistant'

# Reset HA password (then restart HA, see gotchas)
ssh pi "sudo kubectl exec -n home-assistant deploy/home-assistant -- hass --script auth -c /config change_password 'devlinjack123@hotmail.com' '<NEW>'"

# Sync ArgoCD parent (so new commits in argocd-apps/ take effect)
ssh pi 'sudo kubectl patch application -n argocd argocd --type merge -p "{\"operation\":{\"sync\":{}}}"'

# Sync a specific app (replace <name>)
ssh pi 'sudo kubectl patch application -n argocd <name> --type merge -p "{\"operation\":{\"sync\":{}}}"'

# Apply the manually-managed PrometheusRule (until task #16 done)
cat applications/kube-prometheus-stack/prometheus-rules/smart-home-alerts.yaml | ssh pi 'sudo kubectl apply -f -'
```

## Working style with this user

- Lead with the concrete answer (URL, command, credential, file path). Skip generic explanations — he built the infra, he just doesn't remember the specifics between sessions.
- He explicitly wants candid critique when asked for opinions, not vague validation. Specific named problems beat general praise.
- He wants the foundations rock-solid *before* scaling devices. Current vision: full house lights, heating, motion/presence sensors, 4 security cameras, all local, all resilient.
