# How to direct outgoing traffics using HTTPS_PROXY through Egress Gateway in Service Mesh

## Summary
This test is for fixing [Direct the traffic to the external proxy through an egress gateway](https://deploy-preview-6069--preliminary-istio.netlify.app/docs/tasks/traffic-management/egress/http-proxy/#direct-the-traffic-to-the-external-proxy-through-an-egress-gateway) correctly.

The all traffic of the client with HTTPS_PROXY would be forwarded to the external proxy through egress gateway of the Istio for accessing the external target service.

```
curl with HTTPS_PROXY in client pod(istio-proxy) --CONNECT(1)--> egress gateway(proxy) --CONNECT(2)--> external proxy(squid) --> target service
```

It has been tested in OpenShift Service Mesh 2.1.1.

## Environments information

Endpoint Name|IP|Port|Description
-|-|-|-
myproxy.istio-system.svc.cluster.local|10.0.0.15|8080|Headless Service Name mapped with the external squid proxy

Test project has been created as "test-proxy" in advance.
And you can also deploy a test pod referred as sleep pod in this test through [Before you begin](https://istio.io/latest/docs/tasks/traffic-management/egress/egress-gateway/#before-you-begin).

## Manifests
*NOTE:*
Most important thing is the service name resolution at both istio-system and user's namespace of the external proxy.
And we should define two VirtualService and DestinationRule each for egress-gateway and external proxy routings.

### ServiceEntry and k8s Service and Endpoint for the external proxy
```yaml
kind: Service
apiVersion: v1
metadata:
  name: myproxy
  namespace: istio-system
spec:
  clusterIP: None
  type: ClusterIP
  ports:
  - protocol: TCP
    port: 8080
    name: tcp
---
kind: Endpoints
apiVersion: v1
metadata:
  name: myproxy
  namespace: istio-system
subsets:
- addresses:
  - ip: 10.0.0.15
  ports:
  - name: tcp
    port: 8080
    protocol: TCP
---
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: proxy-se
  namespace: test-proxy
spec:
  exportTo:              # IMPORTANT, missing this, it would not work.
  - istio-system
  - test-proxy
  hosts:
  - myproxy.istio-system.svc.cluster.local
  addresses:
  - 10.0.0.15/32
  ports:
  - number: 8080
    name: tcp
    protocol: TCP
  location: MESH_EXTERNAL
```

### Gateway, DestinationRule and VirtualService
```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: istio-egressgateway-gw
  namespace: test-proxy
spec:
  selector:
    istio: egressgateway
  servers:
  - port:
      number: 7777
      name: tcp
      protocol: TCP
    hosts:
    - myproxy.istio-system.svc.cluster.local
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: myproxy
  namespace: test-proxy
spec:
  host: myproxy.istio-system.svc.cluster.local
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: egressgateway-for-proxy
  namespace: test-proxy
spec:
  host: istio-egressgateway.istio-system.svc.cluster.local
  subsets:
  - name: proxy
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: direct-egress-vs
  namespace: test-proxy
spec:
  hosts:
  - myproxy.istio-system.svc.cluster.local
  gateways:
  - mesh
  tcp:
  - match:
    - gateways:
      - mesh
      destinationSubnets:
      - 10.0.0.15/32
      port: 8080
    route:
    - destination:
        host: istio-egressgateway.istio-system.svc.cluster.local
        subset: proxy
        port:
          number: 7777
        weight: 100
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: direct-external-proxy-vs
  namespace: test-proxy
spec:
  hosts:
  - myproxy.istio-system.svc.cluster.local
  gateways:
  - istio-egressgateway-gw
  tcp:
  - match:
    - gateways:
      - istio-egressgateway-gw
      port: 7777
    route:
    - destination:
        host: myproxy.istio-system.svc.cluster.local
        port:
          number: 8080
      weight: 100
```

## Test results

### Access using HTTPS_PROXY from test pod(sleep pods).
```shell
$ kubectl exec -c sleep deploy/sleep -n test-proxy -- sh -c "HTTPS_PROXY=10.0.0.15:8080 curl -vs https://en.wikipedia.org/wiki/Main_Page" | grep -o "<title>.*</title>"

* Uses proxy env variable HTTPS_PROXY == '10.0.0.15:8080'
*   Trying 10.0.0.15:8080...
* Connected to 10.0.0.15 (10.0.0.15) port 8080 (#0)
* allocate connect buffer!
* Establish HTTP proxy tunnel to en.wikipedia.org:443
> CONNECT en.wikipedia.org:443 HTTP/1.1
> Host: en.wikipedia.org:443
> User-Agent: curl/7.82.0-DEV
> Proxy-Connection: Keep-Alive
> 
< HTTP/1.1 200 Connection established
< 
* Proxy replied 200 to CONNECT request
* CONNECT phase completed!
* ALPN, offering h2
* ALPN, offering http/1.1
*  CAfile: /cacert.pem
*  CApath: none
} [5 bytes data]
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
} [512 bytes data]
* TLSv1.3 (IN), TLS handshake, Server hello (2):
{ [122 bytes data]
* TLSv1.3 (IN), TLS handshake, Encrypted Extensions (8):
{ [19 bytes data]
* TLSv1.3 (IN), TLS handshake, Certificate (11):
{ [4510 bytes data]
* TLSv1.3 (IN), TLS handshake, CERT verify (15):
{ [79 bytes data]
* TLSv1.3 (IN), TLS handshake, Finished (20):
{ [52 bytes data]
* TLSv1.3 (OUT), TLS change cipher, Change cipher spec (1):
} [1 bytes data]
* TLSv1.3 (OUT), TLS handshake, Finished (20):
} [52 bytes data]
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384
* ALPN, server accepted to use h2
* Server certificate:
*  subject: CN=*.wikipedia.org
*  start date: Mar 12 07:44:59 2022 GMT
*  expire date: Jun 10 07:44:58 2022 GMT
*  subjectAltName: host "en.wikipedia.org" matched cert's "*.wikipedia.org"
*  issuer: C=US; O=Let's Encrypt; CN=R3
*  SSL certificate verify ok.
* Using HTTP2, server supports multiplexing
:
<title>Wikipedia, the free encyclopedia</title>
* Connection #0 to host 10.0.0.15 left intact
```

### the istio-proxy logs of the client pod(sleep pod)
```shell
[2022-04-20T06:40:24.541Z] "- - -" 0 - - - "-" 989 90274 18975 - "-" "-" "-" "-" "10.131.1.111:7777" outbound|7777|proxy|istio-egressgateway.istio-system.svc.cluster.local 10.131.1.250:33002 10.0.0.15:8080 10.131.1.250:38380 - - "-"
[2022-04-20T06:41:12.111Z] "- - -" 0 - - - "-" 989 90305 3341 - "-" "-" "-" "-" "10.131.1.111:7777" outbound|7777|proxy|istio-egressgateway.istio-system.svc.cluster.local 10.131.1.250:34650 10.0.0.15:8080 10.131.1.250:40028 - - "-"
```

### the egress gateway pod logs
```
[2022-04-20T06:40:24.542Z] "- - -" 0 - - - "-" 989 90274 18973 - "-" "-" "-" "-" "10.0.0.15:8080" outbound|8080||myproxy.istio-system.svc.cluster.local 10.131.1.111:39040 10.131.1.111:7777 10.131.1.250:33002 - - "-"
[2022-04-20T06:41:12.112Z] "- - -" 0 - - - "-" 989 90305 3339 - "-" "-" "-" "-" "10.0.0.15:8080" outbound|8080||myproxy.istio-system.svc.cluster.local 10.131.1.111:40688 10.131.1.111:7777 10.131.1.250:34650 - - "-"
```

### the external proxy(squid) logs
```shell
1650436832.158  18967 10.0.0.98 TCP_TUNNEL/200 90274 CONNECT en.wikipedia.org:443 - HIER_DIRECT/208.80.154.224 -
1650436864.094   3333 10.0.0.98 TCP_TUNNEL/200 90305 CONNECT en.wikipedia.org:443 - HIER_DIRECT/208.80.154.224 -
```
### Kiali graph visualization

![update](https://github.com/bysnupy/handson/blob/master/images/handson__https_proxy1.png)

