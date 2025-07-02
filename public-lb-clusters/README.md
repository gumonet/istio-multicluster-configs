# Connect two clusters using Istio multicluster configuration

## Generate new ca-certs process:

Before starting please make sure that you kubeconfig files has the contexts for both clusters, `demo-cluster1` and `demo-cluster2`, and that you have Istio installed on both clusters.

We will generate new CA certificates for the clusters using Istio's self-signed certificate generation tools.
git clone https://github.com/istio/istio.git

cd istio/tools/certs
make -f Makefile.selfsigned.mk root-ca
make -f Makefile.selfsigned.mk cluster1-cacerts
make -f Makefile.selfsigned.mk cluster2-cacerts

kubectl create secret generic cacerts -n istio-system \
 --from-file=cluster1/ca-cert.pem \
 --from-file=cluster1/ca-key.pem \
 --from-file=cluster1/root-cert.pem \
 --from-file=cluster1/cert-chain.pem --context=cluster1

kubectl create secret generic cacerts -n istio-system \
 --from-file=cluster2/ca-cert.pem \
 --from-file=cluster2/ca-key.pem \
 --from-file=cluster2/root-cert.pem \
 --from-file=cluster2/cert-chain.pem --context=cluster2

## Configurte Istio connections and gateways

### Cluster 1: cluster1

1. Set default network for cluster 1
   cd /cluster1/istio
   kubectl apply -f 1-cfg-namespace.yaml --context cluster1

2. Install eastwgtway
   istioctl install -f 2-istio-eastwgtway.yaml --context cluster1

3. Expose service in cluster 1
   kubectl apply -f 3-gateway.yaml --context cluster1

### Cluster 2: cluster2

1. Set default network for cluster 2
   cd cluster2/istio
   kubectl apply -f 1-cfg-namespace.yaml --context cluster2

2. Install eastwgtway
   istioctl install -f 2-istio-eastwgtway.yaml --context cluster2

3. Expose service in cluster 2
   kubectl apply -f 3-gateway.yaml --context cluster2

Now you have Istio Eastwest gateways installed and configured in both clusters. Now we gonna connect both clusters using Istio multicluster configuration.

## Enable endpoints discovery

```bash
istioctl create-remote-secret \
 --context=cluster1 \
 --name=cluster1 | \
 kubectl apply -f - --context=cluster2

istioctl create-remote-secret \
 --context=cluster2 \
 --name=cluster2 | \
 kubectl apply -f - --context=cluster1
```

## View Cluster Connections

```bash
istioctl remote-clusters --context=cluster1
istioctl remote-clusters --context=cluster2
```

## Test the connection

You can test the connection by deploying a sample application in one cluster and accessing it from the other cluster.
These samples applications run a web server and expose in port 8085 for cluster1 and 80 for cluster2 using a k8s service. Add and istio virtual service and destination rule to expose the application into the mesh.

### Deploy sample application in cluster 1

```bash
cd cluster1/sample-app
kubectl --context cluster1 apply -f demo_app/namespace.yaml
kubectl --context cluster1 apply -f demo_app/service-account.yaml
kubectl --context cluster1 apply -f demo_app/deployment.yaml
kubectl --context cluster1 apply -f demo_app/service.yaml
kubectl --context cluster1 apply -f demo_app/istio-vs.yaml
kubectl --context cluster1 apply -f demo_app/istio-destination-rule.yaml
```

### Deploy sample application in cluster 2

```bash
cd cluster2/sample-app
kubectl --context cluster2 apply -f demo_app/namespace.yaml
kubectl --context cluster2 apply -f demo_app/service-account.yaml
kubectl --context cluster2 apply -f demo_app/deployment.yaml
kubectl --context cluster2 apply -f demo_app/service.yaml
kubectl --context cluster2 apply -f demo_app/istio-vs.yaml
kubectl --context cluster2 apply -f demo_app/istio-destination-rule.yaml
```

### Restar istiod in both clusters

```bash
kubectl rollout restart deployment istiod -n istio-system --context=cluster1
kubectl rollout restart deployment istiod -n istio-system --context=cluster2
kubectl rollout restart deployment istio-eastwestgateway -n istio-system --context=cluster1
kubectl rollout restart deployment istio-eastwestgateway -n istio-system --context=cluster2
```

### Testing webservers application

To test the mesh traffic you need to access in cluster1 webserver application pod and execute:

```bash
#install curl
apt update && apt install curl
# Now test the traffic
curl api.cluster2-webserver.svc.cluster.local:80
```

Do the same for the pod in cluster2 only change the endpoint

```bash
#install curl
apt update && apt install curl
# Now test the traffic
curl api.cluster1-webserver.svc.cluster.local:8085
```
