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

```
$ oc import-image docker.io/openshiftroadshow/parksmap-katacoda:1.2.0 --confirm
imagestream.image.openshift.io/parksmap-katacoda imported

$ oc get is
NAME                IMAGE REPOSITORY                                                                       TAGS     UPDATED
parksmap-katacoda   default-route-openshift-image-registry.apps.ocp4.example.local/test/parksmap-katacoda   1.2.0    4 seconds ago

$ oc describe is parksmap-katacoda 
Name:			parksmap-katacoda
Namespace:		test
Created:		19 minutes ago
Labels:			<none>
Annotations:		openshift.io/image.dockerRepositoryCheck=2020-05-18T07:26:53Z
Image Repository:	default-route-openshift-image-registry.apps.ocp4.example.local/test/parksmap-katacoda
Image Lookup:		local=false
Unique Images:		1
Tags:			1

1.2.0
  tagged from docker.io/openshiftroadshow/parksmap-katacoda:1.2.0

  * docker.io/openshiftroadshow/parksmap-katacoda@sha256:XXX...SHA256 Identifier of the certain image tags...XXX
      19 minutes ago
```

Note the above sha256 identifier for comparing for the image change trigger target configured in Deployment "image:" section after completing the config.

### Check the manifest of Deployment object before setting the image change trigger

This Deployment has just one container and the container refer the image directly using registry URL.

```
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
  resourceVersion: "52678941"
  selfLink: /apis/extensions/v1beta1/namespaces/test/deployments/test
  uid: eb5c3772-02b8-4629-a4f8-1207ff81f229
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: test
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
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
  availableReplicas: 1
  conditions:
  - lastTransitionTime: "2020-05-07T05:34:46Z"
    lastUpdateTime: "2020-05-07T05:36:07Z"
    message: ReplicaSet "test-7b9bbb87cf" has successfully progressed.
    reason: NewReplicaSetAvailable
    status: "True"
    type: Progressing
  - lastTransitionTime: "2020-05-16T12:28:26Z"
    lastUpdateTime: "2020-05-16T12:28:26Z"
    message: Deployment has minimum availability.
    reason: MinimumReplicasAvailable
    status: "True"
    type: Available
  observedGeneration: 1
  readyReplicas: 1
  replicas: 1
  updatedReplicas: 1
```

### Set the image change trigger using "oc set triggers" CLI

Set the trigger with ImageStream name using "--from-image" option as follows.

```
$ oc set triggers deployment/test -c parksmap-katacoda --from-image parksmap-katacoda:1.2.0
deployment.extensions/test triggers updated
```
You can see "image.openshift.io/triggers" annotation and "image:" section was changed same with the ImageStreamTag sha256 identifier.

```
$ oc get deploy/test -o yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "2"
    image.openshift.io/triggers: '[{"from":{"kind":"ImageStreamTag","name":"parksmap-katacoda:1.2.0"},"fieldPath":"spec.template.spec.containers[?(@.name==\"parksmap-katacoda\")].image"}]'
  creationTimestamp: "2020-05-07T01:41:27Z"
  generation: 3
  labels:
    app: test
  name: test
  namespace: test
  resourceVersion: "54767679"
  selfLink: /apis/extensions/v1beta1/namespaces/test/deployments/test
  uid: eb5c3772-02b8-4629-a4f8-1207ff81f229
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: test
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
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
  availableReplicas: 1
  conditions:
  - lastTransitionTime: "2020-05-16T12:28:26Z"
    lastUpdateTime: "2020-05-16T12:28:26Z"
    message: Deployment has minimum availability.
    reason: MinimumReplicasAvailable
    status: "True"
    type: Available
  - lastTransitionTime: "2020-05-07T05:34:46Z"
    lastUpdateTime: "2020-05-18T07:28:15Z"
    message: ReplicaSet "test-5c9bc7567f" has successfully progressed.
    reason: NewReplicaSetAvailable
    status: "True"
    type: Progressing
  observedGeneration: 3
  readyReplicas: 1
  replicas: 1
  updatedReplicas: 1
```

Done.
