# Difference between Stacked Master and external etcd node Configurations

> TLDR  
> Stacked : Master and etcd Services are ran on the same node, but HA provided by the means of multiple Master nodes (A minimum of 3 nodes)  
> External etcd : etcd run on separate nodes to the Master (A minimum of 6 nodes)

## Stacked Master and etcd Topology

![Stacked etcd nodes](https://github.com/benswinney/homelab-multinode/blob/master/docs/stacked-etcd.png)

A Stacked Master and etcd cluster are a topology where the distributed data storage cluster (*Source of all truth*) provided by etcd is stacked on top of the master nodes.

Each master node runs an instance of the `kube-apiserver`, `kube-scheduler`, and `kube-controller-manager`. The `kube-apiserver` is exposed to worker nodes using a load balancer (e.g. HAProxy). A local etcd member is created and this etcd member communicates only with the `kube-apiserver` running on this same node. The same applies to the local `kube-controller-manager` and `kube-scheduler` instances.

This topology couples the master and etcd members on the same node where they run. It is simpler to set up than a cluster with external etcd nodes, and simpler to manage for replication.

However, a Stacked Master and etcd cluster runs into the risk of failed coupling. If one node goes down, both an etcd member and a master instance are lost, and redundancy is compromised. You can help mitigate this risk by adding additional master nodes.

A minimum of three nodes should be used for a Stacked Master and etcd cluster.

This is the default topology in `kubeadm`. A local etcd member is created automatically on master nodes when using `kubeadm init` and `kubeadm join --control-plane`.

## External etcd with Stacked Master Topology

![External etcd nodes](https://github.com/benswinney/homelab-multinode/blob/master/docs/external-etcd.png)

An External Etcd with Stacked Master cluster is a topology where the distributed data storage cluster (*Source of all truth*) provided by etcd is external to the cluster formed by the master nodes.

Similar to a Stacked Master and etcd topology, each master node in an external etcd with Stacked Master cluster runs an instance of the `kube-apiserver`, `kube-scheduler`, and `kube-controller-manager`. The `kube-apiserver` is exposed to worker nodes by using a load balancer (e.g. HAProxy).

Unlike a Stacked Master and etcd topology, the etcd members run on separate nodes, and each etcd node communicates with the `kube-apiserver` of each master node.

This topology decouples the master and etcd member. It therefore provides a setup where losing a master instance or an etcd member has less impact and does not affect the cluster redundancy as much as the Stacked Master and etcd topology.

However, this topology requires twice the number of nodes as the Stacked Master and etcd topology. A minimum of three nodes for master and three nodes for etcd are required for a cluster with this topology.

This is the topology we'll be using. :)  
