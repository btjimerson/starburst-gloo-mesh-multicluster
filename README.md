# Gloo Mesh Ambient Multi-cluster

![image.png](images/multi-cluster.png)

# Introduction

This walkthrough demonstrates a multi-cluster setup in Gloo Mesh. It goes through the following:

1. Installing the Gloo Operator and Service Mesh Controller on both clusters.
2. Configuring applications to use Istio Ambient mode.
3. Configure multi-cluster services in Istio.
4. Configuring a Gateway and associated HTTP routes for applications
5. Configuring a Waypoint proxy for applications.
6. Demonstrating Ambient features such as:
    1. Routing to specific application versions.
    2. Dynamic routing based on information such as headers.
    3. Isolating namespaces through authorization policies.
    4. Resiliency and fault injection.
7. Installing Gloo Mesh Management Server on the 1st cluster for observability.

# Prerequisites

Make sure you have the following available:

1. 2 Kubernetes clusters
2. The `kubectl` CLI - [installation](https://kubernetes.io/docs/tasks/tools/#kubectl)
3. The `helm` CLI - [installation](https://helm.sh/docs/intro/install/)
4. The `meshctl` tool - [installation](https://docs.solo.io/gloo-mesh/latest/setup/prepare/cli/)
5. (Optional) Solo’s version of `istioctl` - [installation](https://support.solo.io/hc/en-us/articles/4414409064596-Istio-images-built-by-Solo-io)

Clone this repository and change to the root directory of the repository.

Add the Gloo Platform Helm repository if necessary:

```bash
helm repo add gloo-platform https://storage.googleapis.com/gloo-platform/helm-charts
helm repo update
```

# Set Up Environment

Set the following environment variables. CLUSTER1 and CLUSTER2 should be the name of your Kubernetes contexts:

```bash
export CLUSTER1=<cluster1-context>
export CLUSTER2=<cluster2-context>
export CLUSTER1_NAME=cluster1
export CLUSTER2_NAME=cluster2
export GLOO_VERSION=2.8.1
export ISTIO_VERSION=1.25.3
export GLOO_MESH_LICENSE_KEY=<gloo-mesh-license-key>
```

Verify they’re set properly:

```bash
echo "Cluster 1: $CLUSTER1"
echo "Cluster 1 Name: $CLUSTER1_NAME"
echo "Cluster 2: $CLUSTER2"
echo "Cluster 2 Name: $CLUSTER2_NAME"
echo "Gloo Version: $GLOO_VERSION"
echo "Istio Version: $ISTIO_VERSION"
```

# Deploy Sample Applications

## Bookinfo

Install the bookinfo application in both clusters:

```bash
for context in $CLUSTER1 $CLUSTER2; do
  kubectl --context $context create namespace bookinfo
  kubectl --context $context apply -n bookinfo -f https://raw.githubusercontent.com/istio/istio/release-1.24/samples/bookinfo/platform/kube/bookinfo.yaml
  kubectl --context $context apply -n bookinfo -f https://raw.githubusercontent.com/istio/istio/release-1.24/samples/bookinfo/platform/kube/bookinfo-versions.yaml
  kubectl set env -n bookinfo --context $context deployments/reviews-v1 CLUSTER_NAME=$context
  kubectl set env -n bookinfo --context $context deployments/reviews-v2 CLUSTER_NAME=$context
  kubectl set env -n bookinfo --context $context deployments/reviews-v3 CLUSTER_NAME=$context
done
```

## Command Runner

Install the command runner application in cluster 1. This will allow you to easily execute Bash commands in a web browser:

```bash
kubectl apply --context $CLUSTER1 -f https://raw.githubusercontent.com/btjimerson/command-runner/refs/heads/main/kubernetes/command-runner.yaml
```

# Install Istio

## Kubernetes Gateway API

Install the Kubernetes Gateway CRDs:

```bash
for context in $CLUSTER1 $CLUSTER2; do
  kubectl apply --context $context -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml
done
```

## Installation

Create the Istio namespaces:

```bash
for context in $CLUSTER1 $CLUSTER2; do
  kubectl --context=$context create ns istio-system || true
  kubectl --context=$context create ns istio-gateways || true
done
```

Install the self-signed certificate chain:

```bash
kubectl --context=$CLUSTER1 create secret generic cacerts -n istio-system \
--from-file=./certs/cluster1/ca-cert.pem \
--from-file=./certs/cluster1/ca-key.pem \
--from-file=./certs/cluster1/root-cert.pem \
--from-file=./certs/cluster1/cert-chain.pem
kubectl --context=$CLUSTER2 create secret generic cacerts -n istio-system \
--from-file=./certs/cluster2/ca-cert.pem \
--from-file=./certs/cluster2/ca-key.pem \
--from-file=./certs/cluster2/root-cert.pem \
--from-file=./certs/cluster2/cert-chain.pem
```

Install the Gloo Istio operator:

```bash
for context in $CLUSTER1 $CLUSTER2; do
  helm upgrade --install --kube-context $context gloo-operator \
  oci://us-docker.pkg.dev/solo-public/gloo-operator-helm/gloo-operator \
  --version 0.2.0-beta.1 \
  -n gloo-system \
  --create-namespace \
  --set manager.env.SOLO_ISTIO_LICENSE_KEY=$GLOO_MESH_LICENSE_KEY
done
```

Wait for the operator to become available:

```bash
for context in $CLUSTER1 $CLUSTER2; do
  kubectl rollout status --context $context -n gloo-system deployments/gloo-operator
done
```

Create service mesh controllers to install Istio on both clusters:

```bash
kubectl --context $CLUSTER1 apply -f- <<EOF
apiVersion: operator.gloo.solo.io/v1
kind: ServiceMeshController
metadata:
  name: istio
spec:
  version: $ISTIO_VERSION
  cluster: $CLUSTER1_NAME
  network: $CLUSTER1_NAME
EOF

kubectl --context $CLUSTER2 apply -f- <<EOF
apiVersion: operator.gloo.solo.io/v1
kind: ServiceMeshController
metadata:
  name: istio
spec:
  version: $ISTIO_VERSION
  cluster: $CLUSTER2_NAME
  network: $CLUSTER2_NAME
EOF
```

Wait for the istio control planes to become available:

```bash
for context in $CLUSTER1 $CLUSTER2; do
  kubectl rollout status --context $context -n istio-system deployments/istiod-gloo
done
```

Join workloads to the mesh:

```bash
for context in $CLUSTER1 $CLUSTER2; do
  kubectl --context $context label namespace bookinfo istio.io/dataplane-mode=ambient --overwrite
done

kubectl --context $CLUSTER1 label namespace command-runner istio.io/dataplane-mode=ambient --overwrite
```

Enforce strict mTLS with a peer authentication policy:

```bash
for context in $CLUSTER1 $CLUSTER2; do
kubectl apply --context $context -f- <<EOF
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT
EOF
done
```

## Multi-cluster Services

Deploy east-west gateways in both clusters:
    
```bash
kubectl apply --context $CLUSTER1 -f- <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  labels:
    istio.io/expose-istiod: "15012"
    topology.istio.io/network: $CLUSTER1_NAME
  name: istio-eastwest
  namespace: istio-gateways
spec:
  gatewayClassName: istio-eastwest
  listeners:
  - name: cross-network
    port: 15008
    protocol: HBONE
    tls:
      mode: Passthrough
  - name: xds-tls
    port: 15012
    protocol: TLS
    tls:
      mode: Passthrough
EOF

kubectl apply --context $CLUSTER2 -f- <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  labels:
    istio.io/expose-istiod: "15012"
    topology.istio.io/network: $CLUSTER2_NAME
  name: istio-eastwest
  namespace: istio-gateways
spec:
  gatewayClassName: istio-eastwest
  listeners:
  - name: cross-network
    port: 15008
    protocol: HBONE
    tls:
      mode: Passthrough
  - name: xds-tls
    port: 15012
    protocol: TLS
    tls:
      mode: Passthrough
EOF
```

Wait for the east-west gateways to become available:

```bash
for context in $CLUSTER1 $CLUSTER2; do
  kubectl wait gateway/istio-eastwest --for=condition=Programmed --context $context -n istio-gateways
done
```

Get the addresses of the 2 clusters’ east-west gateways and link them with Gateways
    
```bash
export CLUSTER1_EW_ADDRESS=$(kubectl get svc -n istio-gateways istio-eastwest --context $CLUSTER1 -o jsonpath="{.status.loadBalancer.ingress[0]['hostname','ip']}")
export CLUSTER2_EW_ADDRESS=$(kubectl get svc -n istio-gateways istio-eastwest --context $CLUSTER2 -o jsonpath="{.status.loadBalancer.ingress[0]['hostname','ip']}")

echo "Cluster 1 east-west gateway: $CLUSTER1_EW_ADDRESS"
echo "Cluster 2 east-west gateway: $CLUSTER2_EW_ADDRESS"

kubectl apply --context $CLUSTER1 -f- <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  annotations:
    gateway.istio.io/service-account: istio-eastwest
    gateway.istio.io/trust-domain: $CLUSTER2_NAME
  labels:
    topology.istio.io/network: $CLUSTER2_NAME
  name: istio-remote-peer-$CLUSTER2_NAME
  namespace: istio-gateways
spec:
  addresses:
  - type: IPAddress
    value: $CLUSTER2_EW_ADDRESS
  gatewayClassName: istio-remote
  listeners:
  - name: cross-network
    port: 15008
    protocol: HBONE
    tls:
      mode: Passthrough
  - name: xds-tls
    port: 15012
    protocol: TLS
    tls:
      mode: Passthrough
EOF

kubectl apply --context $CLUSTER2 -f- <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  annotations:
    gateway.istio.io/service-account: istio-eastwest
    gateway.istio.io/trust-domain: $CLUSTER1_NAME
  labels:
    topology.istio.io/network: $CLUSTER1_NAME
  name: istio-remote-peer-$CLUSTER1_NAME
  namespace: istio-gateways
spec:
  addresses:
  - type: IPAddress
    value: $CLUSTER1_EW_ADDRESS
  gatewayClassName: istio-remote
  listeners:
  - name: cross-network
    port: 15008
    protocol: HBONE
    tls:
      mode: Passthrough
  - name: xds-tls
    port: 15012
    protocol: TLS
    tls:
      mode: Passthrough
EOF
```

Label the product page to enable multi-cluster routing:

```bash
for context in $CLUSTER1 $CLUSTER2; do
  kubectl --context $context -n bookinfo label service productpage solo.io/service-scope=global --overwrite
  kubectl --context $context -n bookinfo annotate service productpage networking.istio.io/traffic-distribution=Any --overwrite
done
```

Get the service entry and workload entry created for the global product page:

```bash
for context in $CLUSTER1 $CLUSTER2; do
  echo "Service entries and workload entries for cluster $context:"
  echo ""
  kubectl get serviceentry --context $context -n istio-system
  kubectl get workloadentry --context $context -n istio-system
  echo ""
done
```

## Gateway Configuration

Create a gateway for cluster 1 ingress:

```bash
kubectl --context=$CLUSTER1 apply -f- <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: http-gateway
  namespace: istio-gateways
spec:
  gatewayClassName: istio
  listeners:
  - name: http-80
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: All
  - name: http-8080
    port: 8080
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: All
EOF
```

Wait for the gateway and backing service to become available:

```bash
kubectl wait gateway/http-gateway --for=condition=Programmed --context $CLUSTER1 -n istio-gateways
```

Get the address of the gateway:

```bash
export GATEWAY_ADDRESS=$(kubectl get svc -n istio-gateways http-gateway-istio --context $CLUSTER1 -o jsonpath="{.status.loadBalancer.ingress[0]['hostname','ip']}")
echo "Gateway address: $GATEWAY_ADDRESS"
```

## HTTP Route Configuration


Create a route for the command runner application:

```bash
kubectl --context $CLUSTER1 apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: command-runner-route
  namespace: command-runner
spec:
  parentRefs:
  - name: http-gateway
    namespace: istio-gateways
    sectionName: http-8080
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: command-runner
      port: 8080
EOF
```

Open the command runner application to make sure the route is working:

```bash
open $(echo http://$GATEWAY_ADDRESS:8080/)
```

Create a route for the global product page destination:

```bash
kubectl --context $CLUSTER1 apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: bookinfo-route
  namespace: bookinfo
spec:
  parentRefs:
  - name: http-gateway
    namespace: istio-gateways
    sectionName: http-80
  rules:
  - matches:
    - path:
        type: Exact
        value: /productpage
    - path:
        type: PathPrefix
        value: /static
    - path:
        type: Exact
        value: /login
    - path:
        type: Exact
        value: /logout
    # Reference the global destination address, 
    # not the local service
    backendRefs:
    - kind: Hostname
      group: networking.istio.io
      name: productpage.bookinfo.mesh.internal
      port: 9080
EOF
```

Open the product page. Refresh the page and check the cluster name in the reviews to verify that traffic is routed across clusters:

```bash
open $(echo http://$GATEWAY_ADDRESS/productpage)
```

Scale the product page in cluster 1 to 0 replicas:

```bash
kubectl scale --context $CLUSTER1 -n bookinfo --replicas=0 deployments/productpage-v1 
```

Refresh the product page again. All traffic should be routed to cluster 2 (make note of the cluster name in the reviews).

Scale the product page in cluster 1 back to 1 replica:

```bash
kubectl scale --context $CLUSTER1 -n bookinfo --replicas=1 deployments/productpage-v1 
```

## Waypoint Configuration

Deploy a waypoint to the `bookinfo` namespace for layer 7 policies:

```bash
for context in $CLUSTER1 $CLUSTER2; do
kubectl apply --context $context -f- <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: waypoint
  namespace: bookinfo
spec:
  gatewayClassName: istio-waypoint
  listeners:
  - name: mesh
    port: 15008
    protocol: HBONE
EOF

kubectl label namespace bookinfo --context $context istio.io/use-waypoint=waypoint
done
```
Deploy a waypoint for the command runner namespace:

```bash
kubectl apply --context $CLUSTER1 -f- <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: waypoint
  namespace: command-runner
spec:
  gatewayClassName: istio-waypoint
  listeners:
  - name: mesh
    port: 15008
    protocol: HBONE
EOF

kubectl label namespace command-runner --context $CLUSTER1 istio.io/use-waypoint=waypoint
```

# Ambient Features

## Route to Specific Version

Route all traffic to reviews v1:

```bash
for context in $CLUSTER1 $CLUSTER2; do
kubectl apply --context $context -f- <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: reviews
  namespace: bookinfo
spec:
  parentRefs:
  - group: ""
    kind: Service
    name: reviews
    port: 9080
  rules:
  - backendRefs:
    - name: reviews-v1
      port: 9080
EOF
done
```

Show that all traffic is served by v1 of reviews (note the pod name in the reviews; it should be reviews-v1):

```bash
open $(echo http://$GATEWAY_ADDRESS/productpage)
```

Clean up the route:

```bash
for context in $CLUSTER1 $CLUSTER2; do
  kubectl delete httproute --context $context -n bookinfo reviews
done
```

## Route Based on User

Route traffic based on user:

```bash
for context in $CLUSTER1 $CLUSTER2; do
kubectl apply --context $context -f- <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: reviews
  namespace: bookinfo
spec:
  parentRefs:
  - group: ""
    kind: Service
    name: reviews
    port: 9080
  rules:
  - matches:
    - headers:
      - name: end-user
        value: jason
    backendRefs:
    - name: reviews-v2
      port: 9080
  - backendRefs:
    - name: reviews-v1
      port: 9080
EOF
done
```

Open the product page and login in with jason / jason. When logged in as jason, reviews-v2 pods should be used. Otherwise reviews-v1 pods should be displayed:

```bash
open $(echo http://$GATEWAY_ADDRESS/productpage)
```

Clean up routes:

```bash
for context in $CLUSTER1 $CLUSTER2; do
  kubectl delete httproute --context $context -n bookinfo reviews
done
```

## Namespace Isolation

Apply an authorization policy that isolates traffic to the bookinfo namespace:

```bash
for context in $CLUSTER1 $CLUSTER2; do
kubectl apply --context $context -f- <<EOF
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: namespace-isolation
  namespace: bookinfo
spec:
  action: ALLOW
  rules:
  - from:
    - source:
        namespaces: ["bookinfo"]
  targetRefs:
  - kind: Gateway
    group: gateway.networking.k8s.io
    name: waypoint
EOF
done
```

Open the command runner application and try to curl the reviews API. You should get a 'RBAC: access denied' response:

```bash
curl -ivk http://reviews.bookinfo:9080/reviews/1 
```

Remove the authorization policy:

```bash
for context in $CLUSTER1 $CLUSTER2; do
  kubectl delete authorizationpolicy namespace-isolation --context $context -n bookinfo
done
```

Try to curl the reviews API again:

```bash
curl -ivk http://reviews.bookinfo:9080/reviews/1 
```

Apply an authorization policy that allows L7 GET traffic to the reviews service from the product page principal (SPIFFE ID):

```bash
for context in $CLUSTER1 $CLUSTER2; do
kubectl apply --context $context -f- <<EOF
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: reviews-policy
  namespace: bookinfo
spec:
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster1/ns/bookinfo/sa/bookinfo-productpage", "cluster2/ns/bookinfo/sa/bookinfo-productpage", "cluster1/ns/bookinfo/sa/bookinfo-reviews", "cluster2/ns/bookinfo/sa/bookinfo-reviews"]
    to:
    - operation:
        methods: ["GET"]
  targetRefs:
  - kind: Gateway
    group: gateway.networking.k8s.io
    name: waypoint
EOF
done
```

Open the command runner application and try to curl the reviews API:

```bash
curl -ivk http://reviews.bookinfo:9080/reviews/1 
```

You should get a 'RBAC: access denied' response. Now refresh the product page and ensure you can still access the reviews service.

Clean up the authorization policy:

```bash
for context in $CLUSTER1 $CLUSTER2; do
  kubectl delete authorizationpolicy --context $context -n bookinfo reviews-policy
done
```

## Request Timeouts

Route traffic to v2 of reviews (which calls the ratings service):

```bash
for context in $CLUSTER1 $CLUSTER2; do
kubectl apply --context $context -f- <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: reviews
  namespace: bookinfo
spec:
  parentRefs:
  - group: ""
    kind: Service
    name: reviews
    port: 9080
  rules:
  - backendRefs:
    - name: reviews-v2
      port: 9080
EOF
done
```

Add a 2 second delay to the ratings service:

```bash
for context in $CLUSTER1 $CLUSTER2; do
kubectl apply --context $context -f- <<EOF
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: ratings
  namespace: bookinfo
spec:
  hosts:
  - ratings
  http:
  - fault:
      delay:
        fixedDelay: 2s
        percentage:
          value: 100.0
    route:
    - destination:
        host: ratings
EOF
done
```

Open the product page and notice the 2 second delay:

```bash
open $(echo http://$GATEWAY_ADDRESS/productpage)
```

Update the reviews route to use a 0.5 second timeout:

```bash
for context in $CLUSTER1 $CLUSTER2; do
kubectl apply --context $context -f- <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: reviews
  namespace: bookinfo
spec:
  parentRefs:
  - group: ""
    kind: Service
    name: reviews
    port: 9080
  rules:
  - backendRefs:
    - name: reviews-v2
      port: 9080
    # Add a 0.5 second timeout
    timeouts:
      request: 500ms
EOF
done
```

Open the product page again. The reviews should be unavailable because the timeout is less than the delay:

```bash
open $(echo http://$GATEWAY_ADDRESS/productpage)
```

Clean up the virtual services and routes:

```bash
for context in $CLUSTER1 $CLUSTER2; do
  kubectl delete httproute reviews --context $context -n bookinfo
  kubectl delete virtualservice ratings --context $context -n bookinfo
done
```

# Workload identities

Enable access logging for the bookinfo namespace using a Telemetry object:

```bash
kubectl apply -f- <<EOF
apiVersion: telemetry.istio.io/v1
kind: Telemetry
metadata:
  name: access-logging
  namespace: bookinfo
spec:
  accessLogging:
    - providers:
        - name: envoy
EOF
```

Find the ztunnel running on the same node as the product page:

```bash
NODE_NAME=$(kubectl get pod -n bookinfo --context $CLUSTER1 -l app=productpage -o jsonpath="{.items[0].spec.nodeName}")
ZTUNNEL_POD=$(kubectl get pods -n istio-system --context $CLUSTER1 -l app=ztunnel --field-selector spec.nodeName=$NODE_NAME -o jsonpath="{.items[0].metadata.name}")
echo $ZTUNNEL_POD
```

Curl the reviews API from the command runner application:

```bash
curl -ivk http://reviews.bookinfo:9080/reviews/1 
```

View the access logs for ztunnel, and note the workload identities (`src.identity` and `dst.identity`): 

```bash
kubectl logs -n istio-system $ZTUNNEL_POD | jq
```

# Install Gloo Management Plane

Install the management plane in cluster 1:

```bash
helm upgrade --install gloo-platform-crds gloo-platform/gloo-platform-crds \
  -n gloo-mesh \
  --kube-context $CLUSTER1 \
  --version=$GLOO_VERSION
  
helm upgrade --install gloo-platform gloo-platform/gloo-platform \
  -n gloo-mesh \
  --kube-context $CLUSTER1 \
  --version=$GLOO_VERSION \
  --set licensing.glooMeshLicenseKey=$GLOO_MESH_LICENSE_KEY \
  -f- <<EOF
common:
  cluster: cluster1
glooAgent:
  enabled: true
  runAsSidecar: true
  relay:
    serverAddress: gloo-mesh-mgmt-server.gloo-mesh:9900
glooAnalyzer:
  enabled: true
glooMgmtServer:
  enabled: true
  registerCluster: true
  policyApis:
    enabled: true
glooInsightsEngine:
  enabled: true
glooUi:
  enabled: true
prometheus:
  enabled: true
redis:
  deployment:
    enabled: true
telemetryCollector:
  enabled: true
  mode: deployment
  replicaCount: 1
telemetryGateway:
  enabled: true
installEnterpriseCrds: false
featureGates:
  ConfigDistribution: true
EOF
```

Wait for the management server to become available:

```bash
kubectl rollout status --context $CLUSTER1 -n gloo-mesh deployments/gloo-mesh-mgmt-server
```

Get the management server’s server and telemetry gateway addresses:

```bash
export TELEMETRY_GATEWAY_ADDRESS=$(kubectl get svc -n gloo-mesh gloo-telemetry-gateway --context $CLUSTER1 -o jsonpath="{.status.loadBalancer.ingress[0]['hostname','ip']}"):4317
echo "Telemetry gateway address = $TELEMETRY_GATEWAY_ADDRESS"

export MANAGEMENT_SERVER_ADDRESS=$(kubectl get svc -n gloo-mesh gloo-mesh-mgmt-server --context $CLUSTER1 -o jsonpath="{.status.loadBalancer.ingress[0]['hostname','ip']}"):9900
echo "Management server address = $MANAGEMENT_SERVER_ADDRESS"
```

Create a Kubernetes Cluster resource for cluster 2 in the management server:

```bash
kubectl apply --context $CLUSTER1 -f- <<EOF
apiVersion: admin.gloo.solo.io/v2
kind: KubernetesCluster
metadata:
   name: $CLUSTER2_NAME
   namespace: gloo-mesh
spec:
   clusterDomain: cluster.local
EOF
```

Install the Gloo mesh agent in cluster 2:

```bash
kubectl get secret relay-root-tls-secret -n gloo-mesh --context $CLUSTER1 -o jsonpath='{.data.ca\.crt}' | base64 -d > ca.crt
kubectl create secret generic relay-root-tls-secret -n gloo-mesh --context $CLUSTER2 --from-file ca.crt=ca.crt
rm ca.crt

kubectl get secret relay-identity-token-secret -n gloo-mesh --context $CLUSTER1 -o jsonpath='{.data.token}' | base64 -d > token
kubectl create secret generic relay-identity-token-secret -n gloo-mesh --context $CLUSTER2 --from-file token=token
rm token

helm upgrade --install gloo-platform-crds gloo-platform/gloo-platform-crds \
 --namespace=gloo-mesh \
 --version=$GLOO_VERSION \
 --set installEnterpriseCrds=false \
 --kube-context $CLUSTER2

helm upgrade --install gloo-platform gloo-platform/gloo-platform \
  --kube-context $CLUSTER2 \
  -n gloo-mesh \
  --version $GLOO_VERSION \
  --set common.cluster=$CLUSTER2_NAME \
  --set glooAgent.enabled=true \
  --set glooAgent.authority=gloo-meshmgmt-server.gloo-mesh \
  --set glooAgent.relay.serverAddress=$MANAGEMENT_SERVER_ADDRESS \
  --set glooAnalyzer.enabled=true \
  --set installEnterpriseCrds=false \
  --set telemetryCollector.enabled=true \
  --set telemetryCollector.config.exporters.otlp.endpoint=$TELEMETRY_GATEWAY_ADDRESS \
  --set telemetryCollectorCustomization.skipVerify=true
```

Wait for the agent to start properly:

```bash
kubectl rollout status --context $CLUSTER2 -n gloo-mesh deployments/gloo-mesh-agent
```

Ensure that cluster 2 is registered properly:

```bash
meshctl check --kubecontext $CLUSTER1
```

# Observability

Launch the dashboard:

```bash
meshctl dashboard --kubecontext=$CLUSTER1
```

![gloo-ui.png](images/gloo-ui.png)

# Clean up

Clean up all resources:

```bash
# all waypoints
for context in $CLUSTER1 $CLUSTER2; do
  kubectl delete gateway waypoint --context $context -n bookinfo
done
kubectl delete gateway waypoint --context $CLUSTER1 -n command-runner

# application routes and gateway
kubectl delete httproute bookinfo-route --context $CLUSTER1 -n bookinfo
kubectl delete httproute command-runner-route --context $CLUSTER1 -n command-runner
kubectl delete gateway http-gateway --context $CLUSTER1 -n istio-gateways

# istio gateways
kubectl delete gateway istio-remote-peer-cluster1 --context $CLUSTER2 -n istio-gateways
kubectl delete gateway istio-remote-peer-cluster2 --context $CLUSTER1 -n istio-gateways
for context in $CLUSTER1 $CLUSTER2; do
  kubectl --context $context -n bookinfo label service productpage solo.io/service-scope-
  kubectl --context $context -n bookinfo annotate service productpage networking.istio.io/traffic-distribution-
  kubectl delete service istio-eastwest --context $context -n istio-gateways
  kubectl delete gateway istio-eastwest --context $context -n istio-gateways
  kubectl delete namespace istio-gateways --context $context
done

# istio
for context in $CLUSTER1 $CLUSTER2; do
  kubectl delete peerauthentication default --context $context -n istio-system
  kubectl delete servicemeshcontroller istio --context $context
  helm uninstall gloo-operator --kube-context $context -n gloo-system
  kubectl delete namespace gloo-system --context $context
  kubectl delete secret generic cacerts --context $context -n istio-system
  kubectl delete namespace istio-system --context $context
  kubectl delete namespace istio-gateways --context $context
  kubectl get crds --context $context -oname | grep --color=never 'istio.io' | xargs kubectl delete --context $context --ignore-not-found
done


# gloo management plane
helm uninstall gloo-platform --kube-context $CLUSTER2 -n gloo-mesh
helm uninstall gloo-platform-crds --kube-context $CLUSTER2 -n gloo-mesh
kubectl delete secret relay-root-tls-secret --context $CLUSTER2 -n gloo-mesh
kubectl delete secret relay-identity-token-secret --context $CLUSTER2 -n gloo-mesh
kubectl delete kubernetescluster $CLUSTER2_NAME --context $CLUSTER1 -n gloo-mesh
helm uninstall gloo-platform --kube-context $CLUSTER1 -n gloo-mesh
helm uninstall gloo-platform-crds --kube-context $CLUSTER1 -n gloo-mesh
kubectl delete secret relay-client-tls-secret --context $CLUSTER1 -n gloo-mesh
kubectl delete namespace gloo-mesh --context $CLUSTER1
for context in $CLUSTER1 $CLUSTER2; do
    kubectl get crds --context $context -oname | grep --color=never 'solo.io' | xargs kubectl delete --context $context --ignore-not-found
done

# sample apps
kubectl delete --context $CLUSTER1 -f https://raw.githubusercontent.com/btjimerson/command-runner/refs/heads/main/kubernetes/command-runner.yaml
for context in $CLUSTER1 $CLUSTER2; do
  kubectl --context $context delete -n bookinfo -f https://raw.githubusercontent.com/istio/istio/release-1.24/samples/bookinfo/platform/kube/bookinfo.yaml
  kubectl --context $context delete -n bookinfo -f https://raw.githubusercontent.com/istio/istio/release-1.24/samples/bookinfo/platform/kube/bookinfo-versions.yaml
  kubectl --context $context delete namespace bookinfo
done

# gatewapi crds
for context in $CLUSTER1 $CLUSTER2; do
  kubectl delete --context $context -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml
done
```
