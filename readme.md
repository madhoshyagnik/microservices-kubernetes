# Microservices & Kubernetes Learning Lab

Welcome to the automated K3s cluster deployment lab! This repository provisions a fully functional, multi-node K3s Kubernetes cluster on local Vagrant VMs using Ansible, and automatically deploys essential tools like **MetalLB**, **Rancher**, **KubeVirt**, and the **OpenTelemetry Demo**.

## 🚀 Features

- **Automated K3s Cluster**: 1 Control Plane, 4 Worker Nodes.
- **MetalLB**: Provides external LoadBalancer IPs for services.
- **Rancher**: Web UI for multi-cluster management.
- **KubeVirt**: Run Virtual Machines natively alongside containers.
- **OpenTelemetry Demo**: A comprehensive microservices application with observability tools (Grafana, Jaeger, Prometheus).

## 🛠️ Prerequisites

- [Vagrant](https://www.vagrantup.com/) & VirtualBox (or preferred provider)
- [Ansible](https://docs.ansible.com/)
- `kubectl` & `helm` installed on your host machine

---

## 🏃‍♂️ Getting Started

### 1. Bring up the Virtual Machines
This provisions the VMs as defined in the `Vagrantfile`.

```bash
vagrant up
```

### 2. Deploy the Cluster & Addons
The Ansible playbook will install K3s, configure the cluster, copy the kubeconfig to your host, and install all addons (MetalLB, Rancher, KubeVirt, and the OpenTelemetry Demo).

```bash
ansible-playbook -i inventory/inventory.yaml playbooks/deploy-k3s.yaml
```

*Note: The playbook is idempotent. You can safely re-run it if something fails.*

---

## 🎯 Accessing the Applications

Once the playbook completes, you can access your deployed services. Wait a few minutes for all pods to become ready (`kubectl get pods -A -w`).

### 🐮 Rancher
Rancher is exposed via a MetalLB LoadBalancer IP (usually `192.168.56.x`). Check the exact IP:
```bash
kubectl get svc rancher -n cattle-system
```
Access the IP in your browser via `https://<RANCHER_IP>`. (Default bootstrap password: `admin`).

### 📦 KubeVirt Manager
Manage your KubeVirt VMs from a web interface. Check the IP:
```bash
kubectl get svc kubevirt-manager -n kubevirt-manager
```
Access via `http://<KUBEVIRT_MANAGER_IP>`.

### 🔭 OpenTelemetry Demo
The demo frontend is exposed as a LoadBalancer. Check the IP:
```bash
kubectl get svc my-otel-demo-frontendproxy -n default
```
Access via `http://<OTEL_IP>:8080`.

*(Note: The Prometheus crash issue in local environments has been fixed by disabling persistent storage for Prometheus in the automated playbook!)*

---

## 🧹 Cleanup

To completely remove K3s and all installed components from the VMs without destroying the VMs themselves:

```bash
ansible-playbook -i inventory/inventory.yaml playbooks/uninstall-k3s.yaml
```

If you want to destroy the VMs entirely:
```bash
vagrant destroy -f
```

## 📖 Architecture & Details

For deeper insights into the configuration, check out the [Documentation folder](./Documentation), which contains the manual steps that were converted into this automated deployment, and explanations of Kubernetes concepts.