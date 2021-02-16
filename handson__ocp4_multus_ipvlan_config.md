# Management for the multiple networks using the Multus CNI with ipvlan in OpenShift

## Summary

In this article, I will show you how to configure the multiple networks in OpenShift using the Multus CNI based on ipvlan plug-in with other optional CNI plug-ins.
Personally, I think the ipvlan CNI plug-in is the most useful and easy in most cases and environments due to less restrictions to use, I hope this demonstration to help your implementations.
If you need more information about CNI, then it's helpful to refer [Using the Multus CNI in OpenShift](https://www.openshift.com/blog/using-the-multus-cni-in-openshift) first before reading this post.

## What do I show you ?

I will demonstrate to add two ipvlan interface for a test pod, and test to connect each external network configured different networks. The following figure describes this in details.

![ipvlan_flow](https://github.com/bysnupy/handson/blob/master/ocp4__ipvlan_flow_diagram.png)

## Configurations

Add two additional networks that have a routing rule and bandwidth limitation(in/out 1Kbps) on each of that through the Cluster Network Operator(CNO) CustomResource(CR) as follows.

```console
// Create test-ipvlan namespace before adding networks.
$ oc new-project test-ipvlan

$ oc edit networks.operator.openshift.io cluster
apiVersion: operator.openshift.io/v1
kind: Network
metadata:
  name: cluster
spec:
  additionalNetworks: 
  - name: net1
    namespace: test-ipvlan
    type: Raw
    rawCNIConfig: '{
      "cniVersion": "0.3.1",
      "name": "net1",
      "capabilities": {
        "ips": true
      },
      "plugins": [
        {
          "type": "ipvlan",
          "mode": "l2",
          "master": "ens4",
          "ipam": {
            "type": "static"
          }
        },
        {
          "type": "route-override",
          "addroutes": [
            {
              "dst": "192.168.8.0/24",
              "gw": "192.168.12.1"
            }
          ]
        }
      ]
    }'
  - name: net2
    namespace: test-ipvlan
    type: Raw
    rawCNIConfig: '{
      "cniVersion": "0.3.1",
      "name": "net2",
      "plugins": [
        {
          "type": "ipvlan",
          "mode": "l2",
          "master": "ens4",
          "capabilities": {
            "ips": true
          },
          "ipam": {
            "type": "static"
          }
        },
        {
          "type": "route-override",
          "addroutes": [
            {
              "dst": "192.168.9.0/24",
              "gw": "192.168.12.1"
            }
          ]
        }
      ]
    }'
    ...snip...

// List the created NetworkAttachmentDefinition by CNO as follows.
$ oc get net-attach-def -n test-ipvlan
NAME   AGE
net1   3m23s
net2   3m23s
```

Create a test pod with the two networks as follows.
```console
$ oc create -n test-ipvlan -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  annotations:
    k8s.v1.cni.cncf.io/networks: '[
      {
        "name": "net1", 
        "ips": ["192.168.12.80/24"]
      },
      {
        "name": "net2",
        "ips": ["192.168.12.90/24"]
      }
    ]'
  name: test-pod
spec:
  containers:
  - args:
    - bash
    - -c
    - mkdir -p /tmp/test; cd /tmp/test; echo "Test Pod" > index.html; python -m SimpleHTTPServer 8080
    image: registry.redhat.io/rhel7
    name: test-pod
    ports:
    - containerPort: 8080
      name: web
      protocol: TCP
EOF

$  oc get pod -n test-ipvlan
NAME       READY   STATUS    RESTARTS   AGE   IP             NODE       
test-pod   1/1     Running   0          91s   10.128.2.193   worker.ocp4.example.com
```

Verify the added network interfaces on the scheduled node host using SSH or "oc debug node/NODENAME".
```console
// net1 and net2 has been added with specified IP addresses using annotations.
node ~# ip netns exec 719aa0a6-f1e1-4385-a42b-13f652de99ef ip -4 a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
3: eth0@if698: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default  link-netnsid 0
    inet 10.128.2.193/23 brd 10.128.3.255 scope global eth0
       valid_lft forever preferred_lft forever
4: net1@eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default 
    inet 192.168.12.80/24 brd 192.168.12.255 scope global net1
       valid_lft forever preferred_lft forever
5: net2@eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default 
    inet 192.168.12.90/24 brd 192.168.12.255 scope global net2
       valid_lft forever preferred_lft forever

// The added routing rules can list as follows.
node ~# ip netns exec 719aa0a6-f1e1-4385-a42b-13f652de99ef route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.0.0.1        0.0.0.0         UG    0      0        0 eth0
:
192.168.8.0     192.168.12.1    255.255.255.0   UG    0      0        0 net1
192.168.9.0     192.168.12.1    255.255.255.0   UG    0      0        0 net2
192.168.12.0    0.0.0.0         255.255.255.0   U     0      0        0 net1
192.168.12.0    0.0.0.0         255.255.255.0   U     0      0        0 net2
:
```

Configured routing rule test as follows.
```console
// Access test for running two web servers on the different network each other.
node ~# ip netns exec 719aa0a6-f1e1-4385-a42b-13f652de99ef curl 192.168.8.10:8080
Running Web Server, IP is 192.168.8.10

node ~# ip netns exec 719aa0a6-f1e1-4385-a42b-13f652de99ef curl 192.168.9.10:8080
Running Web Server, IP is 192.168.9.10
```

Done.

Thank you for reading.
