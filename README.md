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
| Living Room | SONOFF SNZB-02D | Temperature/Humidity LCD | Environmental monitoring |
| Hallway | SONOFF SNZB-02D | Temperature/Humidity LCD | Environmental monitoring |
| Bedroom | SONOFF SNZB-03 | Motion Sensor | Presence detection |
| Bedroom | SONOFF SNZB-04 | Contact Sensor | Window monitoring |
| Bedroom | 6x Philips Hue | Color E14 Bulbs | Ambient lighting |
| Hallway | Innr SP 242 | Smart Plug | Lamp control + energy monitoring |
| Kitchen | Innr SP 242 | Smart Plug | Heater control + energy monitoring |
| Dining Room | Innr SP 242 | Smart Plug | TV control + energy monitoring |
| Outdoor | Reolink | Security Camera | Surveillance with solar panel |

## 🎨 Lighting Scenes

The bedroom features sophisticated lighting control with predefined scenes:
- **Wake Up**: Gradual 20-minute brightness increase with warm light
- **Daylight**: Bright, cool lighting for productivity
- **Gaming**: Immersive RGB lighting with blue/red contrast
- **Movie**: Minimal ambient lighting for viewing
- **Evening Relax**: Warm, dimmed lighting for winding down
- **Night Reading**: Focused reading light with ambient background
- **Party**: Full RGB color mix for entertainment

## 🚀 Core Components

### 1. ArgoCD (GitOps Engine)
- Automatically syncs your Git repository with the cluster
- Manages all application deployments
- Accessible via Tailscale

### 2. Home Assistant
- Central automation hub
- Custom lighting scenes
- Power monitoring and energy tracking
- Temperature/humidity monitoring
- Accessible via Tailscale

### 3. Zigbee2mqtt
- Manages all Zigbee devices (lights, sensors, plugs)
- Web interface for device management
- Direct MQTT integration with Home Assistant
- Accessible via Tailscale

### 4. Mosquitto (MQTT Broker)
- Message broker for device communication
- Enables reliable IoT device messaging
- Configured for local-only access

### 5. Monitoring Stack
- **Prometheus & Grafana**:
  - Environmental data visualization
  - Power usage monitoring
  - Device status tracking
  - Temperature trends
  - Light usage patterns

## 📊 Current Monitoring

- Real-time power consumption for smart plugs
- Temperature and humidity trends
- Motion detection events
- Window state monitoring
- Light usage patterns
- Device battery levels
- System performance metrics

## 🔒 Security Features

- All services accessible only through Tailscale VPN
- No ports exposed to the internet
- Regular automatic updates via GitOps
- UPS protection against power failures
- Local-only data storage
- Encrypted remote access

## 🎯 Future Plans

### Short Term
- [ ] Implement advanced automation sequences
- [ ] Create more sophisticated lighting scenes
- [ ] Expand energy usage analytics
- [ ] Enhance temperature-based controls

### Long Term
- [ ] Add more environmental sensors
- [ ] Implement presence-based automation
- [ ] Expand security features
- [ ] Develop custom Grafana dashboards

*Building a smarter home, one device at a time!* 🏠✨