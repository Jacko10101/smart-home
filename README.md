# Smart Home Infrastructure

A production-grade home automation platform leveraging Kubernetes, GitOps, and modern DevOps practices. This project implements enterprise infrastructure patterns to create a resilient, maintainable, and automated smart home system.

## Overview

The system runs on a Raspberry Pi 5 with a complete monitoring stack, managing everything from environmental sensors to security cameras through Kubernetes. By applying GitOps principles, the entire infrastructure is defined as code and automatically deployed through ArgoCD.

## Architecture

- **Infrastructure Platform**
  - Kubernetes (K3s) for container orchestration
  - ArgoCD for GitOps-based deployment
  - Prometheus & Grafana for monitoring
  - MQTT for device communication

- **Smart Home Components**
  - Home Assistant for automation control
  - Zigbee2MQTT for device integration
  - Alertmanager for system notifications
  - Custom Grafana dashboards for home metrics

## Key Features

- **Infrastructure as Code**: Complete system configuration version controlled in Git
- **Automated Deployment**: Zero-touch updates through GitOps workflows
- **Enterprise Monitoring**: Full observability with Prometheus metrics
- **Self-Healing**: Automatic recovery from component failures
- **Scalable Architecture**: Simple integration of new devices and services

## Technical Details

### Component Access

- ArgoCD: `https://<host>:30113`
- Grafana: `http://<host>:31295`
- Home Assistant: `http://<host>:8123`
- Prometheus: `http://<host>:30090`
- Alertmanager: `http://<host>:30723`
- Zigbee2MQTT: `http://<host>:31678`

### Supported Integrations

- Temperature/Humidity Sensors
- Motion Detection
- Smart Lighting
- Power Monitoring
- Security Cameras
- Smart Radiator Valves

## Implementation Highlights

- Kubernetes-managed home automation with high availability
- Monitoring and alerting system
- GitOps workflow for infrastructure management
- IoT device integration through MQTT protocol
- Automated deployment pipelines
- Real-time metrics and visualisations

## Development Roadmap

- Enhanced security monitoring
- ML-based presence detection
- Extended automation capabilities
- Network performance optimisation
- Additional device integrations
