# Deployment Guide

## Overview

A typical Cocoon Stack deployment consists of:

- **Worker nodes** running `cocoon` (the MicroVM engine) and `vk-cocoon` (the Virtual Kubelet provider)
- **Control plane** running `cocoon-operator`, `cocoon-webhook`, and optionally `glance`
- **Snapshot storage** via `epoch` backed by S3-compatible object storage and MySQL

## Worker Prerequisites

Each worker node requires:

- `cocoon` binary installed (default: `/usr/local/bin/cocoon`)
- Cloud Hypervisor binary (use [cocoonstack/cloud-hypervisor](https://github.com/cocoonstack/cloud-hypervisor) `dev` branch for Windows support)
- UEFI firmware (use [cocoonstack/rust-hypervisor-firmware](https://github.com/cocoonstack/rust-hypervisor-firmware) `dev` branch for Windows ACPI shutdown)
- Default Cocoon root at `/data01/cocoon`
- Network access to the Kubernetes API server
- CNI plugins (`bridge`, `host-local`, `loopback`)
- KVM enabled

## Environment Variables

### vk-cocoon

| Variable | Description | Default |
|----------|-------------|---------|
| `KUBECONFIG` | kubeconfig path (outside cluster) | in-cluster config |
| `VK_NODE_NAME` | virtual node name | `cocoon-pool` |
| `VK_NODE_IP` | virtual node IP | auto-detected |
| `COCOON_BIN` | cocoon binary path | `/usr/local/bin/cocoon` |
| `VK_TLS_CERT` | kubelet API cert path | auto-generated |
| `VK_TLS_KEY` | kubelet API key path | auto-generated |

### epoch

| Variable | Description | Default |
|----------|-------------|---------|
| `EPOCH_S3_ENDPOINT` | S3-compatible endpoint | required |
| `EPOCH_S3_BUCKET` | S3 bucket name | required |
| `EPOCH_S3_ACCESS_KEY` | S3 access key | required |
| `EPOCH_S3_SECRET_KEY` | S3 secret key | required |
| `EPOCH_DSN` | MySQL connection string | required |
| `EPOCH_LISTEN` | HTTP listen address | `:4300` |

## Quick Start

```bash
# On the worker node
export KUBECONFIG=$HOME/.kube/config
export VK_NODE_NAME=cocoon-pool
export COCOON_BIN=/usr/local/bin/cocoon

./vk-cocoon
```

## Companion Components

| Component | Purpose | Required |
|-----------|---------|----------|
| [epoch](https://github.com/cocoonstack/epoch) | Remote snapshot storage and pull | For snapshot-based provisioning |
| [cocoon-operator](https://github.com/cocoonstack/cocoon-operator) | CocoonSet and Hibernation CRDs | For managed VM groups |
| [cocoon-webhook](https://github.com/cocoonstack/cocoon-webhook) | Sticky scheduling | For multi-node clusters |
| [glance](https://github.com/cocoonstack/glance) | Web dashboard with browser access | Optional |
