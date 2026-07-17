# Setting Up a Local Multi-Node K3s Cluster with Vagrant

> [!IMPORTANT]
> **Troubleshooting: Multi-Node Flannel Networking / Cluster DNS Issues**
> * **Issue**: In Vagrant-based multi-node setups, Flannel CNI defaults to binding on the `eth0` NAT interface, causing all nodes to share the same tunnel IP (`10.0.2.15`). This breaks cross-node pod communication, causing cluster DNS (`coredns`) and services to fail.
> * **Resolution**: Configured K3s on both the control plane and agents to bind Flannel explicitly to the private network interface (`eth1`) by passing the `--flannel-iface eth1` argument.

## Prerequisites

Ensure the following software is installed on the host machine before proceeding:

- VirtualBox
- Vagrant
- Ansible
- SSH key pair (`~/.ssh/id_rsa` and `~/.ssh/id_rsa.pub`)
- At least **20 GB of free RAM** on the host (recommended for running larger workloads)

### Minimal Installation (Debian/Ubuntu)

**VirtualBox:**
```bash
# Download and install official .deb (Debian Bookworm example)
wget https://download.virtualbox.org/virtualbox/7.0.20/virtualbox-7.0_7.0.20-163906~Debian~bookworm_amd64.deb
sudo apt update && sudo apt install ./virtualbox-7.0_7.0.20-163906~Debian~bookworm_amd64.deb
```

**Vagrant:**
```bash
# Download and install official .deb
wget https://releases.hashicorp.com/vagrant/2.4.1/vagrant_2.4.1-1_amd64.deb
sudo apt update && sudo apt install ./vagrant_2.4.1-1_amd64.deb
```

**Ansible:**
```bash
# Install via official Debian repositories
sudo apt update && sudo apt install -y ansible
```

---

## 1. Create the Virtual Machines

Create the following `Vagrantfile`:

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "debian/bookworm64"

  config.vm.define "debian1" do |node|
    node.vm.hostname = "debian1"
    node.vm.network "private_network", ip: "192.168.56.11"

    node.vm.provider "virtualbox" do |vb|
      vb.name = "debian1"
      vb.memory = 8192
      vb.cpus = 4
    end
  end

  config.vm.define "debian2" do |node|
    node.vm.hostname = "debian2"
    node.vm.network "private_network", ip: "192.168.56.12"

    node.vm.provider "virtualbox" do |vb|
      vb.name = "debian2"
      vb.memory = 4096
      vb.cpus = 2
    end
  end

  config.vm.define "debian3" do |node|
    node.vm.hostname = "debian3"
    node.vm.network "private_network", ip: "192.168.56.13"

    node.vm.provider "virtualbox" do |vb|
      vb.name = "debian3"
      vb.memory = 4096
      vb.cpus = 2
    end
  end

  config.vm.define "debian4" do |node|
    node.vm.hostname = "debian4"
    node.vm.network "private_network", ip: "192.168.56.14"

    node.vm.provider "virtualbox" do |vb|
      vb.name = "debian4"
      vb.memory = 4096
      vb.cpus = 2
    end
  end
end
```

Launch the virtual machines:

```bash
vagrant up
```

---

## 2. Node Roles

| Hostname | IP Address | Role |
|----------|------------|------|
| debian1 | 192.168.56.11 | Control Plane |
| debian2 | 192.168.56.12 | Worker |
| debian3 | 192.168.56.13 | Worker |
| debian4 | 192.168.56.14 | Worker |

---

## 3. Remove Existing K3s Installation (Optional)

If the VMs were previously used for Kubernetes, you can automatically reset them to a clean state without destroying and recreating the VMs:

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

## 4. Configure Passwordless SSH

Each VM has a Vagrant-generated private key:

```text
.vagrant/machines/<vm-name>/virtualbox/private_key
```

Copy your public SSH key into each VM.

Example for **debian1**:

```bash
ssh-copy-id -f \
-o IdentityFile=./.vagrant/machines/debian1/virtualbox/private_key \
vagrant@192.168.56.11
```

Repeat for:

- debian2
- debian3
- debian4

Verify connectivity:

```bash
ssh vagrant@192.168.56.11
ssh vagrant@192.168.56.12
ssh vagrant@192.168.56.13
ssh vagrant@192.168.56.14
```

---

## 5. Install the K3s Server

SSH into **debian1**.

Install required packages:

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y curl wget git vim htop jq
```

Install K3s:

```bash
curl -sfL https://get.k3s.io | \
INSTALL_K3S_EXEC="server \
--node-ip 192.168.56.11 \
--flannel-iface eth1 \
--token DkPS01xep_{8" \
sh -
```

Verify:

```bash
sudo systemctl status k3s
```

Retrieve the node join token:

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

---

## 6. Join Worker Nodes

Repeat the following on **debian2**, **debian3**, and **debian4**.

Install required packages:

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y curl wget git vim htop jq
```

Example for **debian2**:

```bash
curl -sfL https://get.k3s.io | \
INSTALL_K3S_EXEC="agent \
--server https://192.168.56.11:6443 \
--token DkPS01xep_{8 \
--node-ip 192.168.56.12 \
--flannel-iface eth1" \
sh -s -
```

Update only the `--node-ip` value:

- debian3 → `192.168.56.13`
- debian4 → `192.168.56.14`

Verify:

```bash
sudo systemctl status k3s-agent
```

---

## 7. Verify Cluster Health

From the control plane:

```bash
sudo kubectl get nodes -o wide
```

Expected:

- Four nodes
- All nodes in the **Ready** state

Verify cluster components:

```bash
sudo kubectl get pods -A
sudo kubectl cluster-info
```

---

## 8. Copy kubeconfig to the Host

On **debian1**:

```bash
sudo cp /etc/rancher/k3s/k3s.yaml /home/vagrant/k3s.yaml
sudo chown vagrant:vagrant /home/vagrant/k3s.yaml
```

On the host:

```bash
mkdir -p ~/.kube

scp vagrant@192.168.56.11:/home/vagrant/k3s.yaml \
~/.kube/k3s-local.yaml
```

Update the API server address:

```bash
sed -i 's/127.0.0.1/192.168.56.11/' ~/.kube/k3s-local.yaml
```

Use the kubeconfig:

```bash
export KUBECONFIG=~/.kube/k3s-local.yaml
```

> **Note:** Using a dedicated kubeconfig prevents overwriting your default `~/.kube/config`, allowing you to continue using other Kubernetes clusters (EKS, AKS, GKE, etc.).

---

## 9. Prevent Workloads from Running on the Control Plane

```bash
kubectl taint nodes debian1 \
node-role.kubernetes.io/control-plane=true:NoSchedule
```

Verify:

```bash
kubectl describe node debian1 | grep Taints
```

---

## 10. Validate the Cluster

Verify all nodes:

```bash
kubectl get nodes -o wide
```

Verify system pods:

```bash
kubectl get pods -A
```

Verify cluster information:

```bash
kubectl cluster-info
```

Verify recent events:

```bash
kubectl get events -A
```

---

## 11. Verify Internet Connectivity

Ensure each node can reach external registries.

```bash
curl -I https://github.com
curl -I https://ghcr.io
curl -I https://grafana.com
```

---

## 12. Verify Image Pulls

Deploy a temporary workload:

```bash
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80
kubectl get pods -w
```

The pod should reach the **Running** state.

Clean up:

```bash
kubectl delete deployment nginx
kubectl delete service nginx
```

---

## 13. Verify Cluster DNS

```bash
kubectl run dns-test \
--image=busybox \
--rm -it \
-- nslookup kubernetes.default
```

DNS resolution should succeed.

---

## 14. Ready for Workloads

The cluster is ready for application deployments once the following have been verified:

- All nodes are **Ready**
- All system pods are **Running**
- Internet connectivity is working
- Container images pull successfully
- DNS resolution works
- No recurring warning events are present

---

## 15. Deploying the OpenTelemetry Demo

Once the cluster is up and healthy, you can install the OpenTelemetry Demo:

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
