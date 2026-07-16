# Provisioning the K3s Cluster Using Ansible

This repository includes an Ansible playbook that automates the complete K3s cluster provisioning process.

The playbook performs the following tasks:

- Updates the package cache
- Upgrades installed packages
- Installs required dependencies
- Reboots nodes if required
- Installs the K3s server
- Joins worker nodes to the cluster
- Waits for all nodes to become Ready
- Taints the control plane
- Retrieves the kubeconfig to the host
- Configures the local `kubectl`

---

## Directory Structure

```text
.
├── inventory.yaml
└── playbook.yaml
```

---

## Inventory

Update the inventory with your node IP addresses and K3s token.

Example:

```yaml
all:
  vars:
    ansible_user: vagrant
    ansible_python_interpreter: /usr/bin/python3

    k3s_server_ip: 192.168.56.11
    k3s_token: "DkPS01xep_{8"

  children:
    control:
      hosts:
        debian1:
          ansible_host: 192.168.56.11

    workers:
      hosts:
        debian2:
          ansible_host: 192.168.56.12

        debian3:
          ansible_host: 192.168.56.13

        debian4:
          ansible_host: 192.168.56.14

    k3s:
      children:
        control:
        workers:
```

---

## Run the Playbook

Execute the playbook:

```bash
ansible-playbook -i inventory.yaml playbook.yaml
```

---

## What the Playbook Does

### Bootstrap

- Updates the package cache
- Upgrades installed packages
- Installs:
  - curl
  - wget
  - git
  - vim
  - htop
  - jq
- Reboots hosts if required

### Control Plane

- Installs the K3s server
- Enables the K3s service
- Waits for the Kubernetes API server

### Worker Nodes

- Installs the K3s agent
- Joins each worker to the cluster
- Enables the K3s agent service

### Cluster Configuration

- Waits until all worker nodes join the cluster
- Taints the control plane
- Copies the kubeconfig from the control plane
- Fetches the kubeconfig to the host
- Updates the API server address
- Verifies the cluster

---

## Expected Output

Successful execution should display all nodes in the **Ready** state.

Example:

```text
NAME      STATUS   ROLES           AGE   VERSION
debian1   Ready    control-plane   2m    v1.36.2+k3s1
debian2   Ready    <none>          1m    v1.36.2+k3s1
debian3   Ready    <none>          1m    v1.36.2+k3s1
debian4   Ready    <none>          1m    v1.36.2+k3s1
```

---

## Verify the Cluster

```bash
kubectl get nodes -o wide

kubectl get pods -A

kubectl cluster-info
```

---

## Re-running the Playbook

The playbook is idempotent.

Re-running it will:

- Skip K3s installation if already installed
- Ensure required packages are installed
- Ensure services are running
- Refresh the local kubeconfig
- Reapply the control-plane taint if necessary

```bash
ansible-playbook -i inventory.yaml playbook.yaml
```

---

## Uninstalling K3s

### Control Plane

```bash
sudo /usr/local/bin/k3s-uninstall.sh
```

### Worker Nodes

```bash
sudo /usr/local/bin/k3s-agent-uninstall.sh
```

(Optional)

```bash
sudo rm -rf /etc/rancher /var/lib/rancher /var/lib/kubelet
sudo reboot
```
