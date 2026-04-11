# vk-cocoon — Design & Architecture

Virtual Kubelet provider that maps Kubernetes pods to Cocoon MicroVMs.

---

## 1. Core Model

Cocoon Stack uses the **Kubernetes pod as the sole control-plane record** for a
MicroVM. There is no separate VM CRD — the pod object itself carries all
VM-specific metadata via annotations under the `vm.cocoonstack.io` domain.

```
Pod (annotations) → vk-cocoon → cocoon CLI → Cloud Hypervisor → Guest VM
```

Key annotations (all prefixed `vm.cocoonstack.io/`):

| Annotation | Written by | Purpose |
|---|---|---|
| `vm.cocoonstack.io/id` | provider | Runtime VMID assigned after boot/clone |
| `vm.cocoonstack.io/name` | operator / user | Stable VM name (deterministic per slot) |
| `vm.cocoonstack.io/ip` | provider | Guest-reachable IP resolved from dnsmasq lease |
| `vm.cocoonstack.io/image` | operator / user | Source cloud image or snapshot reference |
| `vm.cocoonstack.io/os` | operator / user | Guest OS (`linux` or `windows`) |
| `vm.cocoonstack.io/mode` | operator / user | Boot mode: `clone` (default) or `run` |
| `vm.cocoonstack.io/managed` | operator / user | `"true"` (provider drives lifecycle) or `"false"` (external) |
| `vm.cocoonstack.io/snapshot-policy` | operator / user | `always`, `main-only`, or `never` |
| `vm.cocoonstack.io/hibernate` | operator | Hibernate state flag (`"true"` / `"false"`) |
| `vm.cocoonstack.io/hibernate-epoch` | provider | Content-addressed tag tracking the hibernate snapshot |
| `vm.cocoonstack.io/vnc-port` | provider / operator | VNC port (pre-assigned for unmanaged VMs) |

---

## 2. Lifecycle

### 2.1 CreatePod

1. Parse `meta.VMSpec` from pod annotations.
2. If a VM with `spec.VMName` already exists locally, **adopt** it (idempotent
   on restart).
3. Otherwise branch on **Managed** first, then **Mode**:

| Managed | Mode | Behaviour |
|---|---|---|
| `false` | — | Static / externally-managed VM. Skip the runtime entirely; adopt the pre-assigned VMID, IP, and VNCPort the operator pre-wrote into the annotations. |
| `true` | `clone` (default) | Pull the snapshot from epoch via `Puller.PullSnapshot` if not cached locally, then `Runtime.Clone`. |
| `true` | `run` | `Runtime.Run(image=spec.Image, name=spec.VMName)` — fresh boot from a cloud image. |

4. Resolve the guest IP from the **dnsmasq lease file** by MAC address.
5. `meta.VMRuntime{VMID, IP}.Apply(pod)` writes runtime annotations
   (`vm.cocoonstack.io/id`, `vm.cocoonstack.io/ip`) back to the pod.

### 2.2 DeletePod

1. Decode `meta.VMSpec`.
2. `meta.ShouldSnapshotVM(spec)` decides whether to snapshot before destroy:

| Policy | Behaviour |
|---|---|
| `always` | `Runtime.SnapshotSave` → `Pusher.PushSnapshot` |
| `main-only` | Same, but **only** when VM name ends in `-0` |
| `never` | Skip snapshot entirely |

3. `Runtime.Remove(vmID)`.
4. Forget the pod from in-memory tables.

### 2.3 UpdatePod

Only **HibernateState transitions** are acted upon; any other annotation change
is a no-op.

| Transition | Steps |
|---|---|
| `false → true` (hibernate) | `SnapshotSave` → `PushSnapshot(tag=hibernate)` → `Remove`. Compensating **rollback** if `Remove` fails. |
| `true → false` (wake) | `PullSnapshot(tag=hibernate)` → `Clone` → drop hibernate tag annotation. |

---

## 3. Startup Reconcile

**Cluster state is the source of truth.** There is no persistent `pods.json`.

On provider (re)start:

1. List pods assigned to this node via `fieldSelector`.
2. List VMs from `Runtime.List`.
3. **Adopt** every pod that matches a running VM.
4. Apply the configured **orphan policy** for unmatched VMs:

| Policy | Behaviour |
|---|---|
| `alert` (default) | Log a warning + emit a Prometheus metric; leave VM running. |
| `destroy` | `Runtime.Remove` the orphaned VM. |
| `keep` | Silently ignore. |

---

## 4. Networking

IP resolution relies on the **dnsmasq lease file** written by the host-side
DHCP server.

```
cocoon clone → VM boots → DHCP request → dnsmasq lease file
                                                   ↓
                             vk-cocoon ← lease parser (network/)
                                                   ↓
                         pod annotation: vm.cocoonstack.io/ip
```

The provider distinguishes between:

- **Host-side** network bookkeeping (MAC → IP from the lease file, managed by
  the hypervisor/runtime).
- **Guest-visible** IP address reported to Kubernetes via `status.podIP` and
  the `vm.cocoonstack.io/ip` annotation.

---

## 5. Hibernation

Cocoon Stack does not expose a separate VM CRD for hibernation. The existing
pod object stays alive throughout the cycle:

1. `cocoon-operator` sets `vm.cocoonstack.io/hibernate=true` via the
   `Hibernation` CRD.
2. `vk-cocoon` snapshots the VM, pushes to epoch with a **hibernate tag**,
   destroys runtime state, and **leaves the pod object alive**.
3. The hibernate epoch tag is recorded in
   `vm.cocoonstack.io/hibernate-epoch` for content-addressed tracking.
4. Waking (`hibernate=false`) pulls the hibernate snapshot and restores the VM
   via `Clone`.

This preserves Kubernetes ownership semantics while keeping VM state outside the
container abstraction.

---

## 6. CocoonSet

`CocoonSet` is a CRD managed by `cocoon-operator` that provides:

- **Stable slot identities** — each member gets a deterministic
  `vm.cocoonstack.io/name` (e.g. `myagent-0`, `myagent-1`, …).
- **Sub-agent scaling** with **fork-from-main** semantics — new slots are
  cloned from the `-0` (main) VM's snapshot.
- **Toolbox pods** — utility pods associated with the set but not part of the
  numbered slot sequence.
- **Coordinated suspend / unsuspend** — the operator drives hibernate
  transitions across the entire group atomically.

---

## 7. Snapshot Distribution

```
Worker → cocoon snapshot save → epoch push → S3
                                              ↓
Worker ← cocoon vm clone    ← epoch pull  ← S3  (on-demand)
```

- `epoch` stores VM snapshots as **content-addressed blobs** in S3-compatible
  storage.
- It exposes an OCI-style `/v2/` API and tracks metadata in MySQL.
- The `snapshots/` package wraps epoch as a `RegistryClient` interface, with
  dedicated `Puller` (stream snapshot + cloud images via `epoch/snapshot` and
  `epoch/cloudimg`) and `Pusher` (upload after save) helpers.

---

## 8. Windows Support

Windows is a **first-class** guest in the Cocoon lifecycle:

- The pod is annotated with `vm.cocoonstack.io/os=windows`.
- `cocoon` uses **UEFI boot** with `kvm_hyperv=on`.
- **ACPI power-button shutdown** works with the patched CLOUDHV firmware.
- Guest metadata is refreshed from live runtime state.
- Clone and snapshot follow the **same flow** as Linux guests.
- Guest exec falls back to the **RDP help-text shim** (`guest/`) instead of
  SSH.

---

## 9. Component Map

```
┌──────────────────────────────────────────────┐
│              Kubernetes API                   │
├───────────┬───────────┬──────────────────────┤
│  webhook  │  operator │      epoch API       │
│  (admit)  │  (CRDs)   │     (registry)       │
└─────┬─────┴─────┬─────┴──────────┬───────────┘
      │           │                │
      ▼           ▼                ▼
┌──────────────────────────────────────────────┐
│         vk-cocoon  (Virtual Kubelet)         │
│  ┌──────┬──────────┬─────────┬─────────────┐ │
│  │ vm/  │snapshots/│network/ │guest/probes/│ │
│  │      │          │         │metrics/     │ │
│  └──────┴──────────┴─────────┴─────────────┘ │
├──────────────────────────────────────────────┤
│         cocoon  (MicroVM engine)             │
├──────────────────────────────────────────────┤
│      Cloud Hypervisor  +  CLOUDHV.fd         │
├──────────────────────────────────────────────┤
│           KVM  /  Host kernel                │
└──────────────────────────────────────────────┘
```

---

## 10. Layered Architecture

| Layer | Package | Responsibility |
|---|---|---|
| **Application** | `package main` | `CocoonProvider` — lifecycle methods (`CreatePod`, `DeletePod`, `UpdatePod`, `GetPodStatus`, `GetContainerLogs`, `RunInContainer`), startup reconcile, orphan policy. |
| **Cocoon CLI** | `vm/` | `Runtime` interface + the default `CocoonCLI` implementation that shells out to `sudo cocoon …`. |
| **Snapshot SDK** | `snapshots/` | Wraps the epoch SDK as a `RegistryClient` interface, plus `Puller` and `Pusher` that stream snapshots and cloud images via `epoch/snapshot` and `epoch/cloudimg`. |
| **Network** | `network/` | dnsmasq lease parser — resolves a freshly cloned VM's IP by MAC address. |
| **Guest exec** | `guest/` | SSH executor (Linux) and RDP help-text shim (Windows). |
| **Probes** | `probes/` | In-memory readiness / liveness map for the `GetPodStatus` hot path. |
| **Metrics** | `metrics/` | Prometheus collectors for pod lifecycle, snapshot pull / push, VM table size, orphans. |
| **Build metadata** | `version/` | `ldflags`-injected version / revision / built-at strings. |