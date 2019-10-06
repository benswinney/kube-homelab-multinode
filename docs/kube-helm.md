# Helm

> master01

```shell
kubectl create -f helm/helm-rbac.yaml
kubectl create serviceaccount --namespace kube-system tiller
kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
sudo snap install helm --classic
helm init --upgrade --history-max=200
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
