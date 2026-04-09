# Cocoon Stack

MicroVM platform for AI sandboxing, cloud desktops, and ephemeral dev environments. Built on [Cloud Hypervisor](https://github.com/cloud-hypervisor/cloud-hypervisor) and Kubernetes.

### Core

| Repository | Description |
|---|---|
| [cocoon](https://github.com/cocoonstack/cocoon) | Lightweight MicroVM engine — OCI/cloud images, instant snapshot & clone, Windows 11 support, CNI networking, Docker-like CLI |
| [cloud-hypervisor](https://github.com/cocoonstack/cloud-hypervisor) | Patched Cloud Hypervisor fork — DISCARD fix, virtio-net ctrl_queue tolerance, upstream cherry-picks |
| [rust-hypervisor-firmware](https://github.com/cocoonstack/rust-hypervisor-firmware) | Patched UEFI firmware — ACPI ResetSystem fix for Windows graceful shutdown |
| [windows](https://github.com/cocoonstack/windows) | Windows 11 25H2 image factory — unattended QEMU build, Cloud Hypervisor validation (DHCP, RDP, SAC, ACPI shutdown), published to GHCR as OCI artifacts |

### Kubernetes Integration

| Repository | Description |
|---|---|
| [vk-cocoon](https://github.com/cocoonstack/vk-cocoon) | Virtual Kubelet provider — maps pod lifecycle to VM operations (run, clone, snapshot, hibernate) |
| [epoch](https://github.com/cocoonstack/epoch) | Snapshot registry — S3 blob storage, OCI-style API, web UI, instant VM provisioning |
| [cocoon-operator](https://github.com/cocoonstack/cocoon-operator) | Kubernetes operator — Hibernation and CocoonSet CRDs for stateful VM workflows |
| [cocoon-webhook](https://github.com/cocoonstack/cocoon-webhook) | Admission webhook — sticky scheduling for VM-backed pods |
| [cocoon-net](https://github.com/cocoonstack/cocoon-net) | VPC-native networking — provisions alias IPs (GKE) or ENI secondary IPs (Volcengine) for direct VM DHCP |
| [cocoon-common](https://github.com/cocoonstack/cocoon-common) | Shared Go library — metadata, Kubernetes helpers, logging utilities |

### Documentation

| Project | Docs |
|---------|------|
| vk-cocoon | [Design](https://github.com/cocoonstack/.github/blob/main/docs/vk-cocoon/design.md) · [Deployment](https://github.com/cocoonstack/.github/blob/main/docs/vk-cocoon/deployment.md) |
