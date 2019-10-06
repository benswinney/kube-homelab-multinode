# SealedSecrets (bitnami-labs)

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
