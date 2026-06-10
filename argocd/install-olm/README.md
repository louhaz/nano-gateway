# ArgoCD Installation Plan — IBM IKS

## Overview

Install ArgoCD on an IBM IKS (Kubernetes Service) cluster using OLM (Operator Lifecycle Manager) and the `operatorhubio-catalog` CatalogSource that ships with the standard OLM installation.
The resulting instance manages **all namespaces** on the cluster.

---

## Prerequisites

| Requirement | Notes |
|---|---|
| `kubectl` configured against the target IKS cluster | `ibmcloud ks cluster config --cluster <name>` |
| OLM already installed | Verify with `kubectl get catalogsource -n olm` |
| `operatorhubio-catalog` CatalogSource present in `olm` namespace | Verify with `kubectl get catalogsource operatorhubio-catalog -n olm` |
| Cluster-admin rights | Required to create ClusterRoleBinding and OperatorGroup |

Verify OLM and the catalog are healthy before proceeding:

```bash
kubectl get pods -n olm
kubectl get catalogsource operatorhubio-catalog -n olm
```

---

## Manifest Summary

| File | Kind | Purpose |
|---|---|---|
| [`01-namespace.yaml`](01-namespace.yaml) | `Namespace` | Creates the `argocd` namespace |
| [`02-operatorgroup.yaml`](02-operatorgroup.yaml) | `OperatorGroup` | Cluster-scoped group (watches all namespaces) |
| [`03-subscription.yaml`](03-subscription.yaml) | `Subscription` | Subscribes to `argocd-operator` from `operatorhubio-catalog`, `alpha` channel, latest version (`v0.18.0`) |
| [`04-argocd-instance.yaml`](04-argocd-instance.yaml) | `ArgoCD` | ArgoCD operand — all-namespace management, OLM health checks, ApplicationSet enabled |
| [`05-cluster-role-binding.yaml`](05-cluster-role-binding.yaml) | `ClusterRoleBinding` | Grants `argocd-argocd-application-controller` the `cluster-admin` role |
| [`06-ingress.yaml`](06-ingress.yaml) | `Ingress` | IKS nginx Ingress with SSL passthrough to `argocd-server:443` |

---

## Step-by-Step Installation

### Step 1 — Prepare

```bash
# Point kubectl at the target cluster
ibmcloud ks cluster config --cluster <CLUSTER_NAME>

# Confirm OLM is running
kubectl get pods -n olm
```

### Step 2 — Ingress hostname ✅

[`06-ingress.yaml`](06-ingress.yaml) is already configured for this cluster:

```
argocd.mycluster-eu-de-1-bxf-8x3-e7d3d93b8b317d269525bf063b24f98d-0000.eu-de.containers.appdomain.cloud
```

### Step 3 — Apply the manifests

Apply in strict order because each step depends on the previous one being healthy.

#### 3a — Create the namespace

```bash
kubectl apply -f 01-namespace.yaml
```

#### 3b — Create the OperatorGroup and Subscription (OLM installs the operator)

```bash
kubectl apply -f 02-operatorgroup.yaml
kubectl apply -f 03-subscription.yaml
```

Wait until the operator CSV is in the `Succeeded` phase:

```bash
kubectl get csv -n argocd --watch
# Expected:  argocd-operator.v<VERSION>   Succeeded
```

Also confirm the `InstallPlan` was approved and completed:

```bash
kubectl get installplan -n argocd
```

#### 3c — Create the ArgoCD instance

```bash
kubectl apply -f 04-argocd-instance.yaml
```

Wait for all ArgoCD pods to become `Running`:

```bash
kubectl get pods -n argocd --watch
```

Expect the following pods (names are prefixed with `argocd-`):

| Pod | Component |
|---|---|
| `argocd-application-controller-*` | GitOps reconciliation loop |
| `argocd-applicationset-controller-*` | ApplicationSet controller |
| `argocd-dex-server-*` | SSO (disabled in this config — pod may not appear) |
| `argocd-redis-*` | In-cluster Redis |
| `argocd-repo-server-*` | Git repository clone/manifest generation |
| `argocd-server-*` | API server and Web UI |

#### 3d — Grant cluster-wide RBAC

```bash
kubectl apply -f 05-cluster-role-binding.yaml
```

> **Note:** The `ClusterRoleBinding` in [`05-cluster-role-binding.yaml`](05-cluster-role-binding.yaml) uses `cluster-admin`.  
> For a least-privilege alternative, swap the `roleRef.name` to `apic-argo-role` (defined in [`../apic-argo-role.yaml`](../apic-argo-role.yaml)) after reviewing which additional API groups your workloads require.

#### 3e — Expose via Ingress

```bash
kubectl apply -f 06-ingress.yaml
```

Verify the Ingress was accepted:

```bash
kubectl get ingress -n argocd
```

---

## Verification

### Check the ArgoCD server is reachable

```bash
ARGOCD_HOST=$(kubectl get ingress argocd-server-ingress -n argocd \
  -o jsonpath='{.spec.rules[0].host}')

echo "ArgoCD UI: https://$ARGOCD_HOST"
```

### Retrieve the initial admin password

The `argocd-operator` stores the initial `admin` password in a Secret named `argocd-cluster`:

```bash
kubectl get secret argocd-cluster -n argocd \
  -o jsonpath='{.data.admin\.password}' | base64 -d
echo
```

### Log in with the ArgoCD CLI

```bash
argocd login $ARGOCD_HOST \
  --username admin \
  --password $(kubectl get secret argocd-cluster -n argocd \
    -o jsonpath='{.data.admin\.password}' | base64 -d)
```

---

## All-Namespace Management

All-namespace management is achieved through two complementary mechanisms:

1. **OperatorGroup with empty `targetNamespaces`** ([`02-operatorgroup.yaml`](02-operatorgroup.yaml))  
   OLM configures the operator's `WATCH_NAMESPACE` environment variable to `""`, meaning the operator pod watches cluster-wide.

2. **`cluster-admin` ClusterRoleBinding** ([`05-cluster-role-binding.yaml`](05-cluster-role-binding.yaml))  
   The `argocd-application-controller` ServiceAccount can read and write resources in every namespace, which is required for ArgoCD to reconcile Applications whose destination namespace differs from `argocd`.

---

## Connecting to the Bootstrap Application

After ArgoCD is running, apply the bootstrap Application from the parent directory to start the GitOps loop:

```bash
kubectl apply -f ../bootstrap.yaml
```

This creates an ArgoCD `Application` that watches the `./argocd` path in the repository and continuously reconciles all downstream Applications and ApplicationSets.
