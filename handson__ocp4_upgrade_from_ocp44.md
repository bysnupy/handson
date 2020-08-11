# The changes of the updagrde using CLI as of OCP44

## Summary
Before you update your cluster, you must manually create a ConfigMap that contains the signatures of the release images that you use. 
This signature allows the Cluster Version Operator (CVO) to verify that the release images have not been modified by comparing the expected and actual image signatures. If you are upgrading from version 4.4.8 or later, you can use the oc CLI to create the ConfigMap using "--apply-release-image-signature" option. If you are upgrading from an earlier version, you must use the manual method.

In this demonstration, upgrade from 4.5.3 to 4.5.4 on the restricted network envirnment.

## Demonstration

### Set required environment variables after oc CLI binary upgrade
```console
// oc CLI update
$ oc version
Client Version: 4.5.4
Server Version: 4.5.3
Kubernetes Version: v1.18.3+3107688

// change OCP_RELEASE to target version 
export OCP_RELEASE=4.5.4
export LOCAL_REGISTRY='mirror-reg.ocp.rhev.local:5000'
export LOCAL_REPOSITORY='ocp4/openshift4'
export PRODUCT_REPO='openshift-release-dev' 
export LOCAL_SECRET_JSON='/root/ocp45rt/pull-secret-mirroring.json' 
export RELEASE_NAME="ocp-release" 
export ARCHITECTURE=x86_64
```

### Mirror the images of new version and create the signature ConfigMap
```console
// mirror with target version images, 
// if "--apply-release-image-signature" is specified, the required signature configmap is created automatically.
$ oc adm release mirror -a ${LOCAL_SECRET_JSON} --from=quay.io/${PRODUCT_REPO}/${RELEASE_NAME}:${OCP_RELEASE}-${ARCHITECTURE} \
  --to=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY} --apply-release-image-signature
info: Mirroring 110 images to mirror-reg.ocp.rhev.local:5000/ocp4/openshift4 ...
:
sha256:02dfcae8f6a67e715380542654c952c981c59604b1ba7f569b13b9e5d0fbbed3 mirror-reg.ocp.rhev.local:5000/ocp4/openshift4:4.5.4
:

// Verify the signature configmap
# oc get cm -n openshift-config-managed
NAME                                                                      DATA   AGE
:
sha256-02dfcae8f6a67e715380542654c952c981c59604b1ba7f569b13b9e5d0fbbed3   1      10m
```

### Upgrade
```console
// upgrade from 4.5.3 to 4.5.4

$ oc adm upgrade --allow-explicit-upgrade \
   --to-image ${LOCAL_REGISTRY}/${LOCAL_REPOSITORY}@sha256:02dfcae8f6a67e715380542654c952c981c59604b1ba7f569b13b9e5d0fbbed3
warning: The requested upgrade image is not one of the available updates.  You have used --allow-explicit-upgrade to the update to preceed anyway
Updating to release image mirror-reg.ocp.rhev.local:5000/ocp4/openshift4@sha256:02dfcae8f6a67e715380542654c952c981c59604b1ba7f569b13b9e5d0fbbed3
```

## Upgrade state on the web console

### Before upgrade
![ocp45_upgrade_before](https://github.com/bysnupy/handson/blob/master/ocp4_upgrade_from_ocp44_before.png)

### After upgrade
![ocp45_upgrade_after](https://github.com/bysnupy/handson/blob/master/ocp4_upgrade_from_ocp44_after.png)
