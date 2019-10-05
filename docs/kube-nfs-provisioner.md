# NFS Storage Provisioner

To enable automatic provisioning of Persistent Storage for our deployments, I use the NFS Storage Provisioner method, there are many others like Rook.io, OpenEBS, Minio etc, but this one works well for my home lab environment. I'll update the README in the future with detailed instructions on using other methods.

> master01

```shell
kubectl create -f nfs-client/deploy/rbac.yaml
kubectl create -f nfs-client/deploy/deployment.yaml
kubectl create -f nfs-client/deploy/class.yaml
```

### Optional

You can set the NFS Storage Provisioner as the default storage class which will make things a bit simpler.

```shell
kubectl patch deployment nfs-client-provisioner -p '{"spec":{"template":{"spec":{"serviceAccount":"nfs-client-provisioner"}}}}'
kubectl patch storageclass managed-nfs-storage -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```
