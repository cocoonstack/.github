# Architecture

## Core Model

Cocoon Stack uses the Kubernetes pod as the control-plane record for a MicroVM. There is no separate VM CRD — the pod object carries all VM-specific metadata via annotations.

```
Pod (annotations) → vk-cocoon → cocoon → Cloud Hypervisor → Guest VM
```

### Lifecycle

1. The pod carries Cocoon-specific annotations (`cocoon.cis/*`)
2. `vk-cocoon` resolves the desired VM identity and source image
3. Missing snapshots are pulled from `epoch`
4. `cocoon` boots or clones the VM on the target worker
5. The provider publishes live metadata back to the pod annotations

## Restart Recovery

Provider restart must not create duplicate VMs.

Recovery rules:

1. Derive the stable VM identity from `cocoon.cis/vm-id` and `cocoon.cis/vm-name`
2. Inspect existing Cocoon runtime state before calling create/clone
3. If the VM or suspended snapshot already exists, reattach in-memory state
4. Only create a new VM when no recoverable runtime state exists

## Networking

The provider distinguishes between:

- **Host-side** network bookkeeping from the hypervisor/runtime
- **Guest-visible** IP addresses obtained inside the VM

For managed guests, the provider publishes the guest-reachable address to:

- `status.podIP`
- `cocoon.cis/ip`

## Hibernation

Cocoon Stack does not expose a separate VM CRD for hibernation. Instead:

1. `cocoon-operator` annotates pods for hibernation via the `Hibernation` CRD
2. `vk-cocoon` snapshots the VM, destroys runtime state, and leaves the pod object alive
3. Waking the pod restores the VM from the saved snapshot

This preserves Kubernetes ownership semantics while keeping VM state outside the container abstraction.

## CocoonSet

`CocoonSet` is a CRD managed by `cocoon-operator` that provides:

- Stable slot identities for groups of related VM-backed pods
- Sub-agent scaling with fork-from-main-agent semantics
- Toolbox pod management
- Coordinated suspend/unsuspend across the group

## Snapshot Distribution

```
Worker → cocoon snapshot save → epoch push → S3
                                              ↓
Worker ← cocoon vm clone ← epoch pull ← S3 (on-demand)
```

`epoch` stores VM snapshots as content-addressed blobs in S3-compatible storage, exposes an OCI-style `/v2/` API, and tracks metadata in MySQL.

## Windows Support

Windows is a first-class guest in the Cocoon lifecycle:

- The pod is annotated with `cocoon.cis/os=windows`
- `cocoon` uses UEFI boot with `kvm_hyperv=on`
- ACPI power-button shutdown works with the patched firmware
- Guest metadata is refreshed from live runtime state
- Clone and snapshot follow the same flow as Linux guests

## Component Map

```
┌──────────────────────────────────────────┐
│             Kubernetes API               │
├──────────┬──────────┬───────────────────┤
│ webhook  │ operator │    epoch API      │
│ (admit)  │ (CRDs)  │   (registry)      │
└────┬─────┴────┬─────┴────────┬──────────┘
     │          │              │
     ▼          ▼              ▼
┌──────────────────────────────────────────┐
│        vk-cocoon (Virtual Kubelet)       │
├──────────────────────────────────────────┤
│        cocoon (MicroVM engine)           │
├──────────────────────────────────────────┤
│     Cloud Hypervisor + CLOUDHV.fd        │
├──────────────────────────────────────────┤
│          KVM / Host kernel               │
└──────────────────────────────────────────┘
```
