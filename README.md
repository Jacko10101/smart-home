# My Smart Home Setup

A self-hosted home automation system running on Kubernetes. Built with privacy and local control in mind - no cloud services required.

## Overview

This is my personal smart home setup that runs entirely on a Raspberry Pi 5 using K3s Kubernetes. It features:

- Fully local architecture (no cloud dependencies)
- Remote access through Tailscale VPN 
- GitOps deployments with ArgoCD
- Rich monitoring and power usage tracking
- Scalable foundation for future expansion

## Current Setup

### Core Hardware
- Raspberry Pi 5 (8GB) running K3s
- 1TB NVMe SSD
- UPS backup power
- SONOFF Zigbee USB coordinator
- Wired ethernet connection

### Smart Devices

My current device setup spans several rooms:

#### Bedroom
- 6x Philips Hue Color E14 bulbs for adaptive lighting
- SONOFF motion sensor for occupancy detection
- SONOFF contact sensor on window
- Custom scenes for different activities:
 - Wake Up: Gentle morning brightness increase
 - Gaming: Immersive RGB lighting
 - Movie Mode: Subtle ambient lighting
 - Evening Relax: Warm wind-down lighting
 - Night Reading: Focused task lighting
 - Party Mode: Dynamic RGB effects

#### Environmental Monitoring
- SONOFF LCD temp/humidity sensor in living room
- SONOFF LCD temp/humidity sensor in hallway
- Real-time tracking through Grafana dashboards

#### Power Monitoring
- Innr smart plugs with energy tracking:
 - Hallway lamp
 - Kitchen heater 
 - Dining room TV
- Power usage visualization and trending

#### Security
- Reolink outdoor camera with solar power
- Motion and contact sensor alerts
- Secure remote access via Tailscale VPN

## Software Stack

### Core Services
- **Home Assistant**: Central control and automation
- **Zigbee2MQTT**: Device management 
- **Mosquitto**: MQTT message broker
- **ArgoCD**: GitOps deployment
- **Prometheus & Grafana**: Monitoring and visualization

### Key Features
- Temperature and humidity tracking
- Power consumption monitoring
- Custom lighting scenes
- Motion-based automations
- Secure remote access

## Monitoring & Analytics

Current monitoring includes:
- Real-time power usage per device
- Environmental data trends
- Motion events and occupancy
- Light status and usage patterns
- System performance metrics

## Upcoming Projects

### Near Term
- Installing smart TRV radiator valves for room-by-room heating control
- Expanding security setup with more cameras and sensors
- Setting up presence detection
- Creating more advanced automations

### Future Plans  
- Extending coverage to more rooms
- Adding energy optimisation features
- Improving monitoring dashboards
- Integrating additional sensor types
