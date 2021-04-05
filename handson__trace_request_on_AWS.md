# How to trace the request from external sources the L4 traffic on the OpenShift with AWS

## Capture the tcpdump

### Check the nodes running the router pods
```console
$ oc get pod -n openshift-ingress -o wide
NAME                              READY   STATUS    RESTARTS   AGE     IP            NODE                                              NOMINATED NODE   READINESS GATES
router-default-56589566bc-nb52j   1/1     Running   0          4d22h   10.128.2.7    ip-10-0-134-187.ap-northeast-1.compute.internal   <none>           <none>
router-default-56589566bc-tghb9   1/1     Running   1          4d22h   10.129.2.10   ip-10-0-198-72.ap-northeast-1.compute.internal    <none>           <none>
```

### Run each debug pod of the each node running router pod

```console
$ oc debug node/ip-10-0-134-187.ap-northeast-1.compute.internal
Creating debug namespace/openshift-debug-node-8rxtl ...
Starting pod/ip-10-0-134-187ap-northeast-1computeinternal-debug ...
To use host binaries, run `chroot /host`
Pod IP: 10.0.134.187
If you don't see a command prompt, try pressing enter.
sh-4.4# tcpdump -Z root -G 600 -w ./tcpdump_%Y%m%d_%H%M%S.pcap -i any -s 0
tcpdump: listening on any, link-type LINUX_SLL (Linux cooked), capture size 262144 bytes

// At the another terminal
$ oc debug node/ip-10-0-198-72.ap-northeast-1.compute.internal
Creating debug namespace/openshift-debug-node-2dnn5 ...
Starting pod/ip-10-0-198-72ap-northeast-1computeinternal-debug ...
To use host binaries, run `chroot /host`
Pod IP: 10.0.198.72
If you don't see a command prompt, try pressing enter.
sh-4.4# tcpdump -Z root -G 600 -w ./tcpdump_%Y%m%d_%H%M%S.pcap -i any -s 0
tcpdump: listening on any, link-type LINUX_SLL (Linux cooked), capture size 262144 bytes
```

## Enable the ELB access logs

### Create S3 bucket to store the logs with region the ELB created

```console
$ aws s3api create-bucket \
  --bucket test-trace-logs \
  --create-bucket-configuration "LocationConstraint=ap-northeast-1"
{
    "Location": "http://test-trace-logs.s3.amazonaws.com/"
}

$ aws s3 ls | grep test-trace-logs
2021-04-05 12:16:27 test-trace-logs
```

### Put the bucket policy to be relaxed the permission

```console
$ cat <<EOF > s3-bucket-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::582318560864:root"
      },
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::test-trace-logs/AWSLogs/AWS_ACCOUNT_ID/*"
    },
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "delivery.logs.amazonaws.com"
      },
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::test-trace-logs/AWSLogs/AWS_ACCOUNT_ID/*",
      "Condition": {
        "StringEquals": {
          "s3:x-amz-acl": "bucket-owner-full-control"
        }
      }
    },
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "delivery.logs.amazonaws.com"
      },
      "Action": "s3:GetBucketAcl",
      "Resource": "arn:aws:s3:::test-trace-logs"
    }
  ]
}
EOF

$ aws s3api put-bucket-policy \
  --bucket test-trace-logs \
  --policy file://s3-bucket-policy.json

$ aws s3api get-bucket-policy \
  --bucket test-trace-logs | jq -r .Policy | jq
```

### Enable the ELB access logs using annotations of the LoadBalancer Service

```console
$ oc annotate -n openshift-ingress service router-default --overwrite \
    service.beta.kubernetes.io/aws-load-balancer-access-log-enabled="true" \
    service.beta.kubernetes.io/aws-load-balancer-access-log-emit-interval="5" \
    service.beta.kubernetes.io/aws-load-balancer-access-log-s3-bucket-name="test-trace-logs"
service/router-default annotated
```

## Enable the Flow logs on the VPC

### Create flow log in your VPC

We can specify above S3 bucket ARN this time again without additional configurations.

## Access with request-id header

For a while run the JMeter with random request-id header. 

## Get all capturing data and logs for checking a specific request-id request

### Get the related pcap files using "oc copy" after stopping tcpdump cmd in the debug pods
```console
$ oc get pod -A | grep openshift-debug-node
openshift-debug-node-2dnn5zgqp2                    ip-10-0-198-72ap-northeast-1computeinternal-debug                          1/1     Running        0          176m
openshift-debug-node-8rxtlcxcbq                    ip-10-0-134-187ap-northeast-1computeinternal-debug                         1/1     Running        0          3h15m

$ oc cp -n openshift-debug-node-2dnn5zgqp2 \
  ip-10-0-198-72ap-northeast-1computeinternal-debug:tcpdump_20210405_054132.pcap \
  ./tcpdump_20210405_054132.pcap

$ oc cp -n openshift-debug-node-8rxtlcxcbq \
  ip-10-0-134-187ap-northeast-1computeinternal-debug:tcpdump_20210405_063341.pcap \
  ./tcpdump_20210405_063341.pcap
```

Get the specific request session using "request-id: abcdefghijk" header in this test.

```console
$ TZ=UTC tshark -n -t ud  -r tcpdump_20210405_063341.pcap  -Y 'http.request and http contains "request-id: abcdefghijk"'
23783 2021-04-05 06:34:15.428013   10.128.2.7 → 10.131.0.23  HTTP 885 GET /dddddddddd HTTP/1.1 

$ TZ=UTC tshark -n -t ud  -r tcpdump_20210405_063341.pcap  -Y 'http.request and http contains "request-id: abcdefghijk"' -V | \
  grep "Stream index:"
    [Stream index: 794]
```

## Verify each path layer matched a certain request-id header request session

### Take the matched tcp session from the tcpdump on the node running router pods
```console
$ TZ=UTC tshark -n -t ud  -r tcpdump_20210405_063341.pcap \
  -Y "tcp.stream eq 794"

23152 2021-04-05 06:34:14.307291  10.0.14.231 → 10.0.134.187 TCP 76 31566 → 31790 [SYN] Seq=0 Win=26883 Len=0 MSS=8961 SACK_PERM=1 TSval=2042992871 TSecr=0 WS=256
23157 2021-04-05 06:34:14.307408 10.0.134.187 → 10.0.14.231  TCP 76 31790 → 31566 [SYN, ACK] Seq=0 Ack=1 Win=26697 Len=0 MSS=8911 SACK_PERM=1 TSval=2647712296 TSecr=2042992871 WS=128
23158 2021-04-05 06:34:14.307733  10.0.14.231 → 10.0.134.187 TCP 68 31566 → 31790 [ACK] Seq=1 Ack=1 Win=27136 Len=0 TSval=2042992871 TSecr=2647712296
23161 2021-04-05 06:34:14.307828  10.0.14.231 → 10.0.134.187 PROXYv1 117 31566 → 31790 [PSH, ACK] Seq=1 Ack=1 Win=27136 Len=49 TSval=2042992871 TSecr=2647712296
// HTTP Request
23164 2021-04-05 06:34:14.308841  10.0.14.231 → 10.0.134.187 HTTP 635 GET /dddddddddd HTTP/1.1 
23169 2021-04-05 06:34:14.308861 10.0.134.187 → 10.0.14.231  TCP 68 31790 → 31566 [ACK] Seq=1 Ack=617 Win=27904 Len=0 TSval=2647712297 TSecr=2042992871

// HTTP Response
23680 2021-04-05 06:34:15.311728 10.0.134.187 → 10.0.14.231  HTTP 414 HTTP/1.0 200 OK  (text/html)
23684 2021-04-05 06:34:15.312048  10.0.14.231 → 10.0.134.187 TCP 68 31566 → 31790 [ACK] Seq=617 Ack=347 Win=28160 Len=0 TSval=2042993875 TSecr=2647713300
23748 2021-04-05 06:34:15.416932  10.0.14.231 → 10.0.134.187 TCP 68 31566 → 31790 [FIN, ACK] Seq=617 Ack=347 Win=28160 Len=0 TSval=2042993980 TSecr=2647713300
23753 2021-04-05 06:34:15.417029 10.0.134.187 → 10.0.14.231  TCP 68 31790 → 31566 [FIN, ACK] Seq=347 Ack=618 Win=27904 Len=0 TSval=2647713405 TSecr=2042993980
23754 2021-04-05 06:34:15.417310  10.0.14.231 → 10.0.134.187 TCP 68 31566 → 31790 [ACK] Seq=618 Ack=348 Win=28160 Len=0 TSval=2042993981 TSecr=2647713405
```


Check the above tcp session from the flow logs.
```console
$ grep 31566 ./AWS_ACCOUNT_ID_vpcflowlogs_ap-northeast-1_fl-0e34a6dfefc0db8a9_20210405T0635Z_6427407c.log
version account-id interface-id srcaddr dstaddr srcport dstport protocol packets bytes start end action log-status
// ELB ENI
2 AWS_ACCOUNT_ID eni-047359708552d4965 10.0.134.187 10.0.14.231 31790 31568 6 4 562 1617604407 1617604466 ACCEPT OK
2 AWS_ACCOUNT_ID eni-047359708552d4965 10.0.14.231 10.0.134.187 31568 31790 6 7 988 1617604407 1617604466 ACCEPT OK
// EC2 ENI
2 AWS_ACCOUNT_ID eni-0622b40e5b07b6f17 10.0.14.231 10.0.134.187 31568 31790 6 7 988 1617604502 1617604514 ACCEPT OK
2 AWS_ACCOUNT_ID eni-0622b40e5b07b6f17 10.0.134.187 10.0.14.231 31790 31568 6 4 562 1617604502 1617604514 ACCEPT OK
```

Check the external client IP and port from the ELB access logs matched with above tcp session.
```console
$ grep -E "06:34:1[45]" ./AWS_ACCOUNT_ID_elasticloadbalancing_ap-northeast-1_a17a43204638c41818a7d130503ccfa1_20210405T0635Z_52.199.57.168_ysy02mf0.log
2021-04-05T06:34:14.307137Z a17a43204638c41818a7d130503ccfa1 217.178.150.208:35088 10.0.134.187:31790 0.000499 0.000009 0.000013 - - 567 346 "- - - " "-" - -
2021-04-05T06:34:15.426174Z a17a43204638c41818a7d130503ccfa1 217.178.150.208:35111 10.0.134.187:31790 0.00048 0.000009 0.000013 - - 567 346 "- - - " "-" - -
```
