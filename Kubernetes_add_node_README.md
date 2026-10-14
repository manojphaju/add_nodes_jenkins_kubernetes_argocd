# Runbook: Adding Nodes to a Kubernetes (kubeadm) Cluster

Scope: kubeadm cluster on Azure VMs — adding worker and control-plane nodes.
Replace every `<placeholder>` before running anything.

## 0. Pre-checks (on an existing control-plane node)
```bash
kubectl get nodes -o wide          # note the exact version, e.g. v1.30.4
kubectl get pods -n kube-system    # note the CNI (calico / flannel / cilium)
kubectl -n kube-system get cm kubeadm-config -o yaml | grep controlPlaneEndpoint
```
The new node **must** use the same minor version (kubelet may be ≤ control plane, never newer).

## 1. Azure prerequisites
- VM in the same VNet (or peered VNet) as the cluster.
- Unique hostname; static private IP recommended.
- NSG rules (source = cluster subnet):

| Port | Proto | Purpose | Needed on |
|---|---|---|---|
| 6443 | TCP | API server | control plane |
| 2379–2380 | TCP | etcd | control plane |
| 10250 | TCP | kubelet | all |
| 10257, 10259 | TCP | controller-manager, scheduler | control plane |
| 30000–32767 | TCP/UDP | NodePort | workers |
| 179 | TCP | Calico BGP | all (if Calico BGP) |
| 4789 | UDP | Calico VXLAN | all (if Calico VXLAN) |
| 8472 | UDP | Flannel VXLAN | all (if Flannel) |

- If using Calico in BGP mode on Azure: Azure drops BGP-routed traffic — use VXLAN mode, or enable **IP forwarding** on the NIC.

**Existing-cluster side (easy to miss):**
- The rules above must exist **in both directions**. If the new VM is in a different subnet or NSG, update the NSG on the **existing** control-plane and worker nodes to allow the new node's IP/subnet too.
- New node outbound: 443 to `pkgs.k8s.io`, Ubuntu mirrors and container registries (Docker Hub / ACR). If pulling from a private registry, make sure the namespaces that will schedule here have `imagePullSecrets`, or that ACR attachment covers the node.
- If DNS is not used between nodes, add the new node to `/etc/hosts` on every node.

## 2. Prepare the new node (run on the new VM)
```bash
# 2.1 Disable swap
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# 2.2 Kernel modules + sysctl
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay && sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system

# 2.3 containerd with systemd cgroup driver
sudo apt update && sudo apt install -y containerd apt-transport-https ca-certificates curl gpg
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd && sudo systemctl enable containerd

# 2.3b Time sync (cert validation and etcd depend on it)
sudo apt install -y chrony && sudo systemctl enable --now chrony

# 2.4 kubeadm / kubelet / kubectl — match the cluster minor version
K8S_MINOR=v1.30   # <-- set to your cluster's minor
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/${K8S_MINOR}/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/${K8S_MINOR}/deb/ /" \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
apt-cache madison kubeadm | head          # pick the exact patch matching the cluster
VER=1.30.4-1.1                            # <-- set
sudo apt install -y kubelet=$VER kubeadm=$VER kubectl=$VER
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable kubelet

# 2.5 Multi-NIC VMs only: pin kubelet to the cluster-facing private IP
echo "KUBELET_EXTRA_ARGS=--node-ip=<private-ip>" | sudo tee /etc/default/kubelet

# 2.6 Pre-flight connectivity to the control plane
nc -vz <control-plane-endpoint> 6443
```

## 3A. Join as a WORKER node

**On a control-plane node** — generate a fresh join command (tokens expire after 24h):
```bash
kubeadm token create --print-join-command
```

**On the new node:**
```bash
sudo kubeadm join <control-plane-endpoint>:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

**Verify and label (from control plane):**
```bash
kubectl get nodes -w                                   # NotReady -> Ready once CNI pod is up
kubectl label node <node-name> node-role.kubernetes.io/worker=
kubectl label node <node-name> workload=<general|build|db>   # optional, for scheduling

# If kubelet serving-cert bootstrap is enabled (serverTLSBootstrap: true), approve pending CSRs
kubectl get csr | grep Pending
kubectl certificate approve <csr-name>
```

**Then (Azure side):**
- If ingress or apps are exposed via an Azure LB → NodePort or ingress-controller nodes, add the new worker to that **LB backend pool**.
- Calico multi-NIC: confirm the node got the right IP (`kubectl get node <node> -o wide`). If not, set `IP_AUTODETECTION_METHOD=cidr=<cluster-subnet>` on the `calico-node` DaemonSet.
- Optional taint for dedicated nodes (e.g. Jenkins builds only):
  `kubectl taint node <node-name> dedicated=build:NoSchedule`

## 3B. Join as an additional CONTROL-PLANE node (HA)

Requirements:
- Cluster was initialised with `controlPlaneEndpoint` pointing at a load balancer (Azure LB / internal LB on 6443). If that grep in step 0 returns nothing, you **cannot** add control-plane nodes without reconfiguring the cluster.
- Keep an **odd** number of control-plane nodes (3, 5) for etcd quorum.
- Azure LB health probe: TCP 6443 (or HTTPS `/readyz`).
- **Azure internal LB hairpin limitation:** a backend VM cannot reach the ILB frontend that load-balances back to itself. During the join, the new node isn't in the pool yet, so the join works. Afterwards, kubelet on each control-plane node should talk to its local API server. Map the LB DNS name to `127.0.0.1` in `/etc/hosts` on control-plane nodes, or use a public LB / HAProxy in front.
- The new control-plane node's NSG needs every control-plane port from the table in step 1, and the **existing** control-plane nodes need 2379–2380 open **from the new node** for etcd peering.

**On an existing control-plane node:**
```bash
# Upload certs and get a certificate key (valid 2 hours)
sudo kubeadm init phase upload-certs --upload-certs
# Join command
kubeadm token create --print-join-command
```

**On the new node:**
```bash
sudo kubeadm join <lb-endpoint>:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane \
  --certificate-key <certificate-key-from-upload-certs>

mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

**Then:**
- Add the new VM to the Azure LB backend pool for 6443, and confirm the health probe shows it as healthy.
- If you use the hairpin workaround: `echo "127.0.0.1 <lb-dns-name>" | sudo tee -a /etc/hosts` on the new control-plane node.
- Copy the etcd backup job/cron to the new node if backups run per node.
- Verify etcd membership:
```bash
kubectl -n kube-system get pods -l component=etcd -o wide
kubectl get nodes
```

## 4. Post-join checks
```bash
kubectl get nodes -o wide
kubectl -n kube-system get pods -o wide --field-selector spec.nodeName=<node-name>
kubectl run nettest --image=busybox:1.36 --rm -it --restart=Never \
  --overrides='{"spec":{"nodeName":"<node-name>"}}' -- nslookup kubernetes.default
```
Also: install node-exporter / monitoring agents if not DaemonSets; confirm Defender / hardening baseline applied to the new VM.

## 5. Removing a node (for reference)
```bash
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
kubectl delete node <node-name>
# on the removed node
sudo kubeadm reset -f && sudo rm -rf /etc/cni/net.d $HOME/.kube
```

## Kubernetes troubleshooting
| Symptom | Fix |
|---|---|
| `token ... not found` / expired | Regenerate with `kubeadm token create --print-join-command` |
| `certificate-key` invalid | Re-run `kubeadm init phase upload-certs --upload-certs` (2h TTL) |
| Node stuck `NotReady` | CNI pod on the node: `kubectl -n kube-system get pods -o wide \| grep <node>`; check CNI ports in NSG |
| kubelet crash-looping | `journalctl -u kubelet -f`; usually swap on or cgroup driver mismatch |
| `connection refused :6443` | NSG/route to control plane / LB health probe |
| Preflight errors | Do NOT blindly `--ignore-preflight-errors=all`; fix the reported item |
| `x509: certificate has expired or is not yet valid` | Clock skew; check `chronyc tracking` |
| Node `Ready` but pods `ImagePullBackOff` only there | Node outbound 443 / registry auth |
| Control-plane join hangs at etcd | 2379–2380 not open between existing and new control-plane nodes |
| Control-plane node intermittently loses API | ILB hairpin; apply `/etc/hosts` workaround |



---

## Checklist

- [ ] Existing nodes' NSGs updated to allow the new node (both directions)
- [ ] chrony running; `node-ip` set if multi-NIC
- [ ] Pending CSRs approved
- [ ] Worker added to app/ingress LB backend pool
- [ ] (CP) ILB hairpin `/etc/hosts` workaround; etcd ports open between CPs
- [ ] Version matched, NSG ports open
- [ ] Swap off, modules/sysctl, containerd SystemdCgroup
- [ ] Joined, `Ready`, labelled
- [ ] (CP only) added to LB pool, etcd healthy, odd member count
- [ ] Monitoring + hardening baseline applied
