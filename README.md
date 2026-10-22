<p align="center">
  <img src="docs/assets/logo.svg" width="120" height="120" alt="Kube-Logging logo">
</p>

<h1 align="center">Kube-Logging</h1>

<p align="center">
  <strong>Every container log of the cluster, searchable in Kibana, with one Helm install.</strong><br>
  A small chart that deploys Elasticsearch, a Fluentd DaemonSet and Kibana (the EFK stack) in a
  namespace of your choice, with persistence on the storage class you name.
</p>

<p align="center">
  <a href="#quick-start">Quick start</a> &middot;
  <a href="#how-it-works">How it works</a> &middot;
  <a href="#values">Values</a> &middot;
  <a href="#tech-stack">Tech stack</a> &middot;
  <a href="#documentation">Documentation</a>
</p>

<p align="center">
  <img alt="Kubernetes 1.22.7 or later" src="https://img.shields.io/badge/Kubernetes-%3E%3D%201.22.7-326CE5?logo=kubernetes&amp;logoColor=white">
  <img alt="Helm 3" src="https://img.shields.io/badge/Helm-3-0F1689?logo=helm&amp;logoColor=white">
  <img alt="Elasticsearch 7.2.0" src="https://img.shields.io/badge/Elasticsearch-7.2.0-005571?logo=elasticsearch&amp;logoColor=white">
  <img alt="Fluentd 1.4.2" src="https://img.shields.io/badge/Fluentd-1.4.2-0E83C8?logo=fluentd&amp;logoColor=white">
  <img alt="Kibana 7.2.0" src="https://img.shields.io/badge/Kibana-7.2.0-E8478B?logo=kibana&amp;logoColor=white">
</p>

---

Pods come and go, and so do their logs: `kubectl logs` only shows what is still running. A
cluster needs a place where the output of every container is collected, indexed and searchable
after the pod is gone.

**Kube-Logging** packages the usual answer as a Helm chart. A Fluentd DaemonSet runs on every
node (the control plane included, thanks to a toleration), reads `/var/log` and the Docker
container logs, and forwards them to a three-node Elasticsearch StatefulSet. Kibana reads that
cluster and gives the search interface. The only value you must set is the storage class used
for the Elasticsearch volumes; an Ingress for Kibana is optional and disabled by default.

The chart was written in 2022 at ETNA with Flavien Berwick for the logging layer of
[ToDevOps](https://github.com/verovec/ToDevOps), where an Ansible role installs it. The pinned
images are those of that time (Elastic 7.2.0, Fluentd 1.4.2) and the repository is kept as is.

## Quick start

**Prerequisites:** a Kubernetes cluster 1.22.7 or later, Helm 3, `kubectl` connected to the
cluster, and a storage class able to provision `ReadWriteOnce` volumes.

```bash
git clone https://github.com/verovec/Kube-Logging.git && cd Kube-Logging
vi values.yaml        # elasticsearch.storage.storageClassName: <your storage class>
helm install -f values.yaml kube-logging . --namespace kube-logging --create-namespace
kubectl -n kube-logging get pods -w
```

When `es-cluster-0..2`, `kibana` and the `fluentd` pods are running, open Kibana with
`kubectl -n kube-logging port-forward svc/kibana 5601:5601` at `http://localhost:5601` and create
an index pattern on `logstash-*`. Enabling `kibana.ingress` exposes it on a host instead; the
port-forward is the recommended way.

## How it works

```
  Containers    stdout and stderr written by the kubelet under /var/log and /var/lib/docker/containers
     |
  Fluentd       DaemonSet, one pod per node, hostPath mounts, fluentd-kubernetes-daemonset image
     |          FLUENT_ELASTICSEARCH_HOST=elasticsearch.<namespace>.svc.cluster.local:9200 (http)
     |
  Elastic       StatefulSet es-cluster, 3 replicas, headless Service elasticsearch (9200 rest, 9300 inter-node)
  search        cluster k8s-logs, one PVC per pod on your storage class, 512 MB JVM heap
     |
  Kibana        Deployment, ELASTICSEARCH_URL=http://elasticsearch:9200, Service kibana :5601
     |
  Ingress       optional, class and host from values.yaml, path / to kibana:5601
```

Three init containers prepare each Elasticsearch pod: ownership of the data directory,
`vm.max_map_count=262144` and the file descriptor limit. They run privileged, which your
cluster policy must allow.

## Values

| Value | Default | Role |
|-------|---------|------|
| `fullnameOverride` | `kube-logging` | Release name override declared in `values.yaml` (not read by the templates) |
| `kibana.ingress.enabled` | `false` | Render the Kibana Ingress |
| `kibana.ingress.host` | `""` | Host of the Ingress rule |
| `kibana.ingress.ingressClassName` | `""` | `ingressClassName` of the Ingress |
| `elasticsearch.storage.size` | `8Gi` | Size of each `data` volume claim |
| `elasticsearch.storage.storageClassName` | `""` | Storage class of the claims, to set for your cluster |
| `elasticsearch.resources.limits.memory` / `cpu` | `2Gi` / `1000m` | Limits of the Elasticsearch container |
| `elasticsearch.resources.requests.memory` / `cpu` | `1Gi` / `500m` | Requests of the Elasticsearch container |

Replica counts, image tags and the Fluentd and Kibana resources are fixed in the templates; see
[`docs/architecture.md`](docs/architecture.md#templates).

## Tech stack

| Layer | Tech | Location |
|-------|------|----------|
| Packaging | Helm 3 chart, `apiVersion: v1`, version `2022.06.25` | `Chart.yaml`, `values.yaml` |
| Collection | Fluentd 1.4.2 DaemonSet with ServiceAccount, ClusterRole and binding | `templates/fluentd/` |
| Storage and index | Elasticsearch 7.2.0 StatefulSet and headless Service | `templates/elasticsearch/` |
| Interface | Kibana 7.2.0 Deployment, Service and optional Ingress | `templates/kibana/` |

Details in [Architecture](docs/architecture.md).

## Documentation

| Topic | Where |
|-------|-------|
| Templates, every object and value, the log path, recipes | [`docs/architecture.md`](docs/architecture.md) |
| Everything else | [`docs/`](docs/README.md) |

## Contributing

The chart is kept as it was delivered in 2022. Issues are still welcome; a pull request that
bumps the Elastic images should keep the three-node discovery settings in the StatefulSet in
step.
