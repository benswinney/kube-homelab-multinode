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
sudo swapoff -a && sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

The `/etc/fstab` file should now be commented out for the swap mount point

## Install and Hold the Docker packages

```shell
sudo apt-get update && sudo apt-get install -y \
  containerd.io=1.2.10-3 \
  docker-ce=5:19.03.4~3-0~ubuntu-$(lsb_release -cs) \
  docker-ce-cli=5:19.03.4~3-0~ubuntu-$(lsb_release -cs)

sudo apt-mark hold containerd.io=1.2.10-3 docker-ce=5:19.03.4~3-0~ubuntu-$(lsb_release -cs) docker-ce-cli=5:19.03.4~3-0~ubuntu-$(lsb_release -cs)
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

## Create systemd docker directory

```shell
sudo mkdir -p /etc/systemd/system/docker.service.d
```

## Restart docker

```shell
sudo systemctl daemon-reload && sudo systemctl restart docker
```

## Install and Hold Kubernetes Packages (V1.15.8)

```shell
sudo apt install -y kubelet=1.15.8-00 kubeadm=1.15.8-00 kubectl=1.15.8-00 && sudo apt-mark hold kubelet=1.15.8-00 kubeadm=1.15.8-00 kubectl=1.15.8-00
```

## Enable & Start Kubelet (if not already started)

```shell
sudo systemctl enable kubelet && sudo systemctl start kubelet
```

## Modify /etc/hosts

We want to make sure that the /etc/hosts does not resolve the hostname to 127.0.0.1, if it does, comment it out.
