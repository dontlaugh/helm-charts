# Set up the Kafka Brokers Filter with SPIRE

This scenario is identical to scenario 2, but this time it uses spiffe/spire to lock down the connection between the sidecar with observables filter turned on and the sidecar in front of the kafka brokers.

All files for this setup are in the `kafka_brokers/scenario_3` directory.

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

`kubectl get pods -n observables` and wait for all pods to be up.

The kafka broker pods are `kafka-observables-0`, `kafka-observables-1`, and `kafka-observables-2` in the observables namespace.

```bash
cd ..
kubectl apply -f kafka_brokers/scenario_3/b0-configmap.yaml
kubectl apply -f kafka_brokers/scenario_3/b1-configmap.yaml
kubectl apply -f kafka_brokers/scenario_3/b2-configmap.yaml
kubectl apply -f kafka_brokers/scenario_3/b0-deployment.yaml
kubectl apply -f kafka_brokers/scenario_3/b1-deployment.yaml
kubectl apply -f kafka_brokers/scenario_3/b2-deployment.yaml
kubectl apply -f kafka_brokers/scenario_3/b0-svc.yaml
kubectl apply -f kafka_brokers/scenario_3/b1-svc.yaml
kubectl apply -f kafka_brokers/scenario_3/b2-svc.yaml
```

This will deploy 3 sidecars, one for each kafka broker.  You can see the configuration of the kafka_broker filter in the configmap for each sidecar.

Now deploy the fibonacci service. The listener configuration will be identical to [scenario 2](../README.md#scenario-2) but with eventTopic `fibonacci-scenario3`:

```json
"gm_observables": {
    "useKafka": true,
    "topic": "fibonacci",
    "logLevel": "debug",
    "eventTopic": "fibonacci-scenario3",
    "kafkaServerConnection": "localhost:10909,localhost:10919,localhost:10929"
}
```

```bash
kubectl apply -f kafka_brokers/scenario_3/fib-configmap.yaml
kubectl apply -f kafka_brokers/scenario_3/fibonacci-deployment.yaml

greymatter create domain < kafka_brokers/scenario_3/mesh/domain.json
greymatter create listener < kafka_brokers/scenario_3/mesh/listener.json
greymatter create proxy < kafka_brokers/scenario_3/mesh/proxy.json
greymatter create cluster < kafka_brokers/scenario_3/mesh/edge-fibonacci-cluster.json
greymatter create cluster < kafka_brokers/scenario_3/mesh/fibonacci-cluster.json
greymatter create shared_rules < kafka_brokers/scenario_3/mesh/fibonacci-rules.json
greymatter create shared_rules < kafka_brokers/scenario_3/mesh/edge-to-fibonacci-rules.json
greymatter create route < kafka_brokers/scenario_3/mesh/edge-route.json
greymatter create route < kafka_brokers/scenario_3/mesh/edge-route-slash.json
greymatter create route < kafka_brokers/scenario_3/mesh/fibonacci-route.json
curl -k -XPOST --cert ./certs/quickstart.crt --key ./certs/quickstart.key https://localhost:30000/services/catalog/latest/clusters -d "@kafka_brokers/scenario_3/mesh/catalog.json"
```

At first, the observables filter isn't turned on, this is to give the sidecar time to find the kafka broker proxy clusters.

After startup, `greymatter edit listener fibonacci-listener` and add `gm.observables` to the `active_http_filters`.

Make any request to the fibonacci service `https://localhost:30000/services/fibonacci`.

In the fibonacci sidecar logs, you should see:

```bash
INF Message publishing to Kafka Encryption= EncryptionKeyID=0 Filter=Observables Topic=fibonacci
```

If this isn't there, edit the `fibonacci-listener` again, remove `gm.observables` from the list of active http filters, bounce the fibonacci pod and try to enable it again. (looking into why this is happening to me sometimes)

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

For whichever scenario you are testing, you should see `fibonacci-scenario3`.

You can run the following to list topics from different bootstrap servers:

1. `kafka-topics.sh --list --bootstrap-server kafka-observables-0.kafka-observables-headless.observables.svc.cluster.local:9092` to directly list topics from the kafka broker with pod `kafka-observables-0`
2. `kafka-topics.sh --list --bootstrap-server kafka-observables-1.kafka-observables-headless.observables.svc.cluster.local:9092` to directly list topics from the kafka broker with pod `kafka-observables-1`
3. `kafka-topics.sh --list --bootstrap-server kafka-observables-2.kafka-observables-headless.observables.svc.cluster.local:9092` to directly list topics from the kafka broker with pod `kafka-observables-2`

You should see `fibonacci-scenario3`.

Now, the kafka broker proxies are locked down and looking for a certificate with SAN SPIFFE id `spiffe://quickstart.greymatter.io/fibonacci`. This will be the only allowed connection, so, unlike in scenario 2, this kafka client attempting to connect to the kafka broker proxies should not be allowed. Thus, these should all fail:

1. `kafka-topics.sh --list --bootstrap-server kafka-broker0.observables.svc.cluster.local:9092`
2. `kafka-topics.sh --list --bootstrap-server kafka-broker1.observables.svc.cluster.local:9092`
3. `kafka-topics.sh --list --bootstrap-server kafka-broker2.observables.svc.cluster.local:9092`

Run the following to get which broker the observables are being logged in:

```bash
kafka-topics.sh --describe --zookeeper kafka-observables-zookeeper-headless.observables.svc.cluster.local:2181 --topic fibonacci-scenario3
```

You should see something like:

```bash
Topic: fibonacci-scenario3	PartitionCount: 1	ReplicationFactor: 1	Configs: 
	Topic: fibonacci-scenario3	Partition: 0	Leader: 1	Replicas: 1	Isr: 1
```

The number listed after `Leader` is the broker to check for the logs, so in this example it would be broker 1. Exit and exec into the pod for the leader to check the logs for the observables:

```bash
kubectl exec -it kafka-observables-<LEADER-NUMBER> /bin/bash -n observables
cat /bitnami/kafka/data/fibonacci-scenario3-0/00000000000000000000.log
```

You should see the observable!

### Purposefully breaking it

#### Wrong SAN in broker proxy listeners

Go into `kafka_brokers/scenario_3/b0-configmap.yaml`, `kafka_brokers/scenario_3/b0-configmap.yaml`, and `kafka_brokers/scenario_3/b2-configmap.yaml` and change the SAN on line 53 `exact: "spiffe://quickstart.greymatter.io/fibonacci"` to something incorrect, say `exact: "spiffe://quickstart.greymatter.io/wrong-san"`.

Then, apply those files again and delete the pods. When the new pods come up, they will take this configuration.

Then try to turn on observables on the `fibonacci-listener`. In the fibonacci sidecar logs, you should see repeated:

```bash
ERR error creating kafka producer for publishing Encryption= EncryptionKeyID=0 Filter=Observables Topic=fibonacci err="kafka: client has run out of available brokers to talk to (Is your cluster reachable?)"
```

until it ultimately fails.  Exec into any of the broker pods in the observables namespace. Then run `curl localhost:8001/stats`.  Search for the listener stats, prefixed with `listener.0.0.0.0_9092`, and you should see this statistic with a number greater than zero, `listener.0.0.0.0_9092.ssl.fail_verify_san: 120`. This means that the verify san did not match the incoming, and the connection failed.

#### Wrong SAN in fibonacci egress clusters

In `kafka_brokers/scenario_3/fib-configmap.yaml`, change the `match_subject_alt_names`.

In the sidecar logs, you will see repeated:

```bash
ERR error creating kafka producer for publishing Encryption= EncryptionKeyID=0 Filter=Observables Topic=fibonacci err="kafka: client has run out of available brokers to talk to (Is your cluster reachable?)"
```

and it will ultimately fail.
