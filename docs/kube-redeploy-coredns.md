# Redeploy CoreDNS Services

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
