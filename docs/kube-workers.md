# Adding Worker Nodes (worker01, worker02, worker03)

Adding the worker nodes is very simple.

Using the output used for the joining the master nodes, remove the `--control-plane` from the command and away you go.

> worker01

```shell
sudo kubeadm join vip01:6443 --token f28gzi.k4iydf5rxhchivx6 --discovery-token-ca-cert-hash sha256:2e7d738031ea2c05d4154d3636ced92c390a464d1486d4f4824c112b85a2171f
```

> worker02

```shell
sudo kubeadm join vip01:6443 --token f28gzi.k4iydf5rxhchivx6 --discovery-token-ca-cert-hash sha256:2e7d738031ea2c05d4154d3636ced92c390a464d1486d4f4824c112b85a2171f
```

> worker03

```shell
sudo kubeadm join vip01:6443 --token f28gzi.k4iydf5rxhchivx6 --discovery-token-ca-cert-hash sha256:2e7d738031ea2c05d4154d3636ced92c390a464d1486d4f4824c112b85a2171f
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

Add workers if token has expired

```shell
sudo kubeadm token list

sudo kubeadm token create #If expired


```
