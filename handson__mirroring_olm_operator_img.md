# Test mirroring images for using OLM through external registry

## Summary

This test is helpful for understanding more details of the [Using Operator Lifecycle Manager on restricted networks](https://docs.openshift.com/container-platform/4.9/operators/admin/olm-restricted-networks.html).
My all test was conducted on OCPv4.9 in this article. And the network is not restricted in this test, but in theory all tasks and the result is basically the same with the restricted network.

## Test

###  Create your own index image and push it, in this test filtered just only for elasticsearch-operator and clsuter-logging.
```cmd
$ oc patch OperatorHub cluster --type json \
    -p '[{"op": "add", "path": "/spec/disableAllDefaultSources", "value": true}]'

$ oc get pod -n openshift-marketplace
NAME                                    READY   STATUS    RESTARTS   AGE
marketplace-operator-6dc6dd9896-tpzdp   1/1     Running   0          4h33m

$ podman login registry.redhat.io

$ podman run --authfile cred.json -p50051:50051 \
  -it registry.redhat.io/redhat/redhat-operator-index:v4.9

another-terminal $ grpcurl -plaintext localhost:50051 api.Registry/ListPackages > packages.out
another-terminal $ cat packages.out | grep -i -E 'elasticsearch|logging'
  "name": "cluster-logging"
  "name": "elasticsearch-operator"

$ opm index prune \
  -f registry.redhat.io/redhat/redhat-operator-index:v4.9 \
  -p cluster-logging,elasticsearch-operator \
  -t quay.io/daein/myindex:v4.9
INFO[0000] pruning the index                             packages="[cluster-logging elasticsearch-operator]"
INFO[0000] Pulling previous image registry.redhat.io/redhat/redhat-operator-index:v4.9 to get metadata  packages="[cluster-logging elasticsearch-operator]"
INFO[0000] running /usr/bin/podman pull registry.redhat.io/redhat/redhat-operator-index:v4.9  packages="[cluster-logging elasticsearch-operator]"
INFO[0003] running /usr/bin/podman pull registry.redhat.io/redhat/redhat-operator-index:v4.9  packages="[cluster-logging elasticsearch-operator]"
INFO[0005] Getting label data from previous image        packages="[cluster-logging elasticsearch-operator]"
INFO[0005] running podman inspect                        packages="[cluster-logging elasticsearch-operator]"
INFO[0005] running podman create                         packages="[cluster-logging elasticsearch-operator]"
INFO[0005] running podman cp                             packages="[cluster-logging elasticsearch-operator]"
INFO[0009] running podman rm                             packages="[cluster-logging elasticsearch-operator]"
INFO[0009] deleting packages                             pkg=3scale-operator
INFO[0009] packages: [3scale-operator]                   pkg=3scale-operator
:
INFO[0010] deleting packages                             pkg=web-terminal
INFO[0010] packages: [web-terminal]                      pkg=web-terminal
INFO[0010] Generating dockerfile                         packages="[cluster-logging elasticsearch-operator]"
INFO[0010] writing dockerfile: index.Dockerfile736121001  packages="[cluster-logging elasticsearch-operator]"
INFO[0010] running podman build                          packages="[cluster-logging elasticsearch-operator]"
INFO[0010] [podman build --format docker -f index.Dockerfile736121001 -t quay.io/daein/myindex:v4.9 .]  packages="[cluster-logging elasticsearch-operator]"

$ podman images
REPOSITORY                                                                TAG           IMAGE ID      CREATED         SIZE
quay.io/daein/myindex                                                     v4.9          1ea9feb9ea9a  56 seconds ago  101 MB

$ podman login quay.io

$ podman push quay.io/daein/myindex:v4.9
```

### Mirror the filtered images with remote registry, in this test once downloads the mirror images and mirror that with external registry.

```cmd
$ cat cred.json
{
  "auths": {
    "quay.io": {
      "auth": "...REDACTED...",
      "email": "...REDACTED..."
    },
    "registry.redhat.io": {
      "auth": "...REDACTED...",
      "email": "...REDACTED..."
    }
  }
}

$ mkdir /tmp/mirrors
$ oc adm catalog mirror \
  quay.io/daein/myindex:v4.9 \
  file:///mirrors \
  -a cred.json \
  --index-filter-by-os='linux/amd64'
:
info: Mirroring completed in 4m45.42s (9.751MB/s)
wrote mirroring manifests to manifests-myindex-1635429142

To upload local images to a registry, run:

	oc adm catalog mirror file://tmp/mirrors/daein/myindex:v4.9 REGISTRY/REPOSITORY

$ ls -d v2
v2

$ oc adm catalog mirror \
  file://tmp/mirrors/daein/myindex:v4.9 \
  quay.io/daein/mymirrors \
  -a cred.json \
  --index-filter-by-os='linux/amd64'
:
info: Mirroring completed in 39m22.41s (1.178MB/s)
no digest mapping available for file://tmp/mirrors/daein/myindex:v4.9, skip writing to ImageContentSourcePolicy
wrote mirroring manifests to manifests-mirrors/daein/myindex-1635429796
```

While mirroring the images to the quay.io, by default all registry is created with private mode.
So you should change it to public registry at the setting for all registry created by mirroring cmd.
This is just only required if you use quay.io for testing.

Lock mark means private registry at the quay.io.
![vpc_design](https://github.com/bysnupy/handson/blob/master/images/mirror01.png)

Change the registry to a public registry to access from your OCP cluster.
![vpc_design](https://github.com/bysnupy/handson/blob/master/images/mirror02.png)



### Creating the ImageContentSourcePolicy and CatalogSource

```cmd
$ ls -R ./manifests-mirrors/
./manifests-mirrors/:
daein

./manifests-mirrors/daein:
myindex-1635429796

./manifests-mirrors/daein/myindex-1635429796:
catalogSource.yaml  imageContentSourcePolicy.yaml  mapping.txt

$ cat ./manifests-mirrors/daein/myindex-1635429796/imageContentSourcePolicy.yaml
---
apiVersion: operator.openshift.io/v1alpha1
kind: ImageContentSourcePolicy
metadata:
  labels:
    operators.openshift.org/catalog: "true"
  name: mirrors-daein-myindex-0
spec:
  repositoryDigestMirrors:
  - mirrors:
    - quay.io/daein/mymirrors-openshift-logging-elasticsearch-rhel8-operator
    source: tmp/mirrors/daein/myindex/openshift-logging/elasticsearch-rhel8-operator
  - mirrors:
    - quay.io/daein/mymirrors-openshift4-ose-kube-rbac-proxy
    source: tmp/mirrors/daein/myindex/openshift4/ose-kube-rbac-proxy
  - mirrors:
    - quay.io/daein/mymirrors-openshift-logging-log-file-metric-exporter-rhel8
    source: tmp/mirrors/daein/myindex/openshift-logging/log-file-metric-exporter-rhel8
  - mirrors:
    - quay.io/daein/mymirrors-openshift-logging-elasticsearch-operator-bundle
    source: tmp/mirrors/daein/myindex/openshift-logging/elasticsearch-operator-bundle
  - mirrors:
    - quay.io/daein/mymirrors-openshift4-ose-oauth-proxy
    source: tmp/mirrors/daein/myindex/openshift4/ose-oauth-proxy
  - mirrors:
    - quay.io/daein/mymirrors-openshift-logging-cluster-logging-operator-bundle
    source: tmp/mirrors/daein/myindex/openshift-logging/cluster-logging-operator-bundle
  - mirrors:
    - quay.io/daein/mymirrors-openshift-logging-elasticsearch6-rhel8
    source: tmp/mirrors/daein/myindex/openshift-logging/elasticsearch6-rhel8
  - mirrors:
    - quay.io/daein/mymirrors-openshift-logging-elasticsearch-proxy-rhel8
    source: tmp/mirrors/daein/myindex/openshift-logging/elasticsearch-proxy-rhel8
  - mirrors:
    - quay.io/daein/mymirrors-openshift-logging-cluster-logging-rhel8-operator
    source: tmp/mirrors/daein/myindex/openshift-logging/cluster-logging-rhel8-operator
  - mirrors:
    - quay.io/daein/mymirrors-openshift-logging-fluentd-rhel8
    source: tmp/mirrors/daein/myindex/openshift-logging/fluentd-rhel8
  - mirrors:
    - quay.io/daein/mymirrors-openshift-logging-kibana6-rhel8
    source: tmp/mirrors/daein/myindex/openshift-logging/kibana6-rhel8
  - mirrors:
    - quay.io/daein/mymirrors-openshift-logging-logging-curator5-rhel8
    source: tmp/mirrors/daein/myindex/openshift-logging/logging-curator5-rhel8
    
$ oc create -f ./manifests-mirrors/daein/myindex-1635429796/imageContentSourcePolicy.yaml

$ oc get imageContentSourcePolicy
NAME                      AGE
mirrors-daein-myindex-0   26s

$ cat ./manifests-mirrors/daein/myindex-1635429796/catalogSource.yaml
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: mirrors/daein/myindex             <--- modify myindex
  ↓
  name: myindex                           
  namespace: openshift-marketplace
spec:
  image: quay.io/daein/mymirrors-tmp-mirrors-daein-myindex:v4.9
  sourceType: grpc

$ podman run --authfile cred.json -p50051:50051 \
  -it quay.io/daein/mymirrors-tmp-mirrors-daein-myindex:v4.9

another-terminal $ grpcurl -plaintext localhost:50051 api.Registry/ListPackages
{
  "name": "cluster-logging"
}
{
  "name": "elasticsearch-operator"
}

$ oc create -f ./manifests-mirrors/daein/myindex-1635429796/catalogSource.yaml

$ oc get pod -n openshift-marketplace
NAME                                    READY   STATUS    RESTARTS   AGE
marketplace-operator-6dc6dd9896-tpzdp   1/1     Running   0          10h
myindex-nqd9p                           1/1     Running   0          21s
```

### Check the OperatorHub menu at the web console

Before creating your CatalogSource.
![vpc_design](https://github.com/bysnupy/handson/blob/master/images/mirror03.png)

After that.
![vpc_design](https://github.com/bysnupy/handson/blob/master/images/mirror04.png)

Done.
