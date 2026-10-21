# Minimal Gravitee APIM for small Kubernetes clusters

A minimal, single-replica Gravitee API Management install for dev clusters, built on the
official [`apim` Helm chart](helm/) (v4.13.0, vendored from
[gravitee-api-management](https://github.com/gravitee-io/gravitee-api-management) with its
Bitnami MongoDB 16.5.45 and Elasticsearch 22.0.13 subcharts locked in `helm/charts/`).

| Web UI / endpoint | Service | NodePort | Internal port |
| --- | --- | --- | --- |
| Console UI (management) | `apim-ui` | **30081** | 8080 |
| API Gateway http | `apim-gateway` | **30082** | 8082 |
| Management REST API (`/management`, `/portal`) | `apim-api` | **30083** | 8083 |

Everything uses the `local-path` StorageClass (MongoDB 1Gi, Elasticsearch master 2Gi +
data 3Gi). Excluded on purpose: developer portal UI, Alert Engine, Gamma, Cockpit,
Kafka console, autoscaling, and all Ingress objects — NodePort only.

Resource footprint per install: **~1.5 GiB of requests** / ~3.3 GiB of limits —
sized for small clusters (e.g. 12 cores / 7 GiB) that already run other workloads.

## One edit before deploying

The console is a static UI rendered in your **browser**, so it must be pointed at a
node-reachable Management API URL. In `values-dev-minimal-nodeport.yaml`, in the `ui`
section, replace `CHANGE_ME_NODE_IP` with any node IP (`kubectl get nodes -o wide`):

```yaml
ui:
  baseURL: http://CHANGE_ME_NODE_IP:30083/management
  env:
    - name: MGMT_API_URL
      value: http://CHANGE_ME_NODE_IP:30083/management/
```

Both values must stay consistent; the trailing `/management` is required.

## Install (helm CLI)

```bash
helm upgrade --install apim ./helm -n apim-dev --create-namespace \
  -f values-dev-minimal-nodeport.yaml
```

Watch it come up (`kubectl -n apim-dev get pods`), then open
`http://<node-ip>:30081` and log in with **admin / admin**.

## Install (Fleet / GitOps)

`fleet.yaml` at the repo root is ready for Rancher Fleet: point a GitRepo at this repo
(Fleet picks up bundles in Git) targeting your dev cluster; it deploys the vendored chart
with `values-dev-minimal-nodeport.yaml` into namespace `apim-dev`.

## Notes

- Mongo runs as a **single-node replica set** (`rs0`) — the chart's defaults wire the
  connection string to `graviteeio-apim-mongodb-replicaset-headless`.
- The Elasticsearch **data node carries the `ingest` role** (`data.extraRoles`) because
  APIM's analytics pipeline (geoip/user_agent) requires an ingest-capable node; analytics
  target the `graviteeio-apim-elasticsearch-data-hl` service.
- `global.security.allowInsecureImages: true` is required because the chart pins frozen
  Bitnami `bitnamilegacy/*` images (post-catalog-migration), which the Bitnami subcharts'
  image verification otherwise rejects.
- Dev-only defaults to change for anything beyond dev: the default `admin / admin`
  in-memory admin, the chart's default JWT secret, and no auth on MongoDB/Elasticsearch.
- Image versions: the APIM chart at 4.13.0 (from the gravitee-api-management repo) is
  ahead of published images — only `4.13.0-alpha.*` exists, so the values pin the latest
  GA images: `graviteeio/apim-{management-api,gateway}:4.12.23-debian`,
  `graviteeio/apim-management-ui:4.12.23`, and Bitnami legacy build tags
  (`bitnamilegacy/mongodb:6.0.13-debian-11-r21` — the plain `6.0.13` tag does not exist
  in the legacy catalog). Once 4.13.0 GA images publish, the `image.tag` overrides can
  be dropped and the chart version bumped.
- To add the developer portal back, set `portal.enabled: true` (and expose it on a
  NodePort the same way).