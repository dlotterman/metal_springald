## metal_springald

`metal_springald` is intended to be used as a host / worker harness for benchmarking / load generation activities inside of Equinix Metal.

Currently under active development, when stable it will be able to:

- Provision Equinix Metal instances with a base Linux OS (Ubuntu, CentOS etc), where those instances are intended to be used as load generation hosts.
- Configure those hosts appropriately, including both Equinix Metal platform and the host OS itself, so that worker instances can join and participate in the correct network namespaces
- Monitor those load generation (and other) worker hosts with [prometheus](roles/prometheus) + [node_exporter](roles/node_exporter) + [grafana](roles/grafana)

![](https://s3.us-east-1.wasabisys.com/metalstaticassets/springald.PNG)

### Setup
