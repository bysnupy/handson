# Single master machine recovery demonstration on OCP4 with AWS IPI

## Summary
As of OCP4.4, the etcd is managed by etcd Operator, and the recovery steps are changed.
I walk you through the new recovery steps as follows.

## Demonstration
We will explore the demonstration in the OCP 4.5.z, and first remove the etcd member and recovery as recreating master Machine.

### Delete the etcd member of the target master node in advance
```console
$ oc get etcd -o=jsonpath='{range .items[0].status.conditions[?(@.type=="EtcdMembersAvailable")]}{.message}{"\n"}'
2 of 3 members are available, ip-10-0-154-78.ap-northeast-1.compute.internal is unhealthy

$ oc get pods -n openshift-etcd | grep etcd
etcd-ip-10-0-161-211.ap-northeast-1.compute.internal                4/4     Running            0          45m
etcd-ip-10-0-199-228.ap-northeast-1.compute.internal                3/4     CrashLoopBackOff   8          19m
etcd-ip-10-0-248-43.ap-northeast-1.compute.internal                 4/4     Running            0          46m

$ oc rsh -n openshift-etcd etcd-ip-10-0-161-211.ap-northeast-1.compute.internal
Defaulting container name to etcdctl.
Use 'oc describe pod/etcd-ip-10-0-161-211.ap-northeast-1.compute.internal -n openshift-etcd' to see all of the containers in this pod.
sh-4.2# etcdctl member list -w table
+------------------+---------+-------------------------------------------------+---------------------------+---------------------------+------------+
|        ID        | STATUS  |                      NAME                       |        PEER ADDRS         |       CLIENT ADDRS        | IS LEARNER |
+------------------+---------+-------------------------------------------------+---------------------------+---------------------------+------------+
| b146504ffd29848c | started |  ip-10-0-154-78.ap-northeast-1.compute.internal |  https://10.0.154.78:2380 |  https://10.0.154.78:2379 |      false |
| ea9f77af22e527f5 | started |  ip-10-0-248-43.ap-northeast-1.compute.internal |  https://10.0.248.43:2380 |  https://10.0.248.43:2379 |      false |
| fabb350465fd9f36 | started | ip-10-0-161-211.ap-northeast-1.compute.internal | https://10.0.161.211:2380 | https://10.0.161.211:2379 |      false |
+------------------+---------+-------------------------------------------------+---------------------------+---------------------------+------------+
sh-4.2# etcdctl member remove b146504ffd29848c
Member b146504ffd29848c removed from cluster eb864dbd6d183739
sh-4.2# etcdctl member list -w table
+------------------+---------+-------------------------------------------------+---------------------------+---------------------------+------------+
|        ID        | STATUS  |                      NAME                       |        PEER ADDRS         |       CLIENT ADDRS        | IS LEARNER |
+------------------+---------+-------------------------------------------------+---------------------------+---------------------------+------------+
| ea9f77af22e527f5 | started |  ip-10-0-248-43.ap-northeast-1.compute.internal |  https://10.0.248.43:2380 |  https://10.0.248.43:2379 |      false |
| fabb350465fd9f36 | started | ip-10-0-161-211.ap-northeast-1.compute.internal | https://10.0.161.211:2380 | https://10.0.161.211:2379 |      false |
+------------------+---------+-------------------------------------------------+---------------------------+---------------------------+------------+
sh-4.2# exit
exit
```

### Delete a master-0 node using "oc delete node" command and remove the related VM instance using "aws ec2 terminate-instances".

// Recovery master host 
```console
// List current master Machines
$ oc get machine -n openshift-machine-api -l machine.openshift.io/cluster-api-machine-role=master
NAME                                              PHASE     TYPE         REGION           ZONE              AGE
dapark-45ipi-pr4mf-master-0                       Running   m5.xlarge    ap-northeast-1   ap-northeast-1c   6d22h
dapark-45ipi-pr4mf-master-1                       Running   m5.xlarge    ap-northeast-1   ap-northeast-1c   6d22h
dapark-45ipi-pr4mf-master-2                       Running   m5.xlarge    ap-northeast-1   ap-northeast-1c   6d22h

// Get the instance ID of the master-0 Machine
$ oc describe machine dapark-45ipi-pr4mf-master-0 -n openshift-machine-api | grep "Provider ID:"
Provider ID:  aws:///ap-northeast-1c/i-07dd8fbf8625f50bd

// Verify the node name based on the instance ID
$ aws ec2 describe-instances --instance-ids i-07dd8fbf8625f50bd | jq '.Reservations[].Instances[] | .PrivateDnsName'
"ip-10-0-154-78.ap-northeast-1.compute.internal"

// Delete the mater-0 node
$ oc delete node ip-10-0-154-78.ap-northeast-1.compute.internal

$ oc get node -l node-role.kubernetes.io/master=
NAME                                              STATUS   ROLES    AGE     VERSION
ip-10-0-161-211.ap-northeast-1.compute.internal   Ready    master   6d22h   v1.18.3+3107688
ip-10-0-248-43.ap-northeast-1.compute.internal    Ready    master   6d22h   v1.18.3+3107688

// Terminate the instance of the master-0 forcefully for recreating master-0 host.
$ aws ec2 terminate-instances --instance-ids i-07dd8fbf8625f50bd
{
    "TerminatingInstances": [
        {
            "InstanceId": "i-07dd8fbf8625f50bd", 
            "CurrentState": {
                "Code": 32, 
                "Name": "shutting-down"
            }, 
            "PreviousState": {
                "Code": 16, 
                "Name": "running"
            }
        }
    ]
}
```

### Create the master Machine again after removing "status:" and "ProviderID:" from existing Machine manifest.
At that time, new etcd member also added to the existing etcd cluster.

```console
$ oc get machine -n openshift-machine-api -l machine.openshift.io/cluster-api-machine-role=master
NAME                          PHASE     TYPE        REGION           ZONE              AGE
dapark-45ipi-pr4mf-master-0   Failed    m5.xlarge   ap-northeast-1   ap-northeast-1c   6d23h
dapark-45ipi-pr4mf-master-1   Running   m5.xlarge   ap-northeast-1   ap-northeast-1c   6d23h
dapark-45ipi-pr4mf-master-2   Running   m5.xlarge   ap-northeast-1   ap-northeast-1c   6d23h

$ oc get machine dapark-45ipi-pr4mf-master-0 -o yaml  >  master-0.yaml
...ProviderID: aws:とstatus:配下のセクションを削除してから保存してください。...

$ oc create -f master-0.yaml
machine.machine.openshift.io/dapark-45ipi-pr4mf-master-0 created

$ oc get machine -n openshift-machine-api -l machine.openshift.io/cluster-api-machine-role=master
NAME                                              PHASE          TYPE         REGION           ZONE              AGE
dapark-45ipi-pr4mf-master-0                       Provisioning   m5.xlarge    ap-northeast-1   ap-northeast-1c   4s
dapark-45ipi-pr4mf-master-1                       Running        m5.xlarge    ap-northeast-1   ap-northeast-1c   6d23h
dapark-45ipi-pr4mf-master-2                       Running        m5.xlarge    ap-northeast-1   ap-northeast-1c   6d23h

$ oc get node -l node-role.kubernetes.io/master=
NAME                                              STATUS   ROLES    AGE    VERSION
ip-10-0-139-138.ap-northeast-1.compute.internal   Ready    master   93m    v1.18.3+3107688
ip-10-0-161-211.ap-northeast-1.compute.internal   Ready    master   7d1h   v1.18.3+3107688
ip-10-0-248-43.ap-northeast-1.compute.internal    Ready    master   7d1h   v1.18.3+3107688

$ oc get pods -n openshift-etcd | grep etcd
etcd-ip-10-0-139-138.ap-northeast-1.compute.internal                4/4     Running     14         49m
etcd-ip-10-0-161-211.ap-northeast-1.compute.internal                4/4     Running     0          58m
etcd-ip-10-0-248-43.ap-northeast-1.compute.internal                 4/4     Running     0          19s

$ oc rsh etcd-ip-10-0-139-138.ap-northeast-1.compute.internal
Defaulting container name to etcdctl.
Use 'oc describe pod/etcd-ip-10-0-139-138.ap-northeast-1.compute.internal -n openshift-etcd' to see all of the containers in this pod.
sh-4.2# etcdctl member list -w table
+------------------+---------+-------------------------------------------------+---------------------------+---------------------------+------------+
|        ID        | STATUS  |                      NAME                       |        PEER ADDRS         |       CLIENT ADDRS        | IS LEARNER |
+------------------+---------+-------------------------------------------------+---------------------------+---------------------------+------------+
| 88a792a65986de5d | started | ip-10-0-139-138.ap-northeast-1.compute.internal | https://10.0.139.138:2380 | https://10.0.139.138:2379 |      false |
| ea9f77af22e527f5 | started |  ip-10-0-248-43.ap-northeast-1.compute.internal |  https://10.0.248.43:2380 |  https://10.0.248.43:2379 |      false |
| fabb350465fd9f36 | started | ip-10-0-161-211.ap-northeast-1.compute.internal | https://10.0.161.211:2380 | https://10.0.161.211:2379 |      false |
+------------------+---------+-------------------------------------------------+---------------------------+---------------------------+------------+
sh-4.2# exit
``` 

Done.
