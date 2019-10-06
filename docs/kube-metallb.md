# MetalLB as an Internal Load Balancer

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
