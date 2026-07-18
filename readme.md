# Provisioning the K3s Cluster Using Ansible

## Key Concepts

| Term | What It Is |
|------|------------|
| **[K3s](https://k3s.io/)** | A lightweight, CNCF-certified Kubernetes distribution by Rancher. It ships as a single binary, replaces etcd with SQLite by default, and bundles essential components (Flannel, CoreDNS, Traefik), making it ideal for edge, IoT, and local development clusters. |
| **[Flannel](https://github.com/flannel-io/flannel)** | The default Container Network Interface (CNI) plugin bundled with K3s. It creates a virtual overlay network so that every pod in the cluster gets its own IP address and can communicate with pods on other nodes, regardless of the underlying host network. |
| **[CoreDNS](https://coredns.io/)** | The cluster DNS server that ships with K3s. It allows pods to discover services by name (e.g., `my-service.default.svc.cluster.local`) instead of hard-coding IP addresses, which is essential for inter-service communication. |
| **[Ansible](https://docs.ansible.com/)** | An agentless infrastructure automation tool. Playbooks (written in YAML) define the desired state of your servers, and Ansible connects over SSH to enforce that state — no agent installation required on target nodes. |
| **[Vagrant](https://www.vagrantup.com/)** | A tool for building and managing reproducible virtual machine environments using a declarative `Vagrantfile`. It provisions local VMs (via VirtualBox, libvirt, etc.) that simulate real multi-node infrastructure on a single workstation. |
| **[Helm](https://helm.sh/)** | The package manager for Kubernetes. Helm charts bundle all the manifests, configs, and defaults needed to deploy complex applications (like the OpenTelemetry demo) into a cluster with a single command. |
| **[MetalLB](https://metallb.io/)** | A load balancer implementation for bare-metal Kubernetes clusters. It assigns external IP addresses to `LoadBalancer` Services and advertises them on the local network using Layer 2 or BGP modes. |
| **[kubeconfig](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/)** | A YAML configuration file that `kubectl` uses to authenticate and connect to a Kubernetes cluster. This playbook automatically retrieves it from the control plane and configures it on the host. |
| **[Control Plane Tainting](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)** | A Kubernetes mechanism to prevent regular workload pods from being scheduled on control plane nodes. This reserves the control plane for cluster management duties only. |

## Why Local K3s Instead of AWS EKS?

| | Local K3s + Vagrant | AWS EKS |
|---|---|---|
| **Cost** | Free — runs entirely on your workstation | EKS control plane costs ~$0.10/hr (~$73/mo), plus EC2 instance charges for worker nodes |
| **Learning depth** | You provision the cluster from scratch — networking, DNS, node joining, tainting — building real operational understanding | Much of the infrastructure is abstracted away; you interact with a managed API |
| **Iteration speed** | `vagrant up` + `ansible-playbook` gives you a full cluster in minutes, tear it down and rebuild in seconds | Cluster creation takes 10–15 minutes; teardown is slower and may incur costs if forgotten |
| **Offline development** | Works entirely offline once dependencies are cached | Requires an internet connection and AWS credentials at all times |
| **Portability** | Runs on any machine with VirtualBox and Vagrant — Linux, macOS, Windows | Tied to an AWS account and region |
| **Production readiness** | Not designed for production; ideal for learning and experimentation | Production-grade managed Kubernetes with built-in HA, IAM integration, and auto-scaling |

> **Bottom line**: This setup is purpose-built for **learning Kubernetes internals hands-on** — understanding how nodes join a cluster, how CNI networking works, how DNS resolution functions, and how automation tools like Ansible tie it all together. Use EKS (or GKE, AKS) when you need a production-grade managed cluster; use this repo when you want to truly understand what those managed services are doing under the hood.

---

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

> [!NOTE]
> The associated `Vagrantfile` is configured by default to provision heavy nodes (8 GB RAM / 4 CPUs for control plane, and 4 GB RAM / 2 CPUs for each worker) to support heavy workloads like the OpenTelemetry demo. If you have limited host resources, please adjust the CPU and memory allocations inside the Vagrantfile.

---

## Directory Structure

```text
.
├── inventory.yaml
├── deploy-k3s.yaml (Deploys the K3s cluster)
├── uninstall-k3s.yaml (Uninstalls K3s and resets state)
├── Documentation/
│   └── readme-manual.md (Manual VM setup guide)
└── roles/
    └── k3s/ (Modular Ansible role for K3s tasks)
```

---

## Inventory

The inventory defines the control plane and worker nodes, while shared variables are stored separately in `group_vars/all.yml`.

Update `group_vars/all.yml` with your cluster-specific values, and keep the inventory focused on node addresses.

Example inventory:

```yaml
all:
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
        debian5:
          ansible_host: 192.168.56.15
    k3s:
      children:
        control:
        workers:
```

Example `group_vars/all.yml`:

```yaml
ansible_user: vagrant
ansible_python_interpreter: /usr/bin/python3

k3s_server_ip: 192.168.56.11
k3s_token: "DkPS01xep_{8"
```

---

## Run the Playbook

Before applying the changes, you can perform a dry run (check mode) to preview what tasks Ansible will execute:

```bash
ansible-playbook -i inventory/inventory.yaml playbooks/deploy-k3s.yaml --check
```

To run and apply the playbook:

```bash
ansible-playbook -i inventory/inventory.yaml playbooks/deploy-k3s.yaml
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
debian5   Ready    <none>          1m    v1.36.2+k3s1
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
ansible-playbook -i inventory/inventory.yaml playbooks/deploy-k3s.yaml
```

---

## Uninstalling / Reverting K3s

To automatically uninstall K3s from all cluster nodes, clean up configuration folders, and reboot the machines, run the uninstall playbook on the host:

```bash
ansible-playbook -i inventory/inventory.yaml playbooks/uninstall-k3s.yaml
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

4. **Access the demo application using port forwarding, or expose it using a MetalLB `LoadBalancer` (if installed manually or provisioned by the Ansible playbook):**
   ```bash
   kubectl --namespace default port-forward svc/frontend-proxy 8080:8080
   ```
   Once running, open [http://localhost:8080](http://localhost:8080) in your browser to access the OpenTelemetry demo frontend.


---

## Adding MetalLB LoadBalancer Support

K3s does not provide a cloud-provider load balancer in a local Vagrant environment. Without an external load balancer implementation, `LoadBalancer` Services remain backed by a `NodePort` and do not receive an external IP address.

[MetalLB](https://metallb.io/) provides this functionality for bare-metal and local clusters by assigning external IP addresses from a configured pool and advertising them on the local network.

### Install MetalLB

MetalLB can be installed using the official Helm chart:

```bash
helm repo add metallb https://metallb.github.io/metallb
helm repo update

helm install metallb metallb/metallb \
  -n metallb-system \
  --create-namespace
```

The default Helm values are sufficient for this local cluster. A custom `values.yaml` file can be supplied for advanced configuration:

```bash
helm install metallb metallb/metallb \
  -n metallb-system \
  -f values.yaml
```

Verify the installation:

```bash
kubectl get pods -n metallb-system
kubectl get deployment -n metallb-system
```

All MetalLB components should be running before creating the load balancer configuration.

### Configure the IP Address Pool

The Vagrant nodes use the host-only network (`192.168.56.0/24`). A range of unused addresses from this network is allocated for `LoadBalancer` Services.

The allocated range must not overlap with VM addresses or other devices on the network.

Example `metallb-config.yaml` (`kubernetes-manifests/metallb-config.yaml`):

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: lab-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.56.200-192.168.56.220

---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: lab-advertisement
  namespace: metallb-system
```

Apply the configuration:

```bash
kubectl apply -f metallb-config.yaml
```

Verify:

```bash
kubectl get ipaddresspool -n metallb-system
kubectl get l2advertisement -n metallb-system
```

### Expose the OpenTelemetry Frontend

The OpenTelemetry demo frontend uses the `frontend-proxy` Service. Change it from `ClusterIP` to `LoadBalancer`:

```bash
kubectl patch svc frontend-proxy \
  -p '{"spec":{"type":"LoadBalancer"}}'
```

Verify the assigned external IP:

```bash
kubectl get svc frontend-proxy
```

Example:

```text
NAME             TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)
frontend-proxy   LoadBalancer   10.43.53.112    192.168.56.201   8080:32238/TCP
```

The application is now accessible directly:

```text
http://192.168.56.201:8080
```

---

> [!IMPORTANT]
> **Troubleshooting: MetalLB Webhook Endpoint Missing**
>
> * **Issue**: Applying `IPAddressPool` or `L2Advertisement` failed with:
>
>   ```
>   failed calling webhook:
>   no endpoints available for service "metallb-webhook-service"
>   ```
>
> * **Cause**: The MetalLB webhook service was created, but the controller was not yet available as a webhook endpoint when the configuration was applied.
>
> * **Resolution**: Restart the MetalLB controller deployment:
>
>   ```bash
>   kubectl rollout restart deployment metallb-controller -n metallb-system
>   ```
>
>   Verify the webhook endpoint:
>
>   ```bash
>   kubectl get endpoints -n metallb-system metallb-webhook-service
>   ```
>
>   Example:
>
>   ```text
>   NAME                      ENDPOINTS
>   metallb-webhook-service   10.42.2.36:9443
>   ```
>
>   After the endpoint became available, the MetalLB configuration applied successfully.


For advanced configuration, scaling, and custom parameters, refer to the [OpenTelemetry Kubernetes Deployment Documentation](https://opentelemetry.io/docs/demo/kubernetes-deployment/).
