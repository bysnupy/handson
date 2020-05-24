# How do we appropriately configure readiness/liveness probes for suppressing lost requests ?

## Summary
I'd like to show you simple demonstration of the pod rolling update through two patterns.
First pattern is configured probes based on middle ware healthy check.
The other pattern is configured probes based on service healthy check.

You may learn why we configure the probes based on the service health check through comparison of the above two patterns.

![ocp4 rolling_update](https://github.com/bysnupy/handson/blob/master/ocp4_rolling_update_dc.png)

## Compare the testing systems

### Create the testing project
```cmd
$ oc new-project test-rolling-update
```

### Create the testing pod and deploymentConfig

The following command will create deploymentConfig that deploy the pod that it is initialized middle ware first, and after 10 seconds the application service would be initialized.
Such as, when the pod has just started, we can response the "MIDDLE WARE OK" through "http://<service IP>:8080/middleware_health/index.html".
And after 10 seconds we can also get the "SERVICE OK: V1.0" from the pod through "http://<service IP>:8080/service_health/index.html" on the testing pod.
But before 10 seconds, if we access to "http://<service IP>:8080/service_health/index.html", we are not able to get any messages.

```cmd
$ oc run test-pod --image=registry.access.redhat.com/rhel7 \
  -- bash -c \
  'mkdir -p /tmp/test/{service_health,middleware_health}; cd /tmp/test; echo "MIDDLE WARE OK" > middleware_health/index.html; nohup sleep 10 && echo "SERVICE OK: V1.0" > service_health/index.html & python -m SimpleHTTPServer 8080'
```

We are going to test through the following Service IP and port.
```cmd
$ oc create -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  labels:
    run: test-pod
  name: test-pod-svc
spec:
  ports:
  - name: 8080-8080
    port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    run: test-pod
  type: ClusterIP
EOF
```

```cmd
$ oc get svc
NAME           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
test-pod-svc   ClusterIP   172.30.225.73   <none>        8080/TCP   3m
```

### Take the status and request results of the running test pod

Run this command for monitoring the request response and each service status, such as a middle ware and a application servcie.

```cmd
$ while :; do echo $(date '+%H:%M:%S') - $(curl --connect-timeout 1 -s http://172.30.225.73:8080/middleware_health/index.html): $(curl --connect-timeout 1 -s http://172.30.225.73:8080/service_health/index.html) ; sleep 1; done
```

Run this command on other terminal for monitoring pod status transition.
```cmd
$ while :; do echo $(date '+%H:%M:%S') ---; oc get pod; sleep 1; done
```

## Let's test

### Patter 1, configuring the probes based on middle ware health status

```cmd
$ oc edit dc/test-pod
:
        livenessProbe:
          exec:
            command:
            - curl
            - -f
            - http://localhost:8080/middleware_health/index.html
          failureThreshold: 3
          initialDelaySeconds: 60
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 15
        readinessProbe:
          exec:
            command:
            - curl
            - -f
            - http://localhost:8080/middleware_health/index.html
          failureThreshold: 3
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 15
```

Change the "SERVICE OK: V1.0" -> "SERVICE OK: V2.0" for triggering the rolling update of the deploymentConfig.
```cmd
$ oc edit dc/test-pod
```

You can see some failed requests when new pod become "Running" and old pod become "Terminating" as follows

Pod status
```cmd
  13:56:04 ---
  NAME                READY     STATUS    RESTARTS   AGE
  test-pod-6-gpjcp    1/1       Running   0          10m
  test-pod-7-deploy   1/1       Running   0          12s
  test-pod-7-jtv44    0/1       Running   0          9s
  13:56:05 ---
  NAME                READY     STATUS        RESTARTS   AGE
  test-pod-6-gpjcp    1/1       Terminating   0          10m
  test-pod-7-deploy   1/1       Running       0          14s
  test-pod-7-jtv44    1/1       Running       0          11s
  13:56:07 ---
  NAME                READY     STATUS        RESTARTS   AGE
  test-pod-6-gpjcp    1/1       Terminating   0          10m
  test-pod-7-deploy   1/1       Running       0          15s
  test-pod-7-jtv44    1/1       Running       0          12s
```

Request results
```
  13:56:05 - MIDDLE WARE OK: SERVICE OK: V1.0
  13:56:06 - MIDDLE WARE OK: <!DOCTYPE html PUBLIC "-//W3C//DTD HTML 3.2 Final//EN"><html> <title>Directory listing for /service_health/</title> <body> <h2>Directory listing for /service_health/</h2> <hr> <ul> </ul> <hr> </body> </html>
  13:56:07 - MIDDLE WARE OK: <!DOCTYPE html PUBLIC "-//W3C//DTD HTML 3.2 Final//EN"><html> <title>Directory listing for /service_health/</title> <body> <h2>Directory listing for /service_health/</h2> <hr> <ul> </ul> <hr> </body> </html>
  13:56:08 - MIDDLE WARE OK: SERVICE OK: V2.0
```

### Patter 2, configuring the probes based on service health status

```cmd
$ oc edit dc/test-pod
:
        livenessProbe:
          exec:
            command:
            - curl
            - -f
            - http://localhost:8080/service_health/index.html
          failureThreshold: 3
          initialDelaySeconds: 60
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 15
        readinessProbe:
          exec:
            command:
            - curl
            - -f
            - http://localhost:8080/service_health/index.html
          failureThreshold: 3
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 15
```


Change the "SERVICE OK: V2.0" -> "SERVICE OK: V3.0" for triggering the rolling update of the deploymentConfig.
```cmd
$ oc edit dc/test-pod
```

You can see some no lost requests when new pod become "Running" and old pod become "Terminating" as follows

```cmd
  14:00:25 ---
  NAME                READY     STATUS    RESTARTS   AGE
  test-pod-8-2lpwn    1/1       Running   0          2m
  test-pod-9-deploy   1/1       Running   0          18s
  test-pod-9-jkbnw    1/1       Running   0          14s
  14:00:26 ---
  NAME                READY     STATUS        RESTARTS   AGE
  test-pod-8-2lpwn    1/1       Terminating   0          2m
  test-pod-9-deploy   1/1       Running       0          19s
  test-pod-9-jkbnw    1/1       Running       0          15s
  14:00:27 ---
  NAME                READY     STATUS        RESTARTS   AGE
  test-pod-8-2lpwn    1/1       Terminating   0          2m
  test-pod-9-deploy   1/1       Running       0          21s
  test-pod-9-jkbnw    1/1       Running       0          17s
```

```cmd
  14:00:24 - MIDDLE WARE OK: SERVICE OK: V2.0
  14:00:25 - MIDDLE WARE OK: SERVICE OK: V2.0
  14:00:26 - MIDDLE WARE OK: SERVICE OK: V3.0
  14:00:27 - MIDDLE WARE OK: SERVICE OK: V3.0
```

Done.
