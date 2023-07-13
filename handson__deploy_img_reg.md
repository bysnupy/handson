# How to deploy a simple image registry

## Install required Packakges
```cmd
# yum -y install podman httpd-tools
```
## Generate certificates with SAN
```cmd
# mkdir -p /opt/registry/{auth,certs,data}
# DOMAIN=priv.reg.example.com
# openssl req \
    -newkey rsa:2048 -x509 -nodes -keyout /opt/registry/certs/registry.key \
    -new -out /opt/registry/certs/registry.crt -subj /CN=$DOMAIN -reqexts SAN -extensions SAN \
    -config <(cat /etc/pki/tls/openssl.cnf; printf "[SAN]\nsubjectAltName=DNS:$DOMAIN\n\nbasicConstraints=CA:TRUE\n") \
    -sha256 -days 3650
```
## Create a registry credential file
```cmd
# htpasswd -bBc /opt/registry/auth/htpasswd admin redhat
```
## Open a port to interact with the image registry
```cmd
# firewall-cmd --add-port=5000/tcp --zone=public   --permanent
# firewall-cmd --reload
```

## Update the registry CA as trusted CA bundles.
```cmd
# cp /opt/registry/certs/registry.crt /etc/pki/ca-trust/source/anchors/
# update-ca-trust
```

## Save and load a registry image for disconnected network, if you required
```cmd
# podman pull docker.io/library/registry:2
# podman save <image id> -o registry2.tar
# podman load < registry2.tar
```

## Run the image registry
```cmd
# podman run --name priv-reg -p 5000:5000 \
     -v /opt/registry/data:/var/lib/registry:z \
     -v /opt/registry/auth:/auth:z \
     -e "REGISTRY_AUTH=htpasswd" \
     -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
     -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
     -e REGISTRY_STORAGE_DELETE_ENABLED=true \
     -v /opt/registry/certs:/certs:z \
     -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/registry.crt \
     -e REGISTRY_HTTP_TLS_KEY=/certs/registry.key \
     -d docker.io/library/registry:2
```

## Check if the image registry is up and running
```cmd
# podman ps
CONTAINER ID  IMAGE                         COMMAND               CREATED        STATUS            PORTS                   NAMES
f4ba69b16a8e  docker.io/library/registry:2  /etc/docker/regis...  7 seconds ago  Up 8 seconds ago  0.0.0.0:5000->5000/tcp  priv-reg
```
