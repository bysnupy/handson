# Recovery using certificates backup in lost one master node

## Summary

When one master node(ip-10-0-68-220 in this case) is lost due to terminating operation on AWS dashboard, 
you can restore new master and etcd through following recovery procedures.

## Environments

OpenShift 4.3 AWS IPI, master x3, worker x2.


The master nodes are as follows. 
```cmd
$ oc get nodes -l node-role.kubernetes.io/master=
NAME                                            STATUS   ROLES    AGE   VERSION
ip-10-0-68-185.ap-northeast-1.compute.internal   Ready    master   30m   v1.16.2
ip-10-0-68-220.ap-northeast-1.compute.internal   Ready    master   29m   v1.16.2
ip-10-0-68-39.ap-northeast-1.compute.internal    Ready    master   29m   v1.16.2
```

The etcd member list is as follows.
```cmd
sh-4.2# etcdctl member list -w table
+------------------+---------+-----------------------------------------------------------+----------------------------------------------------+-------------------------+
|        ID        | STATUS  |                           NAME                            |                     PEER ADDRS                     |      CLIENT ADDRS       |
+------------------+---------+-----------------------------------------------------------+----------------------------------------------------+-------------------------+
| 36393ece354b4c2a | started | etcd-member-ip-10-0-68-220.ap-northeast-1.compute.internal | https://etcd-1.dapark-ocp43ipi.example.com:2380 | https://10.0.68.220:2379 |
| 7270d5446b8a3c49 | started | etcd-member-ip-10-0-68-185.ap-northeast-1.compute.internal | https://etcd-0.dapark-ocp43ipi.example.com:2380 | https://10.0.68.185:2379 |
| f93356c11e58e088 | started |  etcd-member-ip-10-0-68-39.ap-northeast-1.compute.internal | https://etcd-2.dapark-ocp43ipi.example.com:2380 |  https://10.0.68.39:2379 |
+------------------+---------+-----------------------------------------------------------+----------------------------------------------------+-------------------------+
```

## Recovery procedures

### Prerequisites

You should backup each etcd certificate set before this work, because we restore a etcd using the backup certificate set.

The following command can take the etcd snapshot db file from an etcd, you need not to execute this script on all the etcd hosts.

```cmd
$ sudo /usr/local/bin/etcd-snapshot-backup.sh ./assets/backup
$ sudo tar zcvf assets.tar.gz $HOME/assets
```

But you should take all etcd certificate set from "/etc/kubernetes/static-pod-resorces/etcd-member".

```cmd
$ sudo tar zcvf $(hostname -f)_etcd_certificates.tar.gz /etc/kubernetes/static-pod-resources/etcd-member
```

Copy the backup and etcd cetificate sets to your bastion server.

```cmd
bastion ~$ ls -1
assets.tar.gz
ip-10-0-68-185.ap-northeast-1.compute.internal_etcd_certificates.tar.gz
ip-10-0-68-220.ap-northeast-1.compute.internal_etcd_certificates.tar.gz
ip-10-0-68-39.ap-northeast-1.compute.internal_etcd_certificates.tar.gz
```

## Terminate a master node(ip-10-0-68-220)

```cmd
// search the instance ID of ip-10-0-68-220 and terminate it with "aws" CLI
$ aws ec2 describe-instances | less
$ aws ec2 terminate-instances --instance-ids <instance ID of ip-10-0-68-220>
```

## Check the master machine status after terminating the instance of that

"Machine" resource is used to reconcile every 10 minutes, so it may take time to reflect the changes.

```cmd
$ oc get machine -l machine.openshift.io/cluster-api-machine-type=master
NAME                             PHASE     TYPE        REGION           ZONE              AGE
dapark-ocp43ipi-2n8rc-master-0   Running   m4.xlarge   ap-northeast-1   ap-northeast-1b   57m
dapark-ocp43ipi-2n8rc-master-1   Failed    m4.xlarge   ap-northeast-1   ap-northeast-1b   57m
dapark-ocp43ipi-2n8rc-master-2   Running   m4.xlarge   ap-northeast-1   ap-northeast-1b   57m
```

## Create new master node after delete the failed existing one

Backup the current "Master" resource and create it again using the manifest you modified to remove "status:", "providerID:", "annotations:" sections.

```cmd
$ oc get machine/dapark-ocp43ipi-2n8rc-master-1 -o yaml > master-1.yaml
// Remove "status:", "providerID:" and "annotations:" section from the yaml manifest.
$ vim edit master-1.yaml
$ oc delete machine/dapark-ocp43ipi-2n8rc-master-1
$ oc create -f ./master-1.yaml
$ oc get machine -l machine.openshift.io/cluster-api-machine-type=master
NAME                             PHASE         TYPE        REGION           ZONE              AGE
dapark-ocp43ipi-2n8rc-master-0   Running       m4.xlarge   ap-northeast-1   ap-northeast-1b   69m
dapark-ocp43ipi-2n8rc-master-1   Provisioned   m4.xlarge   ap-northeast-1   ap-northeast-1b   26s
dapark-ocp43ipi-2n8rc-master-2   Running       m4.xlarge   ap-northeast-1   ap-northeast-1b   69m
```

## Modify the etcd record as new master private IP in Route 53 on AWS after running new master node

```cmd
$ oc get node -l node-role.kubernetes.io/master=
NAME                                            STATUS   ROLES    AGE   VERSION
ip-10-0-68-185.ap-northeast-1.compute.internal   Ready    master   74m   v1.16.2
ip-10-0-68-253.ap-northeast-1.compute.internal   Ready    master   69s   v1.16.2   <--- new master node
ip-10-0-68-39.ap-northeast-1.compute.internal    Ready    master   73m   v1.16.2
```

Modify etcd-1 record change as new master node private IP as follows.

```cmd
etcd-1.dapark-ocp43ipi.example.com:
  10.0.68.220 -> 10.0.68.253
```

## Remove the etcd member on terminated master node and add new etcd member to existing etcd cluster

```cmd
$ oc get pod
NAME                                                        READY   STATUS     RESTARTS   AGE
etcd-member-ip-10-0-68-185.ap-northeast-1.compute.internal   2/2     Running    0          76m
etcd-member-ip-10-0-68-253.ap-northeast-1.compute.internal   0/2     Init:1/2   1          4m47s
etcd-member-ip-10-0-68-39.ap-northeast-1.compute.internal    2/2     Running    0          76m
```

Access any master node host where a etcd is running well on and remove the etcd on terminated masster node as follows.
```cmd
$ sudo oc login -u kubeadmin https://localhost:6443 --insecure-skip-tls-verify
$ sudo -E /usr/local/bin/etcd-member-remove.sh etcd-member-ip-10-0-68-220.ap-northeast-1.compute.internal
Creating asset directory ./assets
a5634afc2428b2e7c44dc6fb37085d8e954827469b0807d14fae36f9595d670f
etcdctl version: 3.3.17
API version: 3.3
Trying to backup etcd client certs..
etcd client certs found in /etc/kubernetes/static-pod-resources/kube-apiserver-pod-5 backing up to ./assets/backup/
Member 36393ece354b4c2a removed from cluster c3c85a56b49a373e
etcd member etcd-member-ip-10-0-68-220.ap-northeast-1.compute.internal with 36393ece354b4c2a successfully removed..
```

## Add new etcd to existing etcd cluster as etcd-1 with the backuped certificate set.

WATCH OUT, you should execute the following commands on the new master node host, and transport backuped certificates and assets to new master node in advance.

```cmd
$ sudo tar zxvf assets.tar.gz -C .
$ sudo tar zxvf ip-10-0-68-220.ap-northeast-1.compute.internal_etcd_certificates.tar.gz
$ sudo -i
# cd /etc/kubernetes/static-pod-resources/etcd-member/
# cp -a /home/core/etc/kubernetes/static-pod-resources/etcd-member/* .
# exit
$ sudo oc login -u kubeadmin https://localhost:6443 --insecure-skip-tls-verify
$ sudo -E /usr/local/bin/etcd-member-add.sh 10.0.68.39 etcd-member-ip-10-0-68-253.ap-northeast-1.compute.internal
7f9257bf34686b67f845adf377fa6790dd83c67553ce03fe48218bfde5e4783e
etcdctl version: 3.3.17
API version: 3.3
etcd-member.yaml found in ./assets/backup/
etcd.conf backup upready exists ./assets/backup/etcd.conf
Trying to backup etcd client certs..
etcd client certs already backed up and available ./assets/backup/
Stopping etcd..
Waiting for etcd-member to stop
Backing up etcd data-dir..
Updating etcd membership..
Removing etcd data_dir /var/lib/etcd..
Member d188c27ec07e24b8 added to cluster c3c85a56b49a373e

ETCD_NAME="etcd-member-ip-10-0-68-253.ap-northeast-1.compute.internal"
ETCD_INITIAL_CLUSTER="etcd-member-ip-10-0-68-185.ap-northeast-1.compute.internal=https://etcd-0.dapark-ocp43ipi.example.com:2380,etcd-member-ip-10-0-68-253.ap-northeast-1.compute.internal=https://etcd-1.dapark-ocp43ipi.example.com:2380,etcd-member-ip-10-0-68-39.ap-northeast-1.compute.internal=https://etcd-2.dapark-ocp43ipi.example.com:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://etcd-1.dapark-ocp43ipi.example.com:2380"
ETCD_INITIAL_CLUSTER_STATE="existing"
Starting etcd..
```

You can remove pending CSR for etcd-1 generated when new master starts with "oc delete csr <csr name of new etcd>".

## Verify whether the added etcd and master are recovered without any issue

```cmd
$ oc get pod -n openshift-etcd
NAME                                                        READY   STATUS    RESTARTS   AGE
etcd-member-ip-10-0-68-185.ap-northeast-1.compute.internal   2/2     Running   0          98m
etcd-member-ip-10-0-68-253.ap-northeast-1.compute.internal   2/2     Running   3          24s
etcd-member-ip-10-0-68-39.ap-northeast-1.compute.internal    2/2     Running   0          98m

$ oc get node -l node-role.kubernetes.io/master=
NAME                                            STATUS   ROLES    AGE    VERSION
ip-10-0-68-185.ap-northeast-1.compute.internal   Ready    master   101m   v1.16.2
ip-10-0-68-253.ap-northeast-1.compute.internal   Ready    master   28m    v1.16.2
ip-10-0-68-39.ap-northeast-1.compute.internal    Ready    master   100m   v1.16.2
```

```cmd
sh-4.2#  etcdctl member list -w table
+------------------+---------+-----------------------------------------------------------+----------------------------------------------------+-------------------------+
|        ID        | STATUS  |                           NAME                            |                     PEER ADDRS                     |      CLIENT ADDRS       |
+------------------+---------+-----------------------------------------------------------+----------------------------------------------------+-------------------------+
| 7270d5446b8a3c49 | started | etcd-member-ip-10-0-68-185.ap-northeast-1.compute.internal | https://etcd-0.dapark-ocp43ipi.example.com:2380 | https://10.0.68.185:2379 |
| d188c27ec07e24b8 | started | etcd-member-ip-10-0-68-253.ap-northeast-1.compute.internal | https://etcd-1.dapark-ocp43ipi.example.com:2380 | https://10.0.68.253:2379 |
| f93356c11e58e088 | started |  etcd-member-ip-10-0-68-39.ap-northeast-1.compute.internal | https://etcd-2.dapark-ocp43ipi.example.com:2380 |  https://10.0.68.39:2379 |
+------------------+---------+-----------------------------------------------------------+----------------------------------------------------+-------------------------+
```
