# ocpv — OpenShift Virtualization GitOps

Kustomize-based manifests and bootstrap scripts for setting up **OpenShift Virtualization** (KubeVirt) and deploying RHEL VirtualMachines via **OpenShift GitOps (ArgoCD)**. Includes operator installation, VM definitions, and an ArgoCD Application for GitOps-driven VM lifecycle management.

---

## Repository Structure

```
.
├── bootstrap.sh                          # Bootstrap script: installs GitOps operator (+ optional OCP-V and ODF)
├── operators/
│   ├── openshift-gitops/                 # Kustomize manifests for OpenShift GitOps operator
│   ├── openshift-virtualization/
│   │   ├── operator/                     # Kustomize manifests for OCP Virtualization operator install
│   │   └── instance/                     # HyperConverged instance (activates the virtualization stack)
│   └── odf/                              # Kustomize manifests for OpenShift Data Foundation (ODF) operator
├── vms/
│   └── rhel/                             # VirtualMachine definitions for RHEL 8 and RHEL 9
│       ├── rhel.yaml                     # Base RHEL VM definition
│       ├── rhel8.yaml                    # RHEL 8 VirtualMachine (KubeVirt)
│       ├── rhel9.yaml                    # RHEL 9 VirtualMachine (KubeVirt)
│       └── kustomization.yaml            # Kustomize overlay for RHEL VMs
└── apps/
    └── ap.yaml                           # ArgoCD Application: deploys RHEL VMs via GitOps
```

---

## Components

### `bootstrap.sh` — Cluster Bootstrap

A step-by-step shell script for bootstrapping the cluster. It installs OpenShift GitOps by default, with the Virtualization and ODF operators commented out for optional sequential installation.

**Active (uncomment to enable):**

| Step | Command | Description |
|---|---|---|
| ✅ Active | `oc apply -k ./operators/openshift-gitops` | Install OpenShift GitOps operator |
| 💬 Commented | `oc apply -k ./operators/openshift-virtualization/operator` | Install OCP Virtualization operator |
| 💬 Commented | `oc apply -k ./operators/openshift-virtualization/instance` | Create HyperConverged instance |
| 💬 Commented | `oc apply -k ./operators/odf` | Install ODF operator |
| 💬 Commented | ODF node labeling + console plugin patch | Enable ODF storage and console UI |

```bash
# Run as cluster-admin
bash bootstrap.sh
```

---

### `operators/openshift-gitops` — GitOps Operator

Kustomize manifests to install the **OpenShift GitOps** operator into the `openshift-gitops-operator` namespace.

| File | Kind | Description |
|---|---|---|
| `subscription.yaml` | Subscription | Installs the `openshift-gitops-operator` from OperatorHub |
| `og.yaml` | OperatorGroup | Scopes the operator to all namespaces |
| `crb.yaml` | ClusterRoleBinding | Grants ArgoCD service account cluster-admin privileges |
| `kustomization.yaml` | Kustomization | Bundles all of the above |

```bash
oc create ns openshift-gitops-operator
oc apply -k ./operators/openshift-gitops
```

---

### `operators/openshift-virtualization` — Virtualization Operator

Two-stage Kustomize install for **OpenShift Virtualization (KubeVirt)**:

- `operator/` — Installs the CNV operator into the `openshift-cnv` namespace
- `instance/` — Creates the `HyperConverged` custom resource that activates the full virtualization stack

```bash
oc create ns openshift-cnv
oc apply -k ./operators/openshift-virtualization/operator
# Wait for operator to become ready, then:
oc apply -k ./operators/openshift-virtualization/instance
```

---

### `operators/odf` — OpenShift Data Foundation

Kustomize manifests to install the **ODF operator** and configure storage for VM persistent volumes.

```bash
oc label node -l node-role.kubernetes.io/infra= cluster.ocs.openshift.io/openshift-storage=
oc create ns openshift-storage
oc apply -k ./operators/odf
# Enable ODF console plugin:
oc patch consoles.operator.openshift.io cluster --patch '{ "spec": { "plugins": ["odf-console"] } }' --type=merge
```

---

### `vms/rhel` — RHEL VirtualMachine Definitions

KubeVirt `VirtualMachine` manifests for RHEL 8 and RHEL 9, deployed into the `vm-gitops` namespace.

| File | OS | Instance Type | Storage | StorageClass |
|---|---|---|---|---|
| `rhel8.yaml` | RHEL 8 | `u1.medium` | 30Gi Block (RWX) | `ocs-storagecluster-ceph-rbd` |
| `rhel9.yaml` | RHEL 9 | `u1.medium` | 30Gi Block (RWX) | `ocs-storagecluster-ceph-rbd` |

Both VMs:
- Boot from the official `rhel8` / `rhel9` DataSource in `openshift-virtualization-os-images`
- Are configured with cloud-init (default user: `cloud-user`, password: `cloud@123`)
- Use `ReadWriteMany` block volumes for live migration support

```bash
oc apply -k ./vms/rhel
```

---

### `apps/ap.yaml` — ArgoCD GitOps Application

An ArgoCD `Application` resource that manages the RHEL VMs declaratively via GitOps.

| Field | Value |
|---|---|
| App name | `rhelvm` |
| Source repo | `https://github.com/gmidha1/virtdemos.git` |
| Source path | `vms/rhel` |
| Target cluster | Local cluster (`https://kubernetes.default.svc`) |
| Target namespace | `vm-gitops` (auto-created) |
| Sync policy | `CreateNamespace=true` |

```bash
oc apply -f apps/ap.yaml
```

> **Note:** The ArgoCD Application currently points to an external reference repo (`gmidha1/virtdemos`). Update `spec.source.repoURL` to point to this repository if you want to use the local `vms/rhel` definitions instead.

---

## Prerequisites

- OpenShift cluster with cluster-admin access
- `oc` CLI installed and logged in
- For VM workloads: bare-metal nodes (or nested virtualisation enabled) with ODF providing `ocs-storagecluster-ceph-rbd` StorageClass

## Recommended Installation Order

1. Run `bootstrap.sh` (installs GitOps operator)
2. Uncomment and apply OCP Virtualization operator + instance
3. Uncomment and apply ODF (if not already installed)
4. Apply VM definitions via `oc apply -k ./vms/rhel` or via the ArgoCD Application in `apps/ap.yaml`
