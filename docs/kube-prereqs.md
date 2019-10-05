# Configure the Kubernetes & Docker Repositories on ALL Nodes

Feel free to swap Docker with another CRI (Container Runtime Interface) e.g. CRI-O, containerd

```shell
sudo apt update && sudo apt install -y apt-transport-https curl ca-certificates software-properties-common
```

```shell
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
```

```shell
sudo add-apt-repository \
  "deb https://apt.kubernetes.io/ kubernetes-xenial main"
```

```shell
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```

```shell
sudo add-apt-repository \
  "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
```

```shell
sudo apt update
```

## Disable Swap

```shell
sudo swapoff -a

sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

The `/etc/fstab` file should now be commented out for the swap mount point

## Install the Docker packages

```shell
sudo apt-get install docker-ce=18.06.2~ce~3-0~ubuntu
```

## Hold Docker Version

```shell
sudo apt-mark hold docker-ce=18.06.2~ce~3-0~ubuntu
```

## Modify Docker to use systemd and overlay2 storage drivers

```shell
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
```

## Restart docker

```shell
sudo systemctl daemon-reload && sudo systemctl restart docker
```

## Install the Kubernetes Packages (V1.16)

```shell
sudo apt install -y kubelet kubeadm kubectl
```

## Hold Kubernetes Packages

```shell
sudo apt-mark hold kubelet kubeadm kubectl
```

## Enable & Start Kubelet (if not already started)

```shell
sudo systemctl enable kubelet && sudo systemctl start kubelet
```
