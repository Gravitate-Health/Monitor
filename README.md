# Gravitate Health Monitor

[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

---
## Table of contents

- [Gravitate Health Monitor](#gravitate-health-monitor)
  - [Table of contents](#table-of-contents)
  - [Introduction](#introduction)
  - [Kubernetes Deployment](#kubernetes-deployment)
    - [Prerequisites](#prerequisites)
    - [Prepare the environment](#prepare-the-environment)
    - [Prometheus](#prometheus)
      - [Service discovery](#service-discovery)
    - [Alert Manager](#alert-manager)
    - [Grafana](#grafana)
  - [Usage](#usage)
    - [Dashboards](#dashboards)
  - [Known issues and limitations](#known-issues-and-limitations)
  - [Getting help](#getting-help)
  - [Contributing](#contributing)
  - [License](#license)
  - [Authors and history](#authors-and-history)
  - [Acknowledgments](#acknowledgments)

---
## Introduction

This repository contains the configuration and deployment files necessary to monitor a kubernetes cluster and deployments on top of the cluster, such as nodeJS apps, Mongo databases, Keycloak server, etc. The monitor system consists of a Grafana + Prometheus stack.

This readme will help the reader to deploy the system to a kubernetes cluster, but also to understand the configuration and be able to edit/expand it.

![Monitor stack architecture](./docs/prometheus-grafana-stack.png "Monitor stack architecture")

---
## Kubernetes Deployment

Grafana and Prometheus offer their official Docker image which is ready to deploy and work for a local environment. For a k8s cluster, some considerations must be taken into account.

### Prerequisites

The only prerequesites are a Kubernetes cluster and a gateway/reverse-proxy configured and with a working external url(domain name). The externally accsible url for the gateway will be referenced as `BASE_URL` from now on.

This gateway must recirect petitions with prefix `/grafana/` to grafana (removing the prefix), and petitions with prefix `/prometheus/` to prometheus (NOT removing the prefix):

- `{BASE_URL}/grafana/foo` will be redirected to grafana as `{grafana-url}/foo`
- `{BASE_URL}/prometheus/foo` will be redirected to prometheus as `{prometheus-url}/prometheus/foo`

Currently, the yaml files configure Prometheus to be accessible through a reverse proxy, and not through Kubectl port forwarding or an ingress object. To know how to do it, refer to [official kubernetes documentation](https://kubernetes.io/es/docs/home/)

### Prepare the environment


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

### Prometheus 

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


#### Service discovery

Prometheus config enables service discovery by reading annotations from services with no need for extra configuration to scrape a new endpoint. Services should include these annotations if they want to be scraped by prometheus, and sholud be included in the yaml file describing the service, in section `metadata.annotations`.:

- prometheus.io/scrape: Only scrape services that have a value of `true`.
- prometheus.io/path: If the metrics path is not `/metrics` override this.
- prometheus.io/port: If the metrics are exposed on a different port to the service then set this appropriately.
- prometheus.io/scheme: If the metrics endpoint is secured then you will need to set this to `https` & most likely set the `tls_config` of the scrape config.

### Alert Manager

AlertManager is an open source alerting system taht works with Prometheus. Its service endpoint is already configured at [ Prometheus' config-map ](prometheus/prometheus-config-map.yaml) to send alerts to AlertManager. Alert rules are configured here and dumped to a file called `prometheus.rules`.

Alert manager needs smarthost configuration to be able to send emails.

1. Create the following resources:
```bash
kubectl create -f alertmanager/alert-manager-config-map.yaml
kubectl create -f alertmanager/alert-manager-service.yaml
kubectl create -f alertmanager/alert-manager-deployment.yaml
kubectl create -f alertmanager/alert-template-config-map.yaml
```

After these steps, alert manager web GUI will be accessible through `{BASE_URL}/alertmanager/`.



### Grafana

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


---
## Usage

To use the monitor system, only access to the URLs and use it as you would normally use Grafana + Prometheus

### Dashboards

List of community dashbaords that are ready to use for our environment:

- [nodejs app in kubernetes. Credit: elessar-ch](https://gist.github.com/elessar-ch/42f0eb278aedd27d3b20f4ea490902c7#file-nodejs-kubernetes-grafana-dashboard-json)
- [NodeJS Application Dashboard](https://grafana.com/grafana/dashboards/11159)


---
## Known issues and limitations

---
## Getting help

---
## Contributing

---
## License

This project is distributed under the terms of the [Apache License, Version 2.0 (AL2)](http://www.apache.org/licenses/LICENSE-2.0).  The license applies to this file and other files in the [GitHub repository](https://github.com/Gravitate-Health/Gateway) hosting this file.

```
Copyright 2022 Universidad Politécnica de Madrid

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
```
---
## Authors and history

- Guillermo Mejías ([@gmej](https://github.com/gmej))


---
## Acknowledgments

- [Setup Prometheus monitoring on kubernetes](https://devopscube.com/setup-prometheus-monitoring-on-kubernetes/)
- [Setup Grafana on kubernetes](https://devopscube.com/setup-grafana-kubernetes/)
- [Setup Prometheus Node Exporter on Kubernetes](https://devopscube.com/node-exporter-kubernetes/)

---

Thanks [bibinwilson](https://github.com/bibinwilson) for this YAMLs
- [Prometheus Kubernetes prepared yamls](https://github.com/bibinwilson/kubernetes-prometheus)
- [Grafana Kubernetes prepared yamls](https://github.com/bibinwilson/kubernetes-grafana)
  
- [Basic kubernetes alerts](https://gist.github.com/efossas/c0bba665469186621f89f7bd89b3f0a9#file-prometheus-yml)

---

- [Setting Up Alert Manager on Kubernetes](https://devopscube.com/alert-manager-kubernetes-guide/)
