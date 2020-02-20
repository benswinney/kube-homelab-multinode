# Deploy a CNI Plugin

Now add a CNI (Container Network Interface) plugin. I use Weave as my CNI of choice, with secure communication

```shell
kubectl create secret -n kube-system generic weave-passwd --from-literal=weave-passwd=$(hexdump -n 16 -e '4/4 "%08x" 1 "\n"' /dev/random)
kubectl apply -n kube-system -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')&password-secret=weave-passwd"
```

If using Calcio


```shell
sysctl net.ipv4.conf.all.forwarding=1 # On each node

kubectl apply -f https://docs.projectcalico.org/v3.8/manifests/calico.yaml
```
