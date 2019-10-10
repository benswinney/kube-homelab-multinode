# Architecture Overview

## Nodes

Since I've Proxmox VE in my "home lab" environment, I created eleven (11) nodes for use within the Kubernetes Cluster. You could use any virtualisation platform you desire i.e. VMWare ESXi, Nutanix or even use Bare-metal nodes, if so inclined.

I created my 11 nodes (VM's) with the following requirements as below.

| Total | Role | CPU | RAM | HDD |
|-------|------|-----|-----|-----|
| 3     | master / control-plane | 2 | 4Gb | 50Gb |
| 3     | etcd | 2   | 3Gb | 32Gb |
| 2     | proxy* | 2 | 2Gb | 20Gb |
| 3     | worker | 4 | 8Gb | 50Gb |

\* The Proxy nodes will be used to provide Load Balancing for the masters / control-planes Kube-APIServer via HAProxy

* Under Kubernetes, Master nodes and Control-Plane nodes are one of the same. Some people refer to them as a Master node and others a Control-Plane node. For sake of simplicity, I'll refer to them as a Master node going forward.

### Attention

The nodes I created were done with the minimum requirements specified by the Kubernetes docs, you may be able to use less, but your milage may vary.  

If you find that you're short on resources and can't meet the minimum, then you could combine the Proxy nodes (proxy01 / proxy02) with two (2) of the etcd nodes (e.g. etcd01 / etcd02).

However, if you're even shorter on resources, you could run your etcd services on your Master nodes within what it referred to as a Stacked Master nodes configuration, but I'd recommend that you still run separate Proxy nodes.

## Operating System

I used Ubuntu Server 18.04 LTS as the OS for all the nodes.

## Kubernetes Release

Kubernetes Version: 1.16.1 **Update: Due to KubeFlow V0.6.2 not being compatible with Kubernetes versions > 1.14.7, cluster was built using 1.14.7**

## Network / Hostname Layout

| Role | Hostname | IP Address |
|------|----------|------------|
| Virtual IP (for Load Balancing) | kubevip01 | 192.168.1.49 |
| Master Node 01 | master01 | 192.168.1.50 |
| Master Node 02 | master02 | 192.168.1.51 |
| Master Node 03 | master03 | 192.168.1.52 |
| etcd Node 01 | etcd01 | 192.168.1.53 |
| etcd Node 02 | etcd02 | 192.168.1.54 |
| etcd Node 03 | etcd03 | 192.168.1.55 |
| Proxy Node 01 | proxy01 | 192.168.1.56 |
| Proxy Node 02 | proxy02 | 192.168.1.57 |
| Worker Node 01 | worker01 | 192.168.1.58 |
| Worker Node 02 | worker02 | 192.168.1.59 |
| Worker Node 03 | worker03 | 192.168.1.60 |
