# How to validate the SCCs on the Admission Controller

## Summary

Let's see how to process and applied the SCCs to the Pod through Admission Controller.

## Process flow on the Admission

![ocp4 scc flow chart](https://github.com/bysnupy/handson/blob/master/ocp4_scc_flow_chart.png)

1. Retrieve all security context constraints available for use by the user based on Priority of the SCCs.
2. Loop through the SCCs list in the order of Priority for looking up any one which meet any requests on the pod specifications. 
3. If there are no SCCs within the list, the pod will be rejected.
4. If there is a SCC to meet the reuqested pecifications of the pod, it is accepted.


## Demonstration

### The accepted SCC is "restricted" in this case, because "PRIORITY" of the "restricted" is most highest on the all available SCCs list. 
```
$ oc get scc
NAME                 PRIV      CAPS      SELINUX     RUNASUSER          FSGROUP     SUPGROUP    PRIORITY   READONLYROOTFS   VOLUMES
anyuid               false     []        MustRunAs   RunAsAny           RunAsAny    RunAsAny    10         false            [configMap downwardAPI emptyDir persistentVolumeClaim projected secret]
hostaccess           false     []        MustRunAs   MustRunAsRange     MustRunAs   RunAsAny    <none>     false            [configMap downwardAPI emptyDir hostPath persistentVolumeClaim projected secret]
hostmount-anyuid     false     []        MustRunAs   RunAsAny           RunAsAny    RunAsAny    <none>     false            [configMap downwardAPI emptyDir hostPath nfs persistentVolumeClaim projected secret]
hostnetwork          false     []        MustRunAs   MustRunAsRange     MustRunAs   MustRunAs   <none>     false            [configMap downwardAPI emptyDir persistentVolumeClaim projected secret]
kube-state-metrics   false     []        RunAsAny    RunAsAny           RunAsAny    RunAsAny    <none>     false            [*]
node-exporter        false     []        RunAsAny    RunAsAny           RunAsAny    RunAsAny    <none>     false            [*]
nonroot              false     []        MustRunAs   MustRunAsNonRoot   RunAsAny    RunAsAny    <none>     false            [configMap downwardAPI emptyDir persistentVolumeClaim projected secret]
privileged           true      [*]       RunAsAny    RunAsAny           RunAsAny    RunAsAny    <none>     false            [*]
restricted           false     []        MustRunAs   MustRunAsRange     MustRunAs   RunAsAny    20         false            [configMap downwardAPI emptyDir persistentVolumeClaim projected secret]
```

### Assign the anyuid to the "default" seriviceaccount and run some pod without pod specifications.
```
$ oc adm policy add-scc-to-user anyuid -z default
$ oc run test --image registry.redhat.io/rhel7 -- tail -f /dev/null
$ oc get pod
NAME            READY     STATUS    RESTARTS   AGE
test-1-hgt9l   1/1       Running   0          40m
$ oc get pod test-1-hgt9l -o yaml | grep scc
    openshift.io/scc: restricted
```

### But if you specify "securityContext.runAsUser" in the pod specification, then the pod run as "anyuid", even though "anyuid" is lower than "restricted" on the Priority.
```
$ oc edit dc/test
:
  template:
    :
    spec:
      :
      containers:
      :
      securityContext:
        runAsUser: 1002
:
$ oc get pod
NAME            READY     STATUS    RESTARTS   AGE
test-2-gzdjv   1/1       Running   0          28m
$ oc get pod test-2-gzdjv -o yaml | grep scc
      openshift.io/scc: anyuid
```

Done.
