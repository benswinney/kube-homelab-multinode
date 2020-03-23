# MetalLB as an Internal Load Balancer

Apply MetalLB deployment

> master01

```shell
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
kubectl apply -f https://raw.githubusercontent.com/google/metallb/v0.9.2/manifests/metallb.yaml
```

Next apply the MetalLB ConfigMap

```shell
kubectl create -f metallb/metallb-configmap.yaml
```

You can edit the ConfigMap after deployment by simply editing the configmap

```shell
kubectl edit configmap config -n metallb-system
```
