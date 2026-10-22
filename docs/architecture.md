# Architecture

One Helm chart, three components, nine templates. Everything is created in the release
namespace (`{{ .Release.Namespace }}`), except the two cluster-wide RBAC objects of Fluentd.

## Layout

```
Chart.yaml                   name kube-logging, version 2022.06.25, maintainers Clement and Flavien
values.yaml                  the eight tunable values (ingress, storage, Elasticsearch resources)
templates/
  elasticsearch/
    statefulset.yaml         es-cluster, 3 replicas, init containers, PVC template on the storage class
    service.yaml             elasticsearch, headless (clusterIP None), ports 9200 rest and 9300 inter-node
  fluentd/
    serviceaccount.yaml      fluentd
    clusterrole.yaml         get, list, watch on pods and namespaces
    clusterrolebinding.yaml  binds the role to the service account of the release namespace
    daemonset.yaml           fluentd, one pod per node, hostPath volumes, Elasticsearch output
  kibana/
    deployment.yaml          kibana, 1 replica, ELASTICSEARCH_URL=http://elasticsearch:9200
    service.yaml             kibana, ClusterIP, port 5601
    ingress.yaml             rendered only when kibana.ingress.enabled
docs/                        this documentation and the logo
```

## Log path

```
  kubelet       writes container stdout/stderr under /var/log/containers (symlinks) and /var/lib/docker/containers
     |
  Fluentd       hostPath mounts of both directories (the second read-only), tails and parses them,
                enriches with pod metadata through the Kubernetes API (hence the ClusterRole)
     |
  Elasticsearch FLUENT_ELASTICSEARCH_HOST elasticsearch.<ns>.svc.cluster.local, port 9200, scheme http
     |
  Kibana        queries http://elasticsearch:9200, serves 5601
```

Fluentd targets the Docker log directory, which matches clusters running the Docker engine (as
ToDevOps does with Docker 19.03).

## Templates

### Elasticsearch

| Field | Value |
|-------|-------|
| StatefulSet | `es-cluster`, `serviceName: elasticsearch`, `replicas: 3`, label `app: elasticsearch` |
| Image | `docker.elastic.co/elasticsearch/elasticsearch:7.2.0` |
| Environment | `cluster.name=k8s-logs`, `node.name` from the pod name, `discovery.seed_hosts` and `cluster.initial_master_nodes` listing `es-cluster-0..2`, `ES_JAVA_OPTS=-Xms512m -Xmx512m` |
| Ports | 9200 `rest`, 9300 `inter-node` |
| Resources | from `elasticsearch.resources` (limits 2Gi / 1000m, requests 1Gi / 500m) |
| Init containers | `fix-permissions` (`chown -R 1000:1000` on the data dir), `increase-vm-max-map` (`sysctl -w vm.max_map_count=262144`), `increase-fd-ulimit` (`ulimit -n 65536`), all `busybox`, all privileged |
| Volume | PVC template `data`, `ReadWriteOnce`, `elasticsearch.storage.size`, `elasticsearch.storage.storageClassName`, mounted at `/usr/share/elasticsearch/data` |
| Service | `elasticsearch`, headless, ports 9200 and 9300, selector `app: elasticsearch` |

The replica count and the discovery lists are written together: change one, change the other.

### Fluentd

| Field | Value |
|-------|-------|
| DaemonSet | `fluentd`, label `app: fluentd`, `serviceAccountName: fluentd` |
| Toleration | `node-role.kubernetes.io/master: NoSchedule`, so logs of the control plane are collected too |
| Image | `fluent/fluentd-kubernetes-daemonset:v1.4.2-debian-elasticsearch-1.1` |
| Environment | `FLUENT_ELASTICSEARCH_HOST=elasticsearch.<namespace>.svc.cluster.local`, `FLUENT_ELASTICSEARCH_PORT=9200`, `FLUENT_ELASTICSEARCH_SCHEME=http`, `FLUENTD_SYSTEMD_CONF=disable` |
| Resources | limits 200m / 512Mi, requests 100m / 100Mi |
| Volumes | `/var/log` and `/var/lib/docker/containers` (read-only) from the host |
| RBAC | ServiceAccount `fluentd`; ClusterRole `fluentd` with `get`, `list`, `watch` on `pods` and `namespaces`; ClusterRoleBinding to the account in the release namespace |
| Termination | `terminationGracePeriodSeconds: 30` |

### Kibana

| Field | Value |
|-------|-------|
| Deployment | `kibana`, 1 replica, label `app: kibana` |
| Image | `docker.elastic.co/kibana/kibana:7.2.0` |
| Environment | `ELASTICSEARCH_URL=http://elasticsearch:9200` |
| Resources | limits 500m / 500Mi, requests 100m / 100Mi |
| Service | `kibana`, `ClusterIP`, port 5601, selector `app: kibana` |
| Ingress | `kibana-ingress`, `networking.k8s.io/v1`, `ingressClassName` and `host` from values, path `/` `Prefix` to `kibana:5601`; rendered only when `kibana.ingress.enabled` |

## Values

| Value | Default | Where it is used |
|-------|---------|------------------|
| `fullnameOverride` | `kube-logging` | declared in `values.yaml`, not referenced by any template |
| `kibana.ingress.enabled` | `false` | `templates/kibana/ingress.yaml` guard |
| `kibana.ingress.host` | `""` | Ingress rule host |
| `kibana.ingress.ingressClassName` | `""` | Ingress `spec.ingressClassName` |
| `elasticsearch.storage.size` | `8Gi` | PVC template request |
| `elasticsearch.storage.storageClassName` | `""` | PVC template storage class; empty means the cluster default |
| `elasticsearch.resources.limits.memory` | `2Gi` | Elasticsearch container limit |
| `elasticsearch.resources.limits.cpu` | `1000m` | Elasticsearch container limit |
| `elasticsearch.resources.requests.memory` | `1Gi` | Elasticsearch container request |
| `elasticsearch.resources.requests.cpu` | `500m` | Elasticsearch container request |

Fixed in the templates, not in `values.yaml`: replica counts (3, 1, one per node), image tags,
the JVM heap, the Fluentd and Kibana resources, the ports.

## Recipes

```sh
helm lint .                                                                     # check the chart
helm template kube-logging . -f values.yaml --namespace kube-logging | less     # see the rendered objects
helm install -f values.yaml kube-logging . --namespace kube-logging --create-namespace   # install
helm install kube-logging . -n kube-logging --create-namespace --set elasticsearch.storage.storageClassName=openebs-hostpath   # same, class on the command line
helm upgrade -f values.yaml kube-logging . --namespace kube-logging             # apply a values change
kubectl -n kube-logging get pods -w                                             # wait for es-cluster-0..2, kibana, fluentd-*
kubectl -n kube-logging port-forward svc/kibana 5601:5601                       # http://localhost:5601
helm uninstall kube-logging -n kube-logging                                     # remove the release
kubectl -n kube-logging delete pvc -l app=elasticsearch                         # drop the data volumes too
```

ToDevOps installs it with `--set kibana.host=<domain>`, a value of an older version of the
chart; on this version the ingress values are under `kibana.ingress`.

## Conventions

- Templates are plain Kubernetes manifests with `{{ .Values... }}` substitutions, no helpers.
- Every namespaced object uses `{{ .Release.Namespace }}`; install with `--namespace`.
- Elastic images move together: Elasticsearch and Kibana must share the same version.
