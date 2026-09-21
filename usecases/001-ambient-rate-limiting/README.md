# Ambient Rate Limiting

A PoC for Istio Ingress Gateway rate limiting using Ambient mode.<br />
**Note: Global rate limiting was not PoCed!**

## Setup

```
# create local Kind cluster
kind create cluster --name=istio-pocs-ambient-rate-limiting --image=kindest/node:v1.36.4

# prepare istioctl
curl -O -L https://github.com/istio/istio/releases/download/1.31.0/istioctl-1.31.0-osx-arm64.tar.gz
tar -xvf istioctl-1.31.0-osx-arm64.tar.gz
chmod +x istioctl

# deploy Istio
kubectl apply --server-side -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.6.2/experimental-install.yaml
./istioctl install --set profile=ambient --skip-confirmation

# run Cloud Provider KIND
go install sigs.k8s.io/cloud-provider-kind@latest
sudo cloud-provider-kind

# create Gateway
kubectl create ns ingress
kubectl apply -f gateway.yaml

# deploy echo service
kubectl create ns echo
kubectl apply -f deployment.yaml
kubectl apply -f route.yaml
echo "127.0.0.1 echo.istio-pocs-00" | sudo tee -a /etc/hosts

# rate limiting without tenancy
kubectl apply -f filter-no-tenancy.yaml
for i in {1..5}; do curl -s "http://echo.istio-pocs-001" -o /dev/null -w "%{http_code}\n"; sleep 1; done
kubectl delete -f filter-no-tenancy.yaml

# rate limiting with static tenancy
kubectl apply -f filter-static-tenancy.yaml
for i in {1..5}; do curl -s "http://echo.istio-pocs-001" -H "x-tenant: tenant-a" -o /dev/null -w "%{http_code}\n"; sleep 1; done
for i in {1..5}; do curl -s "http://echo.istio-pocs-001" -H "x-tenant: tenant-b" -o /dev/null -w "%{http_code}\n"; sleep 1; done
kubectl delete -f filter-static-tenancy.yaml

# rate limiting with dynamic tenancy
kubectl apply -f filter-dynamic-tenancy.yaml
for i in {1..5}; do curl -s "http://echo.istio-pocs-001" -H "x-tenant: foo" -o /dev/null -w "%{http_code}\n"; sleep 1; done
for i in {1..5}; do curl -s "http://echo.istio-pocs-001" -H "x-tenant: bar" -o /dev/null -w "%{http_code}\n"; sleep 1; done
kubectl delete -f filter-dynamic-tenancy.yaml

# rate limiting based on token claims
kubectl create ns keycloak
kubectl create ns curl
kubectl apply -f https://raw.githubusercontent.com/keycloak/keycloak-quickstarts/refs/heads/main/kubernetes/keycloak.yaml -n keycloak
kubectl apply -f request-auth.yaml
kubectl apply -f curl.yaml
# port-forward Keycloak and create two clients (client credentials grant type)
export AT=$(kubectl exec -it -n curl <pod> -- curl http://keycloak.keycloak.svc.cluster.local:8080/realms/master/protocol/openid-connect/token \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -d 'grant_type=client_credentials' \
  -d 'client_id=tenant-a' \
  -d 'client_secret=<secret>' | jq -r .access_token)
kubectl apply -f filter-jwt-dynamic-tenancy.yaml
for i in {1..5}; do curl -s "http://echo.istio-pocs-001" -H "Authorization: Bearer $AT" -o /dev/null -w "%{http_code}\n"; sleep 1; done
kubectl delete -f filter-jwt-dynamic-tenancy.yaml
```

## Lessons Learned

- Rate limiting is not supported for Waypoint proxies, see https://istio.io/latest/docs/ambient/migrate/#what-is-not-supported
- Local rate limit state is persisted per Gateway Pod
- Rate limit per tenant is possible (e.g., by using HTTP headers)
- Rate limit token bucket scoped to OAuth 2.0/OIDC token claims is possible; EnvoyFilter leverages provided HTTP header value configured by the RequestAuthentication CR
- EnvoyFilter must target Istio ingress gateway (e.g. via labels)
- EnvoyFilter can be scoped to a particular route
- Tenants must NOT be statically listed in EnvoyFilter, dynamic tenant resolution is possible
- Default rate limit buckets can be set; by default, they are always used, opt-out possible via `always_consume_default_token_bucket` false
- EnvoyFilter must be placed into the same namespace as the Ingress Gateway