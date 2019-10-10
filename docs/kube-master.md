# Configure the Master Nodes

## Copy Certificate and Key file from an etcd node

> etcd01

```shell
export CONTROL_PLANE="bens@192.168.1.50"
scp /etc/kubernetes/pki/etcd/ca.crt "${CONTROL_PLANE}":
scp /etc/kubernetes/pki/apiserver-etcd-client.crt "${CONTROL_PLANE}":
scp /etc/kubernetes/pki/apiserver-etcd-client.key "${CONTROL_PLANE}":
```

## Configure first Master Node (master01)

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
kubernetesVersion: v1.16.1
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

Initialize with `kubeadm`

```shell
sudo kubeadm init --config kubeadm-config.yaml --upload-certs
```

Provided all the above steps have been completed correctly, you'll see output similar to this:

```shell
kubeadm join 192.168.1.49:6443 --token f28gzi.k4iydf5rxhchivx6 --discovery-token-ca-cert-hash sha256:2e7d738031ea2c05d4154d3636ced92c390a464d1486d4f4824c112b85a2171f --control-plane
```

Copy the join output to a text file as it's needed for the adding additional master nodes and worker nodes

Allow a non-root user to run kubectl

```shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
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
