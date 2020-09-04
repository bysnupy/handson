# How to define each type of the IngressGateway on OpenShift ServiceMesh

## Summary

I'd like to show you how to configure additional IngressGateway pod for each type, ClusterIP, NodePort and Loadbalancer.

![servicemesh access flow summary](https://github.com/bysnupy/handson/blob/master/ocp4_ingressgateway_type_access_flow.png)


Ingress name| Type | Description
-|-|-
istio-ingressgateway| ClusterIP | Default
second-ingressgateway| ClusterIP| Added one, accessing through router ingress pod
nodeport-ingressgateway| NodePort| Required to configure your LB, and DNS manually
loadbalancer-ingressgateway| LoadBalancer| Maybe required to configure your Gateway hostname DNS

## Demonstration Steps.

### Create test projects

```console
$ for prj in project-a project-b project-c project-d; do
  oc new-project $prj
done
```

### Add the projects to ServiceMeshMemberRole CR

```yaml
$ oc edit -n istio-system smmr default
:
spec:
  members:
  - project-a
  - project-b
  - project-c
  - project-d
```

### Add each ingressgateway configuration to ServiceMeshControlPlane CR

```yaml
$ oc edit -n istio-system smcp basic-install
:
spec:
  istio:
    gateways:
    # Additional ClusterIP type IngressGateway
    second-ingressgateway:
      enabled: true
      autoscaleEnabled: false
      ior_enabled: true
      resources:
        requests:
          cpu: 100m
          memory: 128Mi
      sds:
        enabled: false
      labels:
        app: second-ingressgateway
        istio: ingressgateway
      type: ClusterIP
      ports:
        - name: status-port
          protocol: TCP
          port: 15020
          targetPort: 15020
        - name: http2
          protocol: TCP
          port: 80
          targetPort: 8080
        - name: https
          protocol: TCP
          port: 443
          targetPort: 8443
        - name: tls
          protocol: TCP
          port: 15443
          targetPort: 15443
      # Additional NodePort type IngressGateway
      nodeport-ingressgateway:
        enabled: true
        autoscaleEnabled: false
        ior_enabled: false
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
        sds:
          enabled: false
        labels:
          app: nodeport-ingressgateway
          istio: ingressgateway
        type: NodePort
        ports:
          - name: status-port
            protocol: TCP
            port: 15020
            targetPort: 15020
          - name: http2
            protocol: TCP
            port: 80
            targetPort: 8080
          - name: https
            protocol: TCP
            port: 443
            targetPort: 8443
          - name: tls
            protocol: TCP
            port: 15443
            targetPort: 15443   
      # Additional LoadBalancer type IngressGateway
      loadbalancer-ingressgateway:
        enabled: true
        autoscaleEnabled: false
        ior_enabled: false
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
        sds:
          enabled: false
        labels:
          app: loadbalancer-ingressgateway
          istio: ingressgateway
        type: LoadBalancer
        ports:
          - name: status-port
            protocol: TCP
            port: 15020
            targetPort: 15020
          - name: http2
            protocol: TCP
            port: 80
            targetPort: 8080
          - name: https
            protocol: TCP
            port: 443
            targetPort: 8443
          - name: tls
            protocol: TCP
            port: 15443
            targetPort: 15443
      # Default ClusterIP type IngressGateway
      istio-ingressgateway:
        autoscaleEnabled: false
        ior_enabled: false
:
```

Check the created Service types

```console
$ oc get svc -l istio=ingressgateway -n istio-system
NAME                          TYPE           CLUSTER-IP       EXTERNAL-IP                                                                    PORT(S)                                                      AGE
istio-ingressgateway          ClusterIP      172.30.216.193   <none>                                                                         15020/TCP,80/TCP,443/TCP,15443/TCP                           8d
loadbalancer-ingressgateway   LoadBalancer   172.30.48.60     a1efdab2b94144f29b6866e66937477e-1470559942.ap-northeast-1.elb.amazonaws.com   15020:30053/TCP,80:31829/TCP,443:32661/TCP,15443:30937/TCP   122m
nodeport-ingressgateway       NodePort       172.30.155.6     <none>                                                                         15020:30789/TCP,80:32601/TCP,443:30950/TCP,15443:30683/TCP   3h51m
second-ingressgateway         ClusterIP      172.30.98.214    <none>                                                                         15020/TCP,80/TCP,443/TCP,15443/TCP                           7h5m
```

### Create test pod and service at each projet

```console
for seqnum in a b c d; do
oc create -n project-${seqnum} -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  annotations:
    sidecar.istio.io/inject: "true"
  labels:
    app: service-${seqnum}
  name: service-${seqnum}
  namespace: project-${seqnum}
spec:
  containers:
  - args:
    - bash
    - -c
    - mkdir -p /tmp/test/svc${seqnum}; cd /tmp/test; echo "SERVICE ${seqnum^^}" > svc${seqnum}/index.html; python
      -m SimpleHTTPServer 8080
    image: registry.access.redhat.com/rhel7
    name: service-${seqnum}
    ports:
    - containerPort: 8080
      name: web
      protocol: TCP
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: service-${seqnum}
  name: service-${seqnum}
  namespace: project-${seqnum}
spec:
  ports:
  - name: 8080-8080
    port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    app: service-${seqnum}
  type: ClusterIP
EOF
done
```

### Create Gateway and VirtualService for each Service on each project

For servie A, it is controlled over by Default IngressGateway
```console
$ oc create -n project-a -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: service-a-gw
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "service-a.ossm.example.com"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: service-a-vsvc
spec:
  hosts:
  - "*"
  gateways:
  - service-a-gw
  http:
  - match:
    - uri:
        prefix: /svca
    route:
    - destination:
        port:
          number: 8080
        host: service-a
EOF
```

For service B, it is controlled over by Additional ClusterIP type IngressGateway, second-ingressgateway.
"service-b.ossm.example.com" should be resolved VIP for Ingress router.
```console
$ oc create -n project-b -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: service-b-gw
spec:
  selector:
    app: second-ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "service-b.ossm.example.com"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: service-b-vsvc
spec:
  hosts:
  - "*"
  gateways:
  - service-b-gw
  http:
  - match:
    - uri:
        prefix: /svcb
    route:
    - destination:
        port:
          number: 8080
        host: service-b
EOF
```

For service C, it is controlled over by Additional NodePort type IngressGateway, nodeport-ingressgateway.
After this configuration, you should also create or configure manually your LB in order to access to this created NodePort.
And then "service-c.ossm.example.com" should be resolved VIP for your LB.

```console
$ oc create -n project-c -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: service-c-gw
spec:
  selector:
    app: nodeport-ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "service-c.ossm.example.com"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: service-c-vsvc
spec:
  hosts:
  - "*"
  gateways:
  - service-c-gw
  http:
  - match:
    - uri:
        prefix: /svcc
    route:
    - destination:
        port:
          number: 8080
        host: service-c
EOF
```

For service D, it is controlled over by Additional Loadbalancer type IngressGateway, loadbalancer-ingressgateway.
Usually, this type will create your LB through API on your cloud platform.
And then "service-d.ossm.example.com" should be resolved VIP for your LB.

```console
$ oc create -n project-d -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: service-d-gw
spec:
  selector:
    app: loadbalancer-ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "service-d.ossm.example.com"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: service-d-vsvc
spec:
  hosts:
  - "*"
  gateways:
  - service-d-gw
  http:
  - match:
    - uri:
        prefix: /svcd
    route:
    - destination:
        port:
          number: 8080
        host: service-d
EOF
```

### Access test and check the flow on the Kiali

```console
$ curl -s http://service-c.ossm.example.com/svcc/
SERVICE A
$ curl -s http://service-c.ossm.example.com/svcc/
SERVICE B
$ curl -s http://service-c.ossm.example.com/svcc/
SERVICE C
$ curl -s http://service-c.ossm.example.com/svcc/
SERVICE D
```

Kiali graph,

![servicemesh access flow from kiali](https://github.com/bysnupy/handson/blob/master/ocp4_kiali_ingressgateway_access_flows.png)


Done.
