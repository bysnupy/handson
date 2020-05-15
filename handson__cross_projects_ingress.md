# OpenShift cross namespace ingress as of OCP4.4

## Summary

I'd like to demonstrate new feature to enable traffic routing cross multiple projects using route object on OCP4.4.
Refer [OCP4.4 release notes - Ingress enhancements](https://docs.openshift.com/container-platform/4.4/release_notes/ocp-4-4-release-notes.html#ocp-4-4-ingress-enhancements) for more details about the new feature.

## Environments

As of OCP4.4, the ingress can allow to access in multiple projects using same hostname name.
And this configuration would be enabled on the cluster level, so you should check whether other influences as aspect of your security policy before this configurations.

![ocp4 cross projects ingress](https://github.com/bysnupy/handson/blob/master/ocp4_cross_project_ingress.png)

## Create required resources for testing

### Create projects

```cmd
$ oc new-project project-a

$ oc new-project project-b
```

### Create test pod on its project

You should consider "DocumentRoot" for access context on its pod, because OCP4 does not provide the rewriting backend path yet.
But as you can see here: https://github.com/openshift/origin/issues/20474, the feature will be added soon on the new OCP4.z. 

```cmd
$ oc run -n project-a poda --image=registry.access.redhat.com/rhel7 -- bash -c 'mkdir -p /tmp/test/svca; cd /tmp/test; echo "SERVICE A" > svca/index.html; python -m SimpleHTTPServer 8080'

$ oc run -n project-b podb --image=registry.access.redhat.com/rhel7 -- bash -c 'mkdir -p /tmp/test/svcb; cd /tmp/test; echo "SERVICE B" > svcb/index.html; python -m SimpleHTTPServer 8080'
```

### Create each service with running each pod label selector on its project

```cmd
$ oc create -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  labels:
    app: service-a
  name: service-a
  namespace: project-a
spec:
  ports:
  - name: 8080-8080
    port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    run: poda
  sessionAffinity: None
  type: ClusterIP
EOF
```

```
$ oc create -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  labels:
    app: service-b
  name: service-b
  namespace: project-b
spec:
  ports:
  - name: 8080-8080
    port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    run: podb
  sessionAffinity: None
  type: ClusterIP
EOF
```

### Create route object for each service on its project

```
$ oc create -f - <<EOF
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  labels:
    app: service-a
  name: route-a
  namespace: project-a
spec:
  host: test.apps.example.com
  path: /svca
  port:
    targetPort: 8080-8080
  to:
    kind: Service
    name: service-a
    weight: 100
EOF
```

```
$ oc create -f - <<EOF
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  labels:
    app: service-b
  name: route-b
  namespace: project-b
spec:
  host: test.apps.example.com
  path: /svcb
  port:
    targetPort: 8080-8080
  to:
    kind: Service
    name: service-b
    weight: 100
EOF
```
## Testing

Check what basedoamin was configured by default in the IngressController CRD before the test, it shows the route hostname need not to match with the default basedoamin in the test.

```cmd
$ oc describe ingresscontroller/default -n openshift-ingress-operator | grep Domain:
  Domain:                  apps.ocp44.test.local
```

Before changing "InterNamespaceAllowed" to IngressController, the same hostname does not be applied on its project as follows.
The "route-b" was not available with "HostAlreadyClaimed".

```
$ oc get route route-a -n project-a 
NAME      HOST/PORT               PATH    SERVICES    PORT        TERMINATION   WILDCARD
route-a   test.apps.example.com   /svca   service-a   8080-8080                 None

$ oc get route route-b -n project-b
NAME      HOST/PORT            PATH     SERVICES    PORT        TERMINATION   WILDCARD
route-b   HostAlreadyClaimed   /svcb    service-b   8080-8080                 None
```

After changing "InterNamespaceAllowed", you can see both routes showed same hostname without "HostAlreadyClaimed" in the "project-b".
```
$ oc -n openshift-ingress-operator patch ingresscontroller/default \
  --patch '{"spec":{"routeAdmission":{"namespaceOwnership":"InterNamespaceAllowed"}}}' \
  --type=merge
ingresscontroller.operator.openshift.io/default patched

$ oc get route route-a -n project-a 
NAME      HOST/PORT               PATH    SERVICES    PORT        TERMINATION   WILDCARD
route-a   test.apps.example.com   /svca   service-a   8080-8080                 None

$ oc get route route-b -n project-b
NAME      HOST/PORT               PATH    SERVICES    PORT        TERMINATION   WILDCARD
route-b   test.apps.example.com   /svcb   service-b   8080-8080                 None
```

Test whether each context, "/svca" and "/svcb", shows each page from its running pod in the different project.
Prerequisite is "test.apps.example.com" should resolve the valid VIP of the LB, and it can route the appropriate route pod on the nodes.

```
$ curl http://test.apps.example.com/svca/
SERVICE A

$ curl http://test.apps.example.com/svcb/
SERVICE B
```

Done.
