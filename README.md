# Building a Kubernetes Cluster with external or stacked etcd configuration for an on-prem "home lab"

I decided to put together a bunch of instructions on building a Kubernetes Cluster. Having built (and broke) many Kubernetes clusters over the past few years, I thought it would be useful to actually document the steps so that anyone wanting a simple set of instructions to follow can use these to quickly spin up a on-prem Kubernetes Cluster.

These instructions are not intended for Production Grade Kubernetes Clusters, as there are many examples of Enterprise / Production Grade Kubernetes Cluster providers e.g. Red Hat OpenShift, IBM Cloud Private, but they will help you build up a Kubernetes Cluster with an external or stacked etcd configuration for use within a non-production environment or as I've done, in my own "home lab" environment.

## Audience

Any one with the passion (or just a bunch of free time) to build their own Kubernetes Cluster.

## Index

1. Planning
   - [Architecture Overview](docs/arch-overview.md)
   - [Stacked or External etcd Configuration](docs/stacked-external-etcd.md)
2. Load Balancer for Kubernetes API-Server
   - [HAProxy and Heartbeat](docs/loadbalancer.md)
3. Kubernetes
   - [Pre-requisites](docs/kube-prereqs.md)
   - [etcd](docs/kube-external-etcd.md)
   1. Masters
      1. External or Stacked Etcd
         - [Configure Master Node with External etcd](docs/kube-master-external-etcd.md)
         - [Configure Master Node with Stacked etcd](docs/kube-master-stacked-etcd.md)
      - [CNI Plugins](docs/kube-cni-plugins.md)
      - [Storage Provisioner](docs/kube-nfs-provisioner.md)
      - [Storage Provisioner - Ceph](doc/kube-storage.md)
   2. Create an Highly Available Kube-APIServer
      - [Join Additional Masters](docs/kube-extra-masters.md)
   3. Workers
      - [Add Worker Nodes](docs/kube-workers.md)
   - [Redeploy CoreDNS](docs/kube-redeploy-coredns.md)
   - [MetalLB - Load Balancer](docs/kube-metallb.md)
   - [Helm](docs/kube-helm.md)
   - [Metric Server](docs/kube-metricserver.md)
   - [Ingress Controller](docs/kube-ingresscontroller.md)
   - [Sealed Secrets](docs/kube-sealedsecrets.md)
   - [Kubernetes Dashboard](docs/kube-dashboard.md)
   4. Optional
      - [DNS Auto-Scaling](docs/kube-dnsautoscale.md)
      - [Internal Docker Registory](docs/kube-docker-reg.md)
