# Rancher Ansible Role

Ansible role to automate the deployment of Rancher on a K3s Kubernetes cluster with cert-manager and MetalLB LoadBalancer service integration.

## Description

This role automates:
1. Verifying preflight requirements (`kubectl`, `helm`, `kubeconfig`, cluster connectivity).
2. Installing `cert-manager` using Helm.
3. Deploying Rancher via Helm in the `cattle-system` namespace.
4. Exposing the Rancher service as a MetalLB `LoadBalancer` with proper pod label selectors (`app: rancher`).

## Role Variables

Settable variables in `defaults/main.yml`:

| Variable | Default Value | Description |
| --- | --- | --- |
| `rancher_namespace` | `"cattle-system"` | Namespace for Rancher deployment |
| `rancher_hostname` | `"rancher.madhoshyagnik.com"` | Hostname for Rancher server |
| `rancher_bootstrap_password` | `"admin"` | Initial bootstrap password |
| `rancher_replicas` | `1` | Number of Rancher replicas |
| `cert_manager_namespace` | `"cert-manager"` | Namespace for cert-manager deployment |
| `install_cert_manager` | `true` | Whether to install cert-manager prior to Rancher |

## Example Usage

```yaml
- name: Install Cluster Addons
  hosts: localhost
  connection: local
  gather_facts: false
  roles:
    - role: ../roles/rancher
```

## Requirements

* `kubectl`
* `helm`
* Active Kubernetes cluster with `kubeconfig` available at `~/.kube/config`
