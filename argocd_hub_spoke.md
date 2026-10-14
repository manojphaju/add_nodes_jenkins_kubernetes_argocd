# Runbook: Adding Clusters to ArgoCD (Hub-and-Spoke)

Scope: ArgoCD on a hub cluster managing spoke clusters (Azure, kubeadm).
Replace every `<placeholder>` before running anything.

**Model:** one **hub** cluster runs ArgoCD; each **spoke** cluster is registered with it. The hub's `argocd-application-controller` talks to each spoke's API server directly. Nothing ArgoCD-related runs on the spokes except a service account.

```
            ┌──────────────── HUB ────────────────┐
 Git ─────► │ argocd (server, repo, app-controller)│
            └───────┬──────────────┬───────────────┘
                    │ 6443         │ 6443
              ┌─────▼────┐   ┌─────▼────┐
              │ spoke-stg │   │ spoke-prd │
              └──────────┘   └──────────┘
```

## 0. Prerequisites

**Network (both sides):**
- Hub → spoke API server reachable on 6443:
  - Azure: VNet peering hub↔spoke (preferred, private), **or** public API endpoint locked down by NSG to the hub's egress IP.
- VNet peering must be created **on both VNets** (hub→spoke and spoke→hub); each side shows *Connected*. Address spaces must not overlap, including pod/service CIDRs if routed.
- **Spoke NSG** (control-plane nodes / API LB): inbound 6443 from the **hub node subnet**. ArgoCD pods egress SNAT'd to the hub node IPs, not the pod CIDR.
- **Hub NSG:** outbound 6443 to the spoke API, and outbound 443 to GitHub for repo pulls.
- If the spoke API uses a private DNS name: link the Azure Private DNS zone to the **hub** VNet, or use the IP (which must be in the cert SAN).
- Spoke API server certificate must include the IP/DNS name ArgoCD will use (check: `openssl s_client -connect <spoke>:6443 </dev/null 2>/dev/null | openssl x509 -noout -text | grep -A1 "Subject Alternative Name"`). If missing, add via `apiServer.certSANs` and regenerate API server certs.
- `argocd` CLI installed locally; kubeconfig with a context for each spoke.

**Hub side:**
- ArgoCD has access to the GitOps repo (once per repo, not per spoke):
  ```bash
  argocd repo add https://github.com/<org>/<gitops-repo>.git \
    --username <github-user> --password <github-pat-read-only>
  argocd repo list        # CONNECTION STATUS = Successful
  ```
- Optional, for faster sync than the default 3-minute poll: GitHub webhook → `https://<argocd-hub-host>/api/webhook` (content type `application/json`, push events).

**Spoke side (platform prerequisites, ideally deployed by ArgoCD itself):**
- Private images: create the pull secret in each app namespace, or let the ApplicationSet deploy it via Sealed/External Secret.
  ```bash
  kubectl --context <spoke-context> -n <app-ns> create secret docker-registry dockerhub-cred \
    --docker-server=https://index.docker.io/v1/ --docker-username=<user> --docker-password=<token>
  ```
- Ingress controller, cert-manager, storage class and monitoring agents in place if apps depend on them.
- Spoke nodes outbound 443 to the image registry.
- Test from a hub pod:
```bash
kubectl -n argocd run nettest --rm -it --image=curlimages/curl --restart=Never -- \
  curl -k https://<spoke-api>:6443/version
```

## Register the spoke — Method A — `argocd` CLI (quickest)

```bash
# 1. Login to hub ArgoCD
argocd login <argocd-hub-host> --username admin --grpc-web

# 2. Confirm spoke context and that its server URL is the reachable address
kubectl config get-contexts
kubectl config view -o jsonpath='{.clusters[?(@.name=="<spoke-cluster-name>")].cluster.server}'
#   must NOT be 127.0.0.1 / localhost — edit the kubeconfig if it is

# 3. Register the spoke
argocd cluster add <spoke-context> \
  --name spoke-staging \
  --label env=staging \
  --label region=uksouth \
  --yes

# 4. Verify
argocd cluster list
```
What it does: creates `argocd-manager` ServiceAccount + ClusterRole (cluster-admin equivalent) in `kube-system` on the spoke, and stores a cluster Secret in the hub's `argocd` namespace.

**Least-privilege variant** (only manage specific namespaces):
```bash
argocd cluster add <spoke-context> --name spoke-staging \
  --namespace app-ns-1 --namespace app-ns-2 --yes
```

## Register the spoke — Method B — Declarative (GitOps-friendly, recommended for prod)

**1. On the SPOKE — create service account, RBAC, token**
```bash
kubectl --context <spoke-context> apply -f - <<'EOF'
apiVersion: v1
kind: ServiceAccount
metadata:
  name: argocd-manager
  namespace: kube-system
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: argocd-manager-role
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
- nonResourceURLs: ["*"]
  verbs: ["*"]
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: argocd-manager-role-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: argocd-manager-role
subjects:
- kind: ServiceAccount
  name: argocd-manager
  namespace: kube-system
apiVersion: v1
kind: Secret
metadata:
  name: argocd-manager-token
  namespace: kube-system
  annotations:
    kubernetes.io/service-account.name: argocd-manager
type: kubernetes.io/service-account-token
EOF

TOKEN=$(kubectl --context <spoke-context> -n kube-system get secret argocd-manager-token -o jsonpath='{.data.token}' | base64 -d)
CA=$(kubectl --context <spoke-context> -n kube-system get secret argocd-manager-token -o jsonpath='{.data.ca\.crt}')   # keep base64
```
Tighten the ClusterRole if the spoke only needs specific namespaces/resources.

**2. On the HUB — create the cluster Secret**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cluster-spoke-staging
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: cluster
    env: staging
    region: uksouth
type: Opaque
stringData:
  name: spoke-staging
  server: https://<spoke-api>:6443
  config: |
    {
      "bearerToken": "<TOKEN>",
      "tlsClientConfig": {
        "insecure": false,
        "caData": "<CA base64>"
      }
    }
```
```bash
kubectl --context <hub-context> apply -f cluster-spoke-staging.yaml
argocd cluster list
```
**Do not commit this Secret in plain text.** Store via Sealed Secrets, External Secrets Operator + Azure Key Vault, or SOPS.

## After registering (both methods)

## 1. Restrict what each spoke can receive (AppProject)
Create this **before** the ApplicationSet in step 4, which references `project: staging`.
```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: staging
  namespace: argocd
spec:
  sourceRepos:
  - https://github.com/<org>/<gitops-repo>.git
  destinations:
  - server: https://<spoke-staging-api>:6443
    namespace: '*'
  clusterResourceWhitelist:
  - group: '*'
    kind: Namespace
```

## 2. Deploy to spokes automatically (ApplicationSet cluster generator)
Any cluster whose Secret carries the matching label gets the app — onboarding a new spoke = add the cluster with the right label.
```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: my-app
  namespace: argocd
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
  - clusters:
      selector:
        matchLabels:
          env: staging
  template:
    metadata:
      name: '{{.name}}-my-app'
    spec:
      project: staging
      source:
        repoURL: https://github.com/<org>/<gitops-repo>.git
        targetRevision: main
        path: 'apps/my-app/overlays/{{index .metadata.labels "env"}}'
      destination:
        server: '{{.server}}'
        namespace: my-app
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
        - CreateNamespace=true
```
Note: the cluster generator excludes the hub's own `in-cluster` unless it has a Secret with matching labels.

## 3. Verify
```bash
argocd cluster list                 # STATUS shows "Unknown" until an app targets the cluster, then "Successful"
argocd app list
argocd cluster get spoke-staging
kubectl -n argocd logs deploy/argocd-applicationset-controller | tail
kubectl -n argocd logs statefulset/argocd-application-controller | grep -i spoke-staging
```

## 4. Operations
- **Rotate token:** delete & recreate `argocd-manager-token` on spoke → update hub Secret (`argocd cluster add` again for Method A).
- **Remove spoke:** `argocd cluster rm https://<spoke-api>:6443` (apps targeting it must be deleted/retargeted first).
- **Scale:** many spokes → enable controller sharding (`ARGOCD_CONTROLLER_REPLICAS` on the application-controller StatefulSet).

## ArgoCD troubleshooting
| Symptom | Fix |
|---|---|
| App `ComparisonError: repository not accessible` | Hub repo credentials missing; `argocd repo add` |
| Synced but pods `ImagePullBackOff` on spoke | Pull secret missing on spoke namespace / spoke outbound 443 |
| Works from your laptop, times out from ArgoCD | Spoke NSG allows your IP but not the hub node subnet; peering only one-way |
| App `project 'x' does not exist` | Create AppProject before the ApplicationSet |
| `x509: certificate is valid for ..., not <ip>` | Add SAN to spoke API server cert, or register using a name already in the SAN |
| `dial tcp ... i/o timeout` | Peering / NSG / route from hub nodes to spoke 6443 |
| `the server has asked for the client to provide credentials` | Token Secret deleted/invalid; regenerate |
| Cluster added but ApplicationSet creates nothing | Labels on cluster Secret don't match `selector.matchLabels` |
| Kubeconfig server is `127.0.0.1` | Edit kubeconfig to real endpoint before `argocd cluster add` |



---

## Checklist

- [ ] VNet peering Connected on both sides; spoke NSG allows hub node subnet on 6443
- [ ] Private DNS zone linked to hub VNet (if private API name)
- [ ] Hub: GitOps repo added (`argocd repo list` Successful)
- [ ] Spoke: pull secrets, ingress, cert-manager, storage ready
- [ ] AppProject created before ApplicationSet
- [ ] Hub → spoke 6443 reachable, cert SAN OK
- [ ] Cluster registered with `env` / `region` labels
- [ ] Credentials stored encrypted (not plain in Git)
- [ ] AppProject restricts destinations
- [ ] ApplicationSet picks up spoke, apps `Synced/Healthy`
