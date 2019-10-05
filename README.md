# Building a Kubernetes Cluster with external etcd nodes for an on-prem "home lab"

I decided to put together a bunch of instructions on building a Kubernetes Cluster. Having built (and broke) many Kubernetes clusters over the past few years, I thought it would be useful to actually document the steps so that anyone wanting a simple set of instructions to follow can use these to quickly spin up a on-prem Kubernetes Cluster.

These instructions are not intended for Production Grade Kubernetes Clusters, as there are many examples of Enterprise / Production Grade Kubernetes Cluster providers e.g. Red Hat OpenShift, IBM Cloud Private, but they will help you build up a Kubernetes Cluster with an external etcd nodes configuration for use within a non-production environment or as I've done, in my own "home lab" environment.

## Audience

Any one with the passion (or just a bunch of free time) to build their own Kubernetes Cluster.

## Index

1. Planning
   - [Architecture Overview](docs/arch-overview.md)
   - [Stacked or External etcd Configuration](docs/stacked-external-etcd.md)
2. Load Balancer for Kubernetes API-Server
   - [HAProxy and Heartbeat](docs/loadbalancer.md)
3. Kubernetes
   - [Pre-requisites](docs/kube-prereqs.md)
   - [etcd](docs/kube-external-etcd.md)
   - [Masters](docs/kube-master.md)
   1. Configuration
      - [CNI Plugins](docs/kube-cni-plugins.md)
      - [Storage Provisioner](docs/kube-nfs-provisioner.md)
   2. Create an Highly Available Kube-APIServer 
      - [Join Additional Masters](docs/kube-extra-masters.md) 
   - [Workers](docs/kube-workers.md)
   - [Redeploy CoreDNS](docs/kube-redeploy-coredns.md)

## 7. Enable DNS Auto-scaling (Optional)

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

## 10. Helm

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

I've updated the Kubernetes Dashboard deployment to use a Load Balancer (aka MetalLB) and include an admin-user Service Account.

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

Apply the mandatory Nginx Ingress Controller yaml file.

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

Again, you could apply the Service Type directly from the Kubernetes website, however that Service type is configured for a NodePort Service type and you would need to edit the Service and change it to a Load Balancer Service type to work with the MetalLB Load Balancer configuration we've installed.

```shell
https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/baremetal/service-nodeport.yaml
```

Remember to modify the Service Type after applying the configuration.

Let's check what External IP address has been provided by MetalLB (take a note of it for later)

```shell
kubectl -n ingress-nginx get svc -n ingress-nginx
```

Once everything has been deployed, let's test by deploying a small Nginx web server application.

Apply the Nginx web server deployment yaml file.

```shell
kubectl apply -f nginx-ingress-controller/test-deployment/nginx-test-deployment.yaml
```

Then apply the Nginx web server ingress yaml file. This file will perform the re-write rules and route traffic to the correct paths.

```shell
kubectl apply -f nginx-ingress-controller/test-deployment/nginx-test-ingress.yaml
```

Test via <http://>External IP Address noted above.

## 14. SealedSecrets (bitnami-labs)

> On all master nodes

```shell
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.8.3/kubeseal-linux-amd64 -O kubeseal
sudo install -m 755 kubeseal /usr/local/bin/kubeseal
rm -f kubeseal

kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.8.3/controller.yaml

mkdir certs

kubeseal --fetch-cert > certs/kubecert.pem
```

Example Usage:

```shell
echo -n <SECRET> | kubectl create secret generic <SECRET-NAME> --dry-run --from-file=<VALUE>=/dev/stdin -o yaml > <SECRET-FILENAME>.yaml
kubeseal --cert certs/kubecert.pem --format yaml < <SECRET-FILENAME>.yaml > <SEALEDSECRET-FILENAME>.yaml
```
