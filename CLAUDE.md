# Smart Home — Claude Context

Personal smart-home automation running on a single Raspberry Pi 5 with k3s. Built and operated solo. The code in this repo is the source of truth for everything that runs on the cluster.

## North star

The goal is a **resilient, smart, useful, elegant, fully-local** smart home. Every decision — device choice, integration, automation, infra change — gets weighed against these, in roughly this order:

- **Fully local — no cloud, ever.** Every integration works with the internet unplugged. No vendor clouds (no Nabu Casa Cloud, no Tuya, no Hue Bridge cloud features, no Ring/Nest), no cloud-only devices, no cloud APIs in the critical path. If a device only works via someone else's server, it does not go in. Tailscale for remote access is fine because the home keeps working when it's down.
- **Resilient.** Survives reboots, power blips, k3s upgrades, a dead SSD, and Jack being away for a month. Failures are observable (alerting actually reaches a phone) and recoverable (off-pi backups, idempotent GitOps, physical fallbacks for anything safety-critical like heating).
- **Smart and useful.** Automations have to pay rent — presence-driven lighting, real heating logic, NVR with actual detection. Not just remote-controlled bulbs that need a phone to toggle.
- **Elegant.** Few moving parts. Boring, well-understood tech preferred over the newest thing. Given two designs that achieve the same outcome, pick the one with fewer components and clearer failure modes. No accidental complexity.

**Decision lens:** would this still work if Anthropic, AWS, Nabu Casa, the vendor's servers, and the household's upstream ISP all went dark simultaneously? If not, redesign.

## Quick orientation

- **Pi access**: `ssh pi` (alias for `raspi` / `raspberry`) → user `admin`, host `192.168.1.235`. `kubectl` requires `sudo` because k3s config is root-readable only.
- **What's running**: Home Assistant, Zigbee2MQTT, Mosquitto (MQTT broker), ArgoCD (GitOps), kube-prometheus-stack (Prometheus + Alertmanager; Grafana disabled), nightly backup CronJob.
- **Service URLs on the LAN** (these are all NodePort, NOT the in-cluster ports the upstream docs suggest):
  | Service | URL |
  |---|---|
  | Home Assistant | `http://192.168.1.235:31123` |
  | Zigbee2MQTT | `http://192.168.1.235:31678` |
  | ArgoCD | `http://192.168.1.235:30113` |
  | Prometheus | `http://192.168.1.235:31090` |
  | Alertmanager | `http://192.168.1.235:30723` |
- **mDNS `homeassistant.local` does NOT broadcast** on this network — always use the IP.

## GitOps flow (and its quirks)

1. The `argocd` app (`argocd-apps/argocd.yaml`) is the "app of apps" — it watches the `argocd-apps/` directory and reconciles all Application objects.
2. **`argocd` app has no auto-sync** — new commits to `main` are detected (OutOfSync) but require manual sync (`kubectl patch application -n argocd argocd --type merge -p '{"operation":{"sync":{}}}'` or via UI).
3. Once the parent syncs, individual app sync behavior varies:
   - `kube-prometheus-stack`: auto-sync + self-heal ON. Inline Helm values in `argocd-apps/kube-prometheus-stack.yaml`.
   - All others (`home-assistant`, `zigbee2mqtt`, `mosquitto`, `backup`): no syncPolicy — manual sync only.
4. **`smart-home-alerts.yaml` IS in ArgoCD** via `argocd-apps/smart-home-alerts.yaml` (auto-sync + self-heal). Commit changes to `applications/kube-prometheus-stack/prometheus-rules/smart-home-alerts.yaml` and they reconcile automatically — no manual `kubectl apply` needed.

## Verified gotchas

- **Zigbee2MQTT CrashLoopBackOff after pi reboot.** The `fix-usb-permissions` init container chowns `/dev/ttyUSB0` to `root:18`, but only runs on pod creation. After a host reboot, udev recreates `/dev/ttyUSB0` with default `root:dialout` (GID 20) and k8s only restarts the failed main container, so the chown never re-runs. **Fix:** `ssh pi 'sudo kubectl rollout restart deployment/zigbee2mqtt -n zigbee'`. Permanent fix queued (task #3 — udev rule on host).
- **HA password reset requires a pod restart.** `hass --script auth change_password` writes to `/config/.storage/auth_provider.homeassistant` on disk, but the running HA process holds the credential cache in memory. Login keeps returning `invalid_auth` until the pod is restarted. After any password reset: `ssh pi 'sudo kubectl rollout restart deployment/home-assistant -n home-assistant'`.
- **HA configmap uses `subPath` mounts** for `configuration.yaml`, `scenes.yaml`, `automations.yaml`, `ui-lovelace.yaml`. ConfigMap updates do NOT propagate to the running pod with subPath mounts. After editing `applications/home-assistant/configmap.yaml`, restart the deployment.
- **Lovelace is in YAML mode**, not UI mode. All dashboard changes happen in `applications/home-assistant/configmap.yaml` under `ui-lovelace.yaml`, not from the HA UI.
- **HA entity IDs are wrong.** The 6 living-room Hue bulbs are `light.bedroom_left_1` through `light.bedroom_right_2` because they were originally paired in the bedroom. Scenes, dashboards, automations all carry the wrong name. Rename queued (task #5) — easier now while there are 6 than later when there are 30.

## Architecture notes

- Single-node k3s on Pi 5 (8GB), 1TB NVMe SSD, UPS, SONOFF Zigbee 3.0 USB coordinator.
- `local-path-provisioner` backs all PVCs at `/var/lib/rancher/k3s/storage/`.
- No remote access currently configured. Tailscale is uninstalled; LAN-only via the NodePorts above.
- Backup CronJob (3am daily) tars HA + z2m PVC contents to `/backups` **on the same NVMe** — not off-pi. Task #2 to fix.
- HA recorder is sqlite (`home-assistant_v2.db` ~11MB currently). Will need MariaDB/Postgres before ~50 entities (task #6).

## Currently fragile, in priority order

1. **Pi is on WiFi**, not ethernet. `eth0` is NO-CARRIER (cable physically disconnected). All traffic goes via brcmfmac driver, which spammed errors before every recorded crash. Strongly suspected as root cause of the recurring hangs (Jan/Feb/Mar 2026, latest March 12 → 2-month outage until May 15). Task #9.
2. **Backups not off-pi.** Same disk dies → backups die. Two-month silent backup gap in 2026 went undetected (no alert wired up). Task #2.
3. **HA long-lived access token committed in plaintext** at `argocd-apps/kube-prometheus-stack.yaml:65` (10-year expiry). Assume compromised, must rotate. Task #13.
4. **Grafana admin password is the default** (`prom-operator`). Task #14.
5. **systemd watchdog is on** (`RuntimeWatchdogSec=30s` via `/etc/systemd/system.conf.d/watchdog.conf`) — hard hangs now auto-reboot within 30 seconds. Big win, recently enabled.

## Roadmap (things Jack wants to do)

Direction, not commitments. Don't push toward these until foundations work above is done — Jack is explicit about not scaling devices on shaky infra.

- **Off-pi backups.** Highest priority foundation item left. Current direction: a LAN-resident target (second device, NAS, or a second Pi) plus a USB SSD rotated physically offsite. No cloud target (violates North Star). Hardware decision ~2 weeks out.
- **Ethernet via powerline adapters.** Eliminate the brcmfmac WiFi-driver hypothesis for the Jan–Mar 2026 hangs. Watchdog mitigates but doesn't fix root cause.
- **Security cameras (×4).** Reolink → Frigate via Coral USB accelerator. Will be a new ArgoCD app; needs to evaluate memory headroom on the Pi and storage planning for recordings.
- **Whole-house lighting.** Open to the wall-switch + dumb-bulb pattern (keep physical control + zigbee dimmer behind it). Currently only living room is wired.
- **Smart TRV in one room first.** Boiler thermostat is upstream and out of scope. Heating is high-stakes — physical fallback (manual TRV behaviour when system is down) is a hard requirement.
- **Presence/motion/temp sensors throughout.** mmWave (Aqara FP2 etc.) preferred over PIR for presence accuracy.
- **Window contact sensors.** `bedroom_window` SONOFF sensor is paired (IEEE `0x00124b002fa64513`, battery 100%) but is **missing the magnet half** so it can't actually detect open/closed. Find or replace the magnet, then more sensors elsewhere.
- **HA recorder → MariaDB/Postgres.** Defer until close to ~30 entities. Currently sqlite ~11MB, fine.
- **Remote access.** No fixed answer yet. Tailscale was tried and removed; revisit when there's a real use case.

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
```

## Working style with this user

- Lead with the concrete answer (URL, command, credential, file path). Skip generic explanations — he built the infra, he just doesn't remember the specifics between sessions.
- He explicitly wants candid critique when asked for opinions, not vague validation. Specific named problems beat general praise.
- He wants the foundations rock-solid *before* scaling devices. Current vision: full house lights, heating, motion/presence sensors, 4 security cameras, all local, all resilient.
- **Apply the North Star (above) as a filter, not a slogan.** When recommending a device, integration, or design: if it requires a cloud, say so explicitly and propose a local alternative. If it adds a moving part without earning it, flag the simpler version. If it creates a new single-point-of-failure, name it.
