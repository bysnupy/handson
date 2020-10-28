# How to configure AWS Network Load Balancer for an Ingress Controller on OpenShift

As of OpenShift 4.6, we can enable AWS Network Load Balancer for an Ingress Controller.
Until now only NLS was configured for Control plane services, and the Ingress Controller was supported only for Classic Load Balancer on AWS by default. 

Ingress Controller Network Load Balancer for AWS
https://docs.openshift.com/container-platform/4.6/release_notes/ocp-4-6-release-notes.html#ocp-4-6-ingress-nlb-aws

I just tested how to configure the NLB to existing here, but you can configure the NLB to new cluster either.
Related official documentation links are below.

* Configuring an Ingress Controller Network Load Balancer on an existing AWS cluster
https://docs.openshift.com/container-platform/4.6/networking/configuring_ingress_cluster_traffic/configuring-ingress-cluster-traffic-aws-network-load-balancer.html#nw-aws-nlb-existing-cluster_configuring-ingress-cluster-traffic-aws-network-load-balancer

* Configuring an Ingress Controller Network Load Balancer on a new AWS cluster
https://docs.openshift.com/container-platform/4.6/installing/installing_aws/installing-aws-network-customizations.html#nw-aws-nlb-new-cluster_installing-aws-network-customizations

## Create additional Ingress Controller with the NLB after installation

### Create the Ingress Controller the specified manifest based on each scope, External and Internal.
```bash
$ oc create -n openshift-ingress-operator -f - <<EOF
apiVersion: operator.openshift.io/v1
kind: IngressController
metadata:
  name: nlb-external
  namespace: openshift-ingress-operator
spec:
  domain: nlb-ext.ocp46.example.com
  endpointPublishingStrategy:
    type: LoadBalancerService
    loadBalancer:
      scope: External
      providerParameters:
        type: AWS
        aws:
          type: NLB
EOF
```

### Verify the configuration

```bash
$ oc get pod,svc -n openshift-ingress
NAME                                       READY   STATUS    RESTARTS   AGE
pod/router-default-89g7c9fv9z-aaaaa        1/1     Running   0          162m
pod/router-default-89g7c9fv9z-bbbbb        1/1     Running   0          162m
pod/router-nlb-external-7d55xi8e33-aaaaa   1/1     Running   0          3m53s
pod/router-nlb-external-7d55xi8e33-bbbbb   1/1     Running   0          3m53s

NAME                                   TYPE           CLUSTER-IP       EXTERNAL-IP                                          PORT(S)                      AGE
service/router-default                 LoadBalancer   172.30.22.68     xxxxxxxxxx-yyyyyy.ap-northeast-1.elb.amazonaws.com   80:31155/TCP,443:32009/TCP   162m
service/router-internal-default        ClusterIP      172.30.149.120   <none>                                               80/TCP,443/TCP,1936/TCP      162m
service/router-internal-nlb-external   ClusterIP      172.30.101.12    <none>                                               80/TCP,443/TCP,1936/TCP      3m53s                                                                              80/TCP,443/TCP,1936/TCP      3m33s
service/router-nlb-external            LoadBalancer   172.30.252.213   1111111111-222222.elb.ap-northeast-1.amazonaws.com   80:30776/TCP,443:30240/TCP   3m53s
```

Test accessibility through created NLB and ingresses using a simple web application pod.

```bash
// Create a test project
$ oc new-project nlb-test

// Create a test pod and service
$ oc create -n nlb-test -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: web
  name: web
  namespace: nlb-test
spec:
  containers:
  - args:
    - bash
    - -c
    - mkdir -p /tmp/test/svc; cd /tmp/test; echo "SERVICE OK" > svc/index.html; python
      -m SimpleHTTPServer 8080
    image: registry.access.redhat.com/rhel7
    name: web
    ports:
    - containerPort: 8080
      name: web
      protocol: TCP
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: web
  name: web-svc
  namespace: nlb-test
spec:
  ports:
  - name: 8080-8080
    port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    app: web
  type: ClusterIP
EOF
```

Create Route using the added NLB ingress wildcard domain for exposing the test service, then "External Type" is exposed by the NLB on the Internet.
```bash
$ oc create route edge web-ext --hostname web-ext.nlb-ext.ocp46.example.com --service web-svc
route.route.openshift.io/web-ext created
```

You can verify the same IP address is resolved between the Route hostname and added NLB hostname.
```bash
$ dig +short web-ext.nlb-ext.ocp46.example.com
x.x.x.x
$ dig +short 1111111111-222222.elb.ap-northeast-1.amazonaws.com
x.x.x.x
```
Yes, we can have access without issue.
```bash
$ curl -ks --connect-timeout 1 https://web-ext.nlb-ext.ocp46.example.com/svc/
SERVICE OK
```

## Remove the added an Ingress Controller for the NLB

If you remove added Ingress Controller resource, then all related resources including the NLB on AWS are also removed. 

```
$ oc delete ingresscontroller nlb-external -n openshift-ingress-operator
ingresscontroller.operator.openshift.io "nlb-external" deleted
```

You can also add the NLB as internal load balancer through the same procedure.

Thank you for reading.
