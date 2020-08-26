# AWS App Mesh Deployment Strategies
This project will provide example code and guidance for the following deployment strategies using AWS App Mesh: 
  
Blue/Green  
Canary  
A/B Testing  

## Prerequisites

In order to successfully carry out the base deployment:

- Make sure to have newest [AWS CLI](https://aws.amazon.com/cli/) installed, that is, version `1.18.82` or above.
- Make sure to have `kubectl` [installed](https://kubernetes.io/docs/tasks/tools/install-kubectl/), at least version `1.13` or above.
- Make sure to have `jq` [installed](https://stedolan.github.io/jq/download/).
- Make sure to have `aws-iam-authenticator` [installed](https://github.com/kubernetes-sigs/aws-iam-authenticator), required for eksctl
- Make sure to have `helm` [installed](https://helm.sh/docs/intro/install/).
- Install [eksctl](https://eksctl.io/). See [appendix](#appendix) for eksctl install instructions. Please make you have version `0.21.0` or above installed

Note that this walkthrough assumes throughout to operate in the `us-west-2` region.

```sh
export AWS_DEFAULT_REGION=us-west-2
```

## Cluster provisioning

Create an EKS cluster with `eksctl` using the following command:

```sh
eksctl create cluster \
--name appmeshtest \
--nodes-min 2 \
--nodes-max 3 \
--nodes 2 \
--auto-kubeconfig \
--full-ecr-access \
--appmesh-access
# ...
# [✔]  EKS cluster "appmeshtest" in "us-west-2" region is ready
```

When completed, update the `KUBECONFIG` environment variable according to the `eksctl` output:

```sh
export KUBECONFIG=~/.kube/eksctl/clusters/appmeshtest
```

## Install App Mesh Kubernetes components

In order to automatically inject App Mesh components and proxies on pod creation we need to create some custom resources on the clusters.

*Install App Mesh Components*

Run the following set of commands to install the App Mesh controller 

```sh
helm repo add eks https://aws.github.io/eks-charts
helm repo update
kubectl create ns appmesh-system
kubectl apply -k https://github.com/aws/eks-charts/stable/appmesh-controller/crds?ref=master
helm upgrade -i appmesh-controller eks/appmesh-controller --namespace appmesh-system

```

The EKS cluster should now be provisioned with App Mesh components that automate injection of Envoy and take care of the life cycle management of the underlying resources such as meshes, virtual nodes, virtual services, and virtual routers.

To check the custom resources for the App Mesh Controller:

```sh
kubectl api-resources --api-group=appmesh.k8s.aws
# NAME              SHORTNAMES   APIGROUP          NAMESPACED   KIND
# meshes                         appmesh.k8s.aws   false        Mesh
# virtualnodes                   appmesh.k8s.aws   true         VirtualNode
# virtualrouters                 appmesh.k8s.aws   true         VirtualRouter
# virtualservices                appmesh.k8s.aws   true         VirtualService
```

