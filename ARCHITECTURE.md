# Project Architecture

This document provides a high-level overview of the microservices and Kubernetes deployment architecture managed by this repository.

## Overview

The infrastructure relies on Vagrant to provision virtual machines and Ansible to configure them. The core system is a multi-node K3s Kubernetes cluster running on these VMs.

## Architecture Diagram

```mermaid
graph TD
    User([User])

    subgraph Host Machine
        Ansible[Ansible Playbooks]
        Vagrant[Vagrant]
        Helm[Helm]
    end

    subgraph "Vagrant VMs (K3s Kubernetes Cluster)"
        direction TB
        CP(Control Plane Node)
        W1(Worker Node 1)
        W2(Worker Node 2)
        W3(Worker Node 3)
        W4(Worker Node 4)

        CP --- W1 & W2 & W3 & W4

        subgraph Cluster Services
            MetalLB[MetalLB LoadBalancer]
            Rancher[Rancher]
            KubeVirt[KubeVirt]
        end

        subgraph Workloads
            OTel[OpenTelemetry Demo]
        end
    end

    Vagrant -->|Provisions| CP
    Vagrant -->|Provisions| W1
    Vagrant -->|Provisions| W2
    Vagrant -->|Provisions| W3
    Vagrant -->|Provisions| W4

    Ansible -->|Deploys K3s & Services| CP

    Helm -->|Manually Deploys| OTel

    MetalLB -.->|Exposes Services| Rancher
    MetalLB -.->|Exposes Services| KubeVirt
    MetalLB -.->|Exposes Services| OTel

    User -->|Access via LoadBalancer IPs| MetalLB
```

### Components

- **Vagrant**: Provisions the base Virtual Machines using VirtualBox (or preferred provider).
- **Ansible**: Automates the installation of K3s and the deployment of core cluster services (MetalLB, Rancher, KubeVirt).
- **K3s Control Plane & Workers**: Lightweight Kubernetes cluster where the control plane manages the 4 worker nodes.
- **MetalLB**: Provides external LoadBalancer IPs for services within the bare-metal/VM environment.
- **Rancher**: Provides a Web UI for managing the Kubernetes cluster.
- **KubeVirt**: Extends Kubernetes to support managing VMs natively alongside containers.
- **OpenTelemetry Demo**: A sample microservices application deployed manually using Helm to demonstrate observability.
