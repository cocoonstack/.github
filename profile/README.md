# Cocoon

Lightweight MicroVM engine built on [Cloud Hypervisor](https://github.com/cloud-hypervisor/cloud-hypervisor).

| Repository | Description |
|---|---|
| [cocoon](https://github.com/cocoonstack/cocoon) | MicroVM engine — CLI, hypervisor, networking, snapshot & clone |
| [cloud-hypervisor](https://github.com/cocoonstack/cloud-hypervisor) | Patched CH fork (Windows fixes: DISCARD, virtio-net ctrl_queue) |
| [rust-hypervisor-firmware](https://github.com/cocoonstack/rust-hypervisor-firmware) | Patched firmware fork (ACPI ResetSystem fix for Windows shutdown) |
| [vk-cocoon](https://github.com/cocoonstack/vk-cocoon) | Virtual Kubelet provider — Pod to MicroVM lifecycle mapping |
| [epoch](https://github.com/cocoonstack/epoch) | Snapshot registry — manifests, blobs, distribution, token auth, optional Google/OIDC UI login |
| [cocoon-operator](https://github.com/cocoonstack/cocoon-operator) | Kubernetes controllers — `CocoonSet` and `Hibernation` |
| [cocoon-webhook](https://github.com/cocoonstack/cocoon-webhook) | Optional admission webhook — sticky scheduling for stateful VM-backed pods |
