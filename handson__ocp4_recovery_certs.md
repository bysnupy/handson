# Test for recovery of the expired certificates after the just installed cluster shutdown within 24 hours

This test is based on [Recovering from expired control plane certificates](https://docs.openshift.com/container-platform/4.6/backup_and_restore/disaster_recovery/scenario-3-expired-certs.html).
And it's conducted to OCP v4.6.1 on AWS.

## Start cluster
```console
$ oc get node
NAME                                              STATUS     ROLES    AGE   VERSION
ip-10-9-188-185.ap-northeast-1.compute.internal   NotReady   master   46h   v1.19.0+d59ce34
ip-10-9-189-165.ap-northeast-1.compute.internal   NotReady   worker   45h   v1.19.0+d59ce34
ip-10-9-189-81.ap-northeast-1.compute.internal    NotReady   worker   45h   v1.19.0+d59ce34
ip-10-9-191-230.ap-northeast-1.compute.internal   NotReady   master   46h   v1.19.0+d59ce34
ip-10-9-194-162.ap-northeast-1.compute.internal   NotReady   worker   45h   v1.19.0+d59ce34
ip-10-9-215-20.ap-northeast-1.compute.internal    NotReady   master   46h   v1.19.0+d59ce34
```
## List CSR for checking if pending CSR is existing or not
```console
$ oc get csr
NAME        AGE     SIGNERNAME                                    REQUESTOR                                                                   CONDITION
csr-2587g   3m34s   kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Pending
csr-49877   3m34s   kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Pending
csr-5z7x5   3m36s   kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Pending
csr-6kjvw   3m27s   kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Pending
csr-78wv4   3m18s   kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Pending
csr-k8k6g   3m34s   kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Pending
```
We can see "node-bootstrapper" CSR is "pending".

```console
$ oc describe csr csr-2587g
Name:               csr-2587g
Labels:             <none>
Annotations:        <none>
CreationTimestamp:  Fri, 30 Oct 2020 11:09:31 +0900
Requesting User:    system:serviceaccount:openshift-machine-config-operator:node-bootstrapper
Signer:             kubernetes.io/kube-apiserver-client-kubelet
Status:             Pending
Subject:
         Common Name:    system:node:ip-10-9-189-81.ap-northeast-1.compute.internal
         Serial Number:  
         Organization:   system:nodes
Events:  <none>
```

## Approve all pending CSR to recovery

```console
$ oc get csr -o name | xargs oc adm certificate approve 
certificatesigningrequest.certificates.k8s.io/csr-2587g approved
certificatesigningrequest.certificates.k8s.io/csr-49877 approved
certificatesigningrequest.certificates.k8s.io/csr-5z7x5 approved
certificatesigningrequest.certificates.k8s.io/csr-6kjvw approved
certificatesigningrequest.certificates.k8s.io/csr-78wv4 approved
certificatesigningrequest.certificates.k8s.io/csr-f9ln7 approved
certificatesigningrequest.certificates.k8s.io/csr-gh4b6 approved
certificatesigningrequest.certificates.k8s.io/csr-k8k6g approved
```

After approving all pending node-bootstrapper CSR, node certificates would be rotated automatically including approvment of the related node certificates CSRs. 

```console
$ oc get csr
NAME        AGE     SIGNERNAME                                    REQUESTOR                                                                   CONDITION
csr-2587g   11m     kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Approved,Issued
csr-49877   11m     kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Approved,Issued
csr-5z7x5   11m     kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Approved,Issued
csr-6kjvw   10m     kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Approved,Issued
csr-78wv4   10m     kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Approved,Issued
csr-f9ln7   6m32s   kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Approved,Issued
csr-fqs44   3m8s    kubernetes.io/kubelet-serving                 system:node:ip-10-9-189-81.ap-northeast-1.compute.internal                  Approved,Issued
csr-gh4b6   6m34s   kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Approved,Issued
csr-k8k6g   11m     kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Approved,Issued
csr-kvskg   3m7s    kubernetes.io/kubelet-serving                 system:node:ip-10-9-189-165.ap-northeast-1.compute.internal                 Approved,Issued
csr-lhnpn   3m11s   kubernetes.io/kubelet-serving                 system:node:ip-10-9-194-162.ap-northeast-1.compute.internal                 Approved,Issued
csr-lr46c   3m13s   kubernetes.io/kubelet-serving                 system:node:ip-10-9-188-185.ap-northeast-1.compute.internal                 Approved,Issued
csr-mtgjr   3m7s    kubernetes.io/kubelet-serving                 system:node:ip-10-9-191-230.ap-northeast-1.compute.internal                 Approved,Issued
csr-zhkp4   3m7s    kubernetes.io/kubelet-serving                 system:node:ip-10-9-215-20.ap-northeast-1.compute.internal                  Approved,Issued
```

Simply the node status could be recovered "Ready" as follows.
```console
$ oc get node
NAME                                              STATUS   ROLES    AGE   VERSION
ip-10-9-188-185.ap-northeast-1.compute.internal   Ready    master   46h   v1.19.0+d59ce34
ip-10-9-189-165.ap-northeast-1.compute.internal   Ready    worker   45h   v1.19.0+d59ce34
ip-10-9-189-81.ap-northeast-1.compute.internal    Ready    worker   45h   v1.19.0+d59ce34
ip-10-9-191-230.ap-northeast-1.compute.internal   Ready    master   46h   v1.19.0+d59ce34
ip-10-9-194-162.ap-northeast-1.compute.internal   Ready    worker   45h   v1.19.0+d59ce34
ip-10-9-215-20.ap-northeast-1.compute.internal    Ready    master   46h   v1.19.0+d59ce34
```

