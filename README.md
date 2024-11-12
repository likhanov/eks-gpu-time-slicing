# **GPU Slicing on Amazon EKS**

## **Overview**
This guide explains how to implement GPU slicing on an Amazon EKS cluster, where each node in the `gpu` node group uses `p3.8xlarge` instances (GPU-optimized instances). GPU slicing allows to allocate virtual GPUs (vGPUs) from physical GPUs, which is useful for running multiple smaller workloads on a single GPU.

## **1. Create a ConfigMap for Time-Slicing**

First, create a ConfigMap that configures the time-slicing strategy for the NVIDIA GPU device plugin. This will define the number of virtual GPUs per physical GPU.

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: nvidia-device-plugin
  namespace: kube-system
data:
  any: |-
    version: v1
    flags:
      migStrategy: none  # MIG strategy disabled, using time-slicing instead
    sharing:
      timeSlicing:
        resources:
        - name: nvidia.com/gpu
          replicas: 4  # 4 virtual GPUs per physical GPU (each pod gets 25% of a GPU)
EOF
```

### Explanation:

- The `replicas: 4` line specifies that each physical GPU will be split into 4 virtual GPUs, with each virtual GPU representing 25% of the physical GPU’s resources.
- The `migStrategy: none` ensures that MIG (Multi-Instance GPU) is not used and the GPU is sliced via time-slicing instead.

## **2. Install NVIDIA Device Plugin with Helm**

Next, deploy the NVIDIA device plugin using Helm. This plugin exposes GPU resources to Kubernetes, enabling pods to request GPUs as resources.

```bash
helm upgrade -i nvdp nvdp/nvidia-device-plugin \
  --namespace kube-system \
  -f nvdp-values.yaml \
  --version 0.14.0 \
  --set config.name=nvidia-device-plugin
```

### Explanation:

- This Helm command installs or upgrades the NVIDIA device plugin to the `kube-system` namespace.
- The device plugin automatically detects GPU devices on nodes and exposes them as resources to Kubernetes, allowing you to schedule pods requesting GPU resources.

## **3. Deploy a Sample Workload**

Now, you can deploy a sample workload that will use the GPU slices. In this case, we'll deploy a TensorFlow application that requests one virtual GPU per pod.

```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tensorflow-deployment
  namespace: gpu-demo
  labels:
    app: tensorflow
spec:
  replicas: 8  # Deploy 8 pods, each requesting one virtual GPU
  selector:
    matchLabels:
      app: tensorflow
  template:
    metadata:
      labels:
        app: tensorflow
    spec:
      containers:
      - name: tensorflow
        image: public.ecr.aws/r5m2h0c9/cifar10_cnn:v2
        resources:
          limits:
            nvidia.com/gpu: 1  # Each pod requests 1 virtual GPU
EOF
```

### Explanation:

This deployment creates 8 pods, each requesting a single virtual GPU (nvidia.com/gpu: 1).
The `limits` section ensures that each pod is allocated 25% of a physical GPU, as configured in the previous step.
You can scale the number of pods depending on the number of virtual GPUs available.

## Karpenter and GPU Slicing
Important Consideration:

While Karpenter is a good tool for autoscaling, it currently has compatibility issues when used with GPU slicing. Specifically, Karpenter does not scale out when GPU time-slicing is enabled. This is because the GPU is treated as an extended resource, which may lead to improper resource allocation.

Known Issue:
[Karpenter doesn’t scale out when using time-slicing for GPUs](https://github.com/kubernetes-sigs/karpenter/issues/729)
