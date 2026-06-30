# Kubernetes Cluster + ArgoCD GitOps Setup

A 2-node Kubernetes cluster built from scratch on AWS EC2 using `kubeadm` and Flannel, with ArgoCD deployed on top for GitOps-based continuous delivery. This README documents the setup process and, more usefully, the real issues hit along the way — with root causes and fixes, not just the happy path.

## Stack

- **Infrastructure**: AWS EC2 (2 nodes — 1 master, 1 worker)
- **Cluster bootstrap**: kubeadm
- **CNI**: Flannel (VXLAN overlay)
- **GitOps / CD**: ArgoCD
- **Pod network CIDR**: `10.244.0.0/16`

## Setup

### Step 1 — Provision AWS infrastructure with Terraform

```bash
cd nodejs-app/terraform

# Initialize Terraform
terraform init

# Preview what will be created
terraform plan

# Create the infrastructure (takes ~2-3 minutes)
terraform apply --auto-approve

# Note the output IPs
# master_public_ip = "x.x.x.x"
# worker_public_ip = "x.x.x.x"
```

### Step 2 — Set up the Kubernetes cluster (kubeadm)

#### 2a. On the master node

```bash
# SSH into master
ssh -i ~/.ssh/id_rsa ec2-user@<MASTER_PUBLIC_IP>

# conntrack and tc tools are missing
sudo yum install -y conntrack tc

# Initialize the cluster
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-cert-extra-sans=<masternodeIP>
# You'll get the kubeadm join command in the output — copy it for the worker node step

# Set up kubectl for ec2-user
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Install Flannel CNI (pod networking)
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

# Verify master is Ready
kubectl get nodes
```

#### 2b. On the worker node

```bash
# SSH into worker
ssh -i ~/.ssh/id_rsa ec2-user@<WORKER_PUBLIC_IP>

# conntrack and tc tools are missing
sudo yum install -y conntrack tc

# Run the join command from master's kubeadm init output
sudo kubeadm join <MASTER_PRIVATE_IP>:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

#### 2c. Verify cluster (back on master)

```bash
kubectl get nodes
# Both nodes should show Ready status
```

### Step 3 — Install ArgoCD

Run these on the **master node**:

```bash
# Create ArgoCD namespace and install
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for pods to be ready (~2 minutes)
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=120s

# Expose ArgoCD UI as NodePort
kubectl patch svc argocd-server -n argocd \
  -p '{"spec": {"type": "NodePort", "ports": [{"port": 443, "nodePort": 30090, "targetPort": 8080}]}}'

# Get the initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

Access the ArgoCD UI at: `https://<MASTER_PUBLIC_IP>:30090`
- Username: `admin`
- Password: (from the command above)

### Step 4 — Register the app with ArgoCD

```bash
# Apply the ArgoCD Application manifest
kubectl apply -f argocd-app.yaml
```

ArgoCD will now:
1. Connect to the `nodejs-manifests` repo
2. Create the `production` namespace
3. Deploy the application
4. Watch for future changes (auto-sync every 3 minutes)

---

## Issues encountered and fixes

### 1. `conntrack not found in system path`

**Error:**
```
error execution phase preflight: [preflight] Some fatal errors occurred:
[ERROR FileExisting-conntrack]: conntrack not found in system path
```

**Cause:** `conntrack` is a Linux connection-tracking tool that `kube-proxy` depends on for NAT and Service routing. It's missing by default on minimal/cloud base images.

**Fix (Amazon Linux / RHEL-based):**
```bash
sudo yum install -y conntrack
```
On Debian/Ubuntu-based images, use `sudo apt install -y conntrack` instead.

### 2. `tc not found in system path`

**Warning:**
```
[WARNING FileExisting-tc]: tc not found in system path
```

**Cause:** `tc` (traffic control) handles QoS/bandwidth shaping at the kernel level. Not fatal on its own, but several CNI plugins (notably Cilium's eBPF datapath) depend on it more heavily than Flannel does.

**Fix (Amazon Linux / RHEL-based):**
```bash
sudo yum install -y tc
```
On Debian/Ubuntu-based images, `tc` ships in the `iproute2` package: `sudo apt install -y iproute2`.

### 3. `x509: certificate is valid for 10.0.1.x, not 3.x.x.x`

**Cause:** By default, `kubeadm init` only signs the API server certificate with internal SANs (private IP, cluster-internal DNS names, etc.). Connecting via the EC2 public IP fails TLS validation because that address isn't in the certificate's Subject Alternative Names (SAN) list.

**Fix:** Add the public IP at cluster init time:
```bash
sudo kubeadm init --apiserver-cert-extra-sans=<public-ip> --pod-network-cidr=10.244.0.0/16
```
Multiple SANs (private IP, public IP, DNS name) can be passed comma-separated if needed.

### 4. Kubeconfig pointing to a stale IP

**Cause:** EC2 instances without an Elastic IP get a new public IP on every stop/start. The `server:` field inside `~/.kube/config` (or `admin.conf`) still points at the old IP after a restart, causing connection timeouts that look like network issues but aren't.

**Fix:** Update the `server:` field manually after any IP change:
```yaml
server: https://<current-public-ip>:6443
```
**Permanent fix:** attach an Elastic IP to the instance so the address never changes.

### 5. `scp: Permission denied` copying `admin.conf`

**Cause:** `/etc/kubernetes/admin.conf` is root-owned with `600` permissions — a non-root user (e.g. `ec2-user`) can't read or copy it directly without elevated privileges.

**Fix — when setting up kubectl locally on the master** (used in Step 2a above):
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
`sudo cp` can read the root-owned source file; the `chown` afterward hands ownership to your user so subsequent `kubectl` calls don't need `sudo`.

**Fix — when copying the kubeconfig off the box** (e.g. to use `kubectl` from your laptop), plain `scp` fails for the same permission reason, since `scp` itself doesn't run with elevated privileges:
```bash
ssh -i key.pem ec2-user@<ip> "sudo cat /etc/kubernetes/admin.conf" > ~/kubeconfig-aws
```
This streams the sudo-read file straight into a local file without leaving a root-owned copy behind on the remote box.

### 6. `ERR_CONNECTION_REFUSED` accessing ArgoCD UI

**Symptom:** Browser refused to connect to `http://<public-ip>:30080`.

**Troubleshooting steps:**
1. Checked the EC2 Security Group — NodePort range `30000-32767/TCP` was already open from `0.0.0.0/0`, ruling out a firewall block.
2. Ran `kubectl get svc -n argocd` — found the actual NodePort mapping. Kubernetes auto-assigns a random port in the NodePort range unless explicitly pinned in the manifest; ArgoCD was listening on a different port than assumed.
3. Ran `kubectl get pods -n argocd` — confirmed all pods were `Running` and `1/1` ready, ruling out a dead backend.
4. Tested from inside the node directly with `curl -kv https://localhost:<actual-nodeport>` to isolate cluster-internal issues from external/network issues.
5. Confirmed protocol — ArgoCD serves HTTPS by default; using `http://` against a TLS-only listener can also cause connection issues.

**Root cause:** Wrong port assumed instead of verified. **Fix:** always confirm the real port with `kubectl get svc`, and optionally pin it explicitly:
```bash
kubectl patch svc argocd-server -n argocd --type='json' \
  -p='[{"op": "replace", "path": "/spec/ports/1/nodePort", "value":30080}]'
```

---

## Concepts worth understanding, not just memorizing

### What is a CNI (Container Network Interface)?

A CNI is the plugin standard that gives pods their networking — IP addresses, routing, and pod-to-pod connectivity across nodes. Without a CNI installed, pods stay stuck in `Pending`/`ContainerCreating`, since kubelet has no way to assign them a network identity.

**How it works:** when a pod is scheduled, kubelet calls the CNI plugin, which creates a virtual network interface, assigns an IP from the configured pod CIDR (`10.244.0.0/16` here), and sets up routing — often via Linux bridges, veth pairs, or an overlay network — so pods can reach each other across nodes.

**Common CNI options:**
| CNI | Notes |
|---|---|
| Flannel | Simple VXLAN overlay, easy setup — used in this build |
| Calico | Overlay or native L3 routing, plus NetworkPolicy enforcement |
| Cilium | eBPF-based, high performance, deep observability/security features |
| Weave Net | Overlay-based, simpler multi-cluster support |

Choosing a CNI is an architectural decision — it affects performance, security policy support, and debugging complexity later on.

### What is a TLS certificate, and how does validation work?

A TLS certificate lets two parties establish an **encrypted and authenticated** connection — not just "encrypted," but verified to be talking to who they claim to be.

**Validation flow:**
1. Client connects to the server.
2. Server presents its certificate, which includes a list of SANs (Subject Alternative Names — the IPs/hostnames it's valid for) and a CA signature.
3. Client checks: does the address I'm connecting to appear in the cert's SAN list?
   - **No → `x509: certificate is valid for X, not Y`**
4. Client checks: is this certificate signed by a CA I trust?
   - **No → `x509: certificate signed by unknown authority`**
5. Both checks pass → encrypted session established.

**Why both errors showed up in this project:**
- The **SAN error** happened because `kubeadm` only signs the API server cert for internal IPs by default — connecting via the public IP failed the SAN check until `--apiserver-cert-extra-sans` was added.
- The **unknown authority error** happened separately, on a laptop connecting fresh. Kubernetes uses its own self-signed cluster CA — not a public CA your OS/browser already trusts. `kubectl` only trusts it because the full kubeconfig file embeds the cluster's CA certificate directly (`certificate-authority-data`). If the kubeconfig is incomplete or hand-edited, that trust anchor breaks.

**Where TLS shows up across Kubernetes:** API server authentication, kubelet-to-API server communication, ingress TLS termination, and service mesh mTLS between pods.

---

## Security note

This setup opens the API server (port `6443`) to `0.0.0.0/0` for convenience in a learning environment. In production, this should be restricted — typically via a bastion host, VPN, or IP allowlist — rather than exposed to the open internet. Worth understanding the tradeoff even when not enforcing it in a lab.

---

## Useful commands

```bash
# Verify cluster status
kubectl get nodes
kubectl get pods -A

# Check ArgoCD service and pods
kubectl get svc -n argocd
kubectl get pods -n argocd

# Get ArgoCD initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```
