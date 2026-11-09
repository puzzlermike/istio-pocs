# Istio Ambient Multi-Cluster Mesh

Istio Ambient Multi-Cluster mesh PoC using two local Kind clusters.<br />
Mesh properties:
- [Multi-primary](https://istio.io/latest/docs/ops/deployment/deployment-models/#control-plane-models)
- [Multi-network](https://istio.io/latest/docs/ops/deployment/deployment-models/#multiple-networks)
- Single [Trust-domain](https://istio.io/latest/docs/ops/deployment/deployment-models/#trust-within-a-mesh)

The PoC consists of two clusters, `ecom-cloud` and `ecom-onprem`, each with their own Istio control plane, communication between the clusters happens via dedicated east-west gateways.

## Local Development

```
# run cloud and onprem cluster
task cluster:clusters-up

# install Gateway API into both clusters
task istio:install-gateway-api-all

# delete Gateway API from both clusters
task istio:delete-gateway-api-all

# cleanup clusters
task cluster:clusters-down
```

# Lessons Learned

- Known limitations are listed here, https://istio.io/latest/docs/ambient/install/multicluster/#known-limitations

# References
- https://istio.io/latest/docs/ops/deployment/deployment-models/
- https://istio.io/latest/docs/ambient/install/multicluster/