# Docker Registory

1. Create own local Docker registry.

Select some free external node for placing your docker registry in there, it must have enough disk space for storing your local Docker images. Then ssh into it and install Docker, if it’s not installed. Also, you need to have installed OpenSSL on this node, after that we can create own registry:

Alternative you can install it on a Master / Control Plane node

> master01
```shell
mkdir -p /opt/registry/{certs,store,auth}
```

We just created a new directory for our new Docker registry, in store sub-directory we’ll keep our docker images, in certs directory we’ll put our self-created SSL certificate and in auth will be htpasswd file with credentials for basic authentication.

Now let’s create a certificate using OpenSSL utility, as we don’t have a FQDN name for this registry we need to put the node public IP into the openssl config file, into [ v3_ca ] section first:

```shell
vi /etc/ssl/openssl.cnf
[ v3_ca ]
subjectAltName=IP:x.x.x.x # Put your master or registory node IP address here
```

Now let’s create a cert and key pair:

```shell
cd /opt/registryregistry

openssl req -newkey rsa:4096 -nodes -sha256 -keyout certs/domain.key -x509 -days 365 -out certs/domain.crt
```

You will see output similar to below

```shell
Generating a RSA private key
.............................................................................++++
........................................................................................++++
writing new private key to 'certs/domain.key'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:
State or Province Name (full name) [VIC]:
Locality Name (eg, city) []:
Organization Name (eg, company) [Some Company Ltd]:
Organizational Unit Name (eg, section) []:
Common Name (e.g. server FQDN or YOUR name) []:x.x.x.x                   
Email Address []:
```

Don’t forget to put the registry node public IP as a server FQDN name. This will create two files in the cert directory, called domain.crt and domain.key.

Now we need to create a password file for the basic authentication:

```shell
cd /opt/registryregistry

docker run --rm --entrypoint htpasswd registry:2 -Bbn admin some-password >> auth/htpasswd
```

OK, we almost did with it, now let’s start our registry using docker command:

```shell
docker run -d -p 5000:5000 --restart=always --name registry \
  -v /opt/registry/certs:/certs \
  -v /opt/registry/store:/var/lib/registry \
  -v /opt/registry/auth:/auth \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
  -e REGISTRY_AUTH=htpasswd \
  -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
  -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
  registry:2
```

Then make sure that our registry container is running well:

```shell
docker ps

CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
7d89a1788086        registry:2          "/entrypoint.sh /etc…"   2 seconds ago       Up 1 second         0.0.0.0:5000->5000/tcp   registry
```

If you’ll get any errors at this point, please use docker logs container_id command for figuring them out.

Now let’s try to login into this new registry, from any node with installed docker run:

```shell
master02# docker login x.x.x.x:5000
Username: admin
Password: some-password
Error response from daemon: Get https://x.x.x.x:5000/v1/users/: x509: certificate signed by unknown authority
```

This error happens because we use a self-signed certificate and docker know nothing about it, if you have a valid certificate, you can use them instead.

In our example, we need to add the self-created certificate to docker trusted certificates, on all nodes that will use this registry.

On all these nodes you need to run:

```shell
mkdir -p /etc/docker/certs.d/x.x.x.x:5000

# Copy in there domain.crt file from registry /opt/registry/certs
scp x.x.x.x:/opt/registry/certs/domain.crt /etc/docker/certs.d/x.x.x.x:5000

# Rename file
cd /etc/docker/certs.d/x.x.x.x:5000
mv domain.crt ca.crt

# Reload docker engine
systemctl reload docker
```

After this will be done, try to log in one more time:

```shell
master02# docker login x.x.x.x:5000
Username: admin
Password: some-password
Login Succeeded
```

Don’t forget to repeat step with adding self signed certificate on all docker for all your Kubernetes cluster nodes too !

Good, now all works as it must, let’s test out registry by putting in there any Docker image, for example, nginx. It may need to pull them from public registry first:

```shell
master02# docker pull nginx

Using default tag: latest
latest: Pulling from library/nginx
27833a3ba0a5: Pull complete 
e83729dd399a: Pull complete 
ebc6a67df66d: Pull complete 
Digest: sha256:dff6326b09c76bef1425ee64c2e218b38737cdb5412b8ccf84ca70740bfa1db2
Status: Downloaded newer image for nginx:latest

master01# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
nginx               latest              2bcb04bdb83f        11 hours ago        109MB

master01# docker tag 2bcb04bdb83f x.x.x.x:5000 nginx

master01# docker push x.x.x..:5000/nginx
The push refers to a repository [x.x.x.x:5000/nginx]
7e274c0effe8: Pushed 
dd0338cdfab3: Pushed 
5dacd731af1b: Pushed 
latest: digest: 
sha256:dc4887a794219dc5e3459abc1575f1fa293da344772c33f5bf0a96767371d4e3 size: 948
```

OK, now we have tested our new private registry by pushing in there some test Docker image, and it works well. You can check the store directory on the registry node and see their new sub-directories with our pushed nginx test image:

```shell
master01# ls /opt/registry/store/docker/registry/v2/repositories/nginx
```

As I said previously you need to add this self-signed certificate to all docker engines on nodes that will communicate with your internal docker registry, or use the valid certificate instead.

Well, it’s time to add this new internal registry to our Kubernetes cluster. First, we need to create some Secret in our Kubernetes cluster for it. This Secret will use information from any node, that you have configured with our self-created certificates and successfully logged in to this local registry after this. After a successful login, the docker engine on these nodes always stores information in config.json file for the future time. So we need to take it from there:

```shell
master01# cat ~/.docker/config.json | base64

ewoJImF1dGhzIjogewoJCSIxOTQuMTMyLjQ5LjExNzo1MDAwIjogewoJCQkiYXV0aCI6ICJZV1J00NmRXaG1lV0p1VmpjNCIKCQl9LAoJCSJsYWJzLmludGVybmV0dmlraW5ncy5zZTo1MDAwIjogoJCQkiYXV0aCI6ICJjbVZuWVdSdGFXNDZjR2h2WVRWQ2RYbz0iCgkJfQoJfQrdf
```

Copy this data and login then to your Kubernetes cluster control node, with configured kubectl, there we’ll add new Secret to our cluster:

```shell
master01# vi registry-secret.yaml
apiVersion: v1
kind: Secret
metadata:
 name: registrypullsecret
data:
 .dockerconfigjson: <base-64-encoded-copied-json-here>
type: kubernetes.io/dockerconfigjson
```

Create this new secret then:

```shell
master01# kubectl create -f registry-secret.yaml

master01# kubectl get secrets
.....
registrypullsecret                   kubernetes.io/dockerconfigjson        1      59s
```

And at this point we can try to deploy previously pushed nginx, from our local registry as a pod:

```shell
master01# vi local-registry-nginx.yaml
apiVersion: v1
kind: Pod
metadata:
 name: nginx-test
spec:
 containers:
 - name: nginx-test
   image: x.x.x.x:5000/nginx
 imagePullSecrets:
 - name: registrypullsecretcontrol
 
kubectl create -f local-registry-nginx.yaml
pod/nginx-test created

master01# kubectl get pods
nginx-test                          1/1     Running   0          9s
```

As you can see, there was successfully downloaded and started the nginx pod, from our local docker registry.

You can sure about it by running kubectl describe pod nginx-test and looking into Containers: section:

```shell
master01# kubectl describe pod nginx-test.... ... ..
Containers:
  nginx-test:
    Container ID:   docker://e31c8c4f444a02467d8352b1ea2b4e7688176969c962e736590931a0ce594e8f
    Image:          x.x.x.x:5000/nginx
    Image ID:       docker-pullable://x.x.x.x:5000/nginx@sha256:gc4887a794219dc5e3459abc1575f1fa293da344772c33f5bf0as6767371s4e3
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Tue, 28 Mar 2020 10:11:45 +0100
    Ready:          True
    Restart Count:  0
    Environment:    <none>
```

This means that you can use this new internal registry for storing any private containers if you don’t wanna share them with the Docker community for security or other reasons. Also, you can build any Kubernetes resources based on these images as well.

If you’ll get any error, when Kubernetes try to pull images, that mean you forgot to add certificate it to Docker engine on all or some of your Kubernetes nodes.