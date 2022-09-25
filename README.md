# Installing and configuring MetalLB for your non-cloud cluster

https://platform9.com/blog/using-metallb-to-add-the-loadbalancer-service-to-kubernetes-environments/

Lets assume you have created a home Kubernetes cluster (e.g. [like this](https://github.com/fireflycons/kubeadm-on-ubuntu-jammy)), then the next task at hand is to be able to expose services to the internal network.

This is one of the first use cases you may run into. What is the next step? Are you going to use the NodePort service and manage your own LoadBalancer to distribute traffic? Are you going to expose the ClusterIP network, or manage LoadBalancers that point to ClusterIPs? If you deploy an ingress controller, how will that be reached from the outside? How would this translate to a production environment where the LoadBalancer service is available?

The answer to this is MetalLB. It supports both [Layer2](https://metallb.universe.tf/concepts/layer2/) and BGP configurations. In this post we are going to focus on Layer2 with IPv4 using ARP.

## Prerequisites

1. A working Kubernetes cluster at v1.13 or later, that you can access with `kubectl`. Clusters built with [kind](https://kind.sigs.k8s.io/) are also supported.
1. A [cluster network configuration](https://metallb.universe.tf/installation/network-addons/) that can coexist with MetalLB.
1. Some free IPv4 addresses on your network for MetalLB to hand out.
1. Traffic on port 7946 must be allowed between nodes.
1. Helm package manager for the installations. Install from [here](https://helm.sh/docs/intro/install/).

## Lab Steps

1. [Install and configure MetalLB](./docs/01-install-metallb.md). After this step, you can expose services of type `LoadBalancer`.
1. [Install and configure Ingress](./docs/02-install-ingress.md). After this step, Ingress will use MetalLB to expose services.

All the inline YAML documents found in the above steps can be also be found in the [Manifests](./manifests/) folder.