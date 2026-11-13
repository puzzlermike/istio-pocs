# Istio Ambient Multi-Cluster Mesh

Istio Ambient Multi-Cluster mesh PoC using two local Kind clusters.<br />
Mesh properties:
- [Multi-primary](https://istio.io/latest/docs/ops/deployment/deployment-models/#control-plane-models)
- [Multi-network](https://istio.io/latest/docs/ops/deployment/deployment-models/#multiple-networks)
- Single [Trust-domain](https://istio.io/latest/docs/ops/deployment/deployment-models/#trust-within-a-mesh)

The PoC consists of two clusters, `ecom-cloud` and `ecom-onprem`, each with their own Istio control plane, communication between the clusters happens via dedicated east-west gateways.

## Architecture

![Architecture](./assets/istio-ambient-mc-arch.svg)

## Local Development

```
# run cloud and onprem cluster
task cluster:clusters-up

# install Gateway API into both clusters
task istio:install-gateway-api-all

# install Istio in both clusters (base, shared root CA + per-cluster cacerts, istiod, CNI, ztunnel)
task istio:install-istio-all

# start cloud-provider-kind in a SEPARATE terminal and keep it running, it assigns
# the LoadBalancer IP of the east-west gateways
sudo cloud-provider-kind --enable-lb-port-mapping=false

# install east-west gateway in both clusters
task istio:install-eastwest-gateway-all

# install ingress gateway in both clusters (NodePort 30080, reachable on localhost:8080 - cloud / localhost:18080 - onprem)
task istio:install-ingress-gateway-all

# create remote secrets (cross-cluster service discovery)
task remote:create-remote-secrets-all

# delete Gateway API from both clusters
task istio:delete-gateway-api-all

# cleanup clusters
task cluster:clusters-down
```

## Scenarios

- `samples/` contains the workloads (curl, helloworld), they are not applied directly
- `scenarios/` composes the samples with everything that defines a behavior (routes, labels, policies)
- scenarios share the same namespaces, run one at a time and delete the current one before applying another

### 001 Load Balancing

Ingress gateway and ztunnel traffic is load balanced across both clusters, no locality failover is configured.

```
# deploy the scenario in both clusters
task scenarios:001-load-balancing-all

# call helloworld through the ingress gateways, traffic is load balanced across clusters because the option AMBIENT_ENABLE_MULTI_NETWORK_INGRESS is set to "true"
curl localhost:8080/hello
curl localhost:18080/hello

# verify app traffic is load balanced across both clusters
kubectl exec --context="kind-ecom-cloud" -n curl -c curl \
    "$(kubectl get pod --context="kind-ecom-cloud" -n curl -l \
    app=curl -o jsonpath='{.items[0].metadata.name}')" \
    -- curl -sS helloworld.helloworld:5000/hello
kubectl exec --context="kind-ecom-onprem" -n curl -c curl \
    "$(kubectl get pod --context="kind-ecom-onprem" -n curl -l \
    app=curl -o jsonpath='{.items[0].metadata.name}')" \
    -- curl -sS helloworld.helloworld:5000/hello

# delete the scenario from both clusters
task scenarios:001-load-balancing-delete-all
```

### 002 Locality Failover

The ingress gateway and ztunnel prefer endpoints in their own network (here network == cluster) by setting the service annotation `networking.istio.io/traffic-distribution: PreferSameZone`. 
The other cluster is only used when no local endpoint is available.
To make the answering cluster visible, `ecom-cloud` runs only `helloworld-v1` and `ecom-onprem` only `helloworld-v2`.

```
# deploy the scenario in both clusters
task scenarios:002-locality-failover-all

# traffic stays local: cloud always answers v1, onprem always answers v2
curl localhost:8080/hello
curl localhost:18080/hello
kubectl exec --context="kind-ecom-cloud" -n curl -c curl \
    "$(kubectl get pod --context="kind-ecom-cloud" -n curl -l \
    app=curl -o jsonpath='{.items[0].metadata.name}')" \
    -- curl -sS helloworld.helloworld:5000/hello
kubectl exec --context="kind-ecom-onprem" -n curl -c curl \
    "$(kubectl get pod --context="kind-ecom-onprem" -n curl -l \
    app=curl -o jsonpath='{.items[0].metadata.name}')" \
    -- curl -sS helloworld.helloworld:5000/hello

# failover: remove the local endpoint in cloud, requests are now answered by v2 from onprem
kubectl --context kind-ecom-cloud -n helloworld scale deploy helloworld-v1 --replicas=0
curl localhost:8080/hello

# failback: v1 answers again
kubectl --context kind-ecom-cloud -n helloworld scale deploy helloworld-v1 --replicas=1
curl localhost:8080/hello

# delete the scenario from both clusters
task scenarios:002-locality-failover-delete-all
```

### 003 Error Failover

Failover when the local workload returns errors, the pod stays Ready. 
The ingress gateway and the waypoint eject the failing local endpoint with outlier detection and fail over to the other cluster, see the `DestinationRule` in `scenarios/003-error-failover/base`.
The `flaky` app answers `hello from <cluster>` and returns `503` while the file `/tmp/fail` exists.
ztunnel does not read DestinationRules and has no outlier detection, that is why the `flaky` service uses a waypoint.

```
# deploy the scenario in both clusters
task scenarios:003-error-failover-all

# traffic stays local
curl localhost:8080/flaky
curl localhost:18080/flaky
kubectl exec --context="kind-ecom-cloud" -n curl -c curl \
    "$(kubectl get pod --context="kind-ecom-cloud" -n curl -l \
    app=curl -o jsonpath='{.items[0].metadata.name}')" \
    -- curl -sS flaky.flaky:80/
kubectl exec --context="kind-ecom-onprem" -n curl -c curl \
    "$(kubectl get pod --context="kind-ecom-onprem" -n curl -l \
    app=curl -o jsonpath='{.items[0].metadata.name}')" \
    -- curl -sS flaky.flaky:80/

# make the flaky pod in cloud return 503
kubectl --context kind-ecom-cloud -n flaky exec deploy/flaky -- touch /tmp/fail

# the first request can return 503 (outlier detection is reactive), afterwards cloud fails over to onprem
curl localhost:8080/flaky
kubectl exec --context="kind-ecom-cloud" -n curl -c curl \
    "$(kubectl get pod --context="kind-ecom-cloud" -n curl -l \
    app=curl -o jsonpath='{.items[0].metadata.name}')" \
    -- curl -sS flaky.flaky:80/

# heal, after the ejection time (1 minute) cloud answers again
kubectl --context kind-ecom-cloud -n flaky exec deploy/flaky -- rm /tmp/fail
curl localhost:8080/flaky

# delete the scenario from both clusters
task scenarios:003-error-failover-delete-all
```

### 004 Sharded Namespace

One application is split across both clusters, each cluster runs a subset of the workloads.
`ecom-cloud` runs `productpage`, `details` and `ratings`, `ecom-onprem` runs `reviews` (v1, v2, v3).
Calls cross the clusters in both directions: `productpage` (cloud) calls `reviews` (onprem), `reviews` (onprem) calls `ratings` (cloud).
Both clusters have the namespace and all four Services, labeled `istio.io/global: "true"`, only the workloads differ.

```
# deploy the scenario in both clusters
task scenarios:004-sharded-namespace-all

# each cluster runs only its own subset of the pods, but has all four Services
kubectl --context kind-ecom-cloud -n bookinfo get pods,svc
kubectl --context kind-ecom-onprem -n bookinfo get pods,svc

# the page shows details (cloud), reviews (onprem, v1 without, v2 black and v3 red stars) and the ratings (cloud)
# open localhost:8080/productpage in browser, refresh multiple times
# open localhost:18080/productpage in browser, refresh multiple times
# verify east-west traffic, curl multiple times and check review version changes
kubectl exec --context="kind-ecom-cloud" -n curl -c curl \
    "$(kubectl get pod --context="kind-ecom-cloud" -n curl -l \
    app=curl -o jsonpath='{.items[0].metadata.name}')" \
    -- curl -sS productpage.bookinfo:9080/productpage | grep -E "reviews-v"
kubectl exec --context="kind-ecom-onprem" -n curl -c curl \
    "$(kubectl get pod --context="kind-ecom-onprem" -n curl -l \
    app=curl -o jsonpath='{.items[0].metadata.name}')" \
    -- curl -sS productpage.bookinfo:9080/productpage | grep -E "reviews-v"

# delete the scenario from both clusters
task scenarios:004-sharded-namespace-delete-all
```

#### Experiments

Services must exist in every cluster, be global and be in the same namespace. Each experiment breaks one of these requirements.
Restore an experiment by running `task scenarios:004-sharded-namespace-all` again.
The predictions come from the Istio source and docs, the last column is filled in after running the experiment.

| # | Change | Prediction | Observed |
|---|--------|------------|----------|
| 1 | Delete the `reviews` Service in cloud (no Service in the cluster without the pods) | DNS lookup of `reviews.bookinfo` fails, the page shows "Sorry, product reviews are currently unavailable" | Confirmed, the call fails |
| 2 | Remove the label `istio.io/global` from `reviews` in cloud (consumer side) | The Service has no remote endpoints, the call fails | Confirmed, the call fails |
| 3 | Remove the label `istio.io/global` from `reviews` in onprem (provider side) | The remote Service is not used by cloud, the call fails | Confirmed, the call fails |
| 4 | Namespace `bookinfo-onprem` instead of `bookinfo` in onprem | Services are not combined, the call fails | Confirmed, the call fails |
| 5 | Rename the port of the `reviews` Service in cloud | Docs: ports with the same number must have the same name | Still works, the port name is not used when ztunnel combines the Services (the source merges ports by service port and target port) |

```
# 1 delete the Service
kubectl --context kind-ecom-cloud -n bookinfo delete svc reviews

# 2 remove the label on the consumer side
kubectl --context kind-ecom-cloud -n bookinfo label svc reviews istio.io/global-

# 3 remove the label on the provider side
kubectl --context kind-ecom-onprem -n bookinfo label svc reviews istio.io/global-

# 4 deploy ecom-onprem in the namespace bookinfo-onprem (delete the scenario first), delete it afterwards with the mismatch delete task
task scenarios:004-sharded-namespace-delete-all
task scenarios:004-sharded-namespace-namespace-mismatch-all
task scenarios:004-sharded-namespace-namespace-mismatch-delete-all

# 5 rename the port
kubectl --context kind-ecom-cloud -n bookinfo patch svc reviews --type=json -p '[{"op":"replace","path":"/spec/ports/0/name","value":"http-reviews"}]'
```

## Lessons Learned

- `cloud-provider-kind` < v0.12.0 on macOS publishes every LoadBalancer port on the host, so the second east-west gateway (same port `15008`) never gets an IP. Use v0.12.0+ and run it with `--enable-lb-port-mapping=false`; clusters reach each other over the docker network
- Known limitations are listed here, https://istio.io/latest/docs/ambient/install/multicluster/#known-limitations
- The `istio-reader-service-account` service account can be used for cross-cluster service discovery
- Services that should be accessed from other clusters must be labeled with `istio.io/global: "true"` 
- In ambient mode, istiod builds the remote network gateway from the peer's east-west Gateway's status.addresses. It does not read spec.addresses for managed gateways
- For a managed Gateway, status.addresses comes from the Service: the external addresses if any exist, otherwise the ClusterIP and hostname. A NodePort Service has no external address by default, so the ClusterIP is reported
- pilot-agent environment variable `AMBIENT_ENABLE_MULTI_NETWORK_INGRESS`, If true and AMBIENT_ENABLE_MULTI_NETWORK is also true, it will enable ingress gateways to route requests to clusters on remote networks while by default ingress gateways will keep traffic local
- Traffic entering the ingress gateway stays within the local cluster when `AMBIENT_ENABLE_MULTI_NETWORK_INGRESS` is "false" (the default). Remote endpoints are not offered to the gateway at all, so there is also no failover to the other cluster. To use remote endpoints, set the flag to "true"; to prefer local endpoints with failover, additionally configure locality failover
- Annotating the Service with `networking.istio.io/traffic-distribution: PreferSameZone` makes ztunnel and the ingress gateway prefer endpoints in their own network (here network == cluster) and use the other cluster only when local endpoints are not Ready or gone
- ztunnel is an L4 proxy: it only follows the traffic-distribution annotation, it does not apply `DestinationRule` settings (`failoverPriority`, `outlierDetection`) and has no outlier detection. Outlier detection and failover customization need a waypoint, see https://istio.io/latest/docs/ambient/install/multicluster/failover/
- Outlier detection ejects an endpoint that returns errors. Only the Envoy proxies apply the `DestinationRule`. The waypoint (for ztunnel traffic) and the ingress gateway (directly, it does not need the waypoint)
- Waypoints in multicluster: same name in every cluster, and the waypoint Service must be labeled `istio.io/global: "true"` (set via `spec.infrastructure.labels` of the Gateway), see https://istio.io/latest/docs/ambient/install/multicluster/#waypoint-requirements
- Ingress traffic ignores the service waypoint by default, routing it through the waypoint needs `istio.io/ingress-use-waypoint` and the istiod flag `ENABLE_INGRESS_WAYPOINT_ROUTING`, see https://istio.io/latest/docs/ambient/usage/waypoint/#ingress-and-waypoints
- Out-of-the-box Istio's `outlierDetection` only counts HTTP status codes and connection failures, matching on headers or body is not possible
- `maxEjectionPercent` defaults to 10%, set it to 100 when a cluster has a single endpoint, otherwise the only local endpoint is never ejected
- The generated waypoint Service gets `networking.istio.io/traffic-distribution: PreferClose` automatically ([istiod chart template](https://github.com/istio/istio/blob/release-1.30/manifests/charts/istio-control/istio-discovery/files/waypoint.yaml#L318-L329), [docs](https://istio.io/latest/docs/ambient/usage/traffic-distribution/)), so ztunnel prefers the waypoint of its own cluster
- Namespace sameness: services are combined across clusters by namespace and name, so the namespace must have the same name in every cluster. A service in a differently named namespace is never merged with the service of the other cluster ([deployment models](https://istio.io/latest/docs/ops/deployment/deployment-models/))
- A service that runs in only one cluster (sharded) still has to be a global service in every cluster: the Service must exist in all clusters, because DNS and virtual IPs are cluster-local ([verify guide](https://istio.io/latest/docs/ambient/install/multicluster/verify/)), and it needs the label `istio.io/global: "true"` on both the provider and the consumer side, otherwise the calls fail

## References
- https://istio.io/latest/docs/ops/deployment/deployment-models/
- https://istio.io/latest/docs/ambient/install/multicluster/
- https://istio.io/latest/docs/reference/commands/pilot-agent/
- https://istio.io/latest/docs/ambient/usage/traffic-distribution/
- https://istio.io/latest/docs/reference/config/networking/destination-rule/#OutlierDetection
- https://istio.io/latest/docs/ambient/install/multicluster/failover/
- https://istio.io/latest/docs/ambient/install/multicluster/#waypoint-requirements
- https://istio.io/latest/docs/ambient/usage/waypoint/
- https://istio.io/latest/docs/examples/bookinfo/
- https://istio.io/latest/docs/ambient/install/multicluster/verify/