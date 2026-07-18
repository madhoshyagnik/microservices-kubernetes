# Rancher Deployment on K3s (v1.36.2) with MetalLB and Cloudflare Tunnel

## Environment

- Kubernetes: **K3s v1.36.2**
- Nodes:
  - `debian1` - Control Plane
  - `debian2` - Worker
  - `debian3` - Worker
  - `debian4` - Worker
  - `debian5` - Worker
- Ingress Controller: Traefik (default K3s)
- Load Balancer: MetalLB
- Domain: `madhoshyagnik.space`
- Tunnel: Cloudflare Remote Managed Tunnel

---

# Installing Rancher

## 1. Add Rancher Helm Repository

```bash
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo update
```

---

## 2. Create Namespace

```bash
kubectl create namespace cattle-system
```

---

## 3. Install cert-manager

Rancher was deployed using **Rancher Generated Certificates (Default)**.

Add the Jetstack repository:

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

Verify installation:

```bash
kubectl get pods -n cert-manager
```

---

# Rancher Chart Compatibility Issue

The Rancher Helm chart did not support Kubernetes **v1.36.2** because the chart's `kubeVersion` constraint did not include this version.

Downloaded the chart locally:

```bash
helm pull rancher-stable/rancher --untar
```

Edited:

```
rancher/Chart.yaml
```

Modified the `kubeVersion` field to include Kubernetes 1.36.x.

---

# Install Rancher

Installed Rancher from the modified local chart:

```bash
helm install rancher ./rancher \
  --namespace cattle-system \
  --set hostname=rancher.madhoshyagnik.space \
  --set tls=external \
  --set bootstrapPassword=admin
```

## Why `tls=external`

Since Rancher would be accessed through a Cloudflare Tunnel, TLS termination was configured to happen externally instead of Rancher managing public certificates itself.

---

# Expose Rancher

After installation, the Rancher Service was changed from `ClusterIP` to `LoadBalancer` so MetalLB could assign an external virtual IP.

---

# Problems Encountered

## Problem 1 – Rancher Not Reachable

MetalLB assigned:

```
192.168.56.202
```

However,

```bash
curl -vk https://192.168.56.202
```

returned:

```
curl: (7) Failed to connect
No route to host
```

---

## Investigation

The Rancher Service showed no endpoints.

```bash
kubectl describe svc rancher -n cattle-system
```

Output:

```
Endpoints:
```

(empty)

Although Rancher pods were running:

```text
app=rancher
```

the Service selector was:

```yaml
selector:
  app: LoadBalancer
```

Because the selector didn't match any pods, Kubernetes created **no Endpoints**, so MetalLB had nowhere to forward traffic.

---

## Root Cause

While patching the Service to change it from `ClusterIP` to `LoadBalancer`, the Service selector was accidentally modified from:

```yaml
selector:
  app: rancher
```

to

```yaml
selector:
  app: LoadBalancer
```

---

## Resolution

Restored the Service selector:

```yaml
selector:
  app: rancher
```

After correcting the selector:

```bash
kubectl get endpoints -n cattle-system rancher
```

showed the Rancher pod IPs, and the service immediately became reachable.

---

# Verification

## Local LoadBalancer IP

```bash
curl -vk https://192.168.56.202
```

Returned:

```
HTTP/1.1 200 OK
```

confirming:

- MetalLB working
- Traefik routing correctly
- Rancher responding

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

Once the Rancher Service selector was corrected, Cloudflare successfully proxied requests to Rancher.

Verification:

```bash
curl -vk https://rancher.madhoshyagnik.space
```

Response:

```
HTTP/2 302
server: cloudflare
location: https://rancher.madhoshyagnik.space/
```

confirming:

- Cloudflare DNS resolution
- Tunnel connectivity
- Origin reachability
- Successful HTTPS proxying

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
                  Traefik Ingress Controller
                            │
                            ▼
             Rancher Service (LoadBalancer)
                            │
                            ▼
                      Rancher Pods
```