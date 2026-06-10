# ArgoCD Installation via Helm — IKS

Cluster: **mycluster-eu-de-1-bxf.8x32** (Frankfurt, VPC Gen2)

## Prerequisites

- `kubectl` configured and pointing at `mycluster-eu-de-1-bxf.8x32`
- `helm` >= 3.x installed
- `ibmcloud` CLI with `ks` plugin installed and logged in
- Cluster admin privileges

---

## Steps

### 1. Add the Argo Helm Repository

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
```

### 2. Create the Namespace

```bash
kubectl create namespace argocd
```

### 3. Deploy the IKS TLS Secret into the `argocd` Namespace

The Ingress controller requires the TLS secret to exist in the same namespace as the Ingress (`argocd`).
Use the `ibmcloud ks` CLI to deploy it — this keeps the secret IBM-managed and auto-renewed:

```bash
ibmcloud ks ingress secret create \
  --cluster mycluster-eu-de-1-bxf.8x32 \
  --cert-crn $(ibmcloud ks ingress secret get \
    --cluster mycluster-eu-de-1-bxf.8x32 \
    --name mycluster-eu-de-1-bxf-8x3-e7d3d93b8b317d269525bf063b24f98d-0000 \
    --namespace default \
    --output json | jq -r '.certCrn') \
  --name mycluster-eu-de-1-bxf-8x3-e7d3d93b8b317d269525bf063b24f98d-0000 \
  --namespace argocd
```

### 4. Install ArgoCD

```bash
helm install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  --values values.yaml
```

Pin a specific version with `--version <VERSION>`.  
Check available versions: `helm search repo argo/argo-cd --versions | head -10`

### 5. Verify the Installation

```bash
kubectl -n argocd get pods
kubectl -n argocd get ingress
```

Wait for all pods to reach `Running` and confirm the Ingress shows the IKS hostname.

### 6. Retrieve the Initial Admin Password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

Open the UI at:

```
https://argocd.mycluster-eu-de-1-bxf-8x3-e7d3d93b8b317d269525bf063b24f98d-0000.eu-de.containers.appdomain.cloud
```

Log in with `admin` and the password above. **Change it immediately.**

### 7. (Optional) Install the ArgoCD CLI

```bash
# macOS
brew install argocd

# Linux
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd && sudo mv argocd /usr/local/bin/
```

```bash
argocd login \
  argocd.mycluster-eu-de-1-bxf-8x3-e7d3d93b8b317d269525bf063b24f98d-0000.eu-de.containers.appdomain.cloud \
  --username admin
```

---

## Upgrading

```bash
helm repo update
helm upgrade argocd argo/argo-cd \
  --namespace argocd \
  --version <NEW_VERSION> \
  --values values.yaml
```

---

## Uninstalling

```bash
helm uninstall argocd --namespace argocd
kubectl delete namespace argocd
```

---

## IKS TLS Certificate

The TLS secret `mycluster-eu-de-1-bxf-8x3-e7d3d93b8b317d269525bf063b24f98d-0000` is the IBM-managed wildcard certificate for this cluster's default Ingress subdomain:

```
*.mycluster-eu-de-1-bxf-8x3-e7d3d93b8b317d269525bf063b24f98d-0000.eu-de.containers.appdomain.cloud
```

This means it covers **any service** exposed via the IKS public ALB on this cluster — not just ArgoCD. The same pattern can be reused for any namespace:

```bash
ibmcloud ks ingress secret create \
  --cluster mycluster-eu-de-1-bxf.8x32 \
  --cert-crn <CRN> \
  --name mycluster-eu-de-1-bxf-8x3-e7d3d93b8b317d269525bf063b24f98d-0000 \
  --namespace <your-namespace>
```

IBM rotates the certificate automatically (~90 days). All namespace copies deployed via `ibmcloud ks ingress secret create` are updated automatically on renewal.

**Scope:**
- ✅ Any `<anything>.<subdomain>.eu-de.containers.appdomain.cloud` hostname on this cluster
- ✅ Auto-renewed by IBM across all managed namespace copies
- ❌ Custom domains (e.g. `myapp.mycompany.com`) — those require a separate cert (cert-manager, Let's Encrypt, etc.)
- ❌ Other clusters — each cluster has its own subdomain and certificate

---

## References

- Argo Helm chart: https://github.com/argoproj/argo-helm/tree/main/charts/argo-cd
- ArgoCD docs: https://argo-cd.readthedocs.io/
- IKS Ingress docs: https://cloud.ibm.com/docs/containers?topic=containers-managed-ingress-about
- Chart values: `helm show values argo/argo-cd`