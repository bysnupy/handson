# Manual upgrade on OCP4 in restricted network

## Summary

I'd like to demonstrate you how to upgrade OCP4.3 cluster from 4.3.0 to 4.3.13 in restricted network.

## Mirror the release images from 4.3.0 to 4.3.13

You should sync the mirror images on the client which can access to both Internet and your private mirror registry.

### Chnage to 4.3.0-x86_64 to 4.3.13-x86_64 before exporting "OCP_RELEASE".
```
$ export OCP_RELEASE=4.3.13-x86_64
```

### Mirror the target version images
``` 
$ export LOCAL_REGISTRY='mirror.priv.example.com:5000' 
$ export LOCAL_REPOSITORY='ocp4/openshift4' 
$ export PRODUCT_REPO='openshift-release-dev' 
$ export LOCAL_SECRET_JSON='pull-secret.txt' 
$ export RELEASE_NAME="ocp-release"
$ oc adm -a ${LOCAL_SECRET_JSON} release mirror \
  --from=quay.io/${PRODUCT_REPO}/${RELEASE_NAME}:${OCP_RELEASE} \
  --to=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY} \
  --to-release-image=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY}:${OCP_RELEASE}
```

### Take a note the digest of your target version release image from the above command output.
```
info: Mirroring 103 images to mirror.priv.example.com:5000/ocp4/openshift4 ...
mirror.priv.example.com:5000/
  ocp4/openshift4
    manifests:
      :
      sha256:xxx...xxx -> 4.3.13-x86_64
:
sha256:xxx...xxx mirror.priv.example.com:5000/ocp4/openshift4:4.3.13-x86_64
:
```

## Upgrade OCP4 cluster manually

You can upgrade the following command and options.

```
oc adm upgrade \
   --to-image mirror.priv.example.com:5000/ocp4/openshift4@sha256:xxx...xxx \
   --allow-explicit-upgrade \
   --force
```

### Progress state in Web console

Look at the desired image digest, it's same with above "--to-image" digest. 

![ocp4 manual upgrade1](https://github.com/bysnupy/handson/blob/master/ocp4_manual_upgrade1.png)

You can see progress percentage to complete at the "Cluster Operators" tab.

![ocp4 manual upgrade1](https://github.com/bysnupy/handson/blob/master/ocp4_manual_upgrade2.png)

You can finally upgrade after taking some times, then the "Last Completed Version" will be changed target version, such as "4.3.13" here.

![ocp4 manual upgrade1](https://github.com/bysnupy/handson/blob/master/ocp4_manual_upgrade3.png)


Done.
