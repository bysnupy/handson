# Elasticsearch Operator update test from 5.2.4-17 to 5.3.1-12.

## Summary
The testing enviroments is OCPv4.8.15.

After installed Elasticsearch and OpenShift Logging in order, deployed ClusterLogging as follows before testing.

```cmd
$ oc get pod,pvc
NAME                                                READY   STATUS    RESTARTS   AGE
pod/cluster-logging-operator-c4c5f4f54-wtnlc        1/1     Running   0          4m18s
pod/elasticsearch-cdm-0p4w6uqm-1-85cd7555cd-c55hc   2/2     Running   0          3m1s
pod/elasticsearch-cdm-0p4w6uqm-2-565f46f64f-cdhpj   2/2     Running   0          3m
pod/elasticsearch-cdm-0p4w6uqm-3-7459ffb88-cccxx    2/2     Running   0          2m59s
pod/fluentd-2d6n8                                   2/2     Running   0          3m1s
pod/fluentd-4525r                                   2/2     Running   0          3m1s
pod/fluentd-bzrms                                   2/2     Running   0          3m
pod/fluentd-m9m85                                   2/2     Running   0          3m
pod/fluentd-pvsqk                                   2/2     Running   0          3m1s
pod/fluentd-sl86n                                   2/2     Running   0          3m
pod/kibana-84b65f558b-kvwcj                         2/2     Running   0          2m58s

NAME                                                               STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/elasticsearch-elasticsearch-cdm-0p4w6uqm-1   Bound    pvc-8c491f3b-ef4b-4994-9b3f-19c5bcc1d751   10Gi       RWO            gp2            3m1s
persistentvolumeclaim/elasticsearch-elasticsearch-cdm-0p4w6uqm-2   Bound    pvc-8e7d5cbd-f75f-4764-905a-e84b4533b03c   10Gi       RWO            gp2            3m1s
persistentvolumeclaim/elasticsearch-elasticsearch-cdm-0p4w6uqm-3   Bound    pvc-b24f949d-19cd-4465-aad6-5dace108138b   10Gi       RWO            gp2            3m1s

$ oc exec -c elasticsearch pod/elasticsearch-cdm-0p4w6uqm-1-85cd7555cd-c55hc -- health
Thu Dec 16 08:37:44 UTC 2021
epoch      timestamp cluster       status node.total node.data shards pri relo init unassign pending_tasks max_task_wait_time active_shards_percent
1639643864 08:37:44  elasticsearch green           3         3     22  11    0    0        0             0                  -                100.0%

$ oc exec -c elasticsearch pod/elasticsearch-cdm-0p4w6uqm-1-85cd7555cd-c55hc -- indices
Thu Dec 16 08:39:31 UTC 2021
health status index        uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   audit-000001 cnJ3qF57Sf26ZpUq-9hK-w   3   1          0            0          0              0
green  open   .kibana_1    -pMpB8CqTlSx4BL_sH0i2g   1   1          0            0          0              0
green  open   app-000001   jE6OIKQVQS6vsrXcpjaJGQ   3   1        566            0          1              0
green  open   infra-000001 TgsDGmD4Qie19Dt903GaZQ   3   1      36469            0         59             29
green  open   .security    r4favAOWTGqZBG2MoZQv7w   1   1          6            0          0              0
```

## Change the channel of the Subscription in Elasticsearch Operator

1. I've changed Elasticsearch Operator channel from stable-5.2 to stable-5.3 at 08:51:49.

![update](https://github.com/bysnupy/handson/blob/master/images/esupdate1.png)

2. old elasticsearch-operator pod was terminated with the following messages.
```json
{"_ts":"2021-12-16T08:52:14.290683224Z","_level":"0","_component":"elasticsearch-operator_controller","_message":"Reconciler error","_error":{"msg":"Operation cannot be fulfilled on kibanas.logging.openshift.io \"kibana\": the object has been modified; please apply your changes to the latest version and try again"},"controller":"kibana-controller","name":"kibana","namespace":"openshift-logging"}
{"_ts":"2021-12-16T08:52:15.31988745Z","_level":"0","_component":"elasticsearch-operator","_message":"Beginning restart of node","cluster":"elasticsearch","namespace":"openshift-logging","node":"elasticsearch-cdm-0p4w6uqm-1"}
{"_ts":"2021-12-16T08:52:15.743309022Z","_level":"0","_component":"elasticsearch-operator","_message":"failed to flush nodes","_error":{"msg":"failed to flush shards in preparation for cluster restart","num_failed_shards":5},"cluster":"elasticsearch","namespace":"openshift-logging"}
{"_ts":"2021-12-16T08:52:15.906283125Z","_level":"0","_component":"elasticsearch-operator_controller","_message":"Reconciler error","_error":{"msg":"Operation cannot be fulfilled on kibanas.logging.openshift.io \"kibana\": the object has been modified; please apply your changes to the latest version and try again"},"controller":"kibana-controller","name":"kibana","namespace":"openshift-logging"}
{"_ts":"2021-12-16T08:52:16.897662918Z","_level":"0","_component":"elasticsearch-operator","_message":"Completed restart of node","cluster":"elasticsearch","namespace":"openshift-logging","node":"elasticsearch-cdm-0p4w6uqm-1"}
{"_ts":"2021-12-16T08:52:16.954563608Z","_level":"0","_component":"elasticsearch-operator","_message":"Beginning restart of node","cluster":"elasticsearch","namespace":"openshift-logging","node":"elasticsearch-cdm-0p4w6uqm-2"}
{"_ts":"2021-12-16T08:52:17.259195499Z","_level":"0","_component":"elasticsearch-operator","_message":"failed to flush nodes","_error":{"msg":"failed to flush shards in preparation for cluster restart","num_failed_shards":8},"cluster":"elasticsearch","namespace":"openshift-logging"}
{"_ts":"2021-12-16T08:52:17.413474703Z","_level":"0","_component":"elasticsearch-operator_controller","_message":"Stopping workers","controller":"kibana-controller"}
{"_ts":"2021-12-16T08:52:17.413595849Z","_level":"0","_component":"elasticsearch-operator_controller","_message":"Stopping workers","controller":"elasticsearch-controller"}
{"_ts":"2021-12-16T08:52:17.413627341Z","_level":"0","_component":"elasticsearch-operator_controller","_message":"Stopping workers","controller":"secret"}
```

3. new elasticsearch-operator was getting up, and after minutes ES service fell into service unavailability status.
1st ES pod was completed without failure, but 2nd ES pod was filed to restart, and 3rd ES pod started restarting without waiting for 2nd ES pod restart.
```json
{"_ts":"2021-12-16T08:52:59.556525682Z","_level":"0","_component":"elasticsearch-operator","_message":"Completed restart of node","cluster":"elasticsearch","namespace":"openshift-logging","node":"elasticsearch-cdm-0p4w6uqm-1"}
{"_ts":"2021-12-16T08:52:59.635208333Z","_level":"0","_component":"elasticsearch-operator","_message":"Beginning restart of node","cluster":"elasticsearch","namespace":"openshift-logging","node":"elasticsearch-cdm-0p4w6uqm-2"}
{"_ts":"2021-12-16T08:53:00.254076909Z","_level":"0","_component":"elasticsearch-operator","_message":"failed to flush nodes","_error":{"msg":"failed to flush shards in preparation for cluster restart","num_failed_shards":3},"cluster":"elasticsearch","namespace":"openshift-logging"}
{"_ts":"2021-12-16T08:53:00.310010706Z","_level":"0","_component":"elasticsearch-operator","_message":"failed to update deployment","elasticsearch-cdm-0p4w6uqm-2":"_error"}
{"_ts":"2021-12-16T08:53:32.680876745Z","_level":"0","_component":"elasticsearch-operator","_message":"Completed restart of node","cluster":"elasticsearch","namespace":"openshift-logging","node":"elasticsearch-cdm-0p4w6uqm-2"}
{"_ts":"2021-12-16T08:53:32.704392389Z","_level":"0","_component":"elasticsearch-operator","_message":"Beginning restart of node","cluster":"elasticsearch","namespace":"openshift-logging","node":"elasticsearch-cdm-0p4w6uqm-3"}
{"_ts":"2021-12-16T08:53:34.073846333Z","_level":"0","_component":"elasticsearch-operator","_message":"failed to flush nodes","_error":{"msg":"failed to flush shards in preparation for cluster restart","num_failed_shards":3},"cluster":"elasticsearch","namespace":"openshift-logging"}
// Pod running status
// After restarted elasticsearch-cdm-0p4w6uqm-1, elasticsearch-cdm-0p4w6uqm-3 restarted without waiting for elasticsearch-cdm-0p4w6uqm-2 restart completion.
NAME                                                 READY   STATUS              RESTARTS   AGE     IP         
elasticsearch-cdm-0p4w6uqm-1-7dc8f88dbf-8sbzb        2/2     Running             0          66s     10.131.0.42
elasticsearch-cdm-0p4w6uqm-2-bf568467d-dbh55         1/2     Running             0          34s     10.128.2.51
elasticsearch-cdm-0p4w6uqm-3-55d767b5f9-cthnh        0/2     ContainerCreating   0          1s      <none>     
```

4. Sudden all ES pods were failed to start and started logging ES service failure logs from elasicsearch-operator logs.

![update](https://github.com/bysnupy/handson/blob/master/images/esupdate2.png)

```json
// Start logging the ES service failure.
{"_ts":"2021-12-16T08:54:44.442324949Z","_level":"0","_component":"elasticsearch-operator","_message":"failed to perform rolling update","_error":{"msg":"Get \"https://elasticsearch.openshift-logging.svc:9200/_cluster/state/nodes\": local error: tls: bad record MAC"}}
{"_ts":"2021-12-16T08:54:44.56969177Z","_level":"0","_component":"elasticsearch-operator","_message":"Unable to list existing templates in order to reconcile stale ones","_error":{"cluster":"elasticsearch","msg":"failed to get list of index templates","namespace":"openshift-logging","response_body":null,"response_error":{"Op":"Get","URL":"https://elasticsearch.openshift-logging.svc:9200/_template","Err":{"Op":"dial","Net":"tcp","Source":null,"Addr":{"IP":"172.30.171.225","Port":9200,"Zone":""},"Err":{"Syscall":"connect","Err":111}}},"response_status":0}}
{"_ts":"2021-12-16T08:54:44.582109011Z","_level":"0","_component":"elasticsearch-operator","_message":"failed to create index template","_error":{"msg":"failed decoding raw response body into `map[string]estypes.GetIndexTemplate` for elasticsearch in namespace openshift-logging: unexpected end of JSON input"},"mapping":"app"}
{"_ts":"2021-12-16T08:54:44.582188807Z","_level":"0","_component":"elasticsearch-operator_controller_elasticsearch-controller","_message":"Reconciler error","_error":{"msg":"failed decoding raw response body into `map[string]estypes.GetIndexTemplate` for elasticsearch in namespace openshift-logging: unexpected end of JSON input"},"name":"elasticsearch","namespace":"openshift-logging"}
{"_ts":"2021-12-16T08:54:45.955572078Z","_level":"0","_component":"elasticsearch-operator","_message":"unable to update node","_error":{"msg":"Get \"https://elasticsearch.openshift-logging.svc:9200/_cluster/state/nodes\": dial tcp 172.30.171.225:9200: connect: connection refused"},"cluster":"elasticsearch","namespace":"openshift-logging"}
// Pod running status
NAME                                                 READY   STATUS      RESTARTS   AGE     IP 
elasticsearch-cdm-0p4w6uqm-1-7dc8f88dbf-8sbzb        1/2     Running     0          2m14s   10.131.0.42
elasticsearch-cdm-0p4w6uqm-2-bf568467d-dbh55         1/2     Running     0          102s    10.128.2.51
elasticsearch-cdm-0p4w6uqm-3-55d767b5f9-cthnh        1/2     Running     0          69s     10.129.2.68
```

5. For a while, the same error messages were shown, then all the ES pods status transitioned to "Running" and all service is also back to available.
```json
:
{"_ts":"2021-12-16T08:55:15.430282952Z","_level":"0","_component":"elasticsearch-operator","_message":"unable to update node","_error":{"msg":"Get \"https://elasticsearch.openshift-logging.svc:9200/_cluster/state/nodes\": dial tcp 172.30.171.225:9200: connect: connection refused"},"cluster":"elasticsearch","namespace":"openshift-logging"}
{"_ts":"2021-12-16T08:55:15.53258508Z","_level":"0","_component":"elasticsearch-operator","_message":"Unable to list existing templates in order to reconcile stale ones","_error":{"cluster":"elasticsearch","msg":"failed to get list of index templates","namespace":"openshift-logging","response_body":null,"response_error":{"Op":"Get","URL":"https://elasticsearch.openshift-logging.svc:9200/_template","Err":{"Op":"dial","Net":"tcp","Source":null,"Addr":{"IP":"172.30.171.225","Port":9200,"Zone":""},"Err":{"Syscall":"connect","Err":111}}},"response_status":0}}
{"_ts":"2021-12-16T08:55:15.538785654Z","_level":"0","_component":"elasticsearch-operator","_message":"failed to create index template","_error":{"msg":"failed decoding raw response body into `map[string]estypes.GetIndexTemplate` for elasticsearch in namespace openshift-logging: unexpected end of JSON input"},"mapping":"app"}
{"_ts":"2021-12-16T08:55:15.538879867Z","_level":"0","_component":"elasticsearch-operator_controller_elasticsearch-controller","_message":"Reconciler error","_error":{"msg":"failed decoding raw response body into `map[string]estypes.GetIndexTemplate` for elasticsearch in namespace openshift-logging: unexpected end of JSON input"},"name":"elasticsearch","namespace":"openshift-logging"}

{"_ts":"2021-12-16T08:55:17.161161983Z","_level":"0","_component":"elasticsearch-operator","_message":"Completed restart of node","cluster":"elasticsearch","namespace":"openshift-logging","node":"elasticsearch-cdm-0p4w6uqm-3"}
// Pod running status
NAME                                                 READY   STATUS      RESTARTS   AGE     IP 
elasticsearch-cdm-0p4w6uqm-1-7dc8f88dbf-8sbzb        2/2     Running     0          2m50s   10.131.0.42
elasticsearch-cdm-0p4w6uqm-2-bf568467d-dbh55         2/2     Running     0          2m18s   10.128.2.51
elasticsearch-cdm-0p4w6uqm-3-55d767b5f9-cthnh        2/2     Running     0          105s    10.129.2.68
```

## Updated all ES and kibana pods status is as follows.
```cmd
==== 08:55:41
NAME                                                 READY   STATUS      RESTARTS   AGE     IP             NODE                                              NOMINATED NODE   READINESS GATES
cluster-logging-operator-c4c5f4f54-wtnlc             1/1     Running     0          23m     10.129.2.60    ip-10-0-171-140.ap-northeast-1.compute.internal   <none>           <none>
elasticsearch-cdm-0p4w6uqm-1-7dc8f88dbf-8sbzb        2/2     Running     0          3m10s   10.131.0.42    ip-10-0-169-247.ap-northeast-1.compute.internal   <none>           <none>
elasticsearch-cdm-0p4w6uqm-2-bf568467d-dbh55         2/2     Running     0          2m38s   10.128.2.51    ip-10-0-232-167.ap-northeast-1.compute.internal   <none>           <none>
elasticsearch-cdm-0p4w6uqm-3-55d767b5f9-cthnh        2/2     Running     0          2m5s    10.129.2.68    ip-10-0-171-140.ap-northeast-1.compute.internal   <none>           <none>
elasticsearch-im-app-27327405-9jfwb                  0/1     Completed   0          10m     10.131.0.35    ip-10-0-169-247.ap-northeast-1.compute.internal   <none>           <none>
elasticsearch-im-audit-27327405-bllf5                0/1     Completed   0          10m     10.128.2.45    ip-10-0-232-167.ap-northeast-1.compute.internal   <none>           <none>
elasticsearch-im-infra-27327405-7wpww                0/1     Completed   0          10m     10.131.0.34    ip-10-0-169-247.ap-northeast-1.compute.internal   <none>           <none>
fluentd-2d6n8                                        2/2     Running     0          21m     10.128.2.39    ip-10-0-232-167.ap-northeast-1.compute.internal   <none>           <none>
fluentd-4525r                                        2/2     Running     0          21m     10.131.0.32    ip-10-0-169-247.ap-northeast-1.compute.internal   <none>           <none>
fluentd-bzrms                                        2/2     Running     0          21m     10.128.0.53    ip-10-0-177-180.ap-northeast-1.compute.internal   <none>           <none>
fluentd-m9m85                                        2/2     Running     0          21m     10.129.0.58    ip-10-0-248-187.ap-northeast-1.compute.internal   <none>           <none>
fluentd-pvsqk                                        2/2     Running     0          21m     10.129.2.61    ip-10-0-171-140.ap-northeast-1.compute.internal   <none>           <none>
fluentd-sl86n                                        2/2     Running     0          21m     10.130.0.57    ip-10-0-186-82.ap-northeast-1.compute.internal    <none>           <none>
ip-10-0-177-180ap-northeast-1computeinternal-debug   1/1     Running     0          15m     10.0.177.180   ip-10-0-177-180.ap-northeast-1.compute.internal   <none>           <none>
kibana-59bb4f66d7-htczk                              2/2     Running     0          3m24s   10.131.0.41    ip-10-0-169-247.ap-northeast-1.compute.internal   <none>           <none>
```

ES health and index status were all ok.
```cmd
$ oc exec elasticsearch-cdm-0p4w6uqm-1-7dc8f88dbf-8sbzb -- health
Defaulted container "elasticsearch" out of: elasticsearch, proxy
Thu Dec 16 08:57:14 UTC 2021
epoch      timestamp cluster       status node.total node.data shards pri relo init unassign pending_tasks max_task_wait_time active_shards_percent
1639645034 08:57:14  elasticsearch green           3         3     24  12    0    0        0             0                  -                100.0%

$ oc exec elasticsearch-cdm-0p4w6uqm-1-7dc8f88dbf-8sbzb -- indices
Defaulted container "elasticsearch" out of: elasticsearch, proxy
Thu Dec 16 08:57:37 UTC 2021
health status index                          uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   audit-000001                   cnJ3qF57Sf26ZpUq-9hK-w   3   1          0            0          0              0
green  open   .kibana_1                      -pMpB8CqTlSx4BL_sH0i2g   1   1          0            0          0              0
green  open   app-000001                     jE6OIKQVQS6vsrXcpjaJGQ   3   1       1653            0          4              2
green  open   infra-000001                   TgsDGmD4Qie19Dt903GaZQ   3   1      83937            0        159             65
green  open   .kibana_-377444158_kubeadmin_1 6zWEbiVbSQ2rlJG6s6Qhkw   1   1          2            0          0              0
green  open   .security                      r4favAOWTGqZBG2MoZQv7w   1   1          6            0          0              0
```

Done.
