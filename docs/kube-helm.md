# Helm

## Helm V3

> master01

```shell
$ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
$ chmod 700 get_helm.sh
$ ./get_helm.sh
```

Add Stable Helm Repo

```shell
helm repo add stable https://kubernetes-charts.storage.googleapis.com/
```

## Helm V2

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
helm init --upgrade --history-max=200
```

> master03

```shell
sudo snap install helm --classic
helm init --upgrade --history-max=200
```
