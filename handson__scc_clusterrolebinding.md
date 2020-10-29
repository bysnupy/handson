# Test for SCC with ClusterRole and ClusterRolebinding

When grantees are received SCCs, create appropriate ClusterRole and ClusterRoleBinding instead and not modifying SCC at all.
Reference: https://bugzilla.redhat.com/show_bug.cgi?id=1833558

## Summary
Check the "users:" sections are not modified after adding "nonroot" SCC to "user1" login account and "test-sa" serviceaccount.
Check if the related RoleBindings or ClusterRoleBinding created for "nonroot" SCC after that.
Verify a pod can run with "nonroot" through "user1" account and "test-sa" serviceacocunt, at that time not using cluster-admin granted account which is allowed to use all SCCs for accurate testing.

Let’s check if the SCC would be changed after granting permissions and if the related RoleBinding and ClusterRoleBinding are created or not first.

## Create test Project: “scc-test” and test ServiceAccount: “test-sa”.
```
$ oc whoami
Kube:admin

$ oc new-project scc-test

$ oc create sa test-sa

$ oc get sa test-sa -n scc-test
NAME      SECRETS   AGE
test-sa     2         22s
```

Check the current “users:” section of the “nonroot” SCC resource
```
$ oc get scc nonroot -o json | jq '.users'
[]
```

Grant “nonroot” SCC to “test-sa” serviceaccount and “user1” account.
```
$ oc adm policy add-scc-to-user nonroot -z test-sa -n scc-test
clusterrole.rbac.authorization.k8s.io/system:openshift:scc:nonroot added: "test-sa"
$ oc adm policy add-scc-to-user nonroot user1
clusterrole.rbac.authorization.k8s.io/system:openshift:scc:nonroot added: "user1"
```

Check again if the “nonroot” SCC resource is changed
```
$ oc get scc nonroot -o json | jq '.users'
[]
```

Check RoleBinding for “test-sa” and ClusterRolebinding for “user1”
```
// In the case of “test-sa” serviceaccount in “scc-test”
$ oc get rolebinding system:openshift:scc:nonroot -n scc-test
NAME                           ROLE                                       AGE
system:openshift:scc:nonroot   ClusterRole/system:openshift:scc:nonroot   8m38s

$ oc get rolebinding system:openshift:scc:nonroot -n scc-test -o json | jq '.subjects'
[
  {
    "kind": "ServiceAccount",
    "name": "test-sa",
    "namespace": "scc-test"
  }
]

// In the case of “user1” normal account with not specifying any project.
$ oc get clusterrolebinding system:openshift:scc:nonroot
NAME                           ROLE                                       AGE
system:openshift:scc:nonroot   ClusterRole/system:openshift:scc:nonroot   4m19s

$ oc get clusterrolebinding system:openshift:scc:nonroot -o json | jq '.subjects'
[
  {
    "apiGroup": "rbac.authorization.k8s.io",
    "kind": "User",
    "name": "user1"
  }
]
```

I was able to verify the SCC resource was not be changed and the related RoleBinding and ClusterRoleBinding were also created expectedly through above testing.

Additionally I’ve tested the SCC permission took effect after granting SCCs through new binding ways.

```
$ oc adm policy add-role-to-user edit user2
clusterrole.rbac.authorization.k8s.io/edit added: "user2"

$ oc whoami
user2

$ oc create -n scc-test -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: scc-test
spec:
  containers:
  - args:
    - tail
    - -f
    - /dev/null
    image: centos/tools
    name: scc-test
    securityContext:
      runAsUser: 6868
  serviceAccountName: test-sa
EOF

$ oc get pod -n scc-test
NAME       READY   STATUS    RESTARTS   AGE
scc-test   1/1     Running   0          8s

$ oc get pod scc-test -n scc-test -o yaml | grep -w -E 'openshift.io/scc:|serviceAccountName:'
openshift.io/scc: nonroot
serviceAccountName: test-sa
    
$ oc rsh -n scc-test scc-test id
uid=6868(6868) gid=0(root) groups=0(root)

// Change the user to "user1" and create pod without specific ServiceAccount which grants "nonroot".
$ oc new-project scc-test2

$ oc adm policy add-role-to-user edit user1 -n scc-test2
clusterrole.rbac.authorization.k8s.io/edit added: "user1"

$ oc whoami
user1

$ oc create -n scc-test2 -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: scc-test2
spec:
  containers:
  - args:
    - tail
    - -f
    - /dev/null
    image: centos/tools
    name: scc-test2
    securityContext:
      runAsUser: 6868
EOF

$ oc get pod -n scc-test2
NAME        READY   STATUS    RESTARTS   AGE
scc-test2   1/1     Running   0          10s

$ oc get pod scc-test2 -n scc-test2 -o yaml | grep -w -E 'openshift.io/scc:|serviceAccountName:'
    openshift.io/scc: nonroot
  serviceAccountName: default

$ oc rsh -n scc-test2 scc-test2 id
uid=6868(6868) gid=0(root) groups=0(root)
```

I was able to verify the pod can work well with specified SCC through ClusterRoleBinding and RoleBinding expectedly.
