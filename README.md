# NVIDIA DRA Driver Pack for SpectroCloud Palette

A SpectroCloud Palette addon pack for the NVIDIA Dynamic Resource Allocation (DRA) Driver, enabling flexible GPU allocation in Kubernetes 1.32+.

## Overview

DRA is a Kubernetes feature that provides flexible request and sharing of hardware resources like GPUs. This pack replaces the traditional NVIDIA device plugin approach with a modern, CEL-based device selection mechanism.

## Prerequisites

- Kubernetes 1.32+ (experimental), 1.34+ (GA)
- NVIDIA GPU Operator 25.3.0+
- CDI-enabled container runtime (containerd 1.7+ or CRI-O 1.28+)
- Node Feature Discovery (NFD) for GPU detection

## Key Features

- **Dynamic GPU Allocation**: Request GPUs using DeviceClass and ResourceClaim resources
- **CEL-based Selection**: Filter GPUs by attributes (memory, architecture, UUID)
- **GPU Sharing**: Multiple pods can share access to the same GPU
- **MIG Support**: Multi-Instance GPU partitioning via `mig.nvidia.com` DeviceClass
- **ComputeDomains**: Multi-Node NVLink (MNNVL) support for GB200 systems

## Pack Contents

```
packs/nvidia-dra-driver-25.8.1/
├── pack.json           # Pack metadata & dependencies
├── values.yaml         # Helm configuration
├── README.md           # Pack documentation
└── charts/
    └── nvidia-dra-driver-gpu-25.8.1.tgz  # Helm chart
```

## Quick Start

### Deploy with Helm

```bash
helm install nvidia-dra-driver ./packs/nvidia-dra-driver-25.8.1/charts/nvidia-dra-driver-gpu-25.8.1.tgz \
  --namespace nvidia-dra-driver-gpu \
  --create-namespace \
  -f packs/nvidia-dra-driver-25.8.1/values.yaml
```

### Verify Installation

```bash
kubectl get pods -n nvidia-dra-driver-gpu
kubectl get deviceclasses
kubectl get resourceslices -A
```

### Request a GPU

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata:
  name: gpu-claim
spec:
  devices:
    requests:
      - name: gpu
        deviceClassName: gpu.nvidia.com
---
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  containers:
    - name: cuda
      image: nvcr.io/nvidia/cuda:13.0-base
      resources:
        claims:
          - name: gpu
  resourceClaims:
    - name: gpu
      resourceClaimName: gpu-claim
```

## Documentation

- [NVIDIA-DRA-How-To.md](NVIDIA-DRA-How-To.md) - Comprehensive deployment and usage guide
- [NVIDIA DRA Driver Docs](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/dra-intro-install.html)
- [Kubernetes DRA Docs](https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/)
- [NVIDIA k8s-dra-driver GitHub](https://github.com/NVIDIA/k8s-dra-driver-gpu)
