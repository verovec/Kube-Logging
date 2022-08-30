![](https://img.shields.io/badge/Kubernetes%20>=-1.22.7-blue)
![](https://img.shields.io/badge/Helm-3-9cf)
![](https://img.shields.io/badge/Elastic%20Search-7.17.5-yellow)
![](https://img.shields.io/badge/Fluentd-1.4.2-%2366A5CE)
![](https://img.shields.io/badge/Kibana-7.17.5-%23E44A91)

# Kube-Logging
Logger Helm chart for Kubernetes cluster working with Elastic Search - Fluentd - Kibana

## Prerequisite
For this chart you must have Kubernetes cluster with version equal or higher to 1.22.7

Helm must be at least version 3

# How To
## Customize the chart
You need to set the storageClass that you have in your cluster for Elastic Search data persistence.

By default the ingress is disabled, the correct way is to use [Port Forward](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/) to acces your Kibana dashboard. If you want to enable it, edit *./values.yaml*, set ingress value to true with your ingressClassName and your host.
## Deploy
For this section we assume you are locally connected to your cluster.

```bash
helm install -f values.yaml kube-logging . --namespace kube-logging --create-namespace
```
