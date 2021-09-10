# Tested automated etcd Defragmentation

# Test procedures

1. Check the current etcd DB size.
```cmd
$ oc rsh etcd-ip-10-0-133-114.ap-southeast-2.compute.internal etcdctl endpoint status -w table
Defaulted container "etcdctl" out of: etcdctl, etcd, etcd-metrics, etcd-health-monitor, setup (init), etcd-ensure-env-vars (init), etcd-resources-copy (init)
+---------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|         ENDPOINT          |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+---------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| https://10.0.133.114:2379 | 5fba30d8bdfd4dbe |   3.5.0 |   88 MB |      true |      false |         5 |     204728 |             204728 |        |
| https://10.0.162.213:2379 | 80bc2a8cc88520cd |   3.5.0 |   88 MB |     false |      false |         5 |     204728 |             204728 |        |
| https://10.0.216.121:2379 | 381d6ccf16f737e3 |   3.5.0 |   88 MB |     false |      false |         5 |     204728 |             204728 |        |
+---------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
```
2. Create some objects to increases the DB size.
```cmd
$ for i in {1..500}; do oc new-project test-project-$i ; done
```
3. Check the Defragmentation logs at the etcd operator as follows
```cmd
I0910 04:19:33.431866       1 event.go:282] Event(v1.ObjectReference{Kind:"Deployment", Namespace:"openshift-etcd-operator", Name:"etcd-operator", UID:"f2cd5f82-690e-4ebd-89e9-a276ff692919", APIVersion:"apps/v1", ResourceVersion:"", FieldPath:""}): type: 'Normal' reason: 'DefragControllerDefragmentSuccess' etcd member has been defragmented: ip-10-0-216-121.ap-southeast-2.compute.internal, memberID: 4043507677147903971
I0910 04:19:35.450261       1 defragcontroller.go:198] etcd member "ip-10-0-162-213.ap-southeast-2.compute.internal" backend store fragmented: 51.50 %, dbSize: 138854400
I0910 04:19:35.450452       1 event.go:282] Event(v1.ObjectReference{Kind:"Deployment", Namespace:"openshift-etcd-operator", Name:"etcd-operator", UID:"f2cd5f82-690e-4ebd-89e9-a276ff692919", APIVersion:"apps/v1", ResourceVersion:"", FieldPath:""}): type: 'Normal' reason: 'DefragControllerDefragmentAttempt' Attempting defrag on member: ip-10-0-162-213.ap-southeast-2.compute.internal, memberID: 9276336116624335053, dbSize: 138854400, dbInUse: 67342336, leader ID: 6897879486729899454
I0910 04:19:36.064292       1 event.go:282] Event(v1.ObjectReference{Kind:"Deployment", Namespace:"openshift-etcd-operator", Name:"etcd-operator", UID:"f2cd5f82-690e-4ebd-89e9-a276ff692919", APIVersion:"apps/v1", ResourceVersion:"", FieldPath:""}): type: 'Normal' reason: 'DefragControllerDefragmentSuccess' etcd member has been defragmented: ip-10-0-162-213.ap-southeast-2.compute.internal, memberID: 9276336116624335053
I0910 04:19:38.092141       1 defragcontroller.go:198] etcd member "ip-10-0-133-114.ap-southeast-2.compute.internal" backend store fragmented: 51.45 %, dbSize: 138878976
I0910 04:19:38.092348       1 event.go:282] Event(v1.ObjectReference{Kind:"Deployment", Namespace:"openshift-etcd-operator", Name:"etcd-operator", UID:"f2cd5f82-690e-4ebd-89e9-a276ff692919", APIVersion:"apps/v1", ResourceVersion:"", FieldPath:""}): type: 'Normal' reason: 'DefragControllerDefragmentAttempt' Attempting defrag on member: ip-10-0-133-114.ap-southeast-2.compute.internal, memberID: 6897879486729899454, dbSize: 138878976, dbInUse: 67424256, leader ID: 6897879486729899454
I0910 04:19:38.687526       1 event.go:282] Event(v1.ObjectReference{Kind:"Deployment", Namespace:"openshift-etcd-operator", Name:"etcd-operator", UID:"f2cd5f82-690e-4ebd-89e9-a276ff692919", APIVersion:"apps/v1", ResourceVersion:"", FieldPath:""}): type: 'Normal' reason: 'DefragControllerDefragmentSuccess' etcd member has been defragmented: ip-10-0-133-114.ap-southeast-2.compute.internal, memberID: 6897879486729899454
:
I0910 06:16:32.818568       1 defragcontroller.go:198] etcd member "ip-10-0-216-121.ap-southeast-2.compute.internal" backend store fragmented: 6.55 %, dbSize: 181358592
I0910 06:16:32.818593       1 defragcontroller.go:198] etcd member "ip-10-0-162-213.ap-southeast-2.compute.internal" backend store fragmented: 6.53 %, dbSize: 181383168
I0910 06:16:32.818599       1 defragcontroller.go:198] etcd member "ip-10-0-133-114.ap-southeast-2.compute.internal" backend store fragmented: 6.57 %, dbSize: 181387264
```
