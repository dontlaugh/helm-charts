# control-api Configuration Options

Autogenerated by `gen-docs.py` at 2020-03-18 18:40:09

### Global Configuration

|             Parameter              |Description|           Default           |
|------------------------------------|-----------|-----------------------------|
|global.image_pull_secret            |           |docker.secret                |
|global.environment                  |           |openshift                    |
|global.zone                         |           |zone-default-zone            |
|global.consul.enabled               |           |False                        |
|global.consul.host                  |           |                             |
|global.consul.port                  |           |                         8500|
|global.control_port                 |           |                        50000|
|global.waiter.image                 |           |deciphernow/k8s-waiter:latest|
|global.waiter.service_account.create|           |True                         |
|global.waiter.service_account.name  |           |waiter-sa                    |
|global.sidecar.version              |           |latest                       |

### Sidecar Configuration

|            Parameter            |Description|                                          Default                                          |
|---------------------------------|-----------|-------------------------------------------------------------------------------------------|
|sidecar.version                  |           |{{- $.Values.global.sidecar.version \| default "latest" }}                                  |
|sidecar.image                    |           |docker.production.deciphernow.com/deciphernow/gm-proxy:{{ tpl $.Values.sidecar.version $ }}|
|sidecar.port                     |           |                                                                                      10808|
|sidecar.metrics_port             |           |                                                                                       8081|
|sidecar.secret.enabled           |           |True                                                                                       |
|sidecar.secret.secret_name       |           |sidecar-certs                                                                              |
|sidecar.secret.mount_point       |           |/etc/proxy/tls/sidecar                                                                     |
|sidecar.secret.secret_keys.ca    |           |ca.crt                                                                                     |
|sidecar.secret.secret_keys.cert  |           |server.crt                                                                                 |
|sidecar.secret.secret_keys.key   |           |server.key                                                                                 |
|sidecar.image_pull_policy        |           |IfNotPresent                                                                               |
|sidecar.resources.limits.cpu     |           |200m                                                                                       |
|sidecar.resources.limits.memory  |           |512Mi                                                                                      |
|sidecar.resources.requests.cpu   |           |100m                                                                                       |
|sidecar.resources.requests.memory|           |128Mi                                                                                      |
