# OCP4.3 installation on RHV 4.3 with Bare-metal method

## Summary

I walk through installation procedures of the OCP4.3 on RHV 4.3 with bare-metal method.
In this hands-on, we create OCP4 cluster which is consist with master x 3 and worker x 2.

VM hostname                 | IPs                                      | Role
----------------------------|------------------------------------------|--------------------------------------- 
infra.ocp43.rhev.local      | 10.0.1.10, 192.168.10.10, 192.168.20.10  | LB, DNS, NW default GW for OCP nodes 
bootstrap.ocp43.rhev.local  | 192.168.10.99                            | bootstrap for OCP installation
master-0.ocp43.rhev.local   | 192.168.10.20, 192.168.20.20             | master-0 for control plane
master-1.ocp43.rhev.local   | 192.168.10.21, 192.168.20.21             | master-1 for control plane
master-2.ocp43.rhev.local   | 192.168.10.22, 192.168.20.22             | master-2 for control plane
etcd-0.ocp43.rhev.local     | 192.168.10.20                            | etcd-0 for control plane
etcd-1.ocp43.rhev.local     | 192.168.10.21                            | etcd-1 for control plane
etcd-2.ocp43.rhev.local     | 192.168.10.22                            | etcd-2 for control plane
worker-0.ocp43.rhev.local   | 192.168.10.23, 192.168.20.23             | worker-0 for worker node
worker-1.ocp43.rhev.local   | 192.168.10.24, 192.168.20.24             | worker-1 for worker node

![ocp4 network diagram](https://github.com/bysnupy/handson/blob/master/ocp4_install_rhv_network.png)

## Prerequisites

First of all, we should create a RHEL7 VM(Bastion-server) on RHV before installation, it works as NW router to connect with Internet, LB to control access to ingress and master, DNS and contents provider server to be required to installation(ignition files and a bios image).
Upload "rhcos-4.3.0-x86_64-installer.iso" to ISO disk domain in advance on your RHV system.

![ocp4 bastion config on rhev](https://github.com/bysnupy/handson/blob/master/ocp4_install_rhv_creatingvm.png)


### Register repositories for installing packages and so on 
```cmd
# subscription-manager register
# subscription-manager attach --pool <appropriate pool id>
# subscription-manager repos --disable "*"
# subscription-manager repos --enable rhel-7-server-rpms --enable rhel-7-server-extras-rpm
# yum install haproxy httpd dnsmasq vim -y

# setenforce Permissive

# systemctl disable firewalld
# systemctl enable httpd haproxy dnsmasq
# systemctl start  httpd haproxy dnsmasq
```

### Configure DNS-related things

```cmd
# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

# shared infra
192.168.10.10 infra.ocp43.rhev.local

# ocp4
# masters
192.168.10.20 master-0.ocp43.rhev.local
192.168.10.21 master-1.ocp43.rhev.local
192.168.10.22 master-2.ocp43.rhev.local
# etcd
192.168.10.20 etcd-0.ocp43.rhev.local
192.168.10.21 etcd-1.ocp43.rhev.local
192.168.10.22 etcd-2.ocp43.rhev.local
# LB
192.168.10.10 api.ocp43.rhev.local   
192.168.10.10 api-int.ocp43.rhev.local
192.168.10.10 oauth-openshift.apps.ocp43.rhev.local
# workers
192.168.10.23 worker-0.ocp43.rhev.local
192.168.10.24 worker-1.ocp43.rhev.local
# bootstrap
192.168.10.99 bootstrap.ocp43.rhev.local
```
```cmd
# cat /etc/dnsmasq.d/ocp4.conf
domain-needed
bogus-priv
expand-hosts
domain=ocp43.rhev.local

address=/apps.ocp43.rhev.local/192.168.10.10

log-facility=/var/log/dnsmasq.log

srv-host=_etcd-server-ssl._tcp.ocp43.rhev.local,etcd-0.ocp43.rhev.local,2380,0,10
srv-host=_etcd-server-ssl._tcp.ocp43.rhev.local,etcd-1.ocp43.rhev.local,2380,0,10
srv-host=_etcd-server-ssl._tcp.ocp43.rhev.local,etcd-2.ocp43.rhev.local,2380,0,10
```

### Configure LB-related things

```cmd
# grep -v -E '^$|^#|^[[:space:]]+#' /etc/haproxy/haproxy.cfg
global
    log         127.0.0.1 local2
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon
    stats socket /var/lib/haproxy/stats
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000
frontend master-api
  bind    *:6443
  option  tcplog
  mode    tcp
  default_backend  master-api-6443
frontend machine-config
  bind    *:22623
  option  tcplog
  mode    tcp
  default_backend  machineconfig-22623
frontend ingress-http
  bind    *:80
  option  tcplog
  mode    tcp
  default_backend  ingress-http-80
frontend ingress-https
  bind    *:443
  option  tcplog
  mode    tcp
  default_backend  ingress-https-443
backend master-api-6443
  mode     tcp
  balance  roundrobin
  option   ssl-hello-chk
  server   bootstrap  bootstrap.ocp43.rhev.local:6443 check
  server   master-0   master-0.ocp43.rhev.local:6443  check
  server   master-1   master-1.ocp43.rhev.local:6443  check
  server   master-2   master-2.ocp43.rhev.local:6443  check
backend machineconfig-22623
  mode     tcp
  balance  roundrobin
  option   ssl-hello-chk
  server   bootstrap  bootstrap.ocp43.rhev.local:22623  check
  server   master-0   master-0.ocp43.rhev.local:22623   check
  server   master-1   master-1.ocp43.rhev.local:22623   check
  server   master-2   master-2.ocp43.rhev.local:22623   check
backend ingress-http-80
  mode     tcp
  balance  roundrobin
  server   worker-0   worker-0.ocp43.rhev.local:80  check
  server   worker-1   worker-1.ocp43.rhev.local:80  check
backend ingress-https-443
  mode     tcp
  balance  roundrobin
  option   ssl-hello-chk
  server   worker-0   worker-0.ocp43.rhev.local:443  check
  server   worker-1   worker-1.ocp43.rhev.local:443  check
```

### Configure httpd web server to use with installation contents provider

```cmd
// Expected directory and included files
/var/www/html/ocp43/
├── ign
│   ├── bootstrap.ign (owner: apache, group: apache, 0644)
│   ├── master.ign    (owner: apache, group: apache, 0644)
│   └── worker.ign    (owner: apache, group: apache, 0644)
└── img
    └── bios.raw.gz   (owner: apache, group: apache, 0400)
```

```cmd
# sed -i s/Listen 80/Listen 8080/g /etc/httpd/conf/httpd.conf
# mkdir /var/www/html/ocp43/{ign,img} -p
# systemctl restart httpd
```

### Configure forwarding rule for NW Gateway for node hosts network(192.168.10.0/24)

```
# cat /etc/sysctl.d/99-sysctl.conf
net.ipv4.ip_forward=1
# sysctl -p /etc/sysctl.d/99-sysctl.conf
# sysctl -a | grep ip_forward
# iptables -A INPUT -m state --state NEW,RELATED,ESTABLISHED -j ACCEPT
# iptables -t nat -A POSTROUTING -o eth0 -s 192.168.10.0/24 -j MASQUERADE
# iptables-save  >  /etc/sysconfig/iptables
# systemctl reload iptables
```

### Install oc CLI and openshift-installer

Access "cloud.redhat.com" and download oc CLI and openshift-installer from OpenShift Cluster Manager, and install those on your bastion server.
You should get "Pull Secret" and "rhcos-4.3.8-x86_64-metal.x86_64.raw.gz" bios image either.

### Generate ignition files and place the contents to httpd web server

Refer [Installing a cluster on bare metal](https://docs.openshift.com/container-platform/4.3/installing/installing_bare_metal/installing-bare-metal.html) for more details.

```cmd
$ ssh-keygen -t rsa -b 4096 -N '' -f ~/.ssh/ocp43_id_rsa
```

```cmd
$ cat $HOME/ocp43/install-config.yaml
apiVersion: v1
baseDomain: rhev.local 
compute:
- hyperthreading: Enabled   
  name: worker
  replicas: 0 
controlPlane:
  hyperthreading: Enabled   
  name: master 
  replicas: 3 
metadata:
  name: ocp43
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14 
    hostPrefix: 23 
  networkType: OpenShiftSDN
  serviceNetwork: 
  - 172.30.0.0/16
platform:
  none: {} 
pullSecret: '<pull secret>' 
sshKey: '<ocp43_id_rsa ssh key>' 
```

```cmd
$ openshift-install create manifests --dir $HOME/ocp43
$ sed -i "s/mastersSchedulable: true/mastersSchedulable: false/g" $HOME/ocp43/manifests/cluster-scheduler-02-config.yml
$ openshift-install create ignition-configs --dir $HOME/ocp43
$ cp $HOME/ocp43/*.ign /var/www/html/ocp43/ign/
$ cp rhcos-4.3.8-x86_64-metal.x86_64.raw.gz /var/www/html/ocp43/img/bios.raw.gz
$ sudo chown apache:apache -R /var/www/html/ocp43/* 
```

### Create VMs for OCP4 cluster before installation

You should create 6 VMs(bootstrapx1, masterx3, workerx2) as following steps on RHEV dashboard.

1. Select "Compute" tab and click "New" button
2. Select appropriate "Cluster" and "Template" for the VM type
3. Select "Operating system" to "RHEL8"
4. Select "Instance Type" with "Custom" and "Optimized for" with "Server"
5. Input each VM name appropriately, such as "master-0.ocp43.rhev.local", "worker-0.ocp43.rhev.local" and "bootstrap.ocp43.rhev.local".
6. Add vNIC as clicking "+" button, such as master and worker cases are required 2 vNICs and bootstrap is just one vNIC.
7. Select "System" tab on the left side pane after expand the tabs as clicking "Hide Advanced Options".
8. Assign CPU and Memory size enough to your workload. I recommend at least CPU x 4 and Memory is 16GB.
9. Complete to create VM profile as clicking "OK" button the bottom on the left side of the dialog.
10. Add disks to the VM as appropriate size, such as 40 ~ 50GB.

### Run each VM with RHCOS installer ISO

You should run each of the bootstrap, master hosts and worker host individually in order with RHCOS ISO through RHEV dashboard.

1. Select "Compute" tab and click target VM name.
2. Expand "Run" button dropbox and clieck "Run Once".
3. Expand "Boot Options" and check "Attach CD" with "rhcos-4.3.0-x86_64-installer.iso".
4. Uncheck "Enable menu to select boot device" and make CD-ROM move to most up side using "Up" button.
5. Complete to click "OK" button the bottom on the left side of the dialog.

![runonce dialog](https://github.com/bysnupy/handson/blob/master/ocp4_install_rhv_runonce.png)

Then the VM will start.

### Input kernel parameter to RHCOS boot page

WATCH OUT, you should recognize your vNIC and disk device name in advance.
Usually, NIC device names as "ensX" format, such as first NIC is "ens3" and second one is "ens4".
Disk device also names as "vdX" format, such as first disk is "vda" and second is "vdb".
 
In case of the bootstrap,

```cmd
coreos.inst=yes
coreos.inst.install_dev=vda
coreos.inst.image_url=http://192.168.10.10:8080/ocp43/img/bios.raw.gz
coreos.inst.ignition_url=http://192.168.10.10:8080/ocp43/ign/bootstrap.ign
ip=192.168.10.99::192.168.10.10:255.255.255.0:bootstrap.ocp43.rhev.local:ens3:none
nameserver=192.168.10.10
```

In case of the master-0 node,
```cmd
coreos.inst=yes
coreos.inst.install_dev=vda
coreos.inst.image_url=http://192.168.10.10:8080/ocp43/img/bios.raw.gz
coreos.inst.ignition_url=http://192.168.10.10:8080/ocp43/ign/master.ign
ip=192.168.10.20::192.168.10.10:255.255.255.0:master-0.ocp43.rhev.local:ens3:none
nameserver=192.168.10.10
ip=192.168.20.20:::255.255.255.0::ens4:none
```

In case of the worker-0 node,
```cmd
coreos.inst=yes
coreos.inst.install_dev=vda
coreos.inst.image_url=http://192.168.10.10:8080/ocp43/img/bios.raw.gz
coreos.inst.ignition_url=http://192.168.10.10:8080/ocp43/ign/worker.ign
ip=192.168.10.23::192.168.10.10:255.255.255.0:worker-0.ocp43.rhev.local:ens3:none
nameserver=192.168.10.10
ip=192.168.20.23:::255.255.255.0::ens4:none
```

### Install OCP4

1. Monitor until the bootstrap is complete

```cmd
$ openshift-install wait-for bootstrap-complete --dir $HOME/ocp43 
```

2. Accept CSR of the worker nodes

```cmd
$ export KUBECONFIG=/auth/kubeconfig
$ oc get node
NAME                       STATUS      ROLES    AGE    VERSION
master-0.ocp43.kvm.local   Ready       master   1h     v1.16.2
master-1.ocp43.kvm.local   Ready       master   1h     v1.16.2
master-2.ocp43.kvm.local   Ready       master   1h     v1.16.2
worker-0.ocp43.kvm.local   NotReady    worker   55s    v1.16.2
worker-1.ocp43.kvm.local   NotReady    worker   59s    v1.16.2
```
```cmd
$ oc get csr -o name | xargs oc adm certificate approve
```

3. Modify image-registry Operator storage type to emptyDir

```cmd
$ oc patch configs.imageregistry.operator.openshift.io cluster --type merge --patch '{"spec":{"storage":{"emptyDir":{}}}}'
```

4. Wait for all Operator is "Available: True"

### Complete Installation of the OCP4 on RHV

After checking all operator is available using "oc get clusteroperators", run the following command.

```cmd
$ openshift-install --dir=bare-metal wait-for install-complete
```

Done.
