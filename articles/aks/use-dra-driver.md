---
title: Use NVIDIA DRA drivers on Azure Kubernetes Service (AKS)
description: Learn how to use NVIDIA's DRA drivers on AKS for flexibility in requesting, configuring, and sharing GPUs.
ms.topic: how-to
ms.custom: devx-track-azurecli
ms.subservice: aks-developer
ms.date: 11/01/2025
author: jackjiang
ms.author: jackjiang

# Customer intent: As a cluster administrator or developer, I want to provision an Azure Kubernetes Service (AKS) cluster NVIDIA's DRA drivers installed so I can flexibily share GPU resources across the cluster.
---

# Use NVIDIA DRA drivers for flexible configuration of GPU resources  

With the release of Kubernetes 1.34 was the graduation to stable of [Dynamic Resource Allocation][k8s-dra] (DRA). DRA allows a user to flexible configure resources to be requested and shared amongst their workloads.

NVIDIA has released their [Kubernetes DRA drivers][nvidia-dra] which introduces new resources that allow users to more comfortably configure GPUs specifically.

This article walks you through the set up of NVIDIA's DRA drivers in your AKS clusters.

## Before you begin

* This article assumes you have an existing AKS cluster. If you don't have a cluster, create one using the [Azure CLI][aks-quickstart-cli], [Azure PowerShell][aks-quickstart-powershell], or the [Azure portal][aks-quickstart-portal].
* You need the Azure CLI version 2.72.2 or later installed to set the `--gpu-driver` field. Run `az --version` to find the version. If you need to install or upgrade, see [Install Azure CLI][install-azure-cli].
* If you have the `aks-preview` Azure CLI extension installed, please update the version to 18.0.0b2 or later.

## Get the credentials for your cluster

Get the credentials for your AKS cluster using the [`az aks get-credentials`][az-aks-get-credentials] command. The following example command gets the credentials for the *myAKSCluster* in the *myResourceGroup* resource group:

```azurecli-interactive
az aks get-credentials --resource-group myResourceGroup --name myAKSCluster
```

## Install NVIDIA GPU operator

Follow instructions [here][nvidia-gpu-operator] to set up the GPU operator, ensure GPUs are schedulable, and GPU workloads can be ran successfully.

You can make sure all your GPU operator components are running and ready via `$ kubectl get pod -n gpu-operator`.

## Installing NVIDIA DRA drivers

The recommended way to install the drivers are via Helm.

```azurecli-interactive
helm install nvidia-dra-driver-gpu nvidia/nvidia-dra-driver-gpu \
    --version="25.3.0" \
    --create-namespace \
    --namespace nvidia-dra-driver-gpu \
    --set nvidiaDriverRoot=/ \
    --set resources.gpus.enabled=false
```

## Verify your installation

Check if all DRA driver components are in a running and ready state,

```azurecli-interactive
$ kubectl get pod -n nvidia-dra-driver-gpu
NAME                                                           READY   STATUS    RESTARTS   AGE
nvidia-dra-driver-k8s-dra-driver-controller-67cb99d84b-5q7kj   1/1     Running   0          ~
nvidia-dra-driver-k8s-dra-driver-kubelet-plugin-7kdg9          1/1     Running   0          ~
nvidia-dra-driver-k8s-dra-driver-kubelet-plugin-bd6gn          1/1     Running   0          ~
nvidia-dra-driver-k8s-dra-driver-kubelet-plugin-bzm6p          1/1     Running   0          ~
nvidia-dra-driver-k8s-dra-driver-kubelet-plugin-xjm4p          1/1     Running   0          ~
```

and all GPU nodes are labeled with clique IDs.

```azurecli-interactive
$ (echo -e "NODE\tLABEL\tCLIQUE"; kubectl get nodes -o json | \
    jq -r '.items[] | [.metadata.name, "nvidia.com/gpu.clique", .metadata.labels["nvidia.com/gpu.clique"]] | @tsv') | \
    column -t
NODE                 LABEL                  CLIQUE
gpu-node-1           nvidia.com/gpu.clique  9277d399-0674-44a9-b64e-d85bb19ce2b0.32766
```

`deviceclasses` and `resourceslices` should also recognize the new GPU devices. You can use `kubectl get deviceclasses` or `kubectl get resourceslices` to confirm.

<!-- LINKS - external -->
[kubectl-apply]: https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#apply
[k8s-dra]: https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/
[nvidia-dra]: https://github.com/NVIDIA/k8s-dra-driver-gpu
[gpu-operator]: https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/getting-started.html

<!-- LINKS - internal -->
[az-aks-create]: /cli/azure/aks#az_aks_create
[az-aks-nodepool-update]: /cli/azure/aks/nodepool#az_aks_nodepool_update
[az-aks-nodepool-add]: /cli/azure/aks/nodepool#az_aks_nodepool_add
[az-aks-get-credentials]: /cli/azure/aks#az_aks_get_credentials
[aks-quickstart-cli]: ./learn/quick-kubernetes-deploy-cli.md
[aks-quickstart-portal]: ./learn/quick-kubernetes-deploy-portal.md
[aks-quickstart-powershell]: ./learn/quick-kubernetes-deploy-powershell.md
[aks-spark]: spark-job.md
[gpu-skus]: /azure/virtual-machines/sizes-gpu
[install-azure-cli]: /cli/azure/install-azure-cli
[azureml-aks]: /azure/machine-learning/how-to-attach-kubernetes-anywhere
[azureml-deploy]: /azure/machine-learning/how-to-deploy-managed-online-endpoints
[azureml-triton]: /azure/machine-learning/how-to-deploy-with-triton
[aks-container-insights]: monitor-aks.md#integrations
[nvidia-gpu-operator]: nvidia-gpu-operator.md
[advanced-scheduler-aks]: operator-best-practices-advanced-scheduler.md
[az-provider-register]: /cli/azure/provider#az-provider-register
[az-feature-register]: /cli/azure/feature#az-feature-register
[az-feature-show]: /cli/azure/feature#az-feature-show
[az-extension-add]: /cli/azure/extension#az-extension-add
[az-extension-update]: /cli/azure/extension#az-extension-update
[NVadsA10]: /azure/virtual-machines/nva10v5-series
