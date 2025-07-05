# Connect two clusters using Istio multicluster configuration with private NLBs

This guide will help you to configure two Istio Clusters that are in different VPCs, using private Network Load Balancers (NLBs) to connect them. We are using VPC Endpoint.

## Prerequisites

Before starting please make sure that you kubeconfig files has the contexts for both clusters, `demo-cluster1` and `demo-cluster2`, and that you have Istio installed on both clusters.
Remove the default ca-certs in both clusters to avoid conflicts with the new certificates we will generate.

```bash
kubectl delete secret istio-ca-secret -n istio-system --context=cluster1
kubectl delete secret istio-ca-secret -n istio-system --context=cluster2
```

## Generate new ca-certs process:

We will generate new CA certificates for the clusters using Istio's self-signed certificate generation tools.
git clone https://github.com/istio/istio.git

```bash
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
```

## Configurte Istio connections and gateways

To start we need create the load balances for each cluster, because the load balancer will be private and we will added a vpc endpoint to connect both clusters because in this case are in different VPCs, so we need to get the DNS names of the load balancers to configure the Istio gateways. This happens because when the LB is private Istio cannot get the DNS names automatically, so we need parse this DNS manually using Istio destination rules.

### Create and configure Istio Eastwest gateways in cluster 1

1. Set default network for cluster 1

```bash
   cd /cluster1/istio
   kubectl apply -f 1-cfg-namespace.yaml --context cluster1
```

2. Create NLB using Service

```bash
   kubectl apply -f 2.0-istio-nlb.yaml --context cluster1
```

This steep will create a Service and create a Network Load Balancer (NLB) in AWS.

3. Install eastwgtway

```bash
   istioctl install -f 2.1-istio-eastwgtway.yaml --context cluster1
```

4. Expose service in cluster 1

```bash
   kubectl apply -f 3-gateway.yaml --context cluster1
```

### Create and configure Istio Eastwest gateways in cluster 2

1. Set default network for cluster 2

```bash
   cd /cluster2/istio
   kubectl apply -f 1-cfg-namespace.yaml --context cluster2
```

2. Create NLB using Service

```bash
   kubectl apply -f 2.0-istio-nlb.yaml --context cluster2
```

This steep will create a Service and create a Network Load Balancer (NLB) in AWS.

3. Install eastwgtway

```bash
   istioctl install -f 2.1-istio-eastwgtway.yaml --context cluster2
```

4. Expose service in cluster 2

```bash
   kubectl apply -f 3-gateway.yaml --context cluster2
```

## Configure Istio multicluster configuration

### Enable endpoints discovery

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

Now both clusters have their Istio Eastwest gateways installed and configured. The next step is to connect both clusters using Istio multicluster configuration.

Before that we need create a VPC Endpoint in AWS to connect both clusters. This VPC Endpoint will be used to connect the private NLBs of both clusters.

### Configure cluster 1 to connect to cluster 2

```
cd /cluster1/istio
```

1. Get the DNS name of the Endpoint created to connect cluster 2. in aws.

2. Use the serviceentry-cluster2.yaml file to create a ServiceEntry in cluster 1 that points to the NLB of cluster 2. Update the `hosts` field with the DNS name you obtained in the previous step. Use the section:

```yaml
endpoints:
  - address: # ⚠️ Use the Endpoint DNS
```

3. Apply the ServiceEntry in cluster 1:
   ```bash
   kubectl apply -f 2.2-serviceentry-cluster2-eastwest.yaml --context cluster1
   ```
4. Create a DestinationRule in cluster 1 to define the TLS settings for the connection to cluster 2.
   Use the `2.3-destinationrule-cluster2-eastwest.yaml` file and apply it:
   ```bash
   kubectl apply -f 2.3-destinationrule-cluster2-eastwest.yaml --context cluster1
   ```

### Configure cluster 2 to connect to cluster 1

```
cd /cluster2/istio
```

1. Get the DNS name of the Endpoint created to connect cluster 1. in aws.

2. Use the serviceentry-cluster1.yaml file to create a ServiceEntry in cluster 2 that points to the NLB of cluster 1. Update the `hosts` field with the DNS name you obtained in the previous step. Use the section:

```yaml
endpoints:
  - address: # ⚠️ Use the Endpoint DNS
```

3. Apply the ServiceEntry in cluster 2:
   ```bash
   kubectl apply -f 2.2-serviceentry-cluster1-eastwest.yaml --context cluster2
   ```
4. Create a DestinationRule in cluster 2 to define the TLS settings for the connection to cluster 1.
   Use the `2.3-destinationrule-cluster1-eastwest.yaml` file and apply it:
   ```bash
   kubectl apply -f 2.3-destinationrule-cluster1-eastwest.yaml --context cluster2
   ```

## Test connectivity

istioctl remote-clusters --context=cluster2
istioctl remote-clusters --context=cluster1

Now you have Istio Eastwest gateways installed and configured in both clusters. Now we gonna connect both clusters using Istio multicluster configuration.

## Test the connection

You can test the connection by deploying a sample application in one cluster and accessing it from the other cluster.
These samples applications run a web server and expose in port 8085 for cluster1 and 80 for cluster2 using a k8s service. Add and istio virtual service and destination rule to expose the application into the mesh.

### Deploy sample application in cluster 1

```bash
cd cluster1/
kubectl --context cluster1 apply -f demo_app/namespace.yaml
kubectl --context cluster1 apply -f demo_app/service-account.yaml
kubectl --context cluster1 apply -f demo_app/deployment.yaml
kubectl --context cluster1 apply -f demo_app/service.yaml
kubectl --context cluster1 apply -f demo_app/istio-vs.yaml
kubectl --context cluster1 apply -f demo_app/istio-destination-rule.yaml
```

### Deploy sample application in cluster 2

```bash
cd cluster2/
kubectl --context cluster2 apply -f demo_app/namespace.yaml
kubectl --context cluster2 apply -f demo_app/service-account.yaml
kubectl --context cluster2 apply -f demo_app/deployment.yaml
kubectl --context cluster2 apply -f demo_app/service.yaml
kubectl --context cluster2 apply -f demo_app/istio-vs.yaml
kubectl --context cluster2 apply -f demo_app/istio-destination-rule.yaml
```

### Restar istiod in both clusters

```bash
kubectl rollout restart deployment istiod -n istio-system --context=demo-cluster1
kubectl rollout restart deployment istiod -n istio-system --context=demo-cluster2
kubectl rollout restart deployment istio-eastwestgateway -n istio-system --context=demo-cluster1
kubectl rollout restart deployment istio-eastwestgateway -n istio-system --context=demo-cluster2
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
