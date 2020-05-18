# How to configure the image trigger to Deployment object on OCP4 like DeploymentConfig ?

## Summary
I'd like to show you how to set the image trigger to Deployment Kubernetes object like DeploymentConfig on OCP4.
OpenShift implemented DeploymentConfig as extended object for deployment process, 
the object has specific features like the image change trigger, automatic rollback and so on unlike Deployment object.

We can simply configure the image change trigger to Deployment for detecting reference image changes using "image.openshift.io/triggers" annotation as follows.
Refer [Kubernetes Resources](https://docs.openshift.com/container-platform/3.11/dev_guide/managing_images.html#image-stream-kubernetes-resources) for more details, even though the source about OpenShift v3.11. But the annotation details is same with OCP4.

## Configuration steps

### Create an ImageStream using a target image

For detecting the image change through ImageStream object, so we create the ImageStreamTag using the target images.

```cmd
$ oc import-image docker.io/openshiftroadshow/parksmap-katacoda:1.0.0 --confirm
imagestream.image.openshift.io/parksmap-katacoda imported

$ oc get is
NAME                IMAGE REPOSITORY                                                                       TAGS     UPDATED
parksmap-katacoda   default-route-openshift-image-registry.apps.ocp4.example.local/test/parksmap-katacoda   1.0.0    4 seconds ago

$ oc tag parksmap-katacoda:1.0.0 parksmap-katacoda:latest

$ oc describe is parksmap-katacoda 
Name:			parksmap-katacoda
Namespace:		test
Created:		12 minutes ago
Labels:			<none>
Annotations:		openshift.io/image.dockerRepositoryCheck=2020-05-18T08:15:41Z
Image Repository:	default-route-openshift-image-registry.apps.ocp43.kvm.local/test/parksmap-katacoda
Image Lookup:		local=false
Unique Images:		1
Tags:			2

latest
  tagged from parksmap-katacoda@sha256:XXX...SHA256 Identifier of the certain image tags...XXX

  * docker.io/openshiftroadshow/parksmap-katacoda@sha256:XXX...SHA256 Identifier of the certain image tags...XXX
      10 minutes ago

1.0.0
  tagged from docker.io/openshiftroadshow/parksmap-katacoda:1.0.0

  * docker.io/openshiftroadshow/parksmap-katacoda@sha256:XXX...SHA256 Identifier of the certain image tags...XXX
      12 minutes ago

```

Note the above sha256 identifier for comparing for the image change trigger target configured in Deployment "image:" section after completing the config.

### Check the manifest of Deployment object before setting the image change trigger

This Deployment has just one container and the container refer the image directly using registry URL.

```cmd
$ oc get deploy/test -o yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
  creationTimestamp: "2020-05-07T01:41:27Z"
  generation: 1
  labels:
    app: test
  name: test
  namespace: test
:
spec:
:
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: test
    spec:
      containers:
      - image: docker.io/openshiftroadshow/parksmap-katacoda:1.2.0
        imagePullPolicy: IfNotPresent
        name: parksmap-katacoda
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
status:
:
```

### Set the image change trigger using "oc set triggers" CLI

Set the trigger with ImageStream name using "--from-image" option as follows.

```cmd
$ oc set triggers deployment/test -c parksmap-katacoda --from-image parksmap-katacoda:latest
deployment.extensions/test triggers updated
```
You can see "image.openshift.io/triggers" annotation and "image:" section was changed same with the ImageStreamTag sha256 identifier.

```cmd
$ oc get deploy/test -o yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "2"
    image.openshift.io/triggers: '[{"from":{"kind":"ImageStreamTag","name":"parksmap-katacoda:latest"},"fieldPath":"spec.template.spec.containers[?(@.name==\"parksmap-katacoda\")].image"}]'
  creationTimestamp: "2020-05-07T01:41:27Z"
  generation: 3
  labels:
    app: test
  name: test
  namespace: test
:
spec:
:
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: test
    spec:
      containers:
      - image: docker.io/openshiftroadshow/parksmap-katacoda@sha256:XXX...SHA256 Identifier of the certain image tags...XXX
        imagePullPolicy: IfNotPresent
        name: parksmap-katacoda
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
status:
:
```

### Testing

Update the ImageStream with new image and watch the "image:" section changed with new image sha256 identifier automatically after that.

```cmd
$ oc tag docker.io/openshiftroadshow/parksmap-katacoda:1.2.0 parksmap-katacoda:latest 
Tag parksmap-katacoda:latest set to docker.io/openshiftroadshow/parksmap-katacoda:1.2.0.

$ oc describe is parksmap-katacoda 
Name:			parksmap-katacoda
Namespace:		test
Created:		19 minutes ago
Labels:			<none>
Annotations:		openshift.io/image.dockerRepositoryCheck=2020-05-18T08:25:53Z
Image Repository:	default-route-openshift-image-registry.apps.ocp43.kvm.local/test/parksmap-katacoda
Image Lookup:		local=false
Unique Images:		2
Tags:			2

latest
  tagged from docker.io/openshiftroadshow/parksmap-katacoda:1.2.0

  * docker.io/openshiftroadshow/parksmap-katacoda@sha256:YYY...SHA256 Identifier of the certain image tags...YYY
      9 minutes ago
    docker.io/openshiftroadshow/parksmap-katacoda@sha256:XXX...SHA256 Identifier of the certain image tags...XXX
      17 minutes ago

1.0.0
  tagged from docker.io/openshiftroadshow/parksmap-katacoda:1.0.0

  * docker.io/openshiftroadshow/parksmap-katacoda@sha256:XXX...SHA256 Identifier of the certain image tags...XXX
      19 minutes ago
$ oc get deploy/test -o yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "2"
    image.openshift.io/triggers: '[{"from":{"kind":"ImageStreamTag","name":"parksmap-katacoda:latest"},"fieldPath":"spec.template.spec.containers[?(@.name==\"parksmap-katacoda\")].image"}]'
  creationTimestamp: "2020-05-07T01:41:27Z"
  generation: 3
  labels:
    app: test
  name: test
  namespace: test
:
spec:
:
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: test
    spec:
      containers:
      - image: docker.io/openshiftroadshow/parksmap-katacoda@sha256:YYY...SHA256 Identifier of the certain image tags...YYY
        imagePullPolicy: IfNotPresent
        name: parksmap-katacoda
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
status:
:
```

Done.
