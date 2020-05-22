# Set up the Kafka Broker Filter

To set up sidecars using the kafka broker filter and test, do the following:

`make install` from the home directory of the helm charts.

Install kafka:

```bash
cd observables
make namespace
kubectl get secret docker.secret --export -o yaml | kubectl apply --namespace=observables -f -
kubectl get secret sidecar-certs --export -o yaml | kubectl apply --namespace=observables -f -
make kafka
```

`kubectl get pods -n observables`,

The kafka broker pods are `kafka-observables-0`, `kafka-observables-1`, and `kafka-observables-2` in the observables namespace.

## Explanation

To explain this setup a little more, there are 3 kafka brokers. Each broker has the following env vars:

```yaml
- name: KAFKA_CFG_LISTENERS
    value: "PLAINTEXT://:$(KAFKA_PORT_NUMBER)"
- name: KAFKA_CFG_ADVERTISED_LISTENERS
    value: "PLAINTEXT://$(MY_POD_NAME).kafka-observables-headless.observables.svc.cluster.local:$(KAFKA_PORT_NUMBER)"
```

`KAFKA_PORT_NUMBER` by default is 9092.  Ultimately, there will be three brokers, one with each of the following listener, advertised listener pair:

```txt
listeners=PLAINTEXT://:9092
advertised.listeners=PLAINTEXT://kafka-observables-0.kafka-observables-headless.observables.svc.cluster.local:9092
```

```txt
listeners=PLAINTEXT://:9092
advertised.listeners=PLAINTEXT://kafka-observables-1.kafka-observables-headless.observables.svc.cluster.local:9092
```

```txt
listeners=PLAINTEXT://:9092
advertised.listeners=PLAINTEXT://kafka-observables-2.kafka-observables-headless.observables.svc.cluster.local:9092
```

The pods for these brokers are `kafka-observables-0`, `kafka-observables-1`. and `kafka-observables-2`.

Now, the files in `kafka_brokers` directory deploy 3 Grey Matter sidecars, one for each broker. Each sidecar will have one listener, with the `kafka_brokers` and `tcp_proxy` filter turned on, and the kafka_brokers filter will be pointing at a broker. You can see these configurations in the `b0-configmap.yaml`, `b1-configmap.yaml`, and `b2-configmap.yaml`.

```bash
kubectl apply -f kafka_brokers/b0-configmap.yaml
kubectl apply -f kafka_brokers/b1-configmap.yaml
kubectl apply -f kafka_brokers/b2-configmap.yaml
kubectl apply -f kafka_brokers/b0-deployment.yaml
kubectl apply -f kafka_brokers/b1-deployment.yaml
kubectl apply -f kafka_brokers/b2-deployment.yaml
kubectl apply -f kafka_brokers/b0-svc.yaml
kubectl apply -f kafka_brokers/b1-svc.yaml
kubectl apply -f kafka_brokers/b2-svc.yaml
```

When this is done, there will be 3 more pods in the observables namespaces, pods with prefix `kafka-broker0*`, `kafka_broker1*`, `kafka_broker2` correspond to these proxies. They each have a kafka_brokers filter pointing at `kafka-observables-0`, `kafka-observables-1`, and `kafka-observables-2` respectively.

Now we will deploy a fibonacci service into the mesh, with the Grey Matter observables filter turned on with 2 different scenarios for its configuration.

## Scenario 1

A sidecar for each kafka broker, the kafka client created by the observables filter connects directly to the broker proxies.

Now we'll deploy a fibonacci service to test observables. The observables configuration for this scenario is:

```json
"gm_observables": {
    "useKafka": true,
    "topic": "fibonacci",
    "logLevel": "debug",
    "eventTopic": "fibonacci-scenario1",
    "kafkaServerConnection": "kafka-broker0.observables.svc.cluster.local:9092,kafka-broker1.observables.svc.cluster.local:9092,kafka-broker2.observables.svc.cluster.local:9092"
}
```

The `kafkaServerConnection` is connecting directly to the kafka broker proxies that we created.

To apply:

```bash
kubectl apply -f kafka_brokers/scenario_1/fibonacci-deployment.yaml

greymatter create domain < kafka_brokers/scenario_1/mesh/domain.json
greymatter create listener < kafka_brokers/scenario_1/mesh/listener.json
greymatter create proxy < kafka_brokers/scenario_1/mesh/proxy.json
greymatter create cluster < kafka_brokers/scenario_1/mesh/edge-fibonacci-cluster.json
greymatter create cluster < kafka_brokers/scenario_1/mesh/fibonacci-cluster.json
greymatter create shared_rules < kafka_brokers/scenario_1/mesh/fibonacci-rules.json
greymatter create shared_rules < kafka_brokers/scenario_1/mesh/edge-to-fibonacci-rules.json
greymatter create route < kafka_brokers/scenario_1/mesh/edge-route.json
greymatter create route < kafka_brokers/scenario_1/mesh/edge-route-slash.json
greymatter create route < kafka_brokers/scenario_1/mesh/fibonacci-route.json
curl -k -XPOST --cert ./certs/quickstart.crt --key ./certs/quickstart.key https://localhost:30000/services/catalog/latest/clusters -d "@kafka_brokers/scenario_1/mesh/catalog.json"
```

Make any request to the fibonacci service `https://localhost:30000/services/fibonacci`.

To test, following the [testing section](#testing).

## Scenario 2

A sidecar for each kafka broker.  The sidecar in the mesh that has the observables filter turned on has 3 extra egress listeners. Each of these listeners have a tcp_proxy filter pointing at a cluster that points at a sidecar for a kafka broker.

Now, deploy a fibonacci service to test observables.  The service has 3 egress listeners, at port 10909, 10919, and 10929.  Each listener has a tcp proxy filter pointing to a cluster for a kafka broker proxy.  For example, the listener at port `10909` has a tcp proxy pointing at cluster `broker0-proxy`, which has address `kafka-broker0.observables.svc.cluster.local:9092` - the address for the kafka broker proxy 0, and so on.

The listener configuration for this scenario is:

```json
"gm_observables": {
    "useKafka": true,
    "topic": "fibonacci",
    "logLevel": "debug",
    "eventTopic": "fibonacci-scenario2",
    "kafkaServerConnection": "localhost:10909,localhost:10919,localhost:10929"
}
```

You can see `"kafkaServerConnection": "localhost:10909,localhost:10919,localhost:10929"` to point at each egress listener, which in turn is pointing at each kafka broker proxy.

```bash
kubectl apply -f kafka_brokers/scenario_2/fib-configmap.yaml
kubectl apply -f kafka_brokers/scenario_2/fibonacci-deployment.yaml

greymatter create domain < kafka_brokers/scenario_2/mesh/domain.json
greymatter create listener < kafka_brokers/scenario_2/mesh/listener.json
greymatter create proxy < kafka_brokers/scenario_2/mesh/proxy.json
greymatter create cluster < kafka_brokers/scenario_2/mesh/edge-fibonacci-cluster.json
greymatter create cluster < kafka_brokers/scenario_2/mesh/fibonacci-cluster.json
greymatter create shared_rules < kafka_brokers/scenario_2/mesh/fibonacci-rules.json
greymatter create shared_rules < kafka_brokers/scenario_2/mesh/edge-to-fibonacci-rules.json
greymatter create route < kafka_brokers/scenario_2/mesh/edge-route.json
greymatter create route < kafka_brokers/scenario_2/mesh/edge-route-slash.json
greymatter create route < kafka_brokers/scenario_2/mesh/fibonacci-route.json
curl -k -XPOST --cert ./certs/quickstart.crt --key ./certs/quickstart.key https://localhost:30000/services/catalog/latest/clusters -d "@kafka_brokers/scenario_2/mesh/catalog.json"
```

At first, the observables filter isn't turned on, this is to give the sidecar time to find the kafka broker proxy clusters.

After startup, `greymatter edit listener fibonacci-listener` and add `gm.observables` to the `active_http_filters`.

Make any request to the fibonacci service `https://localhost:30000/services/fibonacci`.

To test, following the [testing section](#testing).

## Testing

Now, create a kafka client:

```bash
kubectl run kafka-observables-client --rm --tty -i --restart='Never' --image docker.io/bitnami/kafka:2.4.0-debian-9-r22 --namespace observables --command -- bash
```

In that shell, run the following to check the topics from zookeeper:

```bash
kafka-topics.sh --list --zookeeper kafka-observables-zookeeper-headless.observables.svc.cluster.local:2181
```

For whichever scenario you are testing, you should see `fibonacci-scenario<NUMBER>`. You can replace the flag `--zookeeper kafka-observables-zookeeper-headless.observables.svc.cluster.local:2181` in the above command with any of the following:

You can run the following to list topics from different bootstrap servers:

1. `kafka-topics.sh --list --bootstrap-server kafka-observables-0.kafka-observables-headless.observables.svc.cluster.local:9092` to directly list topics from the kafka broker with pod `kafka-observables-0`
2. `kafka-topics.sh --list --bootstrap-server kafka-observables-1.kafka-observables-headless.observables.svc.cluster.local:9092` to directly list topics from the kafka broker with pod `kafka-observables-1`
3. `kafka-topics.sh --list --bootstrap-server kafka-observables-2.kafka-observables-headless.observables.svc.cluster.local:9092` to directly list topics from the kafka broker with pod `kafka-observables-2`
4. `kafka-topics.sh --list --bootstrap-server kafka-broker0.observables.svc.cluster.local:9092` to list topics from the kafka broker **proxy** pointing at broker0
5. `kafka-topics.sh --list --bootstrap-server kafka-broker1.observables.svc.cluster.local:9092` to list topics from the kafka broker **proxy** pointing at broker1
6. `kafka-topics.sh --list --bootstrap-server kafka-broker2.observables.svc.cluster.local:9092` to list topics from the kafka broker **proxy** pointing at broker2

All of the above commands should see the topic `fibonacci-scenario1` or `fibonacci-scenario2` for whichever you are testing.

Now, replace the topic at the end of this command with the scenario number that you are testing for and run the following to get which broker the observables are being logged in:

```bash
kafka-topics.sh --describe --zookeeper kafka-observables-zookeeper-headless.observables.svc.cluster.local:2181 --topic fibonacci-scenario<SCENARIO-NUMBER>
```

You should see something like:

```bash
Topic: fibonacci-scenario<SCENARIO-NUMBER>	PartitionCount: 1	ReplicationFactor: 1	Configs: 
	Topic: fibonacci-scenario<SCENARIO-NUMBER>	Partition: 0	Leader: 1	Replicas: 1	Isr: 1
```

The number listed after `Leader` is the broker to check for the logs, so in this example it would be broker 1. Exec into the pod for broker 1 to check the logs for the observables:

```bash
kubectl exec -it kafka-observables-1 /bin/bash -n observables
cat /bitnami/kafka/data/fibonacci-scenario<SCENARIO-NUMBER>-0/00000000000000000000.log
```

You should see the observables!
