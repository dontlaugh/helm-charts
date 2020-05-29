# Steps to install

`make install`

```bash
cd observables
make namespace
kubectl get secret docker.secret --export -o yaml | kubectl apply --namespace=observables -f -
kubectl get secret sidecar-certs --export -o yaml | kubectl apply --namespace=observables -f -
cd ..
kubectl apply -f kafka_filter_spire/configmap-b0.yaml
kubectl apply -f kafka_filter_spire/configmap-b1.yaml
kubectl apply -f kafka_filter_spire/configmap-b2.yaml
kubectl apply -f kafka_filter_spire/svc-b0.yaml
kubectl apply -f kafka_filter_spire/svc-b1.yaml
kubectl apply -f kafka_filter_spire/svc-b2.yaml
kubectl apply -f kafka_filter_spire/kafka_template.yaml -n observables
```

to create fibonacci:

```bash
kubectl apply -f kafka_filter_spire/fib/fib-configmap.yaml
kubectl apply -f kafka_filter_spire/fib/fibonacci-deployment.yaml

greymatter create domain < kafka_filter_spire/fib/mesh/domain.json
greymatter create listener < kafka_filter_spire/fib/mesh/listener.json
greymatter create proxy < kafka_filter_spire/fib/mesh/proxy.json
greymatter create cluster < kafka_filter_spire/fib/mesh/edge-fibonacci-cluster.json
greymatter create cluster < kafka_filter_spire/fib/mesh/fibonacci-cluster.json
greymatter create shared_rules < kafka_filter_spire/fib/mesh/fibonacci-rules.json
greymatter create shared_rules < kafka_filter_spire/fib/mesh/edge-to-fibonacci-rules.json
greymatter create route < kafka_filter_spire/fib/mesh/edge-route.json
greymatter create route < kafka_filter_spire/fib/mesh/edge-route-slash.json
greymatter create route < kafka_filter_spire/fib/mesh/fibonacci-route.json
curl -k -XPOST --cert ./certs/quickstart.crt --key ./certs/quickstart.key https://localhost:30000/services/catalog/latest/clusters -d "@kafka_filter_spire/fib/mesh/catalog.json"

```

To delete:

```bash
kubectl delete -f kafka_filter_spire/configmap-b0.yaml
kubectl delete -f kafka_filter_spire/configmap-b1.yaml
kubectl delete -f kafka_filter_spire/configmap-b2.yaml
kubectl delete -f kafka_filter_spire/svc-b0.yaml
kubectl delete -f kafka_filter_spire/svc-b1.yaml
kubectl delete -f kafka_filter_spire/svc-b2.yaml
kubectl delete -f kafka_filter_spire/kafka_template.yaml -n observables
```

```bash
kubectl delete -f kafka_filter_spire/fib/fib-configmap.yaml
kubectl delete -f kafka_filter_spire/fib/fibonacci-deployment.yaml
```
