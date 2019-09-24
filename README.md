# homelab-multinode

Using Proxmox VE (or Bare-metal or a n other virtualisation platform i.e. Nutanix), create 10 nodes (VM's) with the following minimum requirements as below:

| Total | Role | CPU | RAM | HDD |
|-------|------|-----|-----|-----|
| 2     | master / control plane | 2 | 4Gb | 50Gb |
| 3     | etcd | 2   | 3Gb | 32Gb |
| 2     | haproxy* | 2 | 2Gb | 20Gb |
| 3     | worker | 4 | 8Gb | 50Gb |

* These nodes will be used to provide LoadBalancing for the masters / control planes

I used Ubuntu Server 18.04 LTS as the OS for all the nodes.

Network layout:
* Virtual IP (for LoadBalancing) : 192.168.1.49
* Master / Control Plane Node 01 (master01) : 192.168.1.50
* Master / Control Plane Node 02 (master02) : 192.168.1.51
* Etcd Node 01 (etcd01) : 192.168.1.52
* Etcd Node 02 (etcd02) : 192.168.1.53
* Etcd Node 03 (etcd03) : 192.168.1.54
* HAProxy Node 01 (proxy01) : 192.168.1.55
* HAProxy Node 02 (proxy02) : 192.168.1.56
* Worker Node 01 (worker01) : 192.168.1.57
* Worker Node 02 (worker02) : 192.168.1.58
* Worker Node 03 (worker03) : 192.168.1.59

If you're short on resources, proxy01/02 could be combined with etcd01/02

## 1. Configure HAProxy and Heartbeat

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

<b>proxy01</b>
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

<b>proxy02</b>
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

<b>etcd01</b>
```shell
export HOST0=192.168.1.52 # etcd01
export HOST1=192.168.1.53 # etcd02
export HOST2=192.168.1.54 # etcd03

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
<b>etd01</b>
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
<b>etcd02</b>
```shell
USER=bens
HOST=${HOST1}
scp -r /tmp/${HOST}/* ${USER}@${HOST}:
ssh ${USER}@${HOST}
USER@HOST $ sudo -Es
root@HOST $ chown -R root:root pki
root@HOST $ mv pki /etc/kubernetes/
```

<b>etcd03</b>
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
<b>etcd01</b>
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

<b>etcd02</b>
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

<b>etcd03</b>
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
<b>etcd01</b>
```shell
sudo kubeadm init phase etcd local --config=/tmp/${HOST0}/kubeadmcfg.yaml
```

<b>etcd02</b>
```shell
kubeadm init phase etcd local --config=/home/bens/kubeadmcfg.yaml
```

<b>etcd03</b>
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
member 238b72cdd26e304f is healthy: got healthy result from https://192.168.1.52:2379
member 8034142cf01c5d1c is healthy: got healthy result from https://192.168.1.53:2379
member fba9d7bc26d1ea21 is healthy: got healthy result from https://192.168.1.54:2379
cluster is healthy
```

## 4. Configure HA Master (aka Control Plane) Nodes 

### Copy Certificate and Key file from Etcd Node

<b>etcd01</b>
```shell
export CONTROL_PLANE="bens@192.168.1.50"
scp /etc/kubernetes/pki/etcd/ca.crt "${CONTROL_PLANE}":
scp /etc/kubernetes/pki/apiserver-etcd-client.crt "${CONTROL_PLANE}":
scp /etc/kubernetes/pki/apiserver-etcd-client.key "${CONTROL_PLANE}":
```

### Configure first Master / Control Plane Node (master01)

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
kubernetesVersion: stable
apiServer:
  certSANs:
  - "192.168.1.49"
controlPlaneEndpoint: "192.168.1.49:6443"
etcd:
    external:
        endpoints:
        - https://192.168.1.52:2379
        - https://192.168.1.53:2379
        - https://192.168.1.54:2379
        caFile: /etc/kubernetes/pki/etcd/ca.crt
        certFile: /etc/kubernetes/pki/apiserver-etcd-client.crt
        keyFile: /etc/kubernetes/pki/apiserver-etcd-client.key
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
coredns-5644d7b6d9-jdrbt               1/1     Running   1          2d16h   10.32.0.2      kubemaster01   <none>           <none>
coredns-5644d7b6d9-rsxwf               1/1     Running   1          2d16h   10.32.0.3      kubemaster01   <none>           <none>
kube-apiserver-kubemaster01            1/1     Running   1          2d16h   192.168.1.50   kubemaster01   <none>           <none>
kube-controller-manager-kubemaster01   1/1     Running   3          2d16h   192.168.1.50   kubemaster01   <none>           <none>
kube-proxy-lwx2k                       1/1     Running   1          2d16h   192.168.1.50   kubemaster01   <none>           <none>
kube-scheduler-kubemaster01            1/1     Running   2          2d16h   192.168.1.50   kubemaster01   <none>           <none>
weave-net-gktp8                        2/2     Running   2          2d16h   192.168.1.50   kubemaster01   <none>           <none>

kubectl get nodes -o wide

NAME           STATUS   ROLES    AGE     VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
kubemaster01   Ready    master   2d16h   v1.16.0   192.168.1.50   <none>        Ubuntu 18.04.3 LTS   4.15.0-64-generic   docker://18.6.2
```

We can now add the second Master / Control Place (master02)

### Configure second Master / Control Plane Node (master02)
