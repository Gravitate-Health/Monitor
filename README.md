# Monitor

Everything is deployed in namespace "monitoring".

A `clusterRole` for the `monitoring` namespace with `get, list and watch` permissions is needed for prometheus to retrieve metrics from other objects. The role is binded to the namespace. See [clusterRole.yaml](./clusterRole.yaml)

## Prometheus

Config is externalized in a configMap (so images does not need to be rebuilt). The deployment mounts the configMap inside `/etc/prometheus` as a volume.


## Grafana

The Prometheus datasource is configured in the [Grafana configMap](./grafana/grafana-datasource-config.yaml)


## TODO 
- [Setting Up Alert Manager on Kubernetes](https://devopscube.com/alert-manager-kubernetes-guide/)
- [Setup Prometheus Node Exporter on Kubernetes](https://devopscube.com/node-exporter-kubernetes/)

### Useful Dashboard templates

- [Kubernetes Deployment Statefulset Daemonset metrics](https://grafana.com/grafana/dashboards/8588)
- [Kubernetes cluster monitoring (via Prometheus)](https://grafana.com/grafana/dashboards/315)


## External documentation and resources

- [Setup Prometheus monitoring on kubernetes](https://devopscube.com/setup-prometheus-monitoring-on-kubernetes/)
- [Setup Grafana on kubernetes](https://devopscube.com/setup-grafana-kubernetes/)
---
Thanks [bibinwilson](https://github.com/bibinwilson) for this YAMLs
- [Prometheus Kubernetes prepared yamls](https://github.com/bibinwilson/kubernetes-prometheus)
- [Grafana Kubernetes prepared yamls](https://github.com/bibinwilson/kubernetes-grafana)
---
- [Kubectl cheatsheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
