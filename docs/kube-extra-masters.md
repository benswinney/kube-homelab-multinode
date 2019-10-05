# Add additional Master nodes to provide an Highly Available Kube-APIServer configuration

## Configure the second Master Node (master02)

Run the join command that you copied to join master02 to the Kubernetes cluster

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

## Configure the third Master Node (master03)

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

Now all 3 Master nodes (master01 / master02 / master03) are configured, running behind an HAProxy Load Balancer (proxy01 / proxy02).

Next we'll configure the workers
