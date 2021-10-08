# How to test ClousterLogForwarder with inseucre and secure Forwarding


## Create target fluentd server first with insecure
```cmd
$ oc new-project test-secureforward
$ oc new-app fluentd --docker-image=fluent/fluentd
$ oc adm policy add-scc-to-user anyuid -z default
$ cat <<EOF > /tmp/fluent.conf
# Receive events from 24224/tcp
# This is used by log forwarding and the fluent-cat command
<source>
  @type forward
  port 24224
</source>
<match **>
  @type stdout
</match>
EOF
$ oc create configmap fluentconf --from-file=/tmp/fluent.conf
$ oc edit deploy fluentd
:
    spec:
      containers:
      - name: fluentd
      :
        volumeMounts:                           <--- ADD 
        - mountPath: /fluentd/etc/fluent.conf   <--- ADD
          name: config                          <--- ADD
          subPath: fluent.conf                  <--- ADD 
:
      volumes:
      - configMap:                              <--- ADD
          defaultMode: 420                      <--- ADD
          items:                                <--- ADD
          - key: fluent.conf                    <--- ADD
            path: fluent.conf                   <--- ADD
          name: fluentconf                      <--- ADD
        name: config                            <--- ADD
:
$ oc delete <existing fluentd pod>
```

## Create ClusterLogForwarder instance with insecure
```yaml
apiVersion: v1
items:
- apiVersion: logging.openshift.io/v1
  kind: ClusterLogForwarder
  metadata:
    name: instance
    namespace: openshift-logging
  spec:
    outputs:
    - name: remote-fluentd
      type: fluentdForward
      url: tcp://fluentd.test-secureforward.svc.cluster.local:24224
    pipelines:
    - inputRefs:
      - application
      - infrastructure
      - audit
      name: fluentd-insecure
      outputRefs:
      - remote-fluentd
```

## Create dummy TLS certificates with Service FQDN and IP
```cmd
$ oc get svc -n test-secureforward
NAME        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)              AGE
fluentd     ClusterIP   172.30.197.149   <none>        5140/TCP,24224/TCP   91m

$ DOMAIN=fluentd.test-secureforward.svc.cluster.local,DNS:172.30.197.149
$ openssl req \
    -newkey rsa:2048 -x509 -nodes -keyout tls.key \
    -new -out tls.crt -subj /CN=$DOMAIN -reqexts SAN -extensions SAN \
    -config <(cat /etc/pki/tls/openssl.cnf; printf "[SAN]\nsubjectAltName=DNS:$DOMAIN\n\nbasicConstraints=CA:TRUE\n") \
    -sha256 -days 3650
$ cp tls.crt ca-bundle.crt
$ ls -1
tls.key
tls.crt
ca-bundle.crt
```

## Create target fluentd server as secure
```cmd
$ oc create secret generic server-cert \
  --from-file=tls.crt=tls.crt \
  --from-file=tls.key=tls.key \
  --from-file=ca-bundle.crt=ca-bundle.crt
$ oc edit deploy fluentd
:
    spec:
      containers:
      - name: fluentd
      :
        volumeMounts:                           
        - mountPath: /fluentd/etc/fluent.conf   
          name: config                          
          subPath: fluent.conf                  
        - mountPath: /fluentd/etc/forward       <--- ADD 
          name: secureforwardcerts              <--- ADD 
          readOnly: true                        <--- ADD 
:
      volumes:
      - configMap:                              
          defaultMode: 420                      
          items:                                
          - key: fluent.conf                    
            path: fluent.conf                   
          name: fluentconf                      
        name: config
      - name: secureforwardcerts               <--- ADD 
        secret:                                <--- ADD 
          defaultMode: 420                     <--- ADD 
          optional: true                       <--- ADD 
          secretName: server-cert              <--- ADD     
:
```

WIP

