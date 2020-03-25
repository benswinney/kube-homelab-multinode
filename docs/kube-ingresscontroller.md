# Ingress Controller (nginx) with Certmanager and Lets Encrypt

Putting an ingress controller in front of our applications provides benefits like basic load balancing, SSL/TLS termination, support for URI rewrites, and upstream SSL/TLS encryption.

Apply the mandatory Nginx Ingress Controller yaml file.

```shell
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.30.0/deploy/static/mandatory.yaml
```

Now we need to create a Load Balancer Service type for the Nginx Ingress Controller. We will use MetalLB as the Bare-Metal LB.

```shell
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.30.0/deploy/static/provider/cloud-generic.yaml
```

Let's check what External IP address has been provided by MetalLB (take a note of it for later)

```shell
kubectl -n ingress-nginx get svc -n ingress-nginx
```

Confirm that the Ingress Controller Pods have started:

```shell
kubectl get pods --all-namespaces -l app.kubernetes.io/name=ingress-nginx
```

Now, confirm that the DigitalOcean Load Balancer was successfully created by fetching the Service details with kubectl:

```shell
kubectl get svc --namespace=ingress-nginx
```

This load balancer receives traffic on HTTP and HTTPS ports 80 and 443, and forwards it to the Ingress Controller Pod. The Ingress Controller will then route the traffic to the appropriate backend Service.

We can now point our DNS records at this external Load Balancer and create some Ingress Resources to implement traffic routing rules. Before we do that, we'll now install cert-manager and letsencrypt to provide secure TLS certificates for our services

Start by creating the cert-manager namespace

```shell
kubectl create namespace cert-manager
```

Next install cert-manager and it's CRD's

```shell
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.14.0/cert-manager.yaml
```

Lets test by using the LetEncrypt staging server to issue a test certificate






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
