# Deploy Ceph Provisioner 

You will need some form of persistent storage on your cluster. A storage class is a general definition that allows containers to create persistent volumes any time they need one. You only need one type of storage, but there are many options. CephFS is a great options that crucially will allow multiple pods to write to the same volume at the same time which is essential for fully scalable pods. Ceph RBD does not allow this. NFS is also a great option if you are trying to share data from another system. Choose whichever option you need, I use CephFS for my containers and NFS to share my media from my unraid NAS to my kubernetes cluster.

## CephFS 

Create Namespace for cephfs

```shell
kubectl create ns cephfs
```

Get Ceph Admin Key from Ceph server

```shell
ceph auth get-key client.admin
```

Create a CephFS Secret

```shell
kubectl create secret generic ceph-secret-admin --from-literal=key="<client.admin key>" -n cephfs
```

Apply CephFS Provisioner

```shell
kubectl apply -f storage/Ceph-FS-Provisioner.yaml
```

Apply Cephfs Storage Class

```shell
kubectl apply -f storage/Ceph-FS-StorageClass.yaml
```

## Ceph RBD

Apply Ceph RBD Provisioner

```shell
kubectl create -f storage/Ceph-RBD-Provisioner.yaml -n kube-system
```

Get Ceph Admin Key from Ceph server

```shell
ceph auth get-key client.admin
```

Create Ceph RBD secret

```shell
kubectl create secret generic ceph-secret \
    --type="kubernetes.io/rbd" \
    --from-literal=key='<client.admin key>' \
    --namespace=kube-system
```

Create a seperate pool for Ceph RBD

```shell
ceph --cluster ceph osd pool create kube 128 128
ceph --cluster ceph auth get-or-create client.kube mon 'allow r' osd 'allow rwx pool=kube'
```

Get Ceph Admin Key for the created pool

```shell
ceph --cluster ceph auth get-key client.kube
```

```shell
kubectl create secret generic ceph-secret-kube \
    --type="kubernetes.io/rbd" \
    --from-literal=key='<client.kube key> \
    --namespace=kube-system
```

Create Ceph RBD Storage Class

```shell
kubectl create -f storage/Ceph-RBD-StorageClass.yaml
```