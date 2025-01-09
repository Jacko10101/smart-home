# 🏠 Smart Home on K3s

A powerful home automation system running on Kubernetes, built with privacy and local control at its heart. All your smart home needs running on a Raspberry Pi - no cloud required!

## ✨ What's This All About?

I've built a modern smart home system that:
- Runs 100% locally (no cloud dependencies)
- Keeps your data private and secure
- Uses enterprise-grade tech (Kubernetes) in a home setting
- Provides fancy features like adaptive lighting and power monitoring
- Scales beautifully as you add more devices

Perfect for both tech enthusiasts looking to self-host their smart home AND anyone wanting a reliable, private home automation setup.

## 🛠 Current Setup

### The Brain
- **Raspberry Pi 5** (8GB) running K3s Kubernetes
- 1TB NVMe SSD for snappy performance
- UPS backup power because reliability matters
- Hardwired ethernet connection
- SONOFF Zigbee USB coordinator for device control

### The Devices

#### Ambient Lighting
- **Bedroom**: 6x Philips Hue Color E14 bulbs featuring:
 - Wake Up: Gentle sunrise simulation
 - Gaming: Immersive RGB lighting
 - Movie Mode: Perfect ambient lighting for films
 - Evening Relax: Warm, cozy lighting
 - Night Reading: Optimized reading light
 - Party Mode: Dynamic RGB effects

#### Smart Power Management
Innr Smart Plugs with real-time power monitoring:
- Hallway lamp
- Kitchen heater
- Dining room TV
- Full energy usage tracking and trends

#### Environmental Monitoring
- Living room temperature & humidity (SONOFF LCD sensor)
- Hallway temperature & humidity (SONOFF LCD sensor)
- Real-time environmental tracking
- Historical trending

#### Security & Safety
- Reolink outdoor camera (solar powered)
- Bedroom motion detection
- Window contact sensing
- All accessible securely from anywhere via Tailscale VPN

## 🚀 Technical Stack

### Core Services
1. **Home Assistant**
  - Central automation hub
  - Custom scenes and routines
  - Device management
  - Accessible via `http://<tailscale-ip>:31123`

2. **Zigbee2MQTT**
  - Manages all Zigbee devices
  - Direct MQTT integration
  - Accessible via `http://<tailscale-ip>:31678`

3. **ArgoCD**
  - GitOps deployment
  - Infrastructure as code
  - Accessible via `http://<tailscale-ip>:30113`

4. **Monitoring Stack**
  - Prometheus metrics collection
  - Grafana dashboards
  - Power usage visualization
  - Environmental data tracking
  - Accessible via `http://<tailscale-ip>:31295`

### For the Techies
All services run as Kubernetes deployments managed by ArgoCD, with:
- Persistent volume claims for data storage
- ConfigMap-based configuration
- NodePort service exposure
- Tailscale VPN integration
- MQTT messaging backbone

## 📊 Current Monitoring

Real-time tracking of:
- Power consumption per device
- Temperature and humidity trends
- Motion events and occupancy
- Light usage patterns
- System performance
- Device battery levels

## 🔒 Security First

- Zero ports exposed to the internet
- Tailscale VPN for secure remote access
- Regular automated updates via GitOps
- UPS protection against power outages
- Local data storage
- Encrypted communications

## 🎯 What's Next?

### Active Projects
- Smart TRV radiator installation for room-by-room heating
- Enhanced security system with more cameras and sensors
- Presence detection improvements
- Advanced automation sequences

### Future Vision
- Whole house coverage
- Energy optimisation
- Advanced monitoring dashboards
- Machine learning integration
