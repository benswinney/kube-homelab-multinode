# Kubernetes Dashboard

## Install the Kubernetes Dashboard

I've updated the Kubernetes Dashboard deployment to use a Load Balancer (aka MetalLB) and include an admin-user Service Account.

The standard Kubernetes Dashboard deployment will use a NodePort and you will need to create an admin-user Service Account.

```shell
kubectl create -f dashboard/dashboard.yaml
```

## Kubernetes Dashboard Token

Kubeadm (which we used to deploy this Kubernetes Cluster) does not support a Kube Config file for logging into the Kubernetes Dashboard. It only supports Token method.

To get the Token to login, we need to run a command to print out that token.

```shell
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')
```

## Kubernetes Dashboard Example

```shell
eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLXNtbTQyIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiI3YjlkMzk4Ni1kYTQyLTQwMTUtOWI4ZC1mYjgzNzgxM2I1YTciLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZXJuZXRlcy1kYXNoYm9hcmQ6YWRtaW4tdXNlciJ9.jjGAVDeJJBIXe7jzSbmC_azlT5MAnH3yemX81m9Bv9W_I5u2Nm9aezTPZyRnO46UN7Eb2piWH5fUeNCiVZylPQt-FI4L4BGLEl5RWJInckollrSRw2bhEBkdtmEdHWjqsKXNQLV2qbuTin6ZE4lpuMa0PbkCkX-wtdpf0ejnq_PIIEdkOAvrYOKzIO6LHAEkCtK4nFObwEGPUH1yDoIbGCbdlg_xbEx-6Uv7Xz8YfbZ3DBDljcL_tyk8LwmaUWmNryTNclWBXNPOKnqrfkx1DEdj6RXTrG9TIbaIJ8YW324PmYPkPt_MDGQNxDDwpWAgH7BsogOcb7XWRGuix16_pQ
```

## Get Kubernetes Dashboard ExternalIP from MetalLB

```shell
kubectl --namespace kubernetes-dashboard get service kubernetes-dashboard
```

Connect via <https://>**ExternalIP**
