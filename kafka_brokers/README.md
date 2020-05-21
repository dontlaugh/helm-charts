# Set up the Kafka Broker Filter

To set up sidecars using the kafka broker filter and test, do the following:

`make install` from the home directory of the helm charts.

Then, `cd observables` and install kafka with `make kafka`.

The kafka broker pods are `kafka-observables-0`, `kafka-observables-1`, and `kafka-observables-2` in the observables namespace.

Come back into this directory and:

```bash
kubectl apply -f b0-configmap.yaml
kubectl apply -f b1-configmap.yaml
kubectl apply -f b2-configmap.yaml
kubectl apply -f b0-deployment.yaml
kubectl apply -f b1-deployment.yaml
kubectl apply -f b2-deployment.yaml
kubectl apply -f b0-svc.yaml
kubectl apply -f b1-svc.yaml
kubectl apply -f b2-svc.yaml
```

This will deploy 3 sidecars, one for each kafka broker.  You can see the configuration of the kafka_broker filter in the configmap for each sidecar.

## To test:

To test, run the following to start up a kafka client:

```bash
kubectl run kafka-observables-client --rm --tty -i --restart='Never' --image docker.io/bitnami/kafka:2.4.0-debian-9-r22 --namespace observables --command -- bash
```

This will start up a kafka client pod.  Then, create a topic with the following command. Note that the bootstrap servers are pointing at the services created for the brokers' sidecars, and not pointing directly at the kafka brokers themselves:

```bash
kafka-topics.sh --bootstrap-server kafka-broker0.observables.svc.cluster.local:9092,kafka-broker1.observables.svc.cluster.local:9092,kafka-broker2.observables.svc.cluster.local:9092 --create --topic test-brokers-topic --replication-factor 2 --partitions 20
```

This should complete without any errors.

Then, to check that the topic was created, exit the client pod (it will delete itself) and exec into any of the kafka broker pods, `kubectl exec -it kafka-observables-0  /bin/sh -n observables`.  List the directories and you should see the topic you just created with 20 partitions.

```bash
ls /bitnami/kafka/data
```