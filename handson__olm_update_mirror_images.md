# How to update the Operator using mirroring images

## Summary

I show you how to update the operator version using mirror images through external registry.
First I will deploy the pipeline operator to OCPv4.8 using mirror images.
After that I will update the pipeline operator as new version after upgrade the OCPv4.9.

## Disable the sources for the default catalogs

```cmd
$ oc patch OperatorHub cluster --type json \
  -p '[{"op": "add", "path": "/spec/disableAllDefaultSources", "value": true}]'

$ oc get pod -n openshift-marketplace
NAME                                    READY   STATUS    RESTARTS   AGE
marketplace-operator-86c47db54b-fs6rc   1/1     Running   1          48m
```

![vpc_design](https://github.com/bysnupy/handson/blob/master/images/olmupdate01.png)

## Creating index image for v4.8 and push it to the external registry

```
$ podman login registry.redhat.io
Login Succeeded!

$ opm index prune \
  -f registry.redhat.io/redhat/redhat-operator-index:v4.8 \
  -p openshift-pipelines-operator-rh \
  -t quay.io/daein/myindex:v4.8.filtered

INFO[0000] pruning the index                             packages="[openshift-pipelines-operator-rh]"
INFO[0000] Pulling previous image registry.redhat.io/redhat/redhat-operator-index:v4.8 to get metadata  packages="[openshift-pipelines-operator-rh]"
INFO[0000] running /usr/bin/podman pull registry.redhat.io/redhat/redhat-operator-index:v4.8  packages="[openshift-pipelines-operator-rh]"
INFO[0004] running /usr/bin/podman pull registry.redhat.io/redhat/redhat-operator-index:v4.8  packages="[openshift-pipelines-operator-rh]"
INFO[0007] Getting label data from previous image        packages="[openshift-pipelines-operator-rh]"
INFO[0007] running podman inspect                        packages="[openshift-pipelines-operator-rh]"
INFO[0007] running podman create                         packages="[openshift-pipelines-operator-rh]"
INFO[0007] running podman cp                             packages="[openshift-pipelines-operator-rh]"
INFO[0011] running podman rm                             packages="[openshift-pipelines-operator-rh]"
:
INFO[0015] Generating dockerfile                         packages="[openshift-pipelines-operator-rh]"
INFO[0015] writing dockerfile: index.Dockerfile900866235  packages="[openshift-pipelines-operator-rh]"
INFO[0015] running podman build                          packages="[openshift-pipelines-operator-rh]"
INFO[0015] [podman build --format docker -f index.Dockerfile900866235 -t quay.io/daein/myindex:v4.8.filtered .]  packages="[openshift-pipelines-operator-rh]"

$ podman images
REPOSITORY                                                                TAG            IMAGE ID      CREATED         SIZE
quay.io/daein/myindex                                                     v4.8.filtered  50d30bf440a0  52 seconds ago  139 MB
:

$ podman login quay.io
Login Succeeded!

$ podman push quay.io/daein/myindex:v4.8.filtered
```

![vpc_design](https://github.com/bysnupy/handson/blob/master/images/olmupdate02.png)

## Mirroring the images

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

$ oc adm catalog mirror \
  quay.io/daein/myindex:v4.8.filtered \
  quay.io/daein \
  -a cred.json \
  --index-filter-by-os='linux/amd64'
:
info: Mirroring completed in 2h34m58.36s (1.36MB/s)
no digest mapping available for quay.io/daein/myindex:v4.8.filtered, skip writing to ImageContentSourcePolicy
wrote mirroring manifests to manifests-myindex-1635469041
```

If you use quay.io as your external registry like this test, you should change it to public registry for all registries you mirrored.

The red lock mark means the registry is private mode.

![vpc_design](https://github.com/bysnupy/handson/blob/master/images/olmupdate03.png)

You can change it at the setting menu of each registry.
![vpc_design](https://github.com/bysnupy/handson/blob/master/images/olmupdate04.png)

## Installing Pipeline Operator after creating ImageContentSourcePolicy and CatalogSource

```cmd
$ ls -R ./manifests-myindex-1635469041/
./manifests-myindex-1635469041/:
catalogSource.yaml  imageContentSourcePolicy.yaml  mapping.txt

$ cat ./manifests-myindex-1635469041/imageContentSourcePolicy.yaml
kind: ImageContentSourcePolicy
metadata:
  labels:
    operators.openshift.org/catalog: "true"
  name: myindex-0
spec:
  repositoryDigestMirrors:
  - mirrors:
    - quay.io/daein/openshift-pipelines-pipelines-triggers-core-interceptors-rhel8
    source: registry.redhat.io/openshift-pipelines/pipelines-triggers-core-interceptors-rhel8
  :
  - mirrors:
    - quay.io/daein/openshift-pipelines-tech-preview-pipelines-operator-proxy-rhel8
    source: registry.redhat.io/openshift-pipelines-tech-preview/pipelines-operator-proxy-rhel8
  - mirrors:
    - quay.io/daein/ocp-tools-43-tech-preview-source-to-image-rhel8
    source: registry.redhat.io/ocp-tools-43-tech-preview/source-to-image-rhel8

$ cat ./manifests-myindex-1635469041/catalogSource.yaml
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: myindex
  namespace: openshift-marketplace
spec:
  image: quay.io/daein/daein-myindex:v4.8.filtered
  sourceType: grpc

$ oc create -f ./manifests-myindex-1635469041/imageContentSourcePolicy.yaml

$ oc create -f ./manifests-myindex-1635469041/catalogSource.yaml

$ oc get pod -n openshift-marketplace
NAME                                    READY   STATUS    RESTARTS   AGE
marketplace-operator-86c47db54b-fs6rc   1/1     Running   1          4h42m
myindex-xvnlq                           1/1     Running   0          28s
```

The Pipeline Opearator is listed at the web console as follows after creating above required resources.

![vpc_design](https://github.com/bysnupy/handson/blob/master/images/olmupdate05.png)

Before installing, in my case as you can see the following capture, the Pipeline Operator version is up-to-date(v1.5.2 at that time).

![vpc_design](https://github.com/bysnupy/handson/blob/master/images/olmupdate06.png)

Installed successfully.

![vpc_design](https://github.com/bysnupy/handson/blob/master/images/olmupdate07.png)

## Upgrade the OCP cluster to the latest version

In this test, I will upgrade the cluster from v4.8.17 to v4.9.0.

...WIP...
