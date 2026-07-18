# KubeVirt Installation on K3s (Vagrant Lab)

This guide documents the installation of KubeVirt and KubeVirt Manager on a K3s cluster running inside Vagrant virtual machines.

## Prerequisites

- Running K3s cluster
- MetalLB installed and configured
- kubectl configured
- Internet access from the control plane

---

# 1. Install KubeVirt

Get the latest stable version:

```bash
export VERSION=$(curl -s https://storage.googleapis.com/kubevirt-prow/release/kubevirt/kubevirt/stable.txt)
echo $VERSION
```

Install the KubeVirt operator:

```bash
kubectl create -f "https://github.com/kubevirt/kubevirt/releases/download/${VERSION}/kubevirt-operator.yaml"
```

Create the KubeVirt custom resource:

```bash
kubectl create -f "https://github.com/kubevirt/kubevirt/releases/download/${VERSION}/kubevirt-cr.yaml"
```

---

# 2. Enable Software Emulation

Since the Kubernetes nodes are Vagrant virtual machines and do not expose VT-x/AMD-V, enable software emulation:

```bash
kubectl -n kubevirt patch kubevirt kubevirt \
  --type=merge \
  --patch '{"spec":{"configuration":{"developerConfiguration":{"useEmulation":true}}}}'
```

---

# 3. Verify Installation

Check that KubeVirt reaches the `Deployed` phase:

```bash
kubectl get kubevirt.kubevirt.io/kubevirt \
  -n kubevirt \
  -o=jsonpath="{.status.phase}"
```

Expected output:

```
Deployed
```

Verify all KubeVirt components:

```bash
kubectl get all -n kubevirt
```

---

# 4. Install virtctl

Get the installed KubeVirt version:

```bash
VERSION=$(kubectl get kubevirt.kubevirt.io/kubevirt \
  -n kubevirt \
  -o=jsonpath="{.status.observedKubeVirtVersion}")
```

Determine system architecture:

```bash
ARCH=$(uname -s | tr A-Z a-z)-$(uname -m | sed 's/x86_64/amd64/')
echo $ARCH
```

Download:

```bash
curl -L -o virtctl \
https://github.com/kubevirt/kubevirt/releases/download/${VERSION}/virtctl-${VERSION}-${ARCH}
```

Install:

```bash
sudo install -m 0755 virtctl /usr/local/bin
```

Verify:

```bash
virtctl version
```

---

# 5. Install CDI

Deploy the Containerized Data Importer (CDI):

```bash
kubectl apply -f \
https://github.com/kubevirt/containerized-data-importer/releases/latest/download/cdi-operator.yaml

kubectl apply -f \
https://github.com/kubevirt/containerized-data-importer/releases/latest/download/cdi-cr.yaml
```

Verify:

```bash
kubectl get pods -n cdi
```

---

# 6. Install KubeVirt Manager

Deploy KubeVirt Manager:

```bash
kubectl apply -f \
https://github.com/kubevirt-manager/kubevirt-manager/releases/download/v1.5.4/bundled-v1.5.4.yaml
```

Verify:

```bash
kubectl get pods -n kubevirt-manager
kubectl get svc -n kubevirt-manager
```

---

# 7. Expose KubeVirt Manager

Patch the service to `LoadBalancer`:

```bash
kubectl patch svc kubevirt-manager \
  -n kubevirt-manager \
  -p '{"spec":{"type":"LoadBalancer"}}'
```

Verify that MetalLB assigns an external IP:

```bash
kubectl get svc -n kubevirt-manager
```

---

# 8. Publish with Cloudflare Tunnel

Create a Cloudflare Tunnel that forwards traffic to the MetalLB external IP assigned to the `kubevirt-manager` LoadBalancer service.

After the tunnel is configured and the DNS record is created, the KubeVirt Manager UI is accessible securely over HTTPS.

---

# 9. Create Virtual Machines

Virtual machines can be created either:

- Using the KubeVirt Manager web UI
- Using `virtctl`
- Using Kubernetes manifests

Verify:

```bash
kubectl get vm
kubectl get vmi
```

---

# Verification

```bash
kubectl get kubevirt -n kubevirt
kubectl get nodes
kubectl get pods -n kubevirt
kubectl get pods -n kubevirt-manager
kubectl get pods -n cdi
kubectl get vm
kubectl get vmi
```

---

# Notes

- K3s was deployed inside Vagrant virtual machines.
- Software emulation (`useEmulation=true`) is required because nested virtualization is unavailable.
- CDI is required for DataVolumes and VM disk provisioning.
- MetalLB provides the external IP used by the `LoadBalancer` service.
- Cloudflare Tunnel publishes the MetalLB IP over HTTPS.
- KubeVirt Manager v1.5.4 was used because it resolves VM creation issues present in v1.5.3.
- Virtual machines can be managed using either the KubeVirt Manager web UI or `virtctl`.