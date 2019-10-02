# Build a Kubernetes Cluster with multiple nodes

These instructions will help build up a Kuberneres Cluster with an external Etcd configuration for use within my homelab.

Using Proxmox VE (or Bare-metal or a n other virtualisation platform i.e. Nutanix), create 11 nodes (VM's) with the following minimum requirements as below:

| Total | Role | CPU | RAM | HDD |
|-------|------|-----|-----|-----|
| 3     | master / control-plane | 2 | 4Gb | 50Gb |
| 3     | etcd | 2   | 3Gb | 32Gb |
| 2     | proxy* | 2 | 2Gb | 20Gb |
| 3     | worker | 4 | 8Gb | 50Gb |

\* The Proxy nodes will be used to provide LoadBalancing for the masters / control-planes API-Server via HAProxy

* Under Kubernetes, Master nodes and Control-Plane nodes are one of the same thing. Some people refer to them as Master nodes and others a Control-Plane nodes. For sake of simplicity, I'll refer to these nodes as Master nodes going forward for the purpose of this document.

I used Ubuntu Server 18.04 LTS as the OS for all the nodes.

Kubernetes Version: 1.16

Network layout:

* Virtual IP (for LoadBalancing) : 192.168.1.49
* Master Node 01 (master01) : 192.168.1.50
* Master Node 02 (master02) : 192.168.1.51
* Master Node 03 (master03) : 192.168.1.52
* Etcd Node 01 (etcd01) : 192.168.1.53
* Etcd Node 02 (etcd02) : 192.168.1.54
* Etcd Node 03 (etcd03) : 192.168.1.55
* Proxy Node 01 (proxy01) : 192.168.1.56
* Proxy Node 02 (proxy02) : 192.168.1.57
* Worker Node 01 (worker01) : 192.168.1.58
* Worker Node 02 (worker02) : 192.168.1.59
* Worker Node 03 (worker03) : 192.168.1.60

If you're short on resources, proxy01/02 could be combined with etcd01/02, if running with External Etcd nodes or if you're even shorter on resources, you could run your Etcd services on your Master nodes within a Stacked Master nodes configuration.

## Differences between Stacked Master and External Etcd Nodes Configurations

> TLDR  
> Stacked : Master and Etcd Services are ran on the same node, but HA provided by multiple Master nodes (A minimum of 3 nodes)  
> External Etcd : Etcd run on seperate nodes to the Master (A minimum of 6 nodes)

### Stacked Master and Etcd Topology

![External Etcd Nodes](https://github.com/benswinney/homelab-multinode/blob/master/docs/stacked-etcd.png)

A Stacked Master and Etcd cluster is a topology where the distributed data storage cluster (*Source of all truth*) provided by Etcd is stacked on top of the master nodes.

Each master node runs an instance of the `kube-apiserver`, `kube-scheduler`, and `kube-controller-manager`. The `kube-apiserver` is exposed to worker nodes using a load balancer (e.g. HAProxy). A local Etcd member is created and this Etcd member communicates only with the `kube-apiserver` running on this same node. The same applies to the local `kube-controller-manager` and `kube-scheduler` instances.

This topology couples the master and Etcd members on the same node where they run. It is simpler to set up than a cluster with external Etcd nodes, and simpler to manage for replication.

However, a Stacked Master and Etcd cluster runs into the risk of failed coupling. If one node goes down, both an Etcd member and a master instance are lost, and redundancy is compromised. You can help mitigate this risk by adding additional master nodes.

A minimum of three nodes should be used for a Stacked Master and Etcd cluster.

This is the default topology in `kubeadm`. A local etcd member is created automatically on master nodes when using `kubeadm init` and `kubeadm join --control-plane`.

### External Etcd with Stacked Master Topology

![External Etcd Nodes](https://github.com/benswinney/homelab-multinode/blob/master/docs/external-etcd.png)

An External Etcd with Stacked Master cluster is a topology where the distributed data storage cluster (*Source of all truth*) provided by Etcd is external to the cluster formed by the master nodes.

Similar to a Stacked Master and Etcd topology, each master node in an External Etcd with Stacked Master cluster runs an instance of the `kube-apiserver`, `kube-scheduler`, and `kube-controller-manager`. The `kube-apiserver` is exposed to worker nodes by using a load balancer (e.g. HAProxy).

Unlike a Stacked Master and Etcd topology, the Etcd members run on separate nodes, and each Etcd node communicates with the `kube-apiserver` of each master node.

This topology decouples the master and Etcd member. It therefore provides a setup where losing a master instance or an Etcd member has less impact and does not affect the cluster redundancy as much as the Stacked Master and Etcd topology.

However, this topology requires twice the number of nodes as the Stacked Master and Etcd topology. A minimum of three nodes for master and three nodes for Etcd are required for a cluster with this topology.

## 1. Create Load Balancer for Kubernetes API-Server

### Configure HAProxy and Heartbeat

On each Proxy node, run the following:

> proxy01

```shell
sudo apt update && sudo apt upgrade -y && sudo apt install haproxy heartbeat -y

sudo mv /etc/haproxy/haproxy.cfg{,.bkp}
```

> proxy02

```shell
sudo apt update && sudo apt upgrade -y && sudo apt install haproxy heartbeat -y

sudo mv /etc/haproxy/haproxy.cfg{,.bkp}
```

Add the below line to the `/etc/systctl.conf` file on each Proxy node

```shell
net.ipv4.ip_nonlocal_bind=1
```

Create a new `/etc/haproxy/haproxy.cfg` on each Proxy node:

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
    server kube-master-0 192.168.1.50:6443 check fall 3 rise 2
    server kube-master-1 192.168.1.51:6443 check fall 3 rise 2
    server kube-master-2 192.168.1.52:6443 check fall 3 rise 2
```

Enable and Start HAProxy & Heartbeat on both Proxy nodes

```shell
sudo systemctl enable haproxy && sudo systemctl start haproxy
sudo systemctl enable heartbeat && sudo systemctl start heartbeat
```

REBOOT each Proxy node

Upon a successful reboot, check to see the that haproxy has started and is listening on the 192.168.1.49 address

```shell
sudo netstat -ntulp
```

Create a authkeys `/etc/ha.d/authkeys` file on both Proxy nodes, ensuring it's readable/writable by root only (`chmod 600 /etc/ha.d/authkeys`).

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

Create `/etc/ha.d/ha.cf` on each Proxy node

They will differ slightly, as can be seen below

> proxy01

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

> proxy02

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
ucast en0 192.168.1.57
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

Create a `/etc/ha.d/haresources` file on both Proxy nodes

```shell
proxy01 192.168.1.49
```

Restart the heartbeat service on both Proxy nodes

```shell
sudo systemctl restart heartbeat
```

## 2. Kubernetes Configuration

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

### Hold Kubernetes Packages

```shell
sudo apt-mark hold kubelet kubeadm kubectl
```

### Enable & Start Kubelet (if not already started)

```shell
sudo systemctl enable kubelet && sudo systemctl start kubelet
```

## 3. Etcd Cluster Configuration

### Create new etcd kubelet services configuration file

On ALL Etcd Nodes, create a new kubelet systemd configuration file, with a higher precedence that the kubelet supplied version just installed

```shell
cat << EOF > /etc/systemd/system/kubelet.service.d/20-etcd-service-manager.conf
[Service]
ExecStart=
ExecStart=/usr/bin/kubelet --address=127.0.0.1 --pod-manifest-path=/etc/kubernetes/manifests --cgroup-driver=systemd
Restart=always
EOF
```

Reload and Restart Kubelet on each of the Etcd Nodes

```shell
sudo systemctl daemon-reload && sudo systemctl restart kubelet
```

### Generate kubeadm configuration files for all Etcd nodes

> etcd01

```shell
export HOST0=192.168.1.53 # etcd01
export HOST1=192.168.1.54 # etcd02
export HOST2=192.168.1.55 # etcd03

mkdir -p /tmp/${HOST0}/ /tmp/${HOST1}/ /tmp/${HOST2}/

ETCDHOSTS=(${HOST0} ${HOST1} ${HOST2})
NAMES=("infra0" "infra1" "infra2")

for i in "${!ETCDHOSTS[@]}"; do
HOST=${ETCDHOSTS[$i]}
NAME=${NAMES[$i]}
cat << EOF > /tmp/${HOST}/kubeadmcfg.yaml
apiVersion: "kubeadm.k8s.io/v1beta2"
kind: ClusterConfiguration
etcd:
    local:
        serverCertSANs:
        - "${HOST}"
        peerCertSANs:
        - "${HOST}"
        extraArgs:
            initial-cluster: ${NAMES[0]}=https://${ETCDHOSTS[0]}:2380,${NAMES[1]}=https://${ETCDHOSTS[1]}:2380,${NAMES[2]}=https://${ETCDHOSTS[2]}:2380
            initial-cluster-state: new
            name: ${NAME}
            listen-peer-urls: https://${HOST}:2380
            listen-client-urls: https://${HOST}:2379
            advertise-client-urls: https://${HOST}:2379
            initial-advertise-peer-urls: https://${HOST}:2380
EOF
done
```

### Generate the Certificate Authority

> etcd01

```shell
sudo kubeadm init phase certs etcd-ca
```

This creates two files
`/etc/kubernetes/pki/etcd/ca.crt`
`/etc/kubernetes/pki/etcd/ca.key`

Create certificates for each Etcd node (etcd01, etcd02, etcd03)

```shell
kubeadm init phase certs etcd-server --config=/tmp/${HOST2}/kubeadmcfg.yaml
kubeadm init phase certs etcd-peer --config=/tmp/${HOST2}/kubeadmcfg.yaml
kubeadm init phase certs etcd-healthcheck-client --config=/tmp/${HOST2}/kubeadmcfg.yaml
kubeadm init phase certs apiserver-etcd-client --config=/tmp/${HOST2}/kubeadmcfg.yaml
cp -R /etc/kubernetes/pki /tmp/${HOST2}/

# cleanup non-reusable certificates
find /etc/kubernetes/pki -not -name ca.crt -not -name ca.key -type f -delete

kubeadm init phase certs etcd-server --config=/tmp/${HOST1}/kubeadmcfg.yaml
kubeadm init phase certs etcd-peer --config=/tmp/${HOST1}/kubeadmcfg.yaml
kubeadm init phase certs etcd-healthcheck-client --config=/tmp/${HOST1}/kubeadmcfg.yaml
kubeadm init phase certs apiserver-etcd-client --config=/tmp/${HOST1}/kubeadmcfg.yaml
cp -R /etc/kubernetes/pki /tmp/${HOST1}/
find /etc/kubernetes/pki -not -name ca.crt -not -name ca.key -type f -delete

kubeadm init phase certs etcd-server --config=/tmp/${HOST0}/kubeadmcfg.yaml
kubeadm init phase certs etcd-peer --config=/tmp/${HOST0}/kubeadmcfg.yaml
kubeadm init phase certs etcd-healthcheck-client --config=/tmp/${HOST0}/kubeadmcfg.yaml
kubeadm init phase certs apiserver-etcd-client --config=/tmp/${HOST0}/kubeadmcfg.yaml

# clean up certs that should not be copied off this host
find /tmp/${HOST2} -name ca.key -type f -delete
find /tmp/${HOST1} -name ca.key -type f -delete
```

### Copy kubeadm configuration and certificates files to the correct etcd node

> etcd02

```shell
USER=bens
HOST=${HOST1}
scp -r /tmp/${HOST}/* ${USER}@${HOST}:
ssh ${USER}@${HOST}
USER@HOST $ sudo -Es
root@HOST $ chown -R root:root pki
root@HOST $ mv pki /etc/kubernetes/
```

> etcd03

```shell
USER=bens
HOST=${HOST2}
scp -r /tmp/${HOST}/* ${USER}@${HOST}:
ssh ${USER}@${HOST}
USER@HOST $ sudo -Es
root@HOST $ chown -R root:root pki
root@HOST $ mv pki /etc/kubernetes/
```

### Confirm that the full list of required files exist on each Etcd node

> etcd01

```shell
/tmp/${HOST0}
└── kubeadmcfg.yaml
---
/etc/kubernetes/pki
├── apiserver-etcd-client.crt
├── apiserver-etcd-client.key
└── etcd
    ├── ca.crt
    ├── ca.key
    ├── healthcheck-client.crt
    ├── healthcheck-client.key
    ├── peer.crt
    ├── peer.key
    ├── server.crt
    └── server.key
```

> etcd02

```shell
$HOME
└── kubeadmcfg.yaml
---
/etc/kubernetes/pki
├── apiserver-etcd-client.crt
├── apiserver-etcd-client.key
└── etcd
    ├── ca.crt
    ├── healthcheck-client.crt
    ├── healthcheck-client.key
    ├── peer.crt
    ├── peer.key
    ├── server.crt
    └── server.key
```

> etcd03

```shell
$HOME
└── kubeadmcfg.yaml
---
/etc/kubernetes/pki
├── apiserver-etcd-client.crt
├── apiserver-etcd-client.key
└── etcd
    ├── ca.crt
    ├── healthcheck-client.crt
    ├── healthcheck-client.key
    ├── peer.crt
    ├── peer.key
    ├── server.crt
    └── server.key
```

### Create Static Pod Manifests on each Etcd node

> etcd01

```shell
sudo kubeadm init phase etcd local --config=/tmp/${HOST0}/kubeadmcfg.yaml
```

> etcd02

```shell
kubeadm init phase etcd local --config=/home/bens/kubeadmcfg.yaml
```

> etcd03

```shell
kubeadm init phase etcd local --config=/home/bens/kubeadmcfg.yaml
```

Once the kubeadm commands have completed, check the Cluster health (it may take a few minutes for the Etcd Cluster to become stable)

```shell
docker run --rm -it \
--net host \
-v /etc/kubernetes:/etc/kubernetes k8s.gcr.io/etcd:${ETCD_TAG} etcdctl \
--cert-file /etc/kubernetes/pki/etcd/peer.crt \
--key-file /etc/kubernetes/pki/etcd/peer.key \
--ca-file /etc/kubernetes/pki/etcd/ca.crt \
--endpoints https://${HOST0}:2379 cluster-health
```

`${HOST0}` should be set to etcd01 IP Address
`${ETCD_TAG}` should be set to the version of etcd image

Output should be similar to below

```shell
member 238b72cdd26e304f is healthy: got healthy result from https://192.168.1.53:2379
member 8034142cf01c5d1c is healthy: got healthy result from https://192.168.1.54:2379
member fba9d7bc26d1ea21 is healthy: got healthy result from https://192.168.1.55:2379
cluster is healthy
```

## 4. Configure the Master Nodes

### Copy Certificate and Key file from Etcd Node

> etcd01

```shell
export CONTROL_PLANE="bens@192.168.1.50"
scp /etc/kubernetes/pki/etcd/ca.crt "${CONTROL_PLANE}":
scp /etc/kubernetes/pki/apiserver-etcd-client.crt "${CONTROL_PLANE}":
scp /etc/kubernetes/pki/apiserver-etcd-client.key "${CONTROL_PLANE}":
```

### Configure first Master Node (master01)

First we need to move the certificates and key files we copied from the etcd01 node to the correct location (`/etc/kubernetes/pki/etcd`)

```shell
sudo mkdir -p /etc/kubernetes/pki/etcd/
sudo cp /home/bens/ca.crt /etc/kubernetes/pki/etcd/
sudo cp /home/bens/apiserver-etcd-client.crt /etc/kubernetes/pki/
sudo cp /home/bens/apiserver-etcd-client.key /etc/kubernetes/pki/
```

Create a file called `kubeadm-config.yaml`

```shell
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: v1.16.0
apiServer:
  certSANs:
  - "192.168.1.49"
controlPlaneEndpoint: "192.168.1.49:6443"
etcd:
    external:
        endpoints:
        - https://192.168.1.53:2379
        - https://192.168.1.54:2379
        - https://192.168.1.55:2379
        caFile: /etc/kubernetes/pki/etcd/ca.crt
        certFile: /etc/kubernetes/pki/apiserver-etcd-client.crt
        keyFile: /etc/kubernetes/pki/apiserver-etcd-client.key
networking:
  podSubnet: 10.11.0.0/16
  serviceSubnet: 10.96.0.0/12
```

Initilise with `kubeadm`

```shell
sudo kubeadm init --config kubeadm-config.yaml --upload-certs
```

Provided all the above steps have been completed correctly, you'll see output similar to this:

```shell
kubeadm join 192.168.1.49:6443 --token f28gzi.k4iydf5rxhchivx6 --discovery-token-ca-cert-hash sha256:2e7d738031ea2c05d4154d3636ced92c390a464d1486d4f4824c112b85a2171f --control-plane
```

Copy the join output to a text file as it's needed for the master02.

Allow a non-root user to run kubectl

```shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Now add a CNI (aka network) plugin. I use Weave as my CNI of choice, with secure communication

```shell
kubectl create secret -n kube-system generic weave-passwd --from-literal=weave-passwd=$(hexdump -n 16 -e '4/4 "%08x" 1 "\n"' /dev/random)
kubectl apply -n kube-system -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')&password-secret=weave-passwd"
```

Check for all the Pods to be deployed and in Running and the status of the cluster

```shell
kubectl get pods -n kube-system -o wide

NAME                                   READY   STATUS    RESTARTS   AGE     IP             NODE           NOMINATED NODE   READINESS GATES
coredns-5644d7b6d9-jdrbt               1/1     Running   1          2d16h   10.32.0.2      master01   <none>           <none>
coredns-5644d7b6d9-rsxwf               1/1     Running   1          2d16h   10.32.0.3      master01   <none>           <none>
kube-apiserver-kubemaster01            1/1     Running   1          2d16h   192.168.1.50   master01   <none>           <none>
kube-controller-manager-kubemaster01   1/1     Running   3          2d16h   192.168.1.50   master01   <none>           <none>
kube-proxy-lwx2k                       1/1     Running   1          2d16h   192.168.1.50   master01   <none>           <none>
kube-scheduler-kubemaster01            1/1     Running   2          2d16h   192.168.1.50   master01   <none>           <none>
weave-net-gktp8                        2/2     Running   2          2d16h   192.168.1.50   master01   <none>           <none>

kubectl get nodes -o wide

NAME           STATUS   ROLES    AGE     VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
master01   Ready    master   2d16h   v1.16.0   192.168.1.50   <none>        Ubuntu 18.04.3 LTS   4.15.0-64-generic   docker://18.6.2
```

We can now add the second Master (master02)

### Configure the second Master Node (master02)

Run the join command that was copied in the previous step to join master02 to the Kubernetes cluster

> master02

```shell
sudo kubeadm join 192.168.1.49:6443 --token f28gzi.k4iydf5rxhchivx6 --discovery-token-ca-cert-hash sha256:2e7d738031ea2c05d4154d3636ced92c390a464d1486d4f4824c112b85a2171f --control-plane
```

Check for all the Pods to be deployed and in Running and the status of the cluster

```shell
kubectl get pods -n kube-system -o wide

NAME                                   READY   STATUS    RESTARTS   AGE     IP             NODE           NOMINATED NODE   READINESS GATES
coredns-5644d7b6d9-jdrbt               1/1     Running   1          2d19h   10.32.0.2      master01   <none>           <none>
coredns-5644d7b6d9-rsxwf               1/1     Running   1          2d19h   10.32.0.3      master01   <none>           <none>
kube-apiserver-kubemaster01            1/1     Running   1          2d19h   192.168.1.50   master01   <none>           <none>
kube-apiserver-kubemaster02            1/1     Running   1          2d18h   192.168.1.51   master02   <none>           <none>
kube-controller-manager-kubemaster01   1/1     Running   3          2d19h   192.168.1.50   master01   <none>           <none>
kube-controller-manager-kubemaster02   1/1     Running   1          2d18h   192.168.1.51   master02   <none>           <none>
kube-proxy-kt6jz                       1/1     Running   1          2d18h   192.168.1.51   master02   <none>           <none>
kube-proxy-lwx2k                       1/1     Running   1          2d19h   192.168.1.50   master01   <none>           <none>
kube-scheduler-kubemaster01            1/1     Running   2          2d19h   192.168.1.50   master01   <none>           <none>
kube-scheduler-kubemaster02            1/1     Running   1          2d18h   192.168.1.51   master02   <none>           <none>
weave-net-gktp8                        2/2     Running   2          2d18h   192.168.1.50   master01   <none>           <none>
weave-net-t87xb                        2/2     Running   2          2d18h   192.168.1.51   master02   <none>           <none>

kubectl get nodes -o wide

NAME           STATUS   ROLES    AGE     VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
master01   Ready    master   2d19h   v1.16.0   192.168.1.50   <none>        Ubuntu 18.04.3 LTS   4.15.0-64-generic   docker://18.6.2
master02   Ready    master   2d18h   v1.16.0   192.168.1.51   <none>        Ubuntu 18.04.3 LTS   4.15.0-64-generic   docker://18.6.2

```

Let's add the 3rd and final Master node

### Configure the third Master Node (master03)

Run the join command again join master03 to the Kubernetes cluster

> master03

```shell
sudo kubeadm join 192.168.1.49:6443 --token f28gzi.k4iydf5rxhchivx6 --discovery-token-ca-cert-hash sha256:2e7d738031ea2c05d4154d3636ced92c390a464d1486d4f4824c112b85a2171f --control-plane
```

Check for all the Pods to be deployed and in Running and the status of the cluster

```shell
kubectl get pods -n kube-system -o wide

NAME                                   READY   STATUS    RESTARTS   AGE     IP             NODE           NOMINATED NODE   READINESS GATES
coredns-5644d7b6d9-jdrbt               1/1     Running   1          2d19h   10.32.0.2      master01   <none>           <none>
coredns-5644d7b6d9-rsxwf               1/1     Running   1          2d19h   10.32.0.3      master01   <none>           <none>
kube-apiserver-kubemaster01            1/1     Running   1          2d19h   192.168.1.50   master01   <none>           <none>
kube-apiserver-kubemaster02            1/1     Running   1          2d18h   192.168.1.51   master02   <none>           <none>
kube-apiserver-kubemaster03            1/1     Running   1          2d18h   192.168.1.52   master02   <none>           <none>
kube-controller-manager-kubemaster01   1/1     Running   3          2d19h   192.168.1.50   master01   <none>           <none>
kube-controller-manager-kubemaster02   1/1     Running   1          2d18h   192.168.1.51   master02   <none>           <none>
kube-controller-manager-kubemaster03   1/1     Running   1          2d18h   192.168.1.51   master03   <none>           <none>
kube-proxy-kt6jz                       1/1     Running   1          2d18h   192.168.1.51   master02   <none>           <none>
kube-proxy-g45jt                       1/1     Running   1          2d18h   192.168.1.52   master03   <none>           <none>
kube-proxy-lwx2k                       1/1     Running   1          2d19h   192.168.1.50   master01   <none>           <none>
kube-scheduler-kubemaster01            1/1     Running   2          2d19h   192.168.1.50   master01   <none>           <none>
kube-scheduler-kubemaster02            1/1     Running   1          2d18h   192.168.1.51   master02   <none>           <none>
kube-scheduler-kubemaster03            1/1     Running   1          2d18h   192.168.1.52   master03   <none>           <none>
weave-net-gktp8                        2/2     Running   2          2d18h   192.168.1.50   master01   <none>           <none>
weave-net-t87xb                        2/2     Running   2          2d18h   192.168.1.51   master02   <none>           <none>
weave-net-r56jk                        2/2     Running   2          2d18h   192.168.1.52   master03   <none>           <none>

kubectl get nodes -o wide

NAME           STATUS   ROLES    AGE     VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
master01   Ready    master   2d19h   v1.16.0   192.168.1.50   <none>        Ubuntu 18.04.3 LTS   4.15.0-64-generic   docker://18.6.2
master02   Ready    master   2d18h   v1.16.0   192.168.1.51   <none>        Ubuntu 18.04.3 LTS   4.15.0-64-generic   docker://18.6.2
master03   Ready    master   2d18h   v1.16.0   192.168.1.52   <none>        Ubuntu 18.04.3 LTS   4.15.0-64-generic   docker://18.6.2
```

Now all 3 Master nodes (master01 / master02 / master03) are configured, running behing an HAProxy LoadBalancer (proxy01 / proxy02).

Next we'll configure the workers.

## 5. Add Worker Nodes (worker01, worker02, worker03)

Adding the additional worker nodes is simple.

> worker01

```shell
sudo kubeadm join 192.168.1.49:6443 --token f28gzi.k4iydf5rxhchivx6 --discovery-token-ca-cert-hash sha256:2e7d738031ea2c05d4154d3636ced92c390a464d1486d4f4824c112b85a2171f
```

> worker02

```shell
sudo kubeadm join 192.168.1.49:6443 --token f28gzi.k4iydf5rxhchivx6 --discovery-token-ca-cert-hash sha256:2e7d738031ea2c05d4154d3636ced92c390a464d1486d4f4824c112b85a2171f
```

> worker03

```shell
sudo kubeadm join 192.168.1.49:6443 --token f28gzi.k4iydf5rxhchivx6 --discovery-token-ca-cert-hash sha256:2e7d738031ea2c05d4154d3636ced92c390a464d1486d4f4824c112b85a2171f
```

After a few minutes, the nodes will start to appear within the cluster

> master01

```shell
kubectl get nodes -o wide

NAME           STATUS   ROLES    AGE     VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
master01   Ready    master   2d19h   v1.16.0   192.168.1.50   <none>        Ubuntu 18.04.3 LTS   4.15.0-64-generic   docker://18.6.2
master02   Ready    master   2d19h   v1.16.0   192.168.1.51   <none>        Ubuntu 18.04.3 LTS   4.15.0-64-generic   docker://18.6.2
master03   Ready    master   2d19h   v1.16.0   192.168.1.52   <none>        Ubuntu 18.04.3 LTS   4.15.0-64-generic   docker://18.6.2
worker01   Ready    <none>   2d19h   v1.16.0   192.168.1.58   <none>        Ubuntu 18.04.3 LTS   4.15.0-64-generic   docker://18.6.2
worker02   Ready    <none>   2d19h   v1.16.0   192.168.1.59   <none>        Ubuntu 18.04.3 LTS   4.15.0-64-generic   docker://18.6.2
worker03   Ready    <none>   2d19h   v1.16.0   192.168.1.60   <none>        Ubuntu 18.04.3 LTS   4.15.0-64-generic   docker://18.6.2
```

## 6. Redeploy CoreDNS Services

Due to a bug / feature of kubeadm, if coredns is deployed without additional nodes available (e.g. another master or worker nodes), then it will deploy entirely on the same node (e.g. master01).

```shell
NAME                                   READY   STATUS    RESTARTS   AGE     IP             NODE           NOMINATED NODE   READINESS GATES
coredns-5644d7b6d9-jdrbt               1/1     Running   1          2d16h   10.32.0.2      master01   <none>           <none>
coredns-5644d7b6d9-rsxwf               1/1     Running   1          2d16h   10.32.0.3      master01   <none>           <none>
```

To resolve this, we can restart the deployment of coredns, which will deploy across different nodes to provide the HA level of redundancy we've come to expect from Kubernetes.

> master01

```shell
kubectl -n kube-system rollout restart deployment coredns
```

```shell
kubectl get pods -A -o wide | grep coredns

kube-system            coredns-58ddcb86c5-cxlnl                      1/1     Running   0          12m     10.42.0.4      worker03   <none>           <none>
kube-system            coredns-58ddcb86c5-dqzh8                      1/1     Running   0          12m     10.36.0.5      worker01   <none>           <none>
```

## 7. Enable DNS Auto-scaling

```shell
kubectl apply -f dns-autoscaling/dns-horizontal-autoscaling-deployment.yaml
```

## 8. Add MetalLB as an Internal Load Balancer

Apply MetalLB deployment

> master01

```shell
kubectl create -f metallb/metallb.yaml
```

Next apply the MetalLB ConfigMap

```shell
kubectl create -f metallb/metallb-configmap.yaml
```

You can edit the ConfigMap after deployment by simply editing the configmap

```shell
kubectl edit configmap config -n metallb-system
```

## 9. Install NFS Storage Provisioner

To enable automatic provisioning of PersistentStorage for our deployments, I use the NFS Storage Provisioner method, there are many others like Rook.io, OpenEBS etc, but this one works well for my homelab environment. I'll update the README.md in the future with detailed instructions on using other methods.

> master01

```shell
kubectl create -f nfs-client/deploy/rbac.yaml
kubectl create -f nfs-client/deploy/deployment.yaml
kubectl create -f nfs-client/deploy/class.yaml
```

### Optional

You can set the NFS Storage Provisioner as the default storage class which will make things a bit simpler.

```shell
kubectl patch deployment nfs-client-provisioner -p '{"spec":{"template":{"spec":{"serviceAccount":"nfs-client-provisioner"}}}}'
kubectl patch storageclass managed-nfs-storage -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

## 10. Helm

> master01

```shell
kubectl create -f helm/helm-rbac.yaml
kubectl create serviceaccount --namespace kube-system tiller
kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
sudo snap install helm --classic
helm init --upgrade
kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'
```

Install the helm cli on the remaining Master nodes. This allows us to interactive with Helm even if the first Master node is unavailable.

> master02

```shell
sudo snap install helm --classic
helm init
```

> master03

```shell
sudo snap install helm --classic
helm init
```

## 11. Install Metric Server

We'll install the Metric Server for insights and as a pre-req to Horizontal Pod Scaling capabilities.

```shell
kubectl create -f metric-server/aggregated-metrics-reader.yaml
kubectl create -f metric-server/auth-delegator.yaml
kubectl create -f metric-server/auth-reader.yaml
kubectl create -f metric-server/metrics-apiservice.yaml
kubectl create -f metric-server/metrics-server-deployment.yaml
kubectl create -f metric-server/metrics-server-service.yaml
kubectl create -f metric-server/resource-reader.yaml
```

## 12. Kubernetes Dashboard

### Install the Kubernetes Dashboard

I've updated the Kubernetes Dashboard deployment to use a LoadBalancer (aka MetalLB) and include a admin-user Service Account.

The standard Kubernetes Dashboard deployment will use a NodePort and you will need to create an admin-user Service Account.

```shell
kubectl create -f dashboard/dashboard.yaml
```

### Kubernetes Dashboard Token

Kubeadm (which we used to deploy this Kubernetes Cluster) does not support a Kube Config file for logging into the Kubernetes Dashboard. It only supports Token method.

To get the Token to login, we need to run a command to print out that token.

```shell
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')
```

### Kubernetes Dashboard Example

```shell
eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLXNtbTQyIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiI3YjlkMzk4Ni1kYTQyLTQwMTUtOWI4ZC1mYjgzNzgxM2I1YTciLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZXJuZXRlcy1kYXNoYm9hcmQ6YWRtaW4tdXNlciJ9.jjGAVDeJJBIXe7jzSbmC_azlT5MAnH3yemX81m9Bv9W_I5u2Nm9aezTPZyRnO46UN7Eb2piWH5fUeNCiVZylPQt-FI4L4BGLEl5RWJInckollrSRw2bhEBkdtmEdHWjqsKXNQLV2qbuTin6ZE4lpuMa0PbkCkX-wtdpf0ejnq_PIIEdkOAvrYOKzIO6LHAEkCtK4nFObwEGPUH1yDoIbGCbdlg_xbEx-6Uv7Xz8YfbZ3DBDljcL_tyk8LwmaUWmNryTNclWBXNPOKnqrfkx1DEdj6RXTrG9TIbaIJ8YW324PmYPkPt_MDGQNxDDwpWAgH7BsogOcb7XWRGuix16_pQ
```

### Get Kubernetes Dashboard ExternalIP from MetalLB

```shell
kubectl --namespace kubernetes-dashboard get service kubernetes-dashboard
```

Connect via <https://>**ExternalIP**

## 13. Ingress Controller (nginx)

Realistically, we can skip an Ingress Contoller and expose our applications directly via the MetalLB load balancer.

Putting an ingress controller in front of our applications provides benefits like basic load balancing, SSL/TLS termination, support for URI rewrites, and upstream SSL/TLS encryption.

> master01

Apply the mandatory Nginx Ingress Controller Yaml file.

```shell
kubectl apply -f nginx-ingress-controller/nginx-ingress-controller-mandatory.yaml
```

Alternatively, you can apply direct from the Kubernetes website:

```shell
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/mandatory.yaml
```

Now we need to create a Load Balancer Service type for the Nginx Ingress Controller.

```shell
kubectl apply -f nginx-ingress-controller/nginx-ingress-controller-svc.yaml
```

Again, you could apply the Service Type directly from the Kubernetes website, however that Service type is configured for a NodePort Service type and you would need to edit the Service and change it to a LoadBalancer Service type to work with the MetalLB LoadBalancer configuration we've installed.

```shell
https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/baremetal/service-nodeport.yaml
```

Remember to modify the Service Type after applying the configuration.

Let's check what External IP address has been provided by MetalLB (take a note of it for later)

```shell
kubectl -n ingress-nginx get svc -n ingress-nginx
```

Once everything has been deployed, let's test to ensure everything is working as we expect. We can test by deploying a small Nginx web server application.

Apply the Nginx web server deployment yaml file.

```shell
kubectl apply -f nginx-ingress-controller/test-deployment/nginx-test-deployment.yaml
```

Then apply the Nginx web server ingress yaml file. This file will perform the re-write rules and route traffic to the correct paths.

```shell
kubectl apply -f nginx-ingress-controller/test-deployment/nginx-test-ingress.yaml
```

Test via <http://>External IP Address noted above.
