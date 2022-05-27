# Monitor

Everything is deployed in namespace "monitoring".

A `clusterRole` for the `monitoring` namespace with `get, list and watch` permissions is needed for prometheus to retrieve metrics from other objects. The role is binded to the namespace. See [clusterRole.yaml](./clusterRole.yaml)

## Prometheus

Config is externalized in a configMap (so images does not need to be rebuilt). The deployment mounts the configMap inside `/etc/prometheus` as a volume.

### Prometheus behind a reverse proxy

Prometheus will be accesed through a reverse proxy with a prefix. The root URL will be something like `{BASE_URL}/prometheus/`. Reverse shouldn't rewrite the url. The prefix should be passed to prometheus.

Prometheus needs to be executed with the following arguments to work behind reverse proxy with a prefix `--web.external-url=https://{BASE_URL}/prometheus/`:

```bash
{COMMAND_TO_EXECUTE_PROMETHEUS} --web.external-url=https://{BASE_URL}/prometheus/
```

## Grafana

- The Prometheus datasource is configured in the [Grafana configMap](./grafana/grafana-datasource-config.yaml).
- To enable grafana behind a reverse proxy with a subpath (i.e a prefix), you must include the subpath at the end of *root_url*, such as `https://gravitate-health.lst.tfo.upm.es:3000/`**`grafana/`**

### Grafana behind a reverse proxy

Grafana will be accesed through a reverse proxy with a prefix. The root URL will be something like `{BASE_URL}/grafana/`. Reverse proxy must rewrite the url and remove the prefix when passing the petition to grafana.

Grafana needs this additional configuration in the `grafana.ini` file to work behind a reverse proxy.

```ini
[server]
    root_url = https://{root_url}/grafana/
```

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
