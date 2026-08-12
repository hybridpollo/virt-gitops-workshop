# virt-gitops

GitOps manifests for deploying demo Fedora virtual machines on OpenShift Virtualization with Argo CD / OpenShift GitOps.

## What this deploys

| Resource | Name | Notes |
|---|---|---|
| Project | `berto-virt-gitops-demo` | OpenShift Project with `cudn: "true"` and Argo CD managed-by label |
| VirtualMachine | `gitops-demo-f44-vm-01` … `gitops-demo-f44-vm-04` | Persistent Fedora disks + cloud-init |

Each VM:

- Boots from the cluster `fedora` DataSource (`openshift-virtualization-os-images`)
- Uses a 30Gi DataVolume root disk
- Attaches to Multus network `vlan-2103-cudn`
- Gets static IPs `172.22.3.101`–`172.22.3.104` via cloud-init `networkData`

## Layout

```
.
├── README.md
├── argocd-application.yaml   # Argo CD Application (apply once)
└── manifests/                # Synced by Argo CD
    ├── kustomization.yaml
    ├── project.yaml          # OpenShift Project (not a bare Namespace)
    ├── vm-01.yaml
    ├── vm-02.yaml
    ├── vm-03.yaml
    └── vm-04.yaml
```

## Prerequisites

- OpenShift Virtualization installed
- OpenShift GitOps (Argo CD) installed
- Fedora DataSource available in `openshift-virtualization-os-images`
- NetworkAttachmentDefinition / CUDN `vlan-2103-cudn` available to the target project
- Cluster storage that can satisfy 30Gi DataVolumes

## Usage

### 1. Point the Application at this repo

Edit `argocd-application.yaml` and set `spec.source.repoURL` to your Git remote.

### 2. Push this repository

Commit and push so Argo CD can read `manifests/`.

### 3. Create the Argo CD Application

```bash
oc apply -f argocd-application.yaml
```

Argo CD syncs `manifests/` into `berto-virt-gitops-demo` with automated prune and self-heal.

### 4. Verify

```bash
oc get application berto-gitops-virt-demo -n openshift-gitops
oc get project berto-virt-gitops-demo --show-labels
oc get vm,dv -n berto-virt-gitops-demo
```

### Optional: sync without Git

For a one-off local apply (no Argo CD):

```bash
oc apply -k manifests/
```

## Customizing

- **Project** — update `manifests/project.yaml` and `spec.destination.namespace` in `argocd-application.yaml`
- **VM count / names** — add or edit `manifests/vm-*.yaml` and list them in `kustomization.yaml`
- **IPs / SSH / password** — edit `cloudInitNoCloud.userData` and `networkData` in each VM
- **Network** — change `multus.networkName` if your CUDN / NAD differs

## Notes

- Credentials in cloud-init are for demo use only; rotate or remove them for non-demo environments.
- The Application does **not** use `CreateNamespace`. The OpenShift `Project` in `manifests/project.yaml` is what creates `berto-virt-gitops-demo` (and its backing namespace) during sync.
