# homelab-multinode
 
 Create 8 nodes (vm's)

 Minimum requirements for this example:

2 servers with 2 CPUs & 2 GB of RAM for the masters (50Gb hdd)
3 servers with 4 CPUs & 4 (8 recommended) GB of RAM for the workers (50Gb hdd)
3 servers with 2 CPUs & 2 GB of RAM for Etcd & HAProxy (32Gb hdd)

All installed with Ubuntu 18.04

Built on Promox Cluster

Network:

VIP: 192.168.1.49
master01: 192.168.1.50
master02: 192.168.1.51
etcd01: 192.168.1.52
etcd02: 192.168.1.53
etcd03: 192.168.1.54
worker01: 192.168.1.55
worker02: 192.168.1.56
worker03: 192.168.1.57

## Configure HAProxy and Heartbeat

Across 3 etcd Nodes

etcd01
```shell
sudo apt update && sudo apt upgrade -y && sudo apt install haproxy -y
```

etcd02
```shell
sudo apt update && sudo apt upgrade -y && sudo apt install haproxy -y
```

etcd03
```shell
sudo apt update && sudo apt upgrade -y && sudo apt install haproxy -y
```

On Each Etd Node
Backup original config
```shell
mv /etc/haproxy/haproxy.cfg{,.back}
```

Create a new one with the following:
```shell
global
    user haproxy
    group haproxy

defaults
    mode http
    log global
    retries 2
    timeout connect 3000ms
    timeout server 5000ms
    timeout client 5000ms

frontend kubernetes
    bind 192.168.1.49:6443
    option tcplog
    mode tcp
    default_backend kubernetes-master-nodes

backend kubernetes-master-nodes
    mode tcp
    balance roundrobin
    option tcp-check
    server k8s-master-0 192.168.1.50:6443 check fall 3 rise 2
    server k8s-master-1 192.168.1.51:6443 check fall 3 rise 2
```

