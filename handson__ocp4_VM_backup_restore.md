# OpenShift Virtualization VirtualMachine backup using PV/PVC backup on OCPv4

VirualMachine data is usually stored in the PV/PVC on OCPv4.
This testing is for checking if the PV/PVC backup is valid to VM backup too.

## Generate some data for checking if the data is remained after backup
```console
$ mkdir /home/fedora/test_dir
$ echo TestData > /home/fedora/test_dir/test.txt
```

## Stop the target VM to backup first.

## Create PVC and PV using storageclass dynamically in this test.
```console
$ oc create -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: new-backup-pvc
  namespace: backup-test
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 15Gi
  storageClassName: gp2
EOF
```

## Create and run a temporary pod for backup task
```console
$ oc create deployment sleep --image=registry.access.redhat.com/rhel7/rhel-tools -- tail -f /dev/null
```

## Setting up both PVC/PV to the deployment of a temporary pod
```console
$ oc set volume deployment/sleep --add -t pvc --name=old-pvc --claim-name=backup-test-rootdisk-3v2kg --mount-path=/old-pvc
$ oc set volume deployment/sleep --add -t pvc --name=new-pvc --claim-name=new-backup-pvc --mount-path=/new-pvc
```

## Access the pod and backup using "rsync" as follows
```console
$ oc get pvc
NAME                               STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
backup-test-rootdisk-3v2kg         Bound    pvc-c7d8dde6-28a5-4803-aaf8-9b93efab4f9e   15Gi       RWO            gp2            40m
new-backup-pvc                     Bound    pvc-28f99c53-76d3-46e4-8960-6adaa441361a   15Gi       RWO            gp2            2m32s

$ oc get pod
NAME                     READY   STATUS    RESTARTS   AGE
sleep-6f979fcb4d-mvdrz   1/1     Running   0          77s

$ oc rsh sleep-6f979fcb4d-mvdrz

sh-4.2$ ls -al /{old,new}-pvc
/new-pvc:
total 20
drwxrwsr-x. 3 root 1000610000  4096 May 26 06:42 .
drwxr-xr-x. 1 root root          36 May 26 06:42 ..

/old-pvc:
total 1025212
drwxrwsr-x. 2 root 1000610000        4096 May 26 06:03 .
drwxr-xr-x. 1 root root                36 May 26 06:42 ..
-rw-rw----. 1 root 1000610000 15727509504 May 26 06:31 disk.img

sh-4.2$ rsync -avxHAX --progress /old-pvc/* /new-pvc
sending incremental file list
disk.img
 15,727,509,504 100%  157.24MB/s    0:01:35 (xfr#1, to-chk=0/1)

sent 15,731,349,359 bytes  received 35 bytes  163,019,164.70 bytes/sec
total size is 15,727,509,504  speedup is 1.00
```

## Create new VirtualMachine using "existing PVC"

Select with "Existing PVC" after "Create Virtual Machine" -> "New with Wizard"

![step1](https://github.com/bysnupy/handson/blob/master/backupvm1.png)

Add disk with "Using an existing PVC" we created previously

![step2](https://github.com/bysnupy/handson/blob/master/backupvm2.png)

Select the disk as "Boot Source"

![step3](https://github.com/bysnupy/handson/blob/master/backupvm3.png)

And run the new VM.

## Verify if the test data is restored

Ok, the data is restored intactly.

![step4](https://github.com/bysnupy/handson/blob/master/backupvm4.png)

Done.
