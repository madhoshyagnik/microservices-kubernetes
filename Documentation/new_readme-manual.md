
# Setting Up a Local Multi-Node K3s Cluster with Vagrant

### **Table of Contents**
1. Prerequisites
2. Create the Virtual Machines
3. Node Roles
4. Configure Passwordless SSH
5. Verify Private Network Connectivity 
6. Remove existing K3s Installation (Optional)
7. Install K3s Server
8. Join Worker Nodes
9. Verify Cluster Health
10. Configure kubectl Access
11. Taint the Control Plane
12. Validate Cluster Networking 
13. Install Helm 
14. Deploy Test Workload
15. Verify Cluster DNS
16. Deploy OpenTelemetry Demo
17. Troubleshooting 

> [!IMPORTANT]
> **Troubleshooting: Multi-Node Flannel Networking / Cluster DNS Issues**
> * **Issue**: In Vagrant-based multi-node setups, Flannel CNI defaults to binding on the `eth0` NAT interface, causing all nodes to share the same tunnel IP (`10.0.2.15`). This breaks cross-node pod communication, causing cluster DNS (`coredns`) and services to fail.
> * **Resolution**: Configured K3s on both the control plane and agents to bind Flannel explicitly to the private network interface (`eth1`) by passing the `--flannel-iface eth1` argument.

## 1. Prerequisites

Ensure the following software is installed on the host machine before proceeding:

- VirtualBox
- Vagrant
- Ansible
- SSH key pair (`~/.ssh/id_rsa` and `~/.ssh/id_rsa.pub`)
- At least **20 GB of free RAM** on the host (recommended for running larger workloads)

#### **HOST Requirements -**

• 4 CPU cores minimum
• 20 GB RAM recommended
• 30 GB free disk space
• VirtualBox 7.x
• Vagrant 2.4+
• Ansible
• OpenSSH

> [!NOTE]
> The Vagrantfile is configured by default to provision heavy nodes (8 GB RAM / 4 CPUs for control plane, and 4 GB RAM / 2 CPUs for each worker) to support heavy workloads like the OpenTelemetry demo. If you have limited host resources, please adjust the CPU and memory allocations inside the Vagrantfile.

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


## 2. Create the Virtual Machines

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

#### Launch the virtual machines

Run:

```bash
vagrant up
```

------------------------------------------------------------------------

## 3. Node Roles

| Hostname | IP Address | Role |
|----------|------------|------|
| debian1 | 192.168.56.11 | Control Plane |
| debian2 | 192.168.56.12 | Worker |
| debian3 | 192.168.56.13 | Worker |
| debian4 | 192.168.56.14 | Worker |

---

## Remove existing K3s Installation (Optional)

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

------------------------------------------------------------------------

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

------------------------------------------------------------------------

## 5. Verify Private Network Connectivity

Verify every VM has two interfaces:

``` bash
ip addr
```

Expected:

-   eth0 → NAT
-   eth1 → Private Network (192.168.56.x)

Verify connectivity:

``` bash
ping -c 3 192.168.56.11
ping -c 3 192.168.56.12
ping -c 3 192.168.56.13
ping -c 3 192.168.56.14
```

### Why `--flannel-iface eth1`?

Vagrant creates two network interfaces:

-   `eth0` → NAT (`10.0.2.x`)
-   `eth1` → Private Network (`192.168.56.x`)

Without specifying:

``` bash
--flannel-iface eth1
```

Flannel may select `eth0`, causing every node to advertise `10.0.2.15`,
which breaks:

-   Pod-to-pod networking
-   CoreDNS
-   Service networking
-   Cross-node communication

------------------------------------------------------------------------

## 6. Install the K3s Server

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

After installation verify:

``` bash
sudo systemctl status k3s
sudo kubectl get nodes
```

Retrieve the node token:

``` bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

------------------------------------------------------------------------

## 7. Join Worker Nodes

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

After each worker joins:

``` bash
sudo systemctl status k3s-agent
```

From the control plane:

``` bash
kubectl get nodes
```

Continue only after every node has joined successfully.

------------------------------------------------------------------------

## 8. Verify Cluster Health

``` bash
kubectl get nodes -o wide
```

Every node should have a unique Internal IP:

-   192.168.56.11
-   192.168.56.12
-   192.168.56.13
-   192.168.56.14

If every node reports `10.0.2.15`, Flannel is using the wrong interface.

Continue with:

``` bash
kubectl get pods -A
kubectl cluster-info
```

------------------------------------------------------------------------

## 9. Configure kubectl on the Host

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

After exporting `KUBECONFIG`:

``` bash
kubectl cluster-info
kubectl version
kubectl get nodes
```

------------------------------------------------------------------------

## 10. Prevent Workloads from Running on the Control Plane

```bash
kubectl taint nodes debian1 \
node-role.kubernetes.io/control-plane=true:NoSchedule
```

Verify:

```bash
kubectl describe node debian1 | grep Taints
```

------------------------------------------------------------------------

## 11. Validate the Cluster

Run:

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

------------------------------------------------------------------------

## 12. Validate Cluster Networking

``` bash
kubectl run network-test \
--image=busybox \
-it --rm -- sh
```

Inside the pod:

``` bash
nslookup kubernetes.default
wget -qO- http://kubernetes.default
ping google.com
```

------------------------------------------------------------------------

## 13. Verify Image Pulls

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

## 14. Verify Cluster DNS

```bash
kubectl run dns-test \
--image=busybox \
--rm -it \
-- nslookup kubernetes.default
```

DNS resolution should succeed.

If the pod stays in `ImagePullBackOff`, verify internet connectivity.

------------------------------------------------------------------------

## 15. Verify Cluster DNS

``` bash
kubectl run dns-test \
--image=busybox \
--rm -it \
-- nslookup kubernetes.default
```

A successful response confirms CoreDNS is functioning correctly.

------------------------------------------------------------------------

## 16. Install Helm

``` bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

helm version
```

------------------------------------------------------------------------

## 17. Deploy the OpenTelemetry Demo

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

After installation:

``` bash
kubectl get all
helm list
```
For advanced configuration, scaling, and custom parameters, refer to the [OpenTelemetry Kubernetes Deployment Documentation](https://opentelemetry.io/docs/demo/kubernetes-deployment/).

------------------------------------------------------------------------

## 18. Troubleshooting

### Node NotReady

``` bash
systemctl status k3s
systemctl status k3s-agent
```

### CoreDNS CrashLoopBackOff

Verify every node was started with:

``` bash
--flannel-iface eth1
```

### Pods Cannot Communicate

``` bash
kubectl get nodes -o wide
```

Each node must advertise its unique `192.168.56.x` address.

### ImagePullBackOff

``` bash
curl -I https://github.com
curl -I https://ghcr.io
```

### Worker Cannot Join

Verify:

-   Correct server IP
-   Correct node token
-   Port 6443 is reachable
