# Deployment Guide

## Overview

A typical Cocoon Stack deployment consists of three tiers:

- **Worker nodes** — each runs `cocoon` (the MicroVM engine) and `vk-cocoon` (the Virtual Kubelet provider). Together they present hardware-backed virtual machines as Kubernetes pods.
- **Control plane** — runs `cocoon-operator` (CRDs for CocoonSets, Hibernation, etc.) and `cocoon-webhook` (sticky scheduling & validation). These are standard Deployments inside the cluster.
- **Snapshot storage** — `epoch` serves VM root-disk snapshots. It is backed by S3-compatible object storage and a MySQL metadata database. Workers pull snapshots over HTTP; pushes require a bearer token.

## Worker Prerequisites

Every worker node that will host MicroVMs must satisfy the following:

| Requirement | Details |
|---|---|
| **cocoon binary** | Default path: `/usr/local/bin/cocoon`. Build from the [cocoonstack/cocoon](https://github.com/cocoonstack/cocoon) repository. |
| **Cloud Hypervisor** | Use the [cocoonstack/cloud-hypervisor](https://github.com/cocoonstack/cloud-hypervisor) **`dev`** branch — it includes Windows guest support and vCPU hot-plug fixes. |
| **UEFI firmware** | Use the [cocoonstack/rust-hypervisor-firmware](https://github.com/cocoonstack/rust-hypervisor-firmware) **`dev`** branch — it includes ACPI shutdown support for Windows guests. |
| **Cocoon root directory** | `/data01/cocoon` by default. This is where VM disks, firmware, and runtime state are stored. |
| **Kubernetes API access** | A valid kubeconfig or in-cluster service-account token so `vk-cocoon` can register its virtual node. |
| **KVM enabled** | `/dev/kvm` must exist and be accessible by the user running `vk-cocoon`. |
| **dnsmasq** | Used for DHCP lease tracking. `vk-cocoon` reads the lease file to discover guest IP addresses. |

## Environment Variables

The table below lists every environment variable recognised by `vk-cocoon`.

| Variable | Default | Description |
|---|---|---|
| `KUBECONFIG` | unset | Path to kubeconfig (in-cluster used otherwise). |
| `VK_NODE_NAME` | `cocoon-pool` | Virtual node name registered with the K8s API. |
| `VK_LOG_LEVEL` | `info` | `projecteru2/core/log` level. |
| `EPOCH_URL` | `http://epoch.cocoon-system.svc:8080` | Epoch base URL. |
| `EPOCH_TOKEN` | unset | Bearer token (only needed for `/v2/` pushes; `/dl/` is anonymous). |
| `VK_LEASES_PATH` | `/var/lib/dnsmasq/dnsmasq.leases` | dnsmasq lease file. |
| `VK_COCOON_BIN` | `/usr/local/bin/cocoon` | Path to the cocoon CLI binary. |
| `VK_SSH_PASSWORD` | unset | SSH password for `kubectl logs / exec` against Linux guests. |
| `VK_ORPHAN_POLICY` | `alert` | `alert`, `destroy`, or `keep`. |
| `VK_METRICS_ADDR` | `:9091` | Plain-HTTP prometheus listener. |

> **Note:** Pod annotations use the `vm.cocoonstack.io/*` prefix.
> For example, `vm.cocoonstack.io/cpu-model` or `vm.cocoonstack.io/firmware`.

## Installation

`vk-cocoon` runs as a **host-level binary** managed by systemd — it is *not* deployed inside the cluster. The `packaging/` directory in the repository contains the service unit and an example environment file.

```
sudo install -m 0755 ./vk-cocoon /usr/local/bin/vk-cocoon
sudo install -m 0644 packaging/vk-cocoon.service /etc/systemd/system/vk-cocoon.service
sudo install -m 0644 packaging/vk-cocoon.env.example /etc/cocoon/vk-cocoon.env
sudo systemctl daemon-reload
sudo systemctl enable --now vk-cocoon
```

The systemd unit expects two files on disk:

| File | Purpose |
|---|---|
| `/etc/cocoon/kubeconfig` | Kubeconfig used to register the virtual node. Copy from your cluster admin credentials or generate a scoped service-account kubeconfig. |
| `/etc/cocoon/vk-cocoon.env` | Environment overrides (one `KEY=VALUE` per line). Edit this file to set `VK_NODE_NAME`, `EPOCH_URL`, etc. |

After starting the service, verify the virtual node is registered:

```
kubectl get nodes -l type=virtual-kubelet
```

## Companion Components

| Component | Repository | Purpose |
|---|---|---|
| **epoch** | [cocoonstack/epoch](https://github.com/cocoonstack/epoch) | Remote snapshot storage and pull service. Backs VM root-disk images with S3 + MySQL. |
| **cocoon-operator** | [cocoonstack/cocoon-operator](https://github.com/cocoonstack/cocoon-operator) | CRDs for CocoonSet, Hibernation, and rolling-update lifecycle management. |
| **cocoon-webhook** | [cocoonstack/cocoon-webhook](https://github.com/cocoonstack/cocoon-webhook) | Mutating & validating admission webhook — handles sticky scheduling and annotation defaults. |
| **cocoon-net** | [cocoonstack/cocoon-net](https://github.com/cocoonstack/cocoon-net) | CNI-adjacent network plumbing for MicroVM tap devices and bridge configuration. |

Each component has its own deployment documentation in the respective repository. Epoch's S3, MySQL, and listener configuration is documented in the [epoch README](https://github.com/cocoonstack/epoch#configuration).