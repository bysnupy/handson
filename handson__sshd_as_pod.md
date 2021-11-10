# How to run sshd as Pod on OpenShift v4

## Summary

It's just a kind of curious. I don't sugget running sshd as pod basically. I have not seen the use case required to run sshd ever.

## 1. Create an image for test sshd server and client as follows
The following Dockerfile is based on [moby/docs/example](https://github.com/moby/moby/blob/docs/docs/examples/running_ssh_service.Dockerfile).

```cmd
$ cat <<EOF ./Dockerfile 
FROM ubuntu:14.04

RUN apt-get update && apt-get install -y openssh-server
RUN mkdir /var/run/sshd
RUN echo 'root:redhat' | chpasswd
RUN sed -i 's/PermitRootLogin without-password/PermitRootLogin yes/' /etc/ssh/sshd_config
RUN sed -i 's/Port 22/Port 2222/' /etc/ssh/sshd_config

# SSH login fix. Otherwise user is kicked off after login
RUN sed 's@session\s*required\s*pam_loginuid.so@session optional pam_loginuid.so@g' -i /etc/pam.d/sshd

ENV NOTVISIBLE "in users profile"
RUN echo "export VISIBLE=now" >> /etc/profile

EXPOSE 2222
CMD ["/usr/sbin/sshd", "-D"]
EOF
```
## 2. Push the image to a external registry, in this time quay.io
```cmd
$ podman build .
:
COMMIT
--> d3a841e219d

$ podman tag d3a841e219d quay.io/daein/sshd:latest
$ podman login quay.io
$ podman push quay.io/daein/sshd:latest
```

## 3. Create test project and deploy the sshd server and client pods using above image
If your oc cli version is less than v4.9.0, you may use "--docker-image" instead of "--image".

```cmd
$ oc new-project test-sshd
$ oc new-app --name sshd-server --image=quay.io/daein/sshd:latest
$ oc adm policy add-scc-to-user privileged -z default
$ oc edit deploy sshd-server
:
    spec:
      containers:
      - image: quay.io/daein/sshd@sha256:...
        imagePullPolicy: IfNotPresent
        securityContext:               <-- ADDED
          privileged: true             <-- ADDED
:
$ oc new-app --name ssh-client --image=quay.io/daein/sshd:latest
```

## 4. Test to access sshd under clusternetworks on OpenShift v4.9.0
```cmd
$ oc get svc
NAME          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
ssh-client    ClusterIP   172.30.168.102   <none>        2222/TCP   3s
sshd-server   ClusterIP   172.30.74.162    <none>        2222/TCP   40s

$ oc get pod -o wide
NAME                           READY   STATUS    RESTARTS   AGE   IP            NODE                                           
ssh-client-7f87d6b594-v44dq    1/1     Running   0          12s   10.128.2.24   ip-10-0-206-193.ap-northeast-1.compute.internal
sshd-server-6cb894f49c-nsbm2   1/1     Running   0          24s   10.128.2.23   ip-10-0-206-193.ap-northeast-1.compute.internal
```
### 4.1. Test to access using sshd-server Pod IP
```cmd
$ oc rsh ssh-client-7f87d6b594-v44dq ssh -p 2222 root@10.128.2.23

Could not create directory '/.ssh'.
The authenticity of host '[10.128.2.23]:2222 ([10.128.2.23]:2222)' can't be established.
ECDSA key fingerprint is 08:46:cf:11:e8:3c:1c:af:11:de:ec:7e:15:1a:f8:76.
Are you sure you want to continue connecting (yes/no)? yes
Failed to add the host to the list of known hosts (/.ssh/known_hosts).

root@10.128.2.23's password: redhat
Welcome to Ubuntu 14.04 LTS (GNU/Linux 4.4.0-170-generic x86_64)

 * Documentation:  https://help.ubuntu.com/

The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

root@sshd-server-6cb894f49c-nsbm2:~# ifconfig
eth0      Link encap:Ethernet  HWaddr 0a:58:0a:80:02:17  
          inet addr:10.128.2.23  Bcast:10.128.3.255  Mask:255.255.254.0  <--- sshd-server Pod IP
:
```
### 4.2. Test using sshd-server Service IP
```cmd
$ oc rsh ssh-client-7f87d6b594-v44dq ssh -p 2222 root@172.30.74.162

Could not create directory '/.ssh'.
The authenticity of host '[172.30.74.162]:2222 ([172.30.74.162]:2222)' can't be established.
ECDSA key fingerprint is 08:46:cf:11:e8:3c:1c:af:11:de:ec:7e:15:1a:f8:76.
Are you sure you want to continue connecting (yes/no)? yes
Failed to add the host to the list of known hosts (/.ssh/known_hosts).

root@172.30.74.162's password: redhat
Welcome to Ubuntu 14.04.6 LTS (GNU/Linux 4.18.0-305.19.1.el8_4.x86_64 x86_64)

 * Documentation:  https://help.ubuntu.com/
Last login: Wed Nov 10 09:38:37 2021 from 10-128-2-24.ssh-client.test-sshd.svc.cluster.local

root@sshd-server-6cb894f49c-nsbm2:~# ifconfig
eth0      Link encap:Ethernet  HWaddr 0a:58:0a:80:02:17  
          inet addr:10.128.2.23  Bcast:10.128.3.255  Mask:255.255.254.0  <--- sshd-server Pod IP
:
```

Done.
