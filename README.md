# virt-gitops

GitOps manifests for deploying demo Fedora virtual machines on OpenShift Virtualization with Argo CD / OpenShift GitOps. This branch targets a home-lab cluster.

## What this deploys

| Resource | Name | Notes |
|---|---|---|
| Project | `berto-virt-gitops-demo` | OpenShift Project with Argo CD managed-by label |
| Secret | `gitops-demo-f44-vm-01-cloudinit` … `vm-04-cloudinit` | Generated from cloud-init overlays; keys `userdata` and `networkdata` |
| VirtualMachine | `gitops-demo-f44-vm-01` … `gitops-demo-f44-vm-04` | Persistent Fedora disks; cloud-init loaded from those Secrets |

Each VM:

- Boots from the cluster `fedora` DataSource (`openshift-virtualization-os-images`)
- Uses a 30Gi DataVolume root disk
- Attaches to Multus network `default/ac4rex-virt-net`
- Gets a static IP via cloud-init `networkData`

| VM | Address |
|---|---|
| `gitops-demo-f44-vm-01` | `172.31.99.231/24` |
| `gitops-demo-f44-vm-02` | `172.31.99.232/24` |
| `gitops-demo-f44-vm-03` | `172.31.99.233/24` |
| `gitops-demo-f44-vm-04` | `172.31.99.234/24` |

Gateway is `172.31.99.1`. Nameserver is `172.31.103.252`.

## Layout

```
.
├── README.md
├── argocd-application.yaml          # Argo CD Application (apply once)
└── manifests/
    ├── kustomization.yaml           # Points at overlays/home-lab
    ├── base/
    │   ├── kustomization.yaml
    │   └── project.yaml             # OpenShift Project
    ├── virtualmachines/
    │   ├── base/
    │   │   ├── kustomization.yaml
    │   │   └── virtualmachine.yaml  # Reference template (not built directly)
    │   └── overlays/
    │       └── vm-01/ … vm-04/      # Per-VM VirtualMachine overlays
    │           ├── kustomization.yaml
    │           └── virtualmachine.yaml
    ├── cloud-init/
    │   ├── base/
    │   │   └── userdata/vm-01 … vm-04   # Reference defaults (not built directly)
    │   └── overlays/
    │       └── home-lab/
    │           ├── kustomization.yaml
    │           ├── userdata/vm-01 … vm-04
    │           └── networkdata/vm-01 … vm-04
    └── overlays/
        └── home-lab/
            ├── kustomization.yaml
            └── patches/multus-network.yaml
```

### Overlay model

| Layer | Path | Purpose |
|---|---|---|
| Base | `manifests/base/` | Shared project and namespace |
| VM overlays | `manifests/virtualmachines/overlays/vm-*/` | One VirtualMachine per overlay |
| Cloud-init overlay | `manifests/cloud-init/overlays/home-lab/` | Per-VM `userdata` and `networkdata` secrets |
| Home-lab overlay | `manifests/overlays/home-lab/` | Composes base, VMs, and cloud-init; patches Multus network |

## Prerequisites

- OpenShift Virtualization installed
- OpenShift GitOps (Argo CD) installed
- Fedora DataSource available in `openshift-virtualization-os-images`
- NetworkAttachmentDefinition `default/ac4rex-virt-net` available to the target project
- Cluster storage that can satisfy 30Gi DataVolumes

## Usage

### 1. Point the Application at this repo

Edit `argocd-application.yaml` and set:

- `spec.source.repoURL` — your Git remote
- `spec.source.path` — `manifests/overlays/home-lab`
- `spec.source.targetRevision` — branch to track (`home-lab`)

### 2. Push this repository

Commit and push so Argo CD can read the overlay path.

### 3. Create the Argo CD Application

```bash
oc apply -f argocd-application.yaml
```

Argo CD syncs `manifests/overlays/home-lab` into `berto-virt-gitops-demo` with automated prune and self-heal.

### 4. Verify

```bash
oc get application berto-gitops-virt-demo -n openshift-gitops
oc get project berto-virt-gitops-demo --show-labels
oc get secret,vm,dv -n berto-virt-gitops-demo
```

### Optional: sync without Git

For a one-off local apply (no Argo CD):

```bash
oc apply -k manifests/
```

Preview rendered manifests:

```bash
kubectl kustomize manifests/overlays/home-lab
```

## Customizing

- **Project** — update `manifests/base/project.yaml` and `spec.destination.namespace` in `argocd-application.yaml`
- **VM count / names** — add `manifests/virtualmachines/overlays/vm-XX/`, list it in `manifests/overlays/home-lab/kustomization.yaml`, and add matching `userdata/vm-XX` and `networkdata/vm-XX` files plus a `secretGenerator` entry in `manifests/cloud-init/overlays/home-lab/kustomization.yaml`
- **User / password / SSH key** — edit `manifests/cloud-init/overlays/home-lab/userdata/vm-*`
- **IPs / DNS / gateway** — edit `manifests/cloud-init/overlays/home-lab/networkdata/vm-*`
- **Multus network** — edit `manifests/overlays/home-lab/patches/multus-network.yaml`

Cloud-init runs at first boot. Changing userdata or networkdata after a VM already has disks will not reconfigure it; recreate the VM (or its DataVolume) if the new values must take effect.

## Notes

- Credentials in cloud-init userdata are for demo use only; rotate or remove them for non-demo environments.
- The Application does **not** use `CreateNamespace`. The OpenShift `Project` in `manifests/base/project.yaml` is what creates `berto-virt-gitops-demo` during sync.
