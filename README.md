# homelab-multinode
 
 Create 10 nodes (vm's)

 Minimum requirements:

| Total | Role | CPU | RAM | HDD |
|-------|------|-----|-----|-----|
| 2     | master | 2 | 4Gb | 50Gb |
| 3     | etcd | 2   | 3Gb | 32Gb |
| 2     | haproxy* | 2 | 2Gb | 20Gb |
| 3     | worker | 4 | 8Gb | 50Gb |

* These nodes will be used to provide LoadBalancing for the masters

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
proxy01: 192.168.1.58
proxy02: 192.168.1.59

If short on resources, proxy01/02 could be combined with etcd01/02

## Configure HAProxy and Heartbeat

On each HAProxy node, run the following:

proxy01
```shell
sudo apt update && sudo apt upgrade -y && sudo apt install haproxy -y

sudo mv /etc/haproxy/haproxy.cfg{,.bkp}
```

proxy02
```shell
sudo apt update && sudo apt upgrade -y && sudo apt install haproxy -y

sudo mv /etc/haproxy/haproxy.cfg{,.bkp}
```

Add the below line to the `/etc/systctl.conf` file on each HAProxy node
```shell
net.ipv4.ip_nonlocal_bind=1
```

Create a new `/etc/haproxy/haproxy.cfg` on each HAProxy node:
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

REBOOT each HAProxy node

Upon a successful reboot, check to see the that haproxy has started and is listening on the 192.168.1.49 address
```shell
netstat -ntulp
```

