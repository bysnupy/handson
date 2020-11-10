# Demonstration for "Replacing an unhealthy etcd member"

This article is for demonstration of the "Replacing an unhealthy etcd member" section procedures.

Replacing an unhealthy etcd member
  https://docs.openshift.com/container-platform/4.6/backup_and_restore/replacing-unhealthy-etcd-member.html

## Test scenario

After removing one master node vm on a hypervisor host, I try the restore of the lost master.
It will be a majority of your masters still available and have an etcd quorum after that. It's a expected status for this test.

## Environments

* OCP version: v4.6.1
* Platform and Installation method: Bare-metal UPI
* Cluster size: masterx3, workerx3

## etcd backup before test

I will take a backup of etcd and VM snapshot for insurance.

Taking etcd backup on a master node.
```console
# oc debug node/master2.ocp46rt.priv.local
Creating debug namespace/openshift-debug-node-xggss ...
Starting pod/master2ocp46rtprivlocal-debug ...
To use host binaries, run `chroot /host`
Pod IP: 192.168.9.33
If you don't see a command prompt, try pressing enter.
sh-4.4# chroot /host
sh-4.4# /usr/local/bin/cluster-backup.sh /home/core/assets/backup
466ab202f016df2f98f0327e39e5e219868d501fcc98a32220b7f8083a81954d
etcdctl version: 3.4.9
API version: 3.4
found latest kube-apiserver-pod: /etc/kubernetes/static-pod-resources/kube-apiserver-pod-17
found latest kube-controller-manager-pod: /etc/kubernetes/static-pod-resources/kube-controller-manager-pod-14
found latest kube-scheduler-pod: /etc/kubernetes/static-pod-resources/kube-scheduler-pod-15
found latest etcd-pod: /etc/kubernetes/static-pod-resources/etcd-pod-3
{"level":"info","ts":1604971892.992505,"caller":"snapshot/v3_snapshot.go:119","msg":"created temporary db file","path":"/home/core/assets/backup/snapshot_2020-11-10_013131.db.part"}
{"level":"info","ts":"2020-11-10T01:31:33.006Z","caller":"clientv3/maintenance.go:200","msg":"opened snapshot stream; downloading"}
{"level":"info","ts":1604971893.007328,"caller":"snapshot/v3_snapshot.go:127","msg":"fetching snapshot","endpoint":"https://192.168.9.33:2379"}
{"level":"info","ts":"2020-11-10T01:31:34.013Z","caller":"clientv3/maintenance.go:208","msg":"completed snapshot read; closing"}
{"level":"info","ts":1604971894.1334417,"caller":"snapshot/v3_snapshot.go:142","msg":"fetched snapshot","endpoint":"https://192.168.9.33:2379","size":"77 MB","took":1.140878497}
{"level":"info","ts":1604971894.1406617,"caller":"snapshot/v3_snapshot.go:152","msg":"saved","path":"/home/core/assets/backup/snapshot_2020-11-10_013131.db"}
Snapshot saved at /home/core/assets/backup/snapshot_2020-11-10_013131.db
snapshot db and kube resources are successfully saved to /home/core/assets/backup
sh-4.4# chown core:core /home/core/assets/backup -R
```

Transfer the etcd backup to local host. You can copy the backup using scp or any ways.
```console
# scp -r -i ~/.ssh/id_rsa core@master2.ocp46rt.priv.local:/home/core/assets/backup ./etcd_backup/
static_kuberesources_2020-11-10_013131.tar.gz                                                                                                             100%   66KB  17.5MB/s   00:00    
snapshot_2020-11-10_013131.db                                                                                                                             100%   74MB  87.7MB/s   00:00  
```

## Remove a master VM

I will remove master1 node VM in this test.
Before removing master1,
```console
# oc get node
NAME                               STATUS   ROLES    AGE     VERSION
master1.ocp46rt.priv.local         Ready    master   6d12h   v1.19.0+d59ce34
master2.ocp46rt.priv.local         Ready    master   6d11h   v1.19.0+d59ce34
master3.ocp46rt.priv.local         Ready    master   6d10h   v1.19.0+d59ce34
worker1.ocp46rt.priv.local         Ready    worker   6d9h    v1.19.0+d59ce34
worker2.ocp46rt.priv.local         Ready    worker   6d9h    v1.19.0+d59ce34
worker3.ocp46rt.priv.local         Ready    worker   6d9h    v1.19.0+d59ce34
```

After removing master1,
```console
# oc get node
NAME                               STATUS     ROLES    AGE     VERSION
master1.ocp46rt.priv.local         NotReady   master   6d12h   v1.19.0+d59ce34
master2.ocp46rt.priv.local         Ready      master   6d11h   v1.19.0+d59ce34
master3.ocp46rt.priv.local         Ready      master   6d10h   v1.19.0+d59ce34
worker1.ocp46rt.priv.local         Ready      worker   6d9h    v1.19.0+d59ce34
worker2.ocp46rt.priv.local         Ready      worker   6d9h    v1.19.0+d59ce34
worker3.ocp46rt.priv.local         Ready      worker   6d9h    v1.19.0+d59ce34
```

## Start to replacing removed master1 with new one.

```console
# oc get etcd -o=jsonpath='{range .items[0].status.conditions[?(@.type=="EtcdMembersAvailable")]}{.message}{"\n"}'
2 of 3 members are available, master1.ocp46rt.priv.local is unhealthy

# oc get nodes -o jsonpath='{range .items[*]}{"\n"}{.metadata.name}{"\t"}{range .spec.taints[*]}{.key}{" "}' | grep unreachable
master1.ocp46rt.priv.local	node-role.kubernetes.io/master node.kubernetes.io/unreachable node.kubernetes.io/unreachable 

# oc get nodes -l node-role.kubernetes.io/master | grep "NotReady"
master1.ocp46rt.priv.local   NotReady   master   6d13h   v1.19.0+d59ce34

# oc get pods -n openshift-etcd -o wide | grep etcd
etcd-master1.ocp46rt.priv.local                3/3     Running     0          6d11h   192.168.9.32   master1.ocp46rt.priv.local   <none>           <none>  <--- WRONG
etcd-master2.ocp46rt.priv.local                3/3     Running     0          6d11h   192.168.9.33   master2.ocp46rt.priv.local   <none>           <none>
etcd-master3.ocp46rt.priv.local                3/3     Running     0          6d11h   192.168.9.34   master3.ocp46rt.priv.local   <none>           <none>
etcd-quorum-guard-644f5747b8-89gx6             1/1     Running     0          5d21h   192.168.9.34   master3.ocp46rt.priv.local   <none>           <none>
etcd-quorum-guard-644f5747b8-knsq9             1/1     Running     0          5d21h   192.168.9.33   master2.ocp46rt.priv.local   <none>           <none>
etcd-quorum-guard-644f5747b8-tm922             1/1     Running     0          5d21h   192.168.9.32   master1.ocp46rt.priv.local   <none>           <none>

# oc logs --all-containers -n openshift-etcd etcd-master1.ocp46rt.priv.local
Error from server: Get "https://192.168.9.32:10250/containerLogs/openshift-etcd/etcd-master1.ocp46rt.priv.local/etcd-ensure-env-vars": dial tcp 192.168.9.32:10250: connect: no route to host
```

### Install new master1

In this case, I will reuse the removed master1 hostname and IP address for new master1 VM.

```console
# oc get node
NAME                               STATUS     ROLES    AGE     VERSION
master1.ocp46rt.priv.local         NotReady   master   6d14h   v1.19.0+d59ce34
master2.ocp46rt.priv.local         Ready      master   6d13h   v1.19.0+d59ce34
master3.ocp46rt.priv.local         Ready      master   6d13h   v1.19.0+d59ce34
worker1.ocp46rt.priv.local         Ready      worker   6d12h   v1.19.0+d59ce34
worker2.ocp46rt.priv.local         Ready      worker   6d12h   v1.19.0+d59ce34
worker3.ocp46rt.priv.local         Ready      worker   6d12h   v1.19.0+d59ce34

# oc get csr
NAME        AGE    SIGNERNAME                                    REQUESTOR                                                                   CONDITION
csr-26tm6   2m3s   kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Pending
csr-mwj77   17m    kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Pending
csr-wz5qw   32m    kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Pending

// approve new master1 client certificate
# oc get csr -o name | xargs oc adm certificate approve
certificatesigningrequest.certificates.k8s.io/csr-26tm6 approved
certificatesigningrequest.certificates.k8s.io/csr-mwj77 approved
certificatesigningrequest.certificates.k8s.io/csr-wz5qw approved

// approve new master1 server certificate
# oc get csr
NAME        AGE     SIGNERNAME                                    REQUESTOR                                                                   CONDITION
csr-26tm6   2m37s   kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Approved,Issued
csr-mwj77   17m     kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Approved,Issued
csr-qplrj   0s      kubernetes.io/kubelet-serving                 system:node:master1.ocp46rt.priv.local                                      Pending
csr-wz5qw   32m     kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Approved,Issued

#  oc get csr -o name | xargs oc adm certificate approve
certificatesigningrequest.certificates.k8s.io/csr-26tm6 approved
certificatesigningrequest.certificates.k8s.io/csr-mwj77 approved
certificatesigningrequest.certificates.k8s.io/csr-qplrj approved
certificatesigningrequest.certificates.k8s.io/csr-wz5qw approved

# oc get node
NAME                               STATUS   ROLES    AGE     VERSION
master1.ocp46rt.priv.local         Ready    master   6d14h   v1.19.0+d59ce34
master2.ocp46rt.priv.local         Ready    master   6d13h   v1.19.0+d59ce34
master3.ocp46rt.priv.local         Ready    master   6d13h   v1.19.0+d59ce34
worker1.ocp46rt.priv.local         Ready    worker   6d12h   v1.19.0+d59ce34
worker2.ocp46rt.priv.local         Ready    worker   6d12h   v1.19.0+d59ce34
worker3.ocp46rt.priv.local         Ready    worker   6d12h   v1.19.0+d59ce34
```

### Restore etcd

After master1 node restore, missing etcd-master1 pod
```console
# oc get pods -n openshift-etcd | grep etcd
etcd-master2.ocp46rt.priv.local                3/3     Running     0          6d13h
etcd-master3.ocp46rt.priv.local                3/3     Running     0          6d13h
etcd-quorum-guard-644f5747b8-89gx6             1/1     Running     0          5d23h
etcd-quorum-guard-644f5747b8-knsq9             1/1     Running     0          5d23h
etcd-quorum-guard-644f5747b8-tm922             0/1     Running     0          5d23h
```

Remove old unhealthy etcd member
```console
# oc rsh -n openshift-etcd etcd-master3.ocp46rt.priv.local
Defaulting container name to etcdctl.
Use 'oc describe pod/etcd-master3.ocp46rt.priv.local -n openshift-etcd' to see all of the containers in this pod.

sh-4.4# etcdctl member list -w table
+------------------+---------+----------------------------+---------------------------+---------------------------+------------+
|        ID        | STATUS  |            NAME            |        PEER ADDRS         |       CLIENT ADDRS        | IS LEARNER |
+------------------+---------+----------------------------+---------------------------+---------------------------+------------+
| 2a1dbc31cb884893 | started | master2.ocp46rt.priv.local | https://192.168.9.33:2380 | https://192.168.9.33:2379 |      false |
| 33da87c182583cb0 | started | master1.ocp46rt.priv.local | https://192.168.9.32:2380 | https://192.168.9.32:2379 |      false |
| f167bc227aff249f | started | master3.ocp46rt.priv.local | https://192.168.9.34:2380 | https://192.168.9.34:2379 |      false |
+------------------+---------+----------------------------+---------------------------+---------------------------+------------+

sh-4.4# etcdctl member remove 33da87c182583cb0
Member 33da87c182583cb0 removed from cluster  4d62032bcb36a17

sh-4.4# etcdctl member list -w table
+------------------+---------+----------------------------+---------------------------+---------------------------+------------+
|        ID        | STATUS  |            NAME            |        PEER ADDRS         |       CLIENT ADDRS        | IS LEARNER |
+------------------+---------+----------------------------+---------------------------+---------------------------+------------+
| 2a1dbc31cb884893 | started | master2.ocp46rt.priv.local | https://192.168.9.33:2380 | https://192.168.9.33:2379 |      false |
| f167bc227aff249f | started | master3.ocp46rt.priv.local | https://192.168.9.34:2380 | https://192.168.9.34:2379 |      false |
+------------------+---------+----------------------------+---------------------------+---------------------------+------------+
sh-4.4# 
```

Remove the old secrets for the unhealthy etcd member that was removed
```console
# oc get secrets -n openshift-etcd | grep master1
etcd-peer-master1.ocp46rt.priv.local              kubernetes.io/tls                     2      6d15h
etcd-serving-master1.ocp46rt.priv.local           kubernetes.io/tls                     2      6d15h
etcd-serving-metrics-master1.ocp46rt.priv.local   kubernetes.io/tls                     2      6d15h

# oc get secrets -n openshift-etcd -o name | grep master1 | xargs oc delete 
secret "etcd-peer-master1.ocp46rt.priv.local" deleted
secret "etcd-serving-master1.ocp46rt.priv.local" deleted
secret "etcd-serving-metrics-master1.ocp46rt.priv.local" deleted
```

After removing the secrets, you can see the new master1 etcd pod is running as follows.
```console
# oc get pods -n openshift-etcd | grep etcd
etcd-master1.ocp46rt.priv.local                3/3     Running             0          18s
etcd-master2.ocp46rt.priv.local                3/3     Running             0          6d13h
etcd-master3.ocp46rt.priv.local                3/3     Running             0          6d13h
etcd-quorum-guard-644f5747b8-89gx6             1/1     Running             0          5d23h
etcd-quorum-guard-644f5747b8-knsq9             1/1     Running             0          5d23h
etcd-quorum-guard-644f5747b8-tm922             1/1     Running             0          5d23h
```

If the output from the previous command only lists two pods, you can manually force an etcd redeployment.
In a terminal that has access to the cluster as a cluster-admin user, run the following command:

```console
$ oc patch etcd cluster -p='{"spec": {"forceRedeploymentReason": "recovery-'"$( date --rfc-3339=ns )"'
```

## Complete the restore tasks and verify the etcd status

```console
# oc rsh etcd-master3.ocp46rt.priv.local
Defaulting container name to etcdctl.
Use 'oc describe pod/etcd-master3.ocp46rt.priv.local -n openshift-etcd' to see all of the containers in this pod.
sh-4.4# etcdctl member list -w table
+------------------+---------+----------------------------+---------------------------+---------------------------+------------+
|        ID        | STATUS  |            NAME            |        PEER ADDRS         |       CLIENT ADDRS        | IS LEARNER |
+------------------+---------+----------------------------+---------------------------+---------------------------+------------+
| 2a1dbc31cb884893 | started | master2.ocp46rt.priv.local | https://192.168.9.33:2380 | https://192.168.9.33:2379 |      false |
| f086625958b22a3a | started | master1.ocp46rt.priv.local | https://192.168.9.32:2380 | https://192.168.9.32:2379 |      false |
| f167bc227aff249f | started | master3.ocp46rt.priv.local | https://192.168.9.34:2380 | https://192.168.9.34:2379 |      false |
+------------------+---------+----------------------------+---------------------------+---------------------------+------------+

sh-4.4# etcdctl endpoint health --cluster
https://192.168.9.34:2379 is healthy: successfully committed proposal: took = 16.820627ms
https://192.168.9.33:2379 is healthy: successfully committed proposal: took = 17.188852ms
https://192.168.9.32:2379 is healthy: successfully committed proposal: took = 22.752886ms
```

Done.
