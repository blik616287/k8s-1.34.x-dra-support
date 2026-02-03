# NVIDIA Dynamic Resource Allocation (DRA) Driver for Kubernetes

## Executive Summary

Dynamic Resource Allocation (DRA) is a Kubernetes API that provides a more flexible and powerful alternative to the traditional device plugin mechanism for managing hardware resources like GPUs. The NVIDIA DRA Driver enables Kubernetes clusters to dynamically allocate NVIDIA GPUs to workloads using this modern API, offering enhanced capabilities for GPU sharing, topology awareness, and fine-grained resource selection.

This document provides a comprehensive guide to understanding, deploying, and using the NVIDIA DRA Driver in Kubernetes environments.

---

## Table of Contents

1. [Introduction to DRA](#introduction-to-dra)
2. [Architecture Overview](#architecture-overview)
3. [Prerequisites and Dependencies](#prerequisites-and-dependencies)
4. [Deployment Guide](#deployment-guide)
5. [Configuration Reference](#configuration-reference)
6. [Usage Guide](#usage-guide)
7. [Advanced Features](#advanced-features)
8. [Troubleshooting](#troubleshooting)
9. [Migration from Device Plugin](#migration-from-device-plugin)

---

## Introduction to DRA

### What is Dynamic Resource Allocation?

Dynamic Resource Allocation (DRA) is a Kubernetes feature that reached General Availability (GA) in Kubernetes 1.34. It provides a generic framework for requesting and sharing hardware resources between pods and containers, replacing the limitations of the traditional device plugin API.

### DRA vs Device Plugin Comparison

| Feature | Device Plugin (Legacy) | DRA (Modern) |
|---------|----------------------|--------------|
| **API Maturity** | Stable since K8s 1.10 | GA in K8s 1.34 |
| **Resource Model** | Simple count-based | Rich attribute-based |
| **GPU Selection** | Random assignment | CEL expressions for precise selection |
| **Sharing** | Limited | Native time-slicing, MIG, MPS support |
| **Topology Awareness** | Basic | Full NUMA/NVLink/PCIe awareness |
| **Resource Claims** | Not supported | First-class ResourceClaim objects |
| **Partitioning** | Static only | Dynamic partitioning |
| **Multi-GPU Selection** | By count only | By attributes (memory, architecture, etc.) |

### Key Benefits of DRA

1. **Fine-grained Selection**: Select GPUs based on specific attributes (memory size, architecture, PCIe topology)
2. **Resource Claims**: Decouple resource specification from pod specification
3. **Sharing Policies**: Native support for GPU sharing strategies
4. **Topology Optimization**: Schedule workloads considering NVLink, PCIe, and NUMA topology
5. **Future-proof**: Extensible framework for new resource types and features

---

## Architecture Overview

### Component Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Kubernetes Cluster                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    Control Plane Components                  │   │
│  │  ┌─────────────────┐  ┌─────────────────┐                   │   │
│  │  │  kube-scheduler │  │  kube-apiserver │                   │   │
│  │  │  (DRA-aware)    │  │                 │                   │   │
│  │  └────────┬────────┘  └────────┬────────┘                   │   │
│  │           │                    │                             │   │
│  │           │         ┌──────────┴──────────┐                 │   │
│  │           │         │                     │                 │   │
│  │           │    ┌────┴────┐          ┌────┴────┐            │   │
│  │           │    │ Device  │          │Resource │            │   │
│  │           │    │ Classes │          │ Slices  │            │   │
│  │           │    └─────────┘          └─────────┘            │   │
│  └───────────┼──────────────────────────────────────────────────┘   │
│              │                                                      │
│  ┌───────────┼──────────────────────────────────────────────────┐   │
│  │           │              GPU Worker Node                      │   │
│  │           ▼                                                   │   │
│  │  ┌─────────────────────────────────────────────────────────┐ │   │
│  │  │           DRA Kubelet Plugin (DaemonSet)                │ │   │
│  │  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐ │ │   │
│  │  │  │ GPU         │  │ Resource    │  │ CDI             │ │ │   │
│  │  │  │ Discovery   │  │ Advertising │  │ Spec Generator  │ │ │   │
│  │  │  └─────────────┘  └─────────────┘  └─────────────────┘ │ │   │
│  │  └─────────────────────────────────────────────────────────┘ │   │
│  │                            │                                  │   │
│  │                            ▼                                  │   │
│  │  ┌─────────────────────────────────────────────────────────┐ │   │
│  │  │              NVIDIA Driver (GPU Operator)               │ │   │
│  │  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐ │ │   │
│  │  │  │ Kernel      │  │ CUDA        │  │ nvidia-smi      │ │ │   │
│  │  │  │ Modules     │  │ Libraries   │  │ Management      │ │ │   │
│  │  │  └─────────────┘  └─────────────┘  └─────────────────┘ │ │   │
│  │  └─────────────────────────────────────────────────────────┘ │   │
│  │                            │                                  │   │
│  │                            ▼                                  │   │
│  │  ┌─────────────────────────────────────────────────────────┐ │   │
│  │  │                    NVIDIA GPUs                          │ │   │
│  │  │    [GPU 0]    [GPU 1]    [GPU 2]    [GPU 3]            │ │   │
│  │  └─────────────────────────────────────────────────────────┘ │   │
│  └───────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Core Components

#### 1. DeviceClass
Cluster-scoped resource that defines a class of devices with selection criteria.

```yaml
apiVersion: resource.k8s.io/v1
kind: DeviceClass
metadata:
  name: gpu.nvidia.com
spec:
  selectors:
  - cel:
      expression: device.driver == 'gpu.nvidia.com' && device.attributes['gpu.nvidia.com'].type == 'gpu'
```

#### 2. ResourceSlice
Represents available devices on a node, created automatically by the DRA kubelet plugin.

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceSlice
metadata:
  name: gpu-node-01-gpu.nvidia.com-xxxxx
spec:
  driver: gpu.nvidia.com
  nodeName: gpu-node-01
  pool:
    name: gpu-node-01
  devices:
  - name: gpu-0
    attributes:
      productName:
        string: "Quadro RTX 6000"
      architecture:
        string: "Turing"
      memory:
        quantity: "24Gi"
```

#### 3. ResourceClaim
Namespaced resource that requests specific devices for a workload.

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata:
  name: my-gpu-claim
spec:
  devices:
    requests:
    - name: gpu
      exactly:
        deviceClassName: gpu.nvidia.com
        count: 1
```

#### 4. DRA Kubelet Plugin
DaemonSet that runs on GPU nodes, responsible for:
- Discovering available GPUs
- Publishing ResourceSlices to the API server
- Preparing devices for pod consumption via CDI

---

## Prerequisites and Dependencies

### Kubernetes Version Requirements

| Component | Minimum Version | Recommended Version |
|-----------|-----------------|---------------------|
| Kubernetes | 1.32.0 | 1.34.x |
| Container Runtime | containerd 1.7+ | containerd 2.0+ |
| CRI-O | 1.28+ | 1.32+ |

### Required Components

#### 1. NVIDIA GPU Operator
The GPU Operator manages the NVIDIA driver and related components.

**Version Compatibility:**
| DRA Driver Version | GPU Operator Version | CUDA Version |
|-------------------|---------------------|--------------|
| v25.8.1 | v25.3.0+ | 13.0+ |
| v25.6.x | v25.1.0+ | 12.8+ |

**Key GPU Operator Settings for DRA:**
```yaml
# GPU Operator must NOT deploy the legacy device plugin
devicePlugin:
  enabled: false  # Disable legacy device plugin when using DRA
```

#### 2. Node Feature Discovery (NFD)
NFD labels nodes with hardware information used for scheduling.

**Required Labels:**
- `feature.node.kubernetes.io/pci-10de.present=true` (NVIDIA PCI vendor ID)
- `nvidia.com/gpu.present=true` (set by GPU Operator)

#### 3. Container Device Interface (CDI)
CDI is the standard for exposing devices to containers. The DRA driver generates CDI specs for GPU allocation.

**CDI Directory:** `/var/run/cdi/`

### Hardware Requirements

- NVIDIA GPU with Compute Capability 5.0+ (Maxwell or newer)
- Supported GPU families: Maxwell, Pascal, Volta, Turing, Ampere, Hopper, Blackwell
- PCIe 3.0+ recommended

### Network Requirements

- Access to `nvcr.io` container registry
- Kubernetes API server accessible from all nodes

---

## Deployment Guide

### Step 1: Install GPU Operator

Deploy the NVIDIA GPU Operator with DRA-compatible settings:

```yaml
# gpu-operator-values.yaml
operator:
  defaultRuntime: containerd

driver:
  enabled: true
  version: "580.105.08"

toolkit:
  enabled: true

devicePlugin:
  enabled: false  # IMPORTANT: Disable for DRA

dcgmExporter:
  enabled: true

gfd:
  enabled: true

nfd:
  enabled: true
```

```bash
helm install gpu-operator nvidia/gpu-operator \
  --namespace gpu-operator \
  --create-namespace \
  -f gpu-operator-values.yaml
```

### Step 2: Verify GPU Operator Health

```bash
# Check all GPU Operator pods are running
kubectl get pods -n gpu-operator

# Verify driver is installed
kubectl logs -n gpu-operator -l app=nvidia-driver-daemonset

# Check GPU node labels
kubectl get nodes --show-labels | grep nvidia
```

### Step 3: Deploy NVIDIA DRA Driver

```yaml
# nvidia-dra-driver-values.yaml
charts:
  nvidia-dra-driver-gpu:
    # Path where GPU Operator installs drivers
    nvidiaDriverRoot: /run/nvidia/driver

    # Enable GPU resources
    gpuResourcesEnabledOverride: true

    resources:
      gpus:
        enabled: true
      computeDomains:
        enabled: false

    image:
      repository: nvcr.io/nvidia/k8s-dra-driver-gpu
      tag: "v25.8.1"
      pullPolicy: IfNotPresent

    # Log verbosity (0-7)
    logVerbosity: "4"

    # Kubelet plugin runs on GPU nodes
    kubeletPlugin:
      priorityClassName: "system-node-critical"
      tolerations:
        - key: nvidia.com/gpu
          operator: Exists
          effect: NoSchedule
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: feature.node.kubernetes.io/pci-10de.present
                    operator: In
                    values:
                      - "true"
              - matchExpressions:
                  - key: nvidia.com/gpu.present
                    operator: In
                    values:
                      - "true"
```

```bash
# Deploy via Helm
helm install nvidia-dra-driver nvidia/nvidia-dra-driver-gpu \
  --namespace nvidia-dra-driver-gpu \
  --create-namespace \
  -f nvidia-dra-driver-values.yaml
```

### Step 4: Verify DRA Driver Deployment

```bash
# Check DRA driver pods
kubectl get pods -n nvidia-dra-driver-gpu -o wide

# Verify DeviceClasses are created
kubectl get deviceclasses

# Check ResourceSlices (available GPUs)
kubectl get resourceslices -o wide

# View GPU details in ResourceSlice
kubectl get resourceslices -o yaml
```

**Expected Output:**
```
NAME                            NODE         DRIVER           POOL         AGE
gpu-node-01-gpu.nvidia.com-xx   gpu-node-01  gpu.nvidia.com   gpu-node-01  5m
```

---

## Configuration Reference

### Helm Values Reference

| Parameter | Description | Default |
|-----------|-------------|---------|
| `nvidiaDriverRoot` | Path to NVIDIA driver installation | `/run/nvidia/driver` |
| `gpuResourcesEnabledOverride` | Enable GPU resource publishing | `false` |
| `resources.gpus.enabled` | Enable GPU device class | `true` |
| `resources.computeDomains.enabled` | Enable compute domains | `false` |
| `logVerbosity` | Log level (0-7) | `4` |
| `image.repository` | Container image repository | `nvcr.io/nvidia/k8s-dra-driver-gpu` |
| `image.tag` | Container image tag | `v25.8.1` |
| `webhook.enabled` | Enable admission webhook | `false` |

### Driver Root Paths

| Scenario | `nvidiaDriverRoot` Value |
|----------|-------------------------|
| GPU Operator (containerized driver) | `/run/nvidia/driver` |
| Host-installed driver | `/` |
| Custom driver location | `/path/to/driver` |

### DeviceClass Configuration

The DRA driver creates the following DeviceClasses by default:

| DeviceClass | Description | Selector |
|-------------|-------------|----------|
| `gpu.nvidia.com` | All NVIDIA GPUs | `device.attributes['gpu.nvidia.com'].type == 'gpu'` |
| `mig.nvidia.com` | MIG instances | `device.attributes['gpu.nvidia.com'].type == 'mig'` |

---

## Usage Guide

### Basic GPU Request

#### 1. Create a ResourceClaim

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata:
  name: single-gpu
  namespace: default
spec:
  devices:
    requests:
    - name: gpu
      exactly:
        deviceClassName: gpu.nvidia.com
        count: 1
```

#### 2. Create a Pod Referencing the Claim

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
  namespace: default
spec:
  restartPolicy: Never
  containers:
  - name: cuda-app
    image: nvcr.io/nvidia/cuda:13.0.0-base-ubuntu22.04
    command: ["nvidia-smi"]
    resources:
      claims:
      - name: gpu
  resourceClaims:
  - name: gpu
    resourceClaimName: single-gpu
```

### Multiple GPUs

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata:
  name: multi-gpu
spec:
  devices:
    requests:
    - name: gpus
      exactly:
        deviceClassName: gpu.nvidia.com
        count: 4
```

### GPU Selection by Attributes

Select GPUs with specific characteristics using CEL expressions:

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata:
  name: high-memory-gpu
spec:
  devices:
    requests:
    - name: gpu
      exactly:
        deviceClassName: gpu.nvidia.com
        count: 1
        selectors:
        - cel:
            expression: device.capacity['memory'].compareTo(quantity('20Gi')) >= 0
```

### Select by GPU Architecture

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata:
  name: ampere-gpu
spec:
  devices:
    requests:
    - name: gpu
      exactly:
        deviceClassName: gpu.nvidia.com
        count: 1
        selectors:
        - cel:
            expression: device.attributes['gpu.nvidia.com'].architecture == 'Ampere'
```

### Select by Product Name

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata:
  name: specific-gpu
spec:
  devices:
    requests:
    - name: gpu
      exactly:
        deviceClassName: gpu.nvidia.com
        count: 1
        selectors:
        - cel:
            expression: device.attributes['gpu.nvidia.com'].productName == 'NVIDIA A100-SXM4-80GB'
```

### ResourceClaimTemplate for Deployments

For Deployments and StatefulSets, use ResourceClaimTemplate:

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaimTemplate
metadata:
  name: gpu-claim-template
spec:
  spec:
    devices:
      requests:
      - name: gpu
        exactly:
          deviceClassName: gpu.nvidia.com
          count: 1
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gpu-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: gpu-app
  template:
    metadata:
      labels:
        app: gpu-app
    spec:
      containers:
      - name: cuda-app
        image: nvcr.io/nvidia/cuda:13.0.0-base-ubuntu22.04
        command: ["sleep", "infinity"]
        resources:
          claims:
          - name: gpu
      resourceClaims:
      - name: gpu
        resourceClaimTemplateName: gpu-claim-template
```

---

## Advanced Features

### GPU Attributes Reference

The DRA driver exposes the following attributes per GPU in ResourceSlices:

| Attribute | Type | Example | Description |
|-----------|------|---------|-------------|
| `productName` | string | `"Quadro RTX 6000"` | GPU product name |
| `architecture` | string | `"Turing"` | GPU architecture |
| `brand` | string | `"Nvidia"` | Brand name |
| `cudaComputeCapability` | version | `7.5.0` | CUDA compute capability |
| `cudaDriverVersion` | version | `13.0.0` | CUDA driver version |
| `driverVersion` | version | `580.105.8` | NVIDIA driver version |
| `uuid` | string | `"GPU-xxxx-xxxx"` | Unique GPU identifier |
| `pcieBusID` | string | `"0000:07:00.0"` | PCIe bus address |
| `type` | string | `"gpu"` or `"mig"` | Device type |

| Capacity | Type | Example | Description |
|----------|------|---------|-------------|
| `memory` | quantity | `24Gi` | GPU memory size |

### CEL Expression Examples

```yaml
# GPU with at least 40GB memory
device.capacity['memory'].compareTo(quantity('40Gi')) >= 0

# Ampere or newer architecture
device.attributes['gpu.nvidia.com'].architecture in ['Ampere', 'Hopper', 'Blackwell']

# Specific PCIe root complex (for topology)
device.attributes['resource.kubernetes.io/pcieRoot'] == 'pci0000:00'

# Exclude specific GPU
device.attributes['gpu.nvidia.com'].uuid != 'GPU-xxxxx'

# Compute capability 8.0 or higher
device.attributes['gpu.nvidia.com'].cudaComputeCapability.isGreaterThan(version('8.0.0'))
```

### MIG (Multi-Instance GPU) Support

For GPUs that support MIG (A100, H100, etc.):

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata:
  name: mig-claim
spec:
  devices:
    requests:
    - name: mig
      exactly:
        deviceClassName: mig.nvidia.com
        count: 1
        selectors:
        - cel:
            expression: device.attributes['gpu.nvidia.com'].migProfile == '3g.40gb'
```

### Topology Constraints

Request GPUs on the same PCIe root complex:

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata:
  name: topology-aware
spec:
  devices:
    requests:
    - name: gpus
      exactly:
        deviceClassName: gpu.nvidia.com
        count: 2
    constraints:
    - requests: ["gpus"]
      matchAttribute: resource.kubernetes.io/pcieRoot
```

---

## Troubleshooting

### Common Issues

#### 1. DRA Pod Stuck in Init

**Symptom:** DRA kubelet-plugin pod stuck in `Init:0/1`

**Cause:** NVIDIA driver not found at expected path

**Solution:**
```bash
# Check init container logs
kubectl logs -n nvidia-dra-driver-gpu <pod-name> -c init-container

# Verify driver path
# If using GPU Operator:
ls /run/nvidia/driver/usr/bin/nvidia-smi

# Ensure nvidiaDriverRoot matches your setup
```

#### 2. No ResourceSlices Created

**Symptom:** `kubectl get resourceslices` returns empty

**Causes & Solutions:**
1. DRA plugin not running on GPU nodes
   ```bash
   kubectl get pods -n nvidia-dra-driver-gpu -o wide
   ```

2. Node affinity not matching
   ```bash
   kubectl get nodes --show-labels | grep -E "pci-10de|nvidia.com/gpu"
   ```

3. Missing tolerations for node taints
   ```bash
   kubectl describe node <gpu-node> | grep Taints
   ```

#### 3. ResourceClaim Stuck in Pending

**Symptom:** ResourceClaim state shows `pending`

**Causes:**
1. No GPUs available matching the request
2. All GPUs already allocated
3. CEL selector too restrictive

**Debug:**
```bash
# Check available GPUs
kubectl get resourceslices -o yaml

# Check claim details
kubectl describe resourceclaim <claim-name>

# Check scheduler logs
kubectl logs -n kube-system -l component=kube-scheduler | grep -i dra
```

#### 4. Pod Scheduled but No GPU Visible

**Symptom:** Pod runs but `nvidia-smi` shows no GPUs

**Causes:**
1. CDI spec not generated
2. Container runtime CDI support issue

**Debug:**
```bash
# Check CDI specs on node
ls /var/run/cdi/

# Check kubelet plugin logs
kubectl logs -n nvidia-dra-driver-gpu <plugin-pod> -c gpus
```

### Diagnostic Commands

```bash
# Full DRA status check
kubectl get deviceclasses,resourceslices,resourceclaims -A

# DRA driver logs
kubectl logs -n nvidia-dra-driver-gpu -l app.kubernetes.io/name=nvidia-dra-driver-gpu --tail=100

# GPU Operator driver status
kubectl logs -n gpu-operator -l app=nvidia-driver-daemonset --tail=50

# Node GPU labels
kubectl get nodes -o custom-columns=\
NAME:.metadata.name,\
GPU_PRESENT:.metadata.labels.nvidia\\.com/gpu\\.present,\
GPU_COUNT:.metadata.labels.nvidia\\.com/gpu\\.count
```

---

## Migration from Device Plugin

### Side-by-Side Comparison

**Legacy Device Plugin Request:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod-legacy
spec:
  containers:
  - name: cuda
    image: nvidia/cuda:13.0-base
    resources:
      limits:
        nvidia.com/gpu: 1
```

**DRA Request:**
```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata:
  name: gpu-claim
spec:
  devices:
    requests:
    - name: gpu
      exactly:
        deviceClassName: gpu.nvidia.com
        count: 1
---
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod-dra
spec:
  containers:
  - name: cuda
    image: nvidia/cuda:13.0-base
    resources:
      claims:
      - name: gpu
  resourceClaims:
  - name: gpu
    resourceClaimName: gpu-claim
```

### Migration Strategy

1. **Phase 1: Deploy DRA alongside Device Plugin**
   - Install DRA driver
   - Keep device plugin enabled
   - Test new workloads with DRA

2. **Phase 2: Migrate Workloads**
   - Convert deployments to use ResourceClaims
   - Update CI/CD pipelines
   - Monitor for issues

3. **Phase 3: Disable Device Plugin**
   - Set `devicePlugin.enabled: false` in GPU Operator
   - Remove legacy resource requests from remaining workloads

### Coexistence Considerations

- DRA and device plugin can run simultaneously
- A GPU allocated via DRA is not visible to device plugin (and vice versa)
- Avoid mixed allocation strategies on the same node during migration

---

## Appendix

### Version History

| Version | Release Date | Key Features |
|---------|--------------|--------------|
| v25.8.1 | 2025-08 | GA support, K8s 1.34+ |
| v25.6.0 | 2025-06 | Beta improvements |
| v25.3.0 | 2025-03 | Initial beta release |

### References

- [Kubernetes DRA Documentation](https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/)
- [NVIDIA GPU Operator Documentation](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/)
- [NVIDIA DRA Driver GitHub](https://github.com/NVIDIA/k8s-dra-driver)
- [Container Device Interface (CDI) Specification](https://github.com/cncf-tags/container-device-interface)

### Glossary

| Term | Definition |
|------|------------|
| **CDI** | Container Device Interface - standard for device exposure to containers |
| **CEL** | Common Expression Language - used for device selection |
| **DeviceClass** | Cluster-scoped definition of a device category |
| **DRA** | Dynamic Resource Allocation - Kubernetes API for hardware resources |
| **MIG** | Multi-Instance GPU - NVIDIA technology for GPU partitioning |
| **ResourceClaim** | Request for specific devices from a DeviceClass |
| **ResourceSlice** | Advertisement of available devices on a node |

---

*Document Version: 1.0*
*Last Updated: February 2026*
*Tested with: Kubernetes 1.34, NVIDIA DRA Driver v25.8.1, GPU Operator v25.3.0*
