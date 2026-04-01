# Cocoon Stack

MicroVM platform for AI sandboxing, cloud desktops, and ephemeral dev environments. Built on [Cloud Hypervisor](https://github.com/cloud-hypervisor/cloud-hypervisor) and Kubernetes.

### Core

| Repository | Description |
|---|---|
| [cocoon](https://github.com/cocoonstack/cocoon) | Lightweight MicroVM engine — OCI/cloud images, instant snapshot & clone, Windows 11 support, CNI networking, Docker-like CLI |
| [cloud-hypervisor](https://github.com/cocoonstack/cloud-hypervisor) | Patched Cloud Hypervisor fork — DISCARD fix, virtio-net ctrl_queue tolerance, upstream cherry-picks |
| [rust-hypervisor-firmware](https://github.com/cocoonstack/rust-hypervisor-firmware) | Patched UEFI firmware — ACPI ResetSystem fix for Windows graceful shutdown |

### Kubernetes Integration

| Repository | Description |
|---|---|
| [vk-cocoon](https://github.com/cocoonstack/vk-cocoon) | Virtual Kubelet provider — maps pod lifecycle to VM operations (run, clone, snapshot, hibernate) |
| [epoch](https://github.com/cocoonstack/epoch) | Snapshot registry — S3 blob storage, OCI-style API, web UI, instant VM provisioning |
| [cocoon-operator](https://github.com/cocoonstack/cocoon-operator) | Kubernetes operator — Hibernation and CocoonSet CRDs for stateful VM workflows |
| [cocoon-webhook](https://github.com/cocoonstack/cocoon-webhook) | Admission webhook — sticky scheduling for VM-backed pods |
| [glance](https://github.com/cocoonstack/glance) | Web dashboard — browser SSH, RDP, VNC, ADB access with auto-discovery |

### Documentation

| Project | Docs |
|---------|------|
| vk-cocoon | [Design](https://github.com/cocoonstack/.github/blob/main/docs/vk-cocoon/design.md) · [Deployment](https://github.com/cocoonstack/.github/blob/main/docs/vk-cocoon/deployment.md) |
