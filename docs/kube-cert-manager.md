# Cert-Manager & Lets Encrypt 

Realistically, we can skip an Ingress Contoller and expose our applications directly via the MetalLB load balancer.

Putting an ingress controller in front of our applications provides benefits like basic load balancing, SSL/TLS termination, support for URI rewrites, and upstream SSL/TLS encryption.

## Easier Ingress Controller install

```shell
helm install ingress stable/nginx-ingress --set controller.hostNetwork=true,controller.kind=DaemonSet
```

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
