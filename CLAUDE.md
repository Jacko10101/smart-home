# Smart Home — Claude Context

Personal smart-home automation on a single Raspberry Pi 5 running k3s. Built and operated solo. The code in this repo is the source of truth for everything that runs on the cluster.

## North star

The goal is a **resilient, smart, useful, elegant, fully-local** smart home. Every decision — device choice, integration, automation, infra change — gets weighed against these, in roughly this order:

- **Fully local — no cloud, ever.** Every integration works with the internet unplugged. No vendor clouds (no Nabu Casa Cloud, no Tuya, no Hue Bridge cloud features, no Ring/Nest), no cloud-only devices, no cloud APIs in the critical path. If a device only works via someone else's server, it does not go in.
- **Resilient.** Survives reboots, power blips, k3s upgrades, a dead SSD, and Jack being away for a month. Failures are observable (alerting actually reaches a phone) and recoverable (off-pi backups, idempotent GitOps, physical fallbacks for anything safety-critical like heating).
- **Smart and useful.** Automations have to pay rent — presence-driven lighting, real heating logic, NVR with actual detection. Not just remote-controlled bulbs that need a phone to toggle.
- **Elegant.** Few moving parts. Boring, well-understood tech preferred over the newest thing. Given two designs that achieve the same outcome, pick the one with fewer components and clearer failure modes. No accidental complexity.

**Decision lens:** would this still work if Anthropic, AWS, Nabu Casa, the vendor's servers, and the household's upstream ISP all went dark simultaneously? If not, redesign.

## Quick orientation

- **Pi access**: `ssh pi` (alias for `raspi` / `raspberry`) → user `admin`, over **wired eth0 at `192.168.1.235`** — the *only* network path now (onboard WiFi is disabled, see Currently fragile). `kubectl` requires `sudo` because k3s config is root-readable only.
  - `.235` is **reserved at the BT hub** (eth0 MAC `2c:cf:67:6d:23:0d`) **and** pinned in k3s (`/etc/rancher/k3s/config.yaml` → `node-ip: 192.168.1.235`). It won't drift. (History: the IP musical-chaired `.229`/`.230`/`.235` on 2026-06-19/20 before this was locked down.)
- **What's running**: Home Assistant, Zigbee2MQTT, Mosquitto (MQTT broker), ArgoCD (GitOps), kube-prometheus-stack (Prometheus + Alertmanager; Grafana disabled), alertmanager-ntfy adapter, daily backup CronJob.
- **Service URLs on the LAN** (NodePort, not the in-cluster ports the upstream docs suggest):
  | Service | URL |
  |---|---|
  | Home Assistant | `http://192.168.1.235:31123` |
  | Zigbee2MQTT | `http://192.168.1.235:31678` |
  | ArgoCD | `http://192.168.1.235:30113` |
  | Prometheus | `http://192.168.1.235:31090` |
  | Alertmanager | `http://192.168.1.235:30723` |
- **mDNS `homeassistant.local` does NOT broadcast** on this network — always use the IP.
- **No remote access currently configured.** LAN only.

## GitOps flow (and its quirks)

1. The `argocd` app (`argocd-apps/argocd.yaml`) is the "app of apps" — it watches the `argocd-apps/` directory and reconciles all Application objects.
2. **`argocd` app has no auto-sync** — new commits to `main` are detected (OutOfSync) but require manual sync.
3. Per-app sync behaviour:
   - `kube-prometheus-stack`, `smart-home-alerts`: auto-sync + self-heal ON.
   - `home-assistant`, `zigbee2mqtt`, `mosquitto`, `backup`, `alertmanager-ntfy`, `argocd-self`: no syncPolicy — manual sync only.
4. **kube-prometheus-stack helm values live inline** in `argocd-apps/kube-prometheus-stack.yaml`, NOT under `applications/`. Don't look for them there.
5. **PrometheusRules must carry the label `release: kube-prometheus-stack`** — that's what the chart's `ruleSelector` matches. The wrong label means the rule applies as a k8s resource but Prometheus silently never loads it.
6. **When a helm values change doesn't reflect in the cluster**, force a hard refresh (see operational commands). ArgoCD's manifest cache can hold stale templating.

## Alerting architecture

- Prometheus → Alertmanager → `alertmanager-ntfy` adapter → ntfy.sh topic `jack-smarthome-050101` → phone.
- **Severity → ntfy priority** (mapped in `applications/alertmanager-ntfy/configmap.yaml`):
  - `critical` firing → `urgent` (priority 5, wakes the phone).
  - `warning` firing → `low` (priority 2, silent tray notification).
  - resolved → `low` (silent).
- **Repeat intervals** (in alertmanager config in `argocd-apps/kube-prometheus-stack.yaml`):
  - `critical`: 6h (1–2 buzzes usually enough to act on a real problem).
  - `warning`: 4h.
- **Alertmanager state persists** to a 1Gi PVC. Without it (the prior emptyDir setup) every pod restart wiped silences + notification dedup state — silences would evaporate mid-outage.
- **Inhibit rule:** `LowBattery` firing suppresses `Zigbee2MQTTNoUpdates` — when we already know a sensor's battery is dying, the "no zigbee updates" alert tells us nothing new. Auto-clears when batteries are replaced.
- **Note (2026-06-18):** there are currently **no battery Zigbee devices** (all removed), so `LowBattery` / `Zigbee2MQTTNoUpdates` have nothing to watch until battery sensors are re-added.

## Zigbee devices & the living-room automation POC

**Current Zigbee inventory (all mains/USB — zero batteries):**
- 6× Hue White & Color E14 bulbs → `group.living_room_lights`.
- 2× Hue White & Color Ambiance **B22** bulbs (model 9290024896; z2m labels them "E27" but they're the B22 twin pack), both onboarded 2026-06-28, `power_on_behavior=off`, RGB only (same `color_temp`-rejection gotcha as the E14s):
  - `bedroom_light` — IEEE `0x001788010daadc60` → HA `light.bedroom_light`.
  - `utility_light` — IEEE `0x001788010daadca8` → HA `light.utility_light`. Utility room by the front door (2nd fridge, pet food); dad's in/out transit point — **prime presence-sensor candidate** (like the LR POC), no sensor yet.
  - NB: this firmware **does** expose working Hue `effect`s over z2m (candle/fireplace/sunset/colorloop/etc.). Bedroom scenes = 5 distinct *static* RGB looks (Bright/Warm/Nightlight/Calm/Sunset) + a `Colour Cycle` scene using the `colorloop` effect. To clear an effect back to static, send `{"effect":"none"}` (the `effect` attribute otherwise lingers stale in HA even after a plain colour command).
- `living_room_presence` — SONOFF **SNZB-06P mmWave** occupancy sensor (USB-powered). Exposes `binary_sensor.living_room_presence_occupancy` + `sensor.living_room_presence_illumination` (dim/bright). `occupancy_timeout` = 30s.
- `living_room_tv` — SP 242 plug on the TV. **Switch-only and never switched** (`power_on_behavior=on`; TV stays powered). Energy metering is junk (see gotcha).
- `hallway_lamp` — SP 242 plug (`switch.hallway_lamp`).
- `kitchen_heater`, `dining_room_tv` — SP 242 plugs physically **MISSING**; still in z2m (show unavailable in HA). Locate or remove.
- **Removed 2026-06-18:** both SNZB-02D temp/humidity sensors + the SNZB-03 PIR (dead batteries; mmWave replaces the PIR) and the SNZB-04 contact sensor (binned — no magnet).

**Living-room presence-lighting POC** (`automations.yaml`, mmWave-driven):
- `lr_presence_lights_on`: occupancy→on AND `sun.sun` `below_horizon` AND lights off → warm lights (brightness 115, `color_temp` 454 ≈ 2200K). Gated on the **sun**, not the 06P illumination (see gotcha).
- `lr_presence_lights_off`: occupancy off for 3 min → lights off. No TV-power guard (mmWave holds a still viewer; SP 242 metering unreliable). **Verified 2026-06-18:** fires ~3 min after the room empties.
- The dashboard "Auto Lighting" card toggles both automations (family pause switch).
- Movie-mode by TV-power sensing is **shelved** — the SP 242 can't measure the TV. `Movie` stays a one-tap scene.

**Dashboard:** 2 tabs (Home, Lights), YAML mode, HACS Mushroom cards. Climate/Energy tabs removed (temp sensors gone, SP 242 metering unusable).

## Verified gotchas

- **k3s `node-ip` drifts with DHCP unless pinned.** With no `node-ip` set, k3s registers whatever IP it sees at startup. When the DHCP lease later moved (`.229`→`.235`), k3s kept the **stale `.229`** as node InternalIP *and* the `kubernetes` API endpoint → a cascade of `TargetDown` / `KubeletInstanceUnreachable` / `KubeAPIInstanceUnreachable` / operator watch-errors (the home kept working — only monitoring broke). Fix: `node-ip: <ip>` in `/etc/rancher/k3s/config.yaml` + `sudo systemctl restart k3s`, **then** `kubectl rollout restart` the monitoring deployments/statefulsets so they reconnect to the corrected endpoint. Now pinned to `.235`.
- **z2m device rename does NOT change the HA entity_id.** HA freezes an MQTT-discovered entity's `entity_id` at first discovery (keyed on `unique_id`). Rename a z2m device *after* HA has already seen it and HA keeps the old id — often `0x<IEEE>_*`. Fixes: rename in z2m **before** first HA discovery, or rename the HA entity via the registry (`config/entity_registry/update` over the websocket API; the prom token has admin). The `living_room_presence` / `living_room_tv` entities came in as `0x..._*` and had to be registry-renamed. Registry renames live in `.storage/core.entity_registry` (in the daily backup, **not** git). **The race is hard to win by hand:** onboarding `bedroom_light` (2026-06-28), even publishing the z2m rename seconds after `interview_successful`, HA had *already* discovered the entity as `light.0x001788010daadc60` — MQTT discovery fires the moment the interview completes. Had to registry-rename all 5 entities (light + select/number/text/update) via the websocket API. Practical takeaway: assume you'll lose the race and just registry-rename afterward. Pure-stdlib websocket renamer used: see git history / scratchpad `ha_ws_rename.py` (handshake → `auth` with the prom token → `config/entity_registry/update`).
- **SP 242 (Tuya UK) plug energy metering is garbage.** `current` pins at 13A and `power` reads ~0–2W under a real ~100W load (verified on the live TV). Useless for power sensing / the Energy dashboard. Switching still works fine.
- **SNZB-06P `illumination` (dim/bright) lags and is circular** — it senses the very lights an automation controls, so it reads `bright` for a beat after lights-off, which caused missed auto-on turn-ons. Gate lighting on `sun.sun` `below_horizon`, not this sensor.
- **The Hue colour bulbs reject `color_temp` over z2m.** They report no valid mireds range, so any `light.turn_on` or scene carrying `color_temp` returns **HTTP 400 and silently no-ops** (this broke auto-on and the Daylight/Relax/Reading/Wake-Up scenes). Use **`rgb_color` for everything** — warm white ≈ `[255, 160, 85]`. All scenes are now RGB-only.
- **Zigbee2MQTT CrashLoopBackOff after pi reboot — FIXED 2026-06-11 via host udev rule.** Historically: the `fix-usb-permissions` init container chowns `/dev/ttyUSB0` to `root:18`, but only runs on pod creation; after a host reboot udev recreated the device as `root:dialout` and the chown never re-ran. Now `/etc/udev/rules.d/99-zigbee-dongle.rules` on the host (`SUBSYSTEM=="tty", ATTRS{idVendor}=="10c4", ATTRS{idProduct}=="ea60", GROUP="18", MODE="0660"`) sets `root:18` on every device (re)creation — verified end-to-end by breaking perms and re-triggering udev. If z2m ever crashloops on serial perms again, check this rule still exists; manual fallback: `ssh pi 'sudo kubectl rollout restart deployment/zigbee2mqtt -n zigbee'`.
- **HA password reset requires a pod restart.** `hass --script auth change_password` writes to `/config/.storage/auth_provider.homeassistant` on disk, but the running HA process holds the credential cache in memory. Login keeps returning `invalid_auth` until the pod is restarted.
- **HA configmap uses `subPath` mounts** for `configuration.yaml`, `scenes.yaml`, `automations.yaml`, `ui-lovelace.yaml`. ConfigMap updates do NOT propagate to the running pod with subPath mounts. After editing `applications/home-assistant/configmap.yaml`, restart the deployment.
- **Lovelace is in YAML mode**, not UI mode. All dashboard changes happen in `applications/home-assistant/configmap.yaml` under `ui-lovelace.yaml`, not from the HA UI.
- **HA's `homeassistant_last_updated_time_seconds` resets on HA pod restart.** Entities re-hydrate from storage at boot, so the metric reports "now" for everything until fresh updates arrive. Battery-conserving zigbee sensors that only heartbeat every 30–60 min will look "stale" for up to an hour after a restart. Alerts using this metric must include an HA-uptime > 2h guard, otherwise they false-positive on every HA restart.
- **SONOFF SNZB-02D temp/humidity sensors die at ~10% battery without further warning.** They report 10% for some time, then go fully silent. The `LowBattery` alert (warning, `< 20%`, iPhones excluded) is what catches it before death. Without it, the only signal is `Zigbee2MQTTNoUpdates` (looks like a network failure).
- **Z2M sensor heartbeat cadence is 30–60 min.** Any "no zigbee updates" threshold must be ≥ 90 min, otherwise it false-positives during stable conditions.
- **Helm `inhibit_rules:` in `alertmanager.config` REPLACES the chart's default inhibit rules** (it doesn't merge). If you want the chart defaults to remain, copy them back when adding new rules.
- **kube-prometheus-stack alertmanager defaults to `emptyDir`** for its data dir. Without an explicit `alertmanagerSpec.storage.volumeClaimTemplate`, every pod restart wipes silences and dedup state. PVC is already configured.

## Architecture notes

- Single-node k3s on Pi 5 (8GB), 1TB NVMe SSD, UPS, SONOFF Zigbee 3.0 USB coordinator.
- `local-path-provisioner` backs all PVCs at `/var/lib/rancher/k3s/storage/`.
- Backup CronJob runs at 3am daily: tars HA + z2m PVCs into `/backups` **on the same NVMe**. `BackupHasntRunRecently` alert (>36h gap) catches silent failures of the cron itself.
- HA recorder is sqlite (`home-assistant_v2.db`, small). Will need MariaDB/Postgres before ~50 entities.
- HA Prometheus integration enabled with `require_auth: true`. Bearer token mounted into Prometheus from the `home-assistant-prom-token` secret (the old in-git token was revoked).
- Mosquitto allows anonymous auth but service is `ClusterIP`-only (not exposed beyond the cluster). Worth tightening before adding heating or security-critical devices.

## Currently fragile

1. **Uplink is wired; onboard WiFi disabled (2026-06-20).** `eth0` (`.235`, reserved at the BT hub + pinned via k3s `node-ip`) runs to a BT WiFi Disc that backhauls to the router; the disc is on the UPS. **Onboard `wlan0` is disabled** (`sudo nmcli radio wifi off`, persists; re-enable with `nmcli radio wifi on`) — it was the source of DHCP churn (its lease drifted and bounced the Pi's IP twice in a day) and is the prime freeze suspect (`brcmfmac`). The Pi now has a single, stable network path. Remaining: (a) confirm the disc's backhaul to the router is solid — it's a mesh disc, check the BT app (suspected source of the *household* WiFi blips, separate from the Pi); (b) `nmcli radio wifi off` deactivates the radio but leaves the `brcmfmac` driver loaded — for full freeze-suspect removal, `dtoverlay=disable-wifi` in `/boot/firmware/config.txt` + reboot kills the chip at firmware level; (c) the freeze fix (wired + WiFi off) is still unvalidated over time — uptime is the proof. As of 2026-06-20 the Pi has been up ~2 days with no freeze.
2. **Backups are on the same NVMe as the data they're backing up.** Disk dies → backups die. Off-pi target (LAN device + offsite USB rotation) is the planned shape; no cloud target (violates North Star).
3. **systemd watchdog is enabled** (`RuntimeWatchdogSec=30s` via `/etc/systemd/system.conf.d/watchdog.conf`) — hard hangs auto-reboot within 30 seconds. Mitigation, not root-cause fix.

## Roadmap (things Jack wants to do)

Direction, not commitments. Don't push toward these until foundations work is solid — Jack is explicit about not scaling devices on shaky infra.

- **Off-pi backups.** LAN-resident target (NAS / second Pi / mini-PC) plus a USB SSD rotated physically offsite. Hardware decision pending.
- **Ethernet** via powerline adapters or a wired run.
- **Compute upgrade for Frigate + local LLM.** Pi 5 stays as the smart-home brain; a second compute node handles heavy workloads (camera object detection, voice STT/TTS, Ollama LLM). Open candidates: Mac mini M4 16/24GB, or Linux SFF with a GPU joined to the k3s cluster.
- **Security cameras (×4).** Reolink PoE → Frigate via Coral USB accelerator. Lives on the new compute node, not the Pi.
- **Whole-house lighting.** Wall-switch + dumb-bulb pattern preferred (always-on physical control + zigbee dimming behind it). Family home — ~20–30 switches.
- **Heating per room.** Smart TRV per radiator; boiler control depends on what's upstream and may be out of scope. High-stakes — physical fallback (TRV behaves like a normal valve when system is down) is a hard requirement.
- **Presence sensors throughout.** mmWave (Aqara FP2 etc.) for major rooms, PIR for transit zones. **Living room done (2026-06-18):** SNZB-06P mmWave drives the presence-lighting POC — the template to replicate per room.
- **Local voice control.** HA Voice Preview Edition satellites or DIY Wyoming protocol — depends on compute node choice.
- **Whole-home energy monitoring.** Shelly EM on the consumer unit so HA's Energy dashboard works at house level, not just per-plug.
- **Window contact sensors.** The `bedroom_window` SONOFF SNZB-04 (IEEE `0x00124b002fa64513`) was binned 2026-06-18 (missing magnet half) but lingered in z2m until **force-removed from the z2m registry 2026-06-28**. Re-add with a fresh contact sensor when expanding.
- **HA recorder → MariaDB/Postgres** before ~30 entities.
- **Remote access.** No fixed answer yet. Tailscale was tried and removed; revisit when there's a real use case.

## Common operational commands

```bash
# All pods status
ssh pi 'sudo kubectl get pods -A'

# Tail Home Assistant / Zigbee2MQTT logs
ssh pi 'sudo kubectl logs -n home-assistant -l app=home-assistant -f'
ssh pi 'sudo kubectl logs -n zigbee -l app=zigbee2mqtt -f'

# Restart HA after a configmap change (needed because of subPath mounts)
ssh pi 'sudo kubectl rollout restart deployment/home-assistant -n home-assistant'

# Reset HA password (then restart HA, see gotchas)
ssh pi "sudo kubectl exec -n home-assistant deploy/home-assistant -- hass --script auth -c /config change_password 'devlinjack123@hotmail.com' '<NEW>'"

# Sync ArgoCD parent (after committing changes to argocd-apps/)
ssh pi 'sudo kubectl patch application -n argocd argocd --type merge -p "{\"operation\":{\"sync\":{}}}"'

# Sync a specific app
ssh pi 'sudo kubectl patch application -n argocd <name> --type merge -p "{\"operation\":{\"sync\":{}}}"'

# Force ArgoCD hard refresh + resync (when helm values changes don't appear)
ssh pi 'sudo kubectl patch application -n argocd <name> --type merge -p "{\"metadata\":{\"annotations\":{\"argocd.argoproj.io/refresh\":\"hard\"}}}"; sleep 3; sudo kubectl patch application -n argocd <name> --type merge -p "{\"operation\":{\"sync\":{}}}"'

# Currently firing alerts
ssh pi 'curl -s http://localhost:30723/api/v2/alerts | python3 -m json.tool | head -40'

# Create an alertmanager silence (replace <ALERTNAME> and <DURATION>, e.g. "30 days")
ssh pi 'START=$(date -u +%Y-%m-%dT%H:%M:%S.000Z); END=$(date -u -d "now + <DURATION>" +%Y-%m-%dT%H:%M:%S.000Z); curl -sS -X POST http://localhost:30723/api/v2/silences -H "Content-Type: application/json" -d "{\"matchers\":[{\"name\":\"alertname\",\"value\":\"<ALERTNAME>\",\"isRegex\":false,\"isEqual\":true}],\"startsAt\":\"$START\",\"endsAt\":\"$END\",\"createdBy\":\"jack\",\"comment\":\"<why>\"}"'

# Active silences
ssh pi 'curl -s http://localhost:30723/api/v2/silences | python3 -c "import sys,json; [print(s[\"id\"][:8], s[\"matchers\"], s[\"endsAt\"]) for s in json.load(sys.stdin) if s[\"status\"][\"state\"]==\"active\"]"'
```

## Working style with this user

- Lead with the concrete answer (URL, command, credential, file path). Skip generic explanations — he built the infra, he just doesn't remember the specifics between sessions.
- He explicitly wants candid critique when asked for opinions, not vague validation. Specific named problems beat general praise.
- He wants the foundations rock-solid *before* scaling devices.
- **Apply the North Star above as a filter, not a slogan.** When recommending a device, integration, or design: if it requires a cloud, say so explicitly and propose a local alternative. If it adds a moving part without earning it, flag the simpler version. If it creates a new single-point-of-failure, name it.
- Watch for over-engineering. If you suggest something complex, also surface the simpler alternative and let him pick.
