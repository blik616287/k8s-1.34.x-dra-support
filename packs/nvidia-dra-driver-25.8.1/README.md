# NVIDIA DRA Driver for GPUs

This pack installs the NVIDIA Dynamic Resource Allocation (DRA) Driver for GPUs, enabling flexible GPU allocation in Kubernetes 1.32+.

## Overview

DRA is a Kubernetes feature that provides flexible request and sharing of hardware resources like GPUs. The NVIDIA DRA Driver replaces the traditional NVIDIA device plugin approach with a more modern, CEL-based device selection mechanism.

## Prerequisites

- Kubernetes 1.32 or newer (DRA is GA in 1.34+)
- NVIDIA GPU Operator 25.3.0+ (for driver management and CDI support)
- CDI enabled in the container runtime (containerd/CRI-O)
- Node Feature Discovery (NFD) for GPU detection

## Key Features

- **Dynamic GPU Allocation**: Request GPUs using DeviceClass and ResourceClaim resources
- **CEL-based Selection**: Filter GPUs by attributes using Common Expression Language
- **GPU Sharing**: Multiple pods can share access to the same GPU
- **ComputeDomains**: Support for Multi-Node NVLink (MNNVL) on GB200 systems

## Configuration

### Driver Root Path

When using with GPU Operator (recommended):
```yaml
nvidiaDriverRoot: /run/nvidia/driver
```

When drivers are installed directly on host:
```yaml
nvidiaDriverRoot: /
```

### Enable/Disable Resources

```yaml
resources:
  gpus:
    enabled: true        # Enable GPU allocation
  computeDomains:
    enabled: false       # Enable for MNNVL systems
```

## Usage

After installation, the DRA driver creates:
- A default `DeviceClass` named `gpu.nvidia.com`
- `ResourceSlice` objects representing available GPUs on each node

### Example ResourceClaimTemplate

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaimTemplate
metadata:
  name: gpu-claim
spec:
  spec:
    devices:
      requests:
        - name: gpu
          deviceClassName: gpu.nvidia.com
```

### Example Pod Using DRA

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  containers:
    - name: cuda
      image: nvidia/cuda:12.0-base
      resources:
        claims:
          - name: gpu
  resourceClaims:
    - name: gpu
      resourceClaimTemplateName: gpu-claim
```

## Documentation

- [NVIDIA DRA Driver Documentation](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/dra-intro-install.html)
- [Kubernetes DRA Documentation](https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/)
- [GitHub Repository](https://github.com/NVIDIA/k8s-dra-driver-gpu)
