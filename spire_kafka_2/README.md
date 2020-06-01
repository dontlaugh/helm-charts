# Steps to install

`make install`

```bash
cd observables
make namespace
kubectl get secret docker.secret --export -o yaml | kubectl apply --namespace=observables -f -
kubectl get secret sidecar-certs --export -o yaml | kubectl apply --namespace=observables -f -
cd ..
kubectl apply -f spire_kafka_2/configmap-b0.yaml
kubectl apply -f spire_kafka_2/configmap-b1.yaml
kubectl apply -f spire_kafka_2/configmap-b2.yaml
kubectl apply -f spire_kafka_2/svc-b0.yaml
kubectl apply -f spire_kafka_2/svc-b1.yaml
kubectl apply -f spire_kafka_2/svc-b2.yaml
kubectl apply -f spire_kafka_2/kafka_template.yaml -n observables
```

## What i am seeing:

This is a second attempt at `kafka_filter` setup with SPIRE:

What I am seeing:

I am just trying to get the kafka brokers and sidecars to be able to talk to each other (locked down). Steps:

- add tls context to each sidecar's listener filter chain (at `0.0.0.0:19092`, `19192` and `19292` respectively) with sds using spiffe identity for that broker sidecar (`kafka-broker-0`, `kafka-broker-1`, and `kafka-broker-2`) with validation context match_subject_alt_names allowing connections from certs with SAN of the other two brokers.
- add tls context to each cluster pointing at the other brokers - so for example kafka broker 0 configmap has clusters `broker1` and `broker2`, now each has tls_context with sds using spiffe identity `kafka-broker-0` and allowing connection to cert w SAN `kafka-broker-1` for cluster `broker1` and `kafka-broker-2` for cluster broker2

I haven't even incorporated allowing fibonacci access yet, The cluster in each sidecar that points at `127.0.0.1:9092`, the plaintext internal listener for the actual kafka broker has no tls_context, indicating plaintext, however, i am seeing this repeated in the logs:

```bash
[2020-06-01 00:02:50.463][21][debug][connection] [external/envoy/source/common/network/connection_impl.cc:192] [C431] closing socket: 0
[2020-06-01 00:02:50.463][21][debug][pool] [external/envoy/source/common/tcp/conn_pool.cc:206] canceling pending request
[2020-06-01 00:02:50.463][21][debug][pool] [external/envoy/source/common/tcp/conn_pool.cc:214] canceling pending connection
[2020-06-01 00:02:50.465][21][debug][connection] [external/envoy/source/common/network/connection_impl.cc:101] [C432] closing data_to_write=0 type=1
[2020-06-01 00:02:50.465][21][debug][connection] [external/envoy/source/common/network/connection_impl.cc:192] [C432] closing socket: 1
[2020-06-01 00:02:50.467][21][debug][pool] [external/envoy/source/common/tcp/conn_pool.cc:124] [C432] client disconnected
[2020-06-01 00:02:50.467][21][debug][conn_handler] [external/envoy/source/server/connection_handler_impl.cc:86] [C431] adding to cleanup list
[2020-06-01 00:02:50.467][21][debug][pool] [external/envoy/source/common/tcp/conn_pool.cc:238] [C432] connection destroyed
[2020-06-01 00:02:50.709][24][debug][filter] [external/envoy/source/common/tcp_proxy/tcp_proxy.cc:233] [C433] new tcp proxy session
[2020-06-01 00:02:50.709][24][debug][filter] [external/envoy/source/common/tcp_proxy/tcp_proxy.cc:378] [C433] Creating connection to cluster my-broker
[2020-06-01 00:02:50.710][24][debug][pool] [external/envoy/source/common/tcp/conn_pool.cc:83] creating a new connection
[2020-06-01 00:02:50.710][24][debug][pool] [external/envoy/source/common/tcp/conn_pool.cc:364] [C434] connecting
[2020-06-01 00:02:50.710][24][debug][connection] [external/envoy/source/common/network/connection_impl.cc:698] [C434] connecting to 127.0.0.1:9092
[2020-06-01 00:02:50.711][24][debug][connection] [external/envoy/source/common/network/connection_impl.cc:707] [C434] connection in progress
[2020-06-01 00:02:50.711][24][debug][pool] [external/envoy/source/common/tcp/conn_pool.cc:109] queueing request due to no available connections
[2020-06-01 00:02:50.711][24][debug][conn_handler] [external/envoy/source/server/connection_handler_impl.cc:353] [C433] new connection
[2020-06-01 00:02:50.712][24][debug][connection] [external/envoy/source/extensions/transport_sockets/tls/ssl_socket.cc:198] [C433] handshake error: 1
[2020-06-01 00:02:50.712][24][debug][connection] [external/envoy/source/extensions/transport_sockets/tls/ssl_socket.cc:226] [C433] TLS error: 268435703:SSL routines:OPENSSL_internal:WRONG_VERSION_NUMBER
```

And in the stats:

```
cluster.my-broker.update_attempt: 98
cluster.my-broker.update_empty: 0
cluster.my-broker.update_failure: 0
cluster.my-broker.update_no_rebuild: 97
cluster.my-broker.update_success: 98
cluster.my-broker.upstream_cx_active: 0
cluster.my-broker.upstream_cx_close_notify: 0
cluster.my-broker.upstream_cx_connect_attempts_exceeded: 0
cluster.my-broker.upstream_cx_connect_fail: 788
cluster.my-broker.upstream_cx_connect_timeout: 0
cluster.my-broker.upstream_cx_destroy: 859
cluster.my-broker.upstream_cx_destroy_local: 859
cluster.my-broker.upstream_cx_destroy_local_with_active_rq: 71
cluster.my-broker.upstream_cx_destroy_remote: 0
cluster.my-broker.upstream_cx_destroy_remote_with_active_rq: 0
cluster.my-broker.upstream_cx_destroy_with_active_rq: 71
cluster.my-broker.upstream_cx_http1_total: 0
cluster.my-broker.upstream_cx_http2_total: 0
cluster.my-broker.upstream_cx_idle_timeout: 0
cluster.my-broker.upstream_cx_max_requests: 0
cluster.my-broker.upstream_cx_none_healthy: 0
cluster.my-broker.upstream_cx_overflow: 0
cluster.my-broker.upstream_cx_pool_overflow: 0
cluster.my-broker.upstream_cx_protocol_error: 0
cluster.my-broker.upstream_cx_rx_bytes_buffered: 0
cluster.my-broker.upstream_cx_rx_bytes_total: 0
cluster.my-broker.upstream_cx_total: 859
cluster.my-broker.upstream_cx_tx_bytes_buffered: 0
cluster.my-broker.upstream_cx_tx_bytes_total: 0
cluster.my-broker.upstream_flow_control_backed_up_total: 0
cluster.my-broker.upstream_flow_control_drained_total: 0
cluster.my-broker.upstream_flow_control_paused_reading_total: 0
cluster.my-broker.upstream_flow_control_resumed_reading_total: 0
cluster.my-broker.upstream_internal_redirect_failed_total: 0
cluster.my-broker.upstream_internal_redirect_succeeded_total: 0
cluster.my-broker.upstream_rq_active: 0
cluster.my-broker.upstream_rq_cancelled: 788
cluster.my-broker.upstream_rq_completed: 0
cluster.my-broker.upstream_rq_maintenance_mode: 0
cluster.my-broker.upstream_rq_pending_active: 0
cluster.my-broker.upstream_rq_pending_failure_eject: 0
cluster.my-broker.upstream_rq_pending_overflow: 0
cluster.my-broker.upstream_rq_pending_total: 859
cluster.my-broker.upstream_rq_per_try_timeout: 0
cluster.my-broker.upstream_rq_retry: 0
cluster.my-broker.upstream_rq_retry_overflow: 0
cluster.my-broker.upstream_rq_retry_success: 0
cluster.my-broker.upstream_rq_rx_reset: 0
cluster.my-broker.upstream_rq_timeout: 0
cluster.my-broker.upstream_rq_total: 71
```

This seems to indicate a problem with the connection from sidecar -> kafka broker at localhost:9092, but I can't figure out exactly why it would be trying to connect with anything other than plaintext.

To delete:

```bash
kubectl delete -f spire_kafka_2/configmap-b0.yaml
kubectl delete -f spire_kafka_2/configmap-b1.yaml
kubectl delete -f spire_kafka_2/configmap-b2.yaml
kubectl delete -f spire_kafka_2/svc-b0.yaml
kubectl delete -f spire_kafka_2/svc-b1.yaml
kubectl delete -f spire_kafka_2/svc-b2.yaml
kubectl delete -f spire_kafka_2/kafka_template.yaml -n observables
```
