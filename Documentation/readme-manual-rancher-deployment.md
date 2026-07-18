# Rancher Deployment on K3s (v1.36.2) with MetalLB and Cloudflare Tunnel

## Environment

* Kubernetes: **K3s v1.36.2**
* Nodes:

  * `debian1` - Control Plane
  * `debian2-5` - Workers
* Load Balancer: **MetalLB**
* Ingress: **Traefik** (default K3s ingress)
* Domain: `madhoshyagnik.space` (Cloudflare)
* Tunnel: **Cloudflare Remote Managed Tunnel**
* TLS: Rancher-generated certificates with Cloudflare terminating public TLS.

---

# 1. Add Rancher Helm Repository

```bash
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo update
```

---

# 2. Create Namespace

```bash
kubectl create namespace cattle-system
```

---

# 3. Install cert-manager

Rancher was deployed using **Rancher Generated Certificates (Default)**.

Add Jetstack repository:

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
```

Install cert-manager:

```bash
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true
```

Verify:

```bash
kubectl get pods -n cert-manager
```

---

# 4. Rancher Chart Compatibility Issue

The latest Rancher chart did not support Kubernetes **v1.36.2** because the chart specified an older supported Kubernetes version.

To work around this:

```bash
helm pull rancher-stable/rancher --untar
```

Edit:

```text
rancher/Chart.yaml
```

Modify the Kubernetes version requirement (`kubeVersion`) to include Kubernetes 1.36.x.

---

# 5. Install Rancher from Local Chart

Instead of installing from the remote repository:

```bash
helm install rancher ./rancher \
  --namespace cattle-system \
  --set hostname=rancher.madhoshyagnik.space \
  --set tls=external \
  --set bootstrapPassword=admin
```

### Why `tls=external`

Since Rancher would be accessed through a Cloudflare Tunnel, TLS termination was intended to happen externally rather than by Rancher itself. Using:

```text
--set tls=external
```

avoided Rancher attempting to manage public TLS certificates itself.

---

# 6. Expose Rancher using MetalLB

The Rancher Service was converted from `ClusterIP` to `LoadBalancer` so MetalLB could advertise an external VIP.

---

# Problems Encountered

## Issue 1 — Rancher inaccessible via LoadBalancer

The assigned MetalLB IP:

```
192.168.56.202
```

returned:

```text
curl: (7) Failed to connect
No route to host
```

Investigation showed:

```bash
kubectl describe svc rancher -n cattle-system
```

displayed:

```text
Endpoints:
```

No endpoints were associated with the Service.

---

## Root Cause

The Rancher Service selector had accidentally become (my bad):

```yaml
selector:
  app: LoadBalancer
```

while the Rancher pods were labelled:

```yaml
app: rancher
```

Because the selector did not match any pods, Kubernetes created no Endpoints.

MetalLB still advertised the VIP, but traffic had nowhere to go.

---

## Resolution

Restore the Service selector:

```yaml
selector:
  app: rancher
```

Immediately after correcting the selector:

```bash
kubectl get endpoints -n cattle-system rancher
```

returned the Rancher pod IPs, and the service became reachable.

---

## Verification

Local access:

```bash
curl -vk https://192.168.56.202
```

Returned:

```
HTTP/1.1 200 OK
```

confirming:

* MetalLB VIP working
* Traefik routing correctly
* Rancher responding

---

# Cloudflare Tunnel

A **Remote Managed Cloudflare Tunnel** was configured.

Origin:

```
https://192.168.56.202
```

Hostname:

```
rancher.madhoshyagnik.space
```

No local `cloudflared` configuration was required.

After Rancher's Service selector was corrected, Cloudflare successfully proxied requests to the origin.

Verification:

```bash
curl -vk https://rancher.madhoshyagnik.space
```

Returned:

```
HTTP/2 302
server: cloudflare
location: https://rancher.madhoshyagnik.space/
```

confirming:

* DNS resolution through Cloudflare
* Tunnel connectivity
* Origin reachability
* Successful end-to-end HTTPS proxying

---

# Final Architecture

```text
Internet
      │
      ▼
Cloudflare DNS
      │
      ▼
Cloudflare Remote Managed Tunnel
      │
      ▼
MetalLB VIP (192.168.56.202)
      │
      ▼
Traefik Ingress
      │
      ▼
Rancher Service (LoadBalancer)
      │
      ▼
Rancher Pods
```