# Provisioning the K3s Cluster Using Ansible

> [!IMPORTANT]
> **Troubleshooting: Multi-Node Flannel Networking / Cluster DNS Issues**
> * **Issue**: In Vagrant-based multi-node setups, Flannel CNI defaults to binding on the `eth0` NAT interface, causing all nodes to share the same tunnel IP (`10.0.2.15`). This breaks cross-node pod communication, causing cluster DNS (`coredns`) and services to fail.
> * **Resolution**: Configured K3s on both the control plane and agents to bind Flannel explicitly to the private network interface (`eth1`) by passing the `--flannel-iface eth1` argument.

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
├── deploy-k3s.yaml (Deploys the K3s cluster)
├── uninstall-k3s.yaml (Uninstalls K3s and resets state)
├── doc/
│   └── readme-manual.md (Manual VM setup guide)
└── roles/
    └── k3s/ (Modular Ansible role for K3s tasks)
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

Before applying the changes, you can perform a dry run (check mode) to preview what tasks Ansible will execute:

```bash
ansible-playbook -i inventory.yaml deploy-k3s.yaml --check
```

To run and apply the playbook:

```bash
ansible-playbook -i inventory.yaml deploy-k3s.yaml
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
ansible-playbook -i inventory.yaml deploy-k3s.yaml
```

---

## Uninstalling / Reverting K3s

To automatically uninstall K3s from all cluster nodes, clean up configuration folders, and reboot the machines, run the uninstall playbook on the host:

```bash
ansible-playbook -i inventory.yaml uninstall-k3s.yaml
```

### Manual Uninstall (Fallback)

If you prefer to uninstall manually on each node:

**Control Plane:**
```bash
sudo /usr/local/bin/k3s-uninstall.sh
```

**Worker Nodes:**
```bash
sudo /usr/local/bin/k3s-agent-uninstall.sh
```

**Cleanup & Reset:**
```bash
sudo rm -rf /etc/rancher /var/lib/rancher /var/lib/kubelet /home/vagrant/k3s.yaml
sudo reboot
```


---

## Deploying the OpenTelemetry Demo

Once your cluster is fully provisioned and healthy, you can deploy the OpenTelemetry Demo:

1. **Add the OpenTelemetry Helm repository:**
   ```bash
   helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
   helm repo update
   ```

2. **Install the OpenTelemetry Demo chart:**
   ```bash
   helm install my-otel-demo open-telemetry/opentelemetry-demo
   ```

   > **Note**: If the Grafana pod crashloops with `OOMKilled` (Exit Code 137) during plugin extraction, or fails with permission errors updating bundled plugins (e.g. `elasticsearch`), increase its memory limits and enable the shadowed plugins volume:
   > ```bash
   > helm upgrade my-otel-demo open-telemetry/opentelemetry-demo --reuse-values \
   >   --set grafana.resources.limits.memory=512Mi \
   >   --set grafana.resources.requests.memory=256Mi \
   >   --set grafana.shadowBundledPlugins=true
   > ```

3. **Verify the installation:**
   ```bash
   kubectl get pods -w
   ```

For advanced configuration, scaling, and custom parameters, refer to the [OpenTelemetry Kubernetes Deployment Documentation](https://opentelemetry.io/docs/demo/kubernetes-deployment/).
