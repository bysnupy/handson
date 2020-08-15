# How to work the Security Context Constraint on OpenShift

## Summary

I simply walk through SCC process with some examples here. 

## Process flow

Basically, the Security Context Constraint(SCC) control over permissions for pods on OpenShift.
The set of SCCs authorized a pod are determined by the operation user identity and specified service account.
We are easy to think that it only considers "Priority" attribute of the SCCs to decide it, 
but additionally requested permissions or capabilities also considers to find an appropriate SCC for the pods or containers.
Let me summarize the process as follows.

![ocp4 scc process_flow](https://github.com/bysnupy/handson/blob/master/ocp4_scc_process_flow.png)

1. Retrieve all available SCCs from user identity or specified service account.
2. Sort the SCCs by "Priority", higher SCC will process first than lower.
3. Check the requested conditions of the pod to each of sorted SCCs.
4. If one SCC can be matched completely with all conditions, then pod will create and run with the SCC.
5. If not, the pod will be failed.

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

## Additional Information

### Security Context Constraints design proposals
https://github.com/openshift/origin/blob/master/docs/proposals/security-context-constraints.md#admission

### "FindApplicableSCCs" is returned SCCs are sorted by priority
https://github.com/openshift/apiserver-library-go/blob/release-4.5/pkg/securitycontextconstraints/sccmatching/matcher.go#L40-L67

### "computeSecurityContext" is returned the valid pod after validation through Admission controller
https://github.com/openshift/apiserver-library-go/blob/release-4.5/pkg/securitycontextconstraints/sccadmission/admission.go#L193-L237

Done.
