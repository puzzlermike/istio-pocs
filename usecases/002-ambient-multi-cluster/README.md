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

# install east-west gateway in both clusters
task istio:install-eastwest-gateway-all

# create remote secrets (cross-cluster service discovery)
task remote:create-remote-secrets-all

# delete Gateway API from both clusters
task istio:delete-gateway-api-all

# cleanup clusters
task cluster:clusters-down
```

## Samples

### Hello World

```
# deploy helloworld app in both clusters
task samples:helloworld-all

# deploy curl app in both clusters
task samples:curl-all

# verify app traffic is load balanced across both clusters
kubectl exec --context="kind-ecom-cloud" -n curl -c curl \
    "$(kubectl get pod --context="kind-ecom-cloud" -n curl -l \
    app=curl -o jsonpath='{.items[0].metadata.name}')" \
    -- curl -sS helloworld.helloworld:5000/hello
```

## Lessons Learned

- Known limitations are listed here, https://istio.io/latest/docs/ambient/install/multicluster/#known-limitations
- The `istio-reader-service-account` service account can be used for cross-cluster service discovery
- Services that should be accessed from other clusters must be labeled with `istio.io/global: "true"` 
- In ambient mode, istiod builds the remote network gateway from the peer's east-west Gateway's status.addresses. It does not read spec.addresses for managed gateways
- For a managed Gateway, status.addresses comes from the Service: the external addresses if any exist, otherwise the ClusterIP and hostname. A NodePort Service has no external address by default, so the ClusterIP is reported

## References
- https://istio.io/latest/docs/ops/deployment/deployment-models/
- https://istio.io/latest/docs/ambient/install/multicluster/