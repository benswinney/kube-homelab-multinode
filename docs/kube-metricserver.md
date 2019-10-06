# Metric Server

We'll install the Metric Server for insights and as a pre-req to Horizontal Pod Scaling capabilities.

```shell
kubectl create -f metric-server/aggregated-metrics-reader.yaml
kubectl create -f metric-server/auth-delegator.yaml
kubectl create -f metric-server/auth-reader.yaml
kubectl create -f metric-server/metrics-apiservice.yaml
kubectl create -f metric-server/metrics-server-deployment.yaml
kubectl create -f metric-server/metrics-server-service.yaml
kubectl create -f metric-server/resource-reader.yaml
```
