# Cocoon Stack

MicroVM platform for AI sandboxing, cloud desktops, and ephemeral dev environments. Built on [Cloud Hypervisor](https://github.com/cloud-hypervisor/cloud-hypervisor) / [Firecracker](https://github.com/firecracker-microvm/firecracker) and Kubernetes.

**Documentation: [cocoonstack.github.io](https://cocoonstack.github.io/)**

### Runtime

| Repository | Description | Docs |
|---|---|---|
| [cocoon](https://github.com/cocoonstack/cocoon) | Lightweight MicroVM engine — OCI/cloud images, instant snapshot & clone, Windows 11 support, CNI networking, Docker-like CLI | [docs](https://cocoonstack.github.io/cocoon/) |
| [cocoon-agent](https://github.com/cocoonstack/cocoon-agent) | In-VM exec agent — vsock command channel behind `cocoon vm exec` / `logs` and per-clone identity reseed, no SSH | [docs](https://cocoonstack.github.io/cocoon-agent/) |
| [cocoon-macos](https://github.com/cocoonstack/cocoon-macos) | macOS guests via QEMU + OpenCore + OVMF, reusing cocoon's cloudimg store and clone/snapshot model | [docs](https://cocoonstack.github.io/cocoon-macos/) |
| [cloud-hypervisor](https://github.com/cocoonstack/cloud-hypervisor) | Patched Cloud Hypervisor fork — DISCARD fix, virtio-net ctrl_queue tolerance, upstream cherry-picks | — |
| [rust-hypervisor-firmware](https://github.com/cocoonstack/rust-hypervisor-firmware) | Patched UEFI firmware — ACPI ResetSystem fix for Windows graceful shutdown | — |
| [windows](https://github.com/cocoonstack/windows) | Windows 11 25H2 image factory — unattended QEMU build, Cloud Hypervisor validation (DHCP, RDP, SAC, ACPI shutdown), published to GHCR as OCI artifacts | — |

### Kubernetes Integration

| Repository | Description | Docs |
|---|---|---|
| [vk-cocoon](https://github.com/cocoonstack/vk-cocoon) | Virtual Kubelet provider — maps pod lifecycle to VM operations (run, clone, snapshot, hibernate) | [docs](https://cocoonstack.github.io/vk-cocoon/) |
| [cocoon-operator](https://github.com/cocoonstack/cocoon-operator) | Kubernetes operator — CocoonSet and CocoonHibernation CRDs for stateful VM workflows | [docs](https://cocoonstack.github.io/cocoon-operator/) |
| [cocoon-webhook](https://github.com/cocoonstack/cocoon-webhook) | Admission webhook — sticky scheduling plus CocoonSet and CocoonHibernation validation | [docs](https://cocoonstack.github.io/cocoon-webhook/) |
| [cocoon-net](https://github.com/cocoonstack/cocoon-net) | VPC-native networking — embedded DHCP plus alias IPs (GKE) or ENI secondary IPs (Volcengine) for direct VM DHCP | [docs](https://cocoonstack.github.io/cocoon-net/) |
| [cocoon-common](https://github.com/cocoonstack/cocoon-common) | Shared Go library — CRD types, annotation contract, Kubernetes helpers, OCI registry + snapshot packages | — |

### Services

| Repository | Description | Docs |
|---|---|---|
| [gateway](https://github.com/cocoonstack/gateway) | Single-binary LLM gateway (`gw`) in Rust — OpenAI/Anthropic-compatible APIs, multi-provider routing, per-key auth / quotas / rate-limits, failover, billing ledger | [docs](https://cocoonstack.github.io/gateway/) |

### Sandbox

| Repository | Description | Docs |
|---|---|---|
| [sandbox](https://github.com/cocoonstack/sandbox) | Fast cold-boot AI-agent sandbox on cocoon — warm pools, in-guest daemon, per-node control plane, Go SDK | [docs](https://cocoonstack.github.io/sandbox/) |
