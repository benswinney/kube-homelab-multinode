# homelab-multinode

Using Proxmox VE (or Bare-metal or a n other virtualisation platform i.e. Nutanix), create 10 nodes (VM's) with the following minimum requirements as below:

| Total | Role | CPU | RAM | HDD |
|-------|------|-----|-----|-----|
| 2     | master | 2 | 4Gb | 50Gb |
| 3     | etcd | 2   | 3Gb | 32Gb |
| 2     | haproxy* | 2 | 2Gb | 20Gb |
| 3     | worker | 4 | 8Gb | 50Gb |

* These nodes will be used to provide LoadBalancing for the masters

I used Ubuntu Server 18.04 LTS as the OS for all the nodes.

Network layout:
* Virtual IP (for LoadBalancing) : 192.168.1.49
* Master Node 01 (master01) : 192.168.1.50
* Master Node 02 (master02) : 192.168.1.51
* Etcd Node 01 (etcd01) : 192.168.1.52
* Etcd Node 02 (etcd02) : 192.168.1.53
* Etcd Node 03 (etcd03) : 192.168.1.54
* HAProxy Node 01 (proxy01) : 192.168.1.55
* HAProxy Node 02 (proxy02) : 192.168.1.56
* Worker Node 01 (worker01) : 192.168.1.57
* Worker Node 02 (worker02) : 192.168.1.58
* Worker Node 03 (worker03) : 192.168.1.59

If you're short on resources, proxy01/02 could be combined with etcd01/02

## Configure HAProxy and Heartbeat

On each HAProxy node, run the following:

<b>proxy01</b>
```shell
sudo apt update && sudo apt upgrade -y && sudo apt install haproxy heartbeat -y

sudo mv /etc/haproxy/haproxy.cfg{,.bkp}
```

<b>proxy02</b>
```shell
sudo apt update && sudo apt upgrade -y && sudo apt install haproxy heartbeat -y

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

Enable and Start HAProxy & Heartbeat
```shell
*proxy01* sudo systemctl enable haproxy && sudo systemctl start haproxy
*proxy01* sudo systemctl enable heartbeat && sudo systemctl start heartbeat
*proxy02* sudo systemctl enable haproxy && sudo systemctl start haproxy
*proxy02* sudo systemctl enable heartbeat && sudo sytemtctl start heartbeat
```

REBOOT each HAProxy node

Upon a successful reboot, check to see the that haproxy has started and is listening on the 192.168.1.49 address
```shell
netstat -ntulp
```

Create authkeys `/etc/ha.d/authkeys` on both HAProxy nodes, ensuring it's readable/writable by root only (`chmod 600 /etc/ha.d/authkeys`).

Firstly generate a md5sum password
```shell
echo -n somepassword | md5sum
9c42a1346e333a770904b2a2b37fa7d3
```

Then add to the `/etc/ha.d/authkeys` file
```shell
auth 1
1 md5 9c42a1346e333a770904b2a2b37fa7d3
```

Create `/etc/ha.d/ha.cf` on each HAProxy node

They will differ slightly, as can be seen below

*proxy01*
```shell
#       keepalive: how many seconds between heartbeats
#
keepalive 2
#
#       deadtime: seconds-to-declare-host-dead
#
deadtime 10
#
#       What UDP port to use for udp or ppp-udp communication?
#
udpport 694
bcast en0
mcast en0 225.255.255.0 694 1 0
ucast en0 192.168.1.55
#
#       What interfaces to heartbeat over?
udp en0
#
#       Facility to use for syslog()/logger (alternative to log/debugfile)
#
logfacility local0
#
#       Tell what machines are in the cluster
#       node    nodename ...    -- must match uname -n
node proxy01
node proxy02
```

*proxy02*
```shell
#       keepalive: how many seconds between heartbeats
#
keepalive 2
#
#       deadtime: seconds-to-declare-host-dead
#
deadtime 10
#
#       What UDP port to use for udp or ppp-udp communication?
#
udpport 694
bcast en0
mcast en0 225.255.255.0 694 1 0
ucast en0 192.168.1.56
#
#       What interfaces to heartbeat over?
udp en0
#
#       Facility to use for syslog()/logger (alternative to log/debugfile)
#
logfacility local0
#
#       Tell what machines are in the cluster
#       node    nodename ...    -- must match uname -n
node proxy01
node proxy02
```

Create a `/etc/ha.d/haresources` on both HAProxy nodes
```shell
proxy01 192.168.1.49
```

Restart the heartbeat service on both HAProxy nodes
```shell
sudo systemctl restart heartbeat
```
## Kubernetes Configuration

### Configure Kubernetes & Docker Repositories on ALL Nodes
```shell
sudo apt update && sudo apt install -y apt-transport-https curl ca-certificates software-properties-common
```

```shell
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
```

```shell
sudo add-apt-repository \
  "deb https://apt.kubernetes.io/ kubernetes-xenial main"
```

```shell
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```

```shell
sudo add-apt-repository \
  "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
```

```shell
sudo apt update
```

### Disable Swap
```shell
sudo swapoff -a

sudo sed -i '/ swap / s/^/#/' /etc/fstab
```
The `/etc/fstab` file should now be commentted out for the swap mount point

### Install Docker packages
```shell
sudo apt-get install docker-ce=18.06.2~ce~3-0~ubuntu
```

### Hold Docker Version
```shell
sudo apt-mark hold docker-ce=18.06.2~ce~3-0~ubuntu
```

### Modify Docker to use systemd driver and overlay2 storage driver
```shell
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
```

### Restart docker
```shell
sudo systemctl daemon-reload && sudo systemctl restart docker
```

### Install Kubernetes Packages
```shell
sudo apt install -y kubelet kubeadm kubectl
```

### Hold Kuberneted Packages
```shell
sudo apt-mark hold kubelet kubeadm kubectl
```