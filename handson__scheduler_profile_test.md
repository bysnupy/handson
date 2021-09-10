# Alter the Scheduler Profile configurations

This feature would show dramatic result on more bigger cluster, not small(masterx3, workerx3) cluster.

# Test procedures

## 1. Create a test project
```cmd
$ oc new-project test-scheduler-profile
```
## 2. Create 90 test pods with "cpu: 20m, memory: 40Mi" and check the running node ratio before test.
```cmd
// ip-10-0-176-92 is already higher resource usage.
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource                    Requests      Limits
  --------                    --------      ------
  cpu                         2300m (65%)   0 (0%)
  memory                      6675Mi (46%)  0 (0%)

$ oc new-app httpd
$ oc label node ip-10-0-176-92.ap-southeast-2.compute.internal resourceusage=high
$ oc patch deploy/httpd --type='json' -p='[{"op": "add", "path": "/spec/template/spec/nodeSelector", "value": {"resourceusage":"high"}}]'
$ oc scale --replicas=90 deploy/httpd
$ echo "$(oc get pod -o wide | grep -c ip-10-0-176-92):\
$(oc get pod -o wide | grep -c ip-10-0-129-58):\
$(oc get pod -o wide | grep -c ip-10-0-211-162)"
90:0:0
```
## 3. After change the scheduler profile to "LowNodeUtilization", more pods would be scheduled to ip-10-0-129-58 and ip-10-0-211-162 than ip-10-0-176-92.
```cmd
$ oc patch scheduler cluster --type='json' -p='[{"op": "add", "path": "/spec/profile", "value": "LowNodeUtilization"}]'
$ oc new-app httpd --name lownodeutil
$ oc scale --replicas=210 deploy/lownodeutil
$ echo "$(oc get pod -o wide | grep -c ip-10-0-176-92):\
$(oc get pod -o wide | grep -c ip-10-0-129-58):\
$(oc get pod -o wide | grep -c ip-10-0-211-162)"
149:76:75
+59:76:75
```
## 4. After change the scheduler profile to "HighNodeUtilization", more pods would be scheduled potd to the ip-10-0-176-92 than "LowNodeUtilization".
```cmd
$ oc patch scheduler cluster --type='json' -p='[{"op": "add", "path": "/spec/profile", "value": "HighNodeUtilization"}]'
$ oc new-app httpd --name highnodeutil
$ oc scale --replicas=210 deploy/highnodeutil
$ echo "$(oc get pod -o wide | grep -c ip-10-0-176-92):\
$(oc get pod -o wide | grep -c ip-10-0-129-58):\
$(oc get pod -o wide | grep -c ip-10-0-211-162)"
167:63:70
+77:63:70
```
## 5. After change the scheduler profile to "NoScoring", it would be scheduled randomly across remained nodes as soon as possible.
```cmd
$ oc patch scheduler cluster --type='json' -p='[{"op": "add", "path": "/spec/profile", "value": "NoScoring"}]'
$ oc new-app httpd --name noscore
$ oc scale --replicas=210 deploy/noscore
$ echo "$(oc get pod -o wide | grep -c ip-10-0-176-92):\
$(oc get pod -o wide | grep -c ip-10-0-129-58):\
$(oc get pod -o wide | grep -c ip-10-0-211-162)"
167:60:73
+77:60:73
```

