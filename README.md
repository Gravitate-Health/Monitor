# 1. Gravitate-Health Monitor system

1.1. Table of contents
-----------------

- [1. Gravitate-Health Monitor system](#1-gravitate-health-monitor-system)
  - [1.1. Table of contents](#11-table-of-contents)
  - [1.2. Introduction](#12-introduction)
  - [1.3. Kubernetes Deployment](#13-kubernetes-deployment)
    - [1.3.1. Prerequisites](#131-prerequisites)
    - [1.3.2. Prepare the environment](#132-prepare-the-environment)
    - [1.3.3. Prometheus](#133-prometheus)
    - [1.3.4. Grafana](#134-grafana)
  - [1.4. Usage](#14-usage)
    - [1.4.1. Dashboards](#141-dashboards)
  - [1.5. Known issues and limitations](#15-known-issues-and-limitations)
  - [1.6. Getting help](#16-getting-help)
  - [1.7. Contributing](#17-contributing)
  - [1.8. License](#18-license)
  - [1.9. Authors and history](#19-authors-and-history)
  - [1.10. Acknowledgments](#110-acknowledgments)
  - [- Setup Prometheus Node Exporter on Kubernetes](#--setup-prometheus-node-exporter-on-kubernetes)
  - [- Grafana Kubernetes prepared yamls](#--grafana-kubernetes-prepared-yamls)


1.2. Introduction
------------

This repository contains the configuration and deployment files necessary to monitor a kubernetes cluster and deployments on top of the cluster, such as nodeJS apps, Mongo databases, Keycloak server, etc. The monitor system consists of a Grafana + Prometheus stack.

This readme will help the reader to deploy the system to a kubernetes cluster, but also to understand the configuration and be able to edit/expand it.

![Monitor stack architecture](./docs/prometheus-grafana-stack.png "Monitor stack architecture")

1.3. Kubernetes Deployment
------------

Grafana and Prometheus offer their official Docker image which is ready to deploy and work for a local environment. For a k8s cluster, some considerations must be taken into account.

### 1.3.1. Prerequisites

The only prerequesites are a Kubernetes cluster and a gateway/reverse-proxy configured and with a working external url(domain name). The externally accsible url for the gateway will be referenced as `BASE_URL` from now on.

This gateway must recirect petitions with prefix `/grafana/` to grafana (removing the prefix), and petitions with prefix `/prometheus/` to prometheus (NOT removing the prefix):

- `{BASE_URL}/grafana/foo` will be redirected to grafana as `{grafana-url}/foo`
- `{BASE_URL}/prometheus/foo` will be redirected to prometheus as `{prometheus-url}/prometheus/foo`

Currently, the yaml files configure Prometheus to be accessible through a reverse proxy, and not through Kubectl port forwarding or an ingress object. To know how to do it, refer to [official kubernetes documentation](https://kubernetes.io/es/docs/home/)

### 1.3.2. Prepare the environment


For Prometheus to be able to scrape information about the cluster or pods within other namespaces, the following steps must be taken:

1. Create the `monitoring` namespace:

```bash
kubectl create namespace monitoring
```

**NOTE: From now on, be sure to deploy everything on the `monitoring` namespace. All yamls files have the `namespace` directive, so if you wish to use anothe namespace, change it from the files.**

2. Deploy the [cluster role](./clusterRole.yaml), which contains a RBAC role that enables Prometheus to get and list nodes, services, endpoints and pods from other namespaces.
   
```bash
kubectl create -f clusterRole.yaml
```

### 1.3.3. Prometheus 

Prometheus configs are externalized to config-maps to avoid needing to build the prometheus image when changing the config. To apply config changes, it is only needed to udpate config maps and restart prometheus pods to apply the new configuration.

Prometheus bases its configuration on a file named `prometheus.yml`, typically located at `/etc/prometheus/prometheus.yml`.

Steps to deploy:

1. Write your own configuration (discovery config and alert rules) to [prometheus-config-map.yaml](prometheus/prometheus-config-map.yaml)

2. Create the following resources:

```bash
kubectl create -f prometheus/prometheus-config-map.yaml
kubectl create -f prometheus/prometheus-service.yaml
kubectl create -f prometheus/prometheus-deployment.yaml
```

After these steps, prometheus web GUI will be accessible through `{BASE_URL}/prometheus/`.

NOTE: To understand prometheus config that enables it to work behind a reverse proxy, take a look at the `--web.external-url` arg for the container specified at the [prometheus deployment](prometheus/prometheus-deployment.yaml)


### 1.3.4. Grafana

As happens with prometheus, grafana's configurations are also externailzed to yaml files. Grafana main configuration file is `grafana.ini` typically placed at `etc/grafana/grafana.ini`.

Steps to deploy:

1. Define the datasources at [grafana-datasource-config.yaml](grafana/grafana-datasource-config.yaml).
2. Configure grafana through [grafana-config-map.yaml](grafana/grafana-config-map.yaml).
3. Create the following resources:

```bash
kubectl create -f grafana/grafana-config-map.yaml
kubectl create -f grafana/grafana-datasource-config.yaml
kubectl create -f grafana/grafana-service.yaml
kubectl create -f grafana/grafana-deployment.yaml
```

After these steps, grafana web GUI will be accessible through `{BASE_URL}/grafana/`.

NOTE: To understand grafana config that enables it to work behind a reverse proxy, take a look at the `server.root_url` config for the container specified at the [grafana-config-map.yaml](grafana/grafana-config-map.yaml)


1.4. Usage
-----

To use the monitor system, only access to the URLs and use it as you would normally use Grafana + Prometheus

### 1.4.1. Dashboards

List of community dashbaords that are ready to use for our environment:

- [nodejs app in kubernetes. Credit: elessar-ch](https://gist.github.com/elessar-ch/42f0eb278aedd27d3b20f4ea490902c7#file-nodejs-kubernetes-grafana-dashboard-json)
- [NodeJS Application Dashboard](https://grafana.com/grafana/dashboards/11159)


1.5. Known issues and limitations
----------------------------

1.6. Getting help
------------

1.7. Contributing
------------

1.8. License
-------

1.9. Authors and history
---------------------------

1.10. Acknowledgments
---------------

- [Setup Prometheus monitoring on kubernetes](https://devopscube.com/setup-prometheus-monitoring-on-kubernetes/)
- [Setup Grafana on kubernetes](https://devopscube.com/setup-grafana-kubernetes/)
- [Setup Prometheus Node Exporter on Kubernetes](https://devopscube.com/node-exporter-kubernetes/)
---
Thanks [bibinwilson](https://github.com/bibinwilson) for this YAMLs
- [Prometheus Kubernetes prepared yamls](https://github.com/bibinwilson/kubernetes-prometheus)
- [Grafana Kubernetes prepared yamls](https://github.com/bibinwilson/kubernetes-grafana)
---
- [Setting Up Alert Manager on Kubernetes](https://devopscube.com/alert-manager-kubernetes-guide/)
