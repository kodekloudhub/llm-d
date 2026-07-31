# Scraping EPP metrics with Prometheus

The EPP (Endpoint Picker / router) exposes the `llm_d_epp_*` metric family
(`llm_d_epp_ready_endpoints`, `llm_d_epp_request_total`,
`llm_d_epp_flow_control_queue_size`, and more — see
[`docs/operations/observability/metrics.md`](../../../../docs/operations/observability/metrics.md))
on its `http-metrics` port (9090) at `/metrics`.

If a PromQL query such as `llm_d_epp_ready_endpoints` returns **no data** even
though the EPP pod is `Running`, the cause is almost always one (or both) of:

1. **No ServiceMonitor targets the EPP.** Prometheus never scrapes port 9090, so
   the entire `llm_d_epp_*` family is absent — not just one metric. Confirm by
   querying `llm_d_epp_request_total`; if that is also empty, the EPP is not
   being scraped at all.
2. **The EPP `/metrics` endpoint is auth-gated and scrapes fail.** The endpoint
   is protected by controller-runtime's authn/authz filter, which validates the
   scrape's bearer token with a Kubernetes `TokenReview` and authorizes it with a
   `SubjectAccessReview`. Unauthenticated scrapes get `HTTP 401`; if the EPP
   ServiceAccount lacks permission to create `TokenReviews`, even a
   token-bearing scrape gets `HTTP 500 "Authentication failed"` and the EPP logs:

   ```
   tokenreviews.authentication.k8s.io is forbidden: User
   "system:serviceaccount:llm-d:quickstart-epp" cannot create resource "tokenreviews"
   ```

This recipe fixes both.

## What it creates

| File | Purpose |
| ---- | ------- |
| `epp-metrics-rbac.yaml` | Binds `system:auth-delegator` to the EPP ServiceAccount (so it can validate scrape tokens), and creates a read-only `quickstart-epp-metrics-reader` ServiceAccount + long-lived token + `/metrics` GET permission for Prometheus to authenticate with. |
| `epp-servicemonitor.yaml` | A `ServiceMonitor` that scrapes the EPP `http-metrics` port over HTTP, presenting the token above. |
| `kustomization.yaml` | Applies both together. |

## Prerequisites

- A Prometheus Operator install (e.g. `kube-prometheus-stack`) whose
  `serviceMonitorSelector` / `serviceMonitorNamespaceSelector` select this
  ServiceMonitor. The manifests carry `release: llmd`; adjust to match your
  release, or the selector your Prometheus uses.
- The EPP deployed as the `quickstart` release in the `llm-d` namespace. For
  other release names/namespaces, replace the `quickstart-epp` prefix and
  `llm-d` namespace throughout both manifests.

## Apply

```bash
kubectl apply -k guides/recipes/observability/epp-metrics/
# or, without kustomize:
kubectl apply -f guides/recipes/observability/epp-metrics/epp-metrics-rbac.yaml
kubectl apply -f guides/recipes/observability/epp-metrics/epp-servicemonitor.yaml
```

## Verify

```bash
# 1. Token is populated:
kubectl -n llm-d get secret quickstart-epp-metrics-token -o jsonpath='{.data.token}' | base64 -d | head -c 20; echo

# 2. Authenticated scrape now returns 200 and the metric:
TOKEN=$(kubectl -n llm-d get secret quickstart-epp-metrics-token -o jsonpath='{.data.token}' | base64 -d)
kubectl -n llm-d port-forward svc/quickstart-epp 19090:9090 &
curl -s -H "Authorization: Bearer $TOKEN" http://localhost:19090/metrics | grep llm_d_epp_ready_endpoints

# 3. Prometheus lists the target as "up" (allow ~1 scrape interval):
#    Prometheus UI -> Status -> Targets -> serviceMonitor/llm-d/quickstart-epp
```

## Helm-native alternative

If you deploy the router via its Helm chart, you can have the chart manage the
ServiceMonitor instead of applying this recipe — see
[`guides/recipes/router/features/monitoring.values.yaml`](../../router/features/monitoring.values.yaml)
(`router.monitoring.prometheus.enabled: true`). This recipe is for existing
deployments where re-running Helm is inconvenient.
