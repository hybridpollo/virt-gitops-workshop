# virt-gitops

GitOps manifests for deploying demo Fedora virtual machines on OpenShift Virtualization with Argo CD / OpenShift GitOps.

## What this deploys

| Resource | Name | Notes |
|---|---|---|
| Project | `berto-virt-gitops-demo` | OpenShift Project with `cudn: "true"` and Argo CD managed-by label |
| Secret | `gitops-demo-f44-vm-01-cloudinit` … `vm-04-cloudinit` | Generated from `manifests/cloud-init/`; keys `userdata` and `networkdata` |
| VirtualMachine | `gitops-demo-f44-vm-01` … `gitops-demo-f44-vm-04` | Persistent Fedora disks; cloud-init loaded from those Secrets |

Each VM:

- Boots from the cluster `fedora` DataSource (`openshift-virtualization-os-images`)
- Uses a 30Gi DataVolume root disk
- Attaches to Multus network `vlan-2103-cudn`
- Gets a static IP via cloud-init `networkData`: `172.22.3.231`–`172.22.3.234`

| VM | Address |
|---|---|
| `gitops-demo-f44-vm-01` | `172.22.3.231/24` |
| `gitops-demo-f44-vm-02` | `172.22.3.232/24` |
| `gitops-demo-f44-vm-03` | `172.22.3.233/24` |
| `gitops-demo-f44-vm-04` | `172.22.3.234/24` |

Gateway is `172.22.3.1`. Nameserver is `1.2.3.4`.

## Layout

```
.
├── README.md
├── argocd-application.yaml   # Argo CD Application (apply once)
└── manifests/                # Synced by Argo CD
    ├── kustomization.yaml    # Resources + cloud-init secretGenerator
    ├── project.yaml          # OpenShift Project (not a bare Namespace)
    ├── vm-01.yaml
    ├── vm-02.yaml
    ├── vm-03.yaml
    ├── vm-04.yaml
    └── cloud-init/           # Per-VM cloud-init (not inline in VMs)
        ├── userdata-vm-01    # User, password, SSH key
        ├── userdata-vm-02
        ├── userdata-vm-03
        ├── userdata-vm-04
        ├── networkdata-vm-01
        ├── networkdata-vm-02
        ├── networkdata-vm-03
        └── networkdata-vm-04
```

Kustomize generates one Secret per VM from those files. Each VM references its Secret with `cloudInitNoCloud.secretRef` and `networkDataSecretRef`. Secret names are stable (`disableNameSuffixHash`).

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
oc get secret,vm,dv -n berto-virt-gitops-demo
```

### Optional: sync without Git

For a one-off local apply (no Argo CD):

```bash
oc apply -k manifests/
```

## Customizing

- **Project** — update `manifests/project.yaml` and `spec.destination.namespace` in `argocd-application.yaml`
- **VM count / names** — add or edit `manifests/vm-*.yaml`, list them in `kustomization.yaml`, add a matching `secretGenerator` entry, and add `cloud-init/userdata-vm-*` plus `cloud-init/networkdata-vm-*` files
- **User / password / SSH key** — edit `manifests/cloud-init/userdata-vm-*` for the VM you want to change
- **IPs / DNS / gateway** — edit `manifests/cloud-init/networkdata-vm-*` for the VM you want to change
- **Network** — change `multus.networkName` if your CUDN / NAD differs

Cloud-init runs at first boot. Changing userdata or networkdata after a VM already has disks will not reconfigure it; recreate the VM (or its DataVolume) if the new values must take effect.

## Notes

- Credentials in `manifests/cloud-init/userdata-vm-*` are for demo use only; rotate or remove them for non-demo environments.
- The Application does **not** use `CreateNamespace`. The OpenShift `Project` in `manifests/project.yaml` is what creates `berto-virt-gitops-demo` (and its backing namespace) during sync.
