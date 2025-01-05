# Smart Home on K3s

A production-ready, fully local home automation system built on Kubernetes (K3s). This project demonstrates how to create a modern, secure, and scalable smart home infrastructure using GitOps principles, Tailscale VPN, and comprehensive monitoring - all while keeping your data private and under your control.

## 🌟 Overview

Transform your home into a smart living space with this enterprise-grade setup that prioritizes:
- Local-first architecture (no cloud dependencies)
- Security and privacy through Tailscale VPN
- GitOps-driven deployments with ArgoCD
- Comprehensive monitoring and alerting
- Scalable Kubernetes (K3s) foundation

## 🏗 Infrastructure

### Hardware Setup
- **Core System**: Raspberry Pi 5 (8GB RAM)
- **Storage**: 1TB NVMe SSD for high-performance data access
- **Power**: CyberPower CP900EPFCLCD UPS for continuous operation
- **Network**: Wired CAT8 Ethernet for reliable connectivity
- **Zigbee**: SONOFF Zigbee 3.0 USB Dongle Plus (coordinator)

### Smart Devices
Currently integrated devices:

| Location | Device | Type | Purpose |
|----------|--------|------|---------|
| Living Room | SONOFF SNZB-02D | Temperature/Humidity | Environmental monitoring |
| Hallway | SONOFF SNZB-02D | Temperature/Humidity | Environmental monitoring |
| Bedroom | SONOFF SNZB-03 | Motion Sensor | Presence detection |
| Bedroom | SONOFF SNZB-04 | Contact Sensor | Window monitoring |

## 🚀 Core Components

### 1. ArgoCD (GitOps Engine)
- Automatically syncs your Git repository with the cluster
- Manages all application deployments
- Enables version control for your entire home automation setup
- Accessible via: `https://<local-ip>:30113` or `http://<tailscale-ip>:30113`

### 2. Home Assistant
- Central automation hub
- Custom configuration via ConfigMap
- Persistent storage for settings and history
- Prometheus metrics integration
- Accessible via: `http://<local-ip>:31123` or `http://<tailscale-ip>:31123`

### 3. Zigbee2mqtt
- Manages all Zigbee devices
- Web interface for device management
- Direct MQTT integration with Home Assistant
- Accessible via: `http://<local-ip>:31678` or `http://<tailscale-ip>:31678`

### 4. Mosquitto (MQTT Broker)
- Message broker for device communication
- Enables reliable IoT device messaging
- Configured for local-only access
- Runs on standard port 1883

### 5. Monitoring Stack
- **Prometheus**: Metrics collection and storage
  - 30-day retention
  - Home Assistant integration
  - Accessible via: `http://<local-ip>:30090`
- **Grafana**: Visualization and dashboards
  - Accessible via: `http://<local-ip>:31295`
- **Alertmanager**: Alert handling
  - Accessible via: `http://<local-ip>:30723`

## 🛠 Installation

1. **Prerequisites**
   ```bash
   # Install K3s
   curl -sfL https://get.k3s.io | sh -

   # Install Helm (required for kube-prometheus-stack)
   curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
   ```

2. **Deploy ArgoCD**
   ```bash
   kubectl create namespace argocd
   kubectl apply -f argocd-apps/argocd.yaml
   ```

3. **Configure Tailscale**
   ```bash
   curl -fsSL https://tailscale.com/install.sh | sh
   sudo tailscale up --advertise-routes=10.43.0.0/16 --accept-routes
   ```

4. **Verify Deployments**
   ```bash
   kubectl get applications -n argocd
   kubectl get pods -A
   ```

## 📊 Monitoring & Management

### Current Metrics
- Device battery levels
- Temperature and humidity readings
- Motion detection events
- Window state changes
- System resource usage

### Grafana Dashboards
- Home Assistant metrics
- Environmental data trends
- Device status overview
- System performance metrics

## 🔒 Security Considerations

- All services accessible only through Tailscale VPN
- No ports exposed to the internet
- Regular automatic updates via GitOps
- UPS protection against power failures
- Local-only data storage

## 🎯 Future Plans

### Short Term
- [ ] Integrate Philips Hue lighting system
- [ ] Add Innr smart plugs with energy monitoring
- [ ] Enhance automation based on motion and temperature
- [ ] Expand Grafana dashboards

### Long Term
- [ ] Implement ML-based presence detection
- [ ] Add energy usage optimization
- [ ] Integrate weather-based controls
- [ ] Develop mobile app interface

*Building a smarter home, one device at a time!* 🏠✨