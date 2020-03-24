# Images Signature Testing

## Install required packages
```cmd
# yum -y install podman httpd-tools -y
```

## Generate certificates for registry
```cmd
# mkdir -p /opt/registry/{auth,certs,data}
# cd /opt/registry/certs
# openssl req -newkey rsa:4096 -nodes -sha256 -keyout domain.key -x509 -days 365 -out domain.crt
:
```

## Generate credentials using htpasswd for registry
```cmd
# htpasswd -bBc /opt/registry/auth/htpasswd regadmin passwd
```

## Add the generated certificates to trusted certificates

```cmd
# cp /opt/registry/certs/domain.crt /etc/pki/ca-trust/source/anchors/
# update-ca-trust
```

## Run private registry
```cmd
# podman run --name private-registry -p 5000:5000 \
     -v /opt/registry/data:/var/lib/registry:z \
     -v /opt/registry/auth:/auth:z \
     -e "REGISTRY_AUTH=htpasswd" \
     -e "REGISTRY_AUTH_HTPASSWD_REALM=Private Registry Realm" \
     -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
     -v /opt/registry/certs:/certs:z \
     -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
     -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
     -d docker.io/library/registry:2

# podman ps -a
CONTAINER ID  IMAGE                         COMMAND               CREATED         STATUS             PORTS                   NAMES
789282acb9b1  docker.io/library/registry:2  /entrypoint.sh /e...  42 seconds ago  Up 41 seconds ago  0.0.0.0:5000->5000/tcp  private-registry

# podman login registry.ocp.rhev.local:5000
Username: regadmin
Password: 
Login Succeeded!
```

## Install httpd for signature store of the images
```cmd
# yum install httpd -y
# mkdir /var/www/html/sigstore -p
# systemctl start httpd
```

## Generate gpg key pair

```cmd
# gpg2 --gen-key
gpg (GnuPG) 2.0.22; Copyright (C) 2013 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Please select what kind of key you want:
   (1) RSA and RSA (default)
   (2) DSA and Elgamal
   (3) DSA (sign only)
   (4) RSA (sign only)
Your selection? 1
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (2048) 
Requested keysize is 2048 bits
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0) 0
Key does not expire at all
Is this correct? (y/N) y

GnuPG needs to construct a user ID to identify your key.

Real name: Private Image Signer
Email address: test@example.com
Comment: Private Image Signer
You selected this USER-ID:
    "Private Image Signer (Private Image Signer) <test@example.com>"

Change (N)ame, (C)omment, (E)mail or (O)kay/(Q)uit? O
You need a Passphrase to protect your secret key.

:

gpg: checking the trustdb
gpg: 3 marginal(s) needed, 1 complete(s) needed, PGP trust model
gpg: depth: 0  valid:   1  signed:   0  trust: 0-, 0q, 0n, 0m, 0f, 1u
pub   2048R/xxx 2020-03-24
      Key fingerprint = ...
uid                  Private Image Signer (Private Image Signer) <test@example.com>
sub   2048R/zzz 2020-03-24

# gpg2 --list-keys
/root/.gnupg/pubring.gpg
pub   2048R/xxx 2020-03-24
uid                  Private Image Signer (Private Image Signer) <test@example.com>
sub   2048R/zzz 2020-03-24
```

## Export and copy your private key to sign images

```cmd
# gpg2 --export-secret-keys -a test@example.com > private-key-imagesigner.asc
# scp private-key-imagesigner.asc your@host-for-signing-images:/path/to/
host-for-signing-images ~# gpg2 --import private-key-imagesigner.asc
host-for-signing-images ~# gpg2 --list-keys
/root/.gnupg/pubring.gpg
pub   2048R/xxx 2020-03-24
uid                  Private Image Signer (Private Image Signer) <test@example.com>
sub   2048R/zzz 2020-03-24# gpg2 --list-keys
```

## Push image with sign

```cmd
// you can also use skopeo to push the local image with sign.
# atomic push --insecure --sign-by test@example.com registry.ocp.rhev.local:5000/test/echo:latest
Registry username: regadmin
Registry password: 
Copying blob 3ad2c2cb58aa done
Copying blob 49577de67301 done
Copying blob 41970e406bb7 done
Copying blob d02565babdb9 done
Copying config 186db29b42 done
Writing manifest to image destination
Signing manifest
Storing signatures

// Look the signature for the pusshed image.
# cd /var/lib/atomic/sigstore/test/
# ll
total 0
drwxr-xr-x. 2 root root 25 Mar 24 18:56 echo@sha256=9043ced01de6c4964c6536c0486c5762d07f2a4c6ee753bd08c953c976a30c47

# cd echo@sha256\=9043ced01de6c4964c6536c0486c5762d07f2a4c6ee753bd08c953c976a30c47
# ls -l
total 1
-rw-r--r--. 1 root root 585 Mar 24 18:56 signature-1

// sync the signature with remote httpd web server for exposing externally.
# rsync -av /var/lib/atomic/sigstore registry.ocp.rhev.local:/var/www/html/sigstore
```

## Configuring Linux container tools to check image signatures


```cmd
// Change the policy to reject all request for image without the signature configured.
# cat /etc/containers/policy.json
{
    "default": [
        {
            "type": "reject"
        }
    ],
    "transports":
        {
            "docker-daemon":
                {
                    "": [{"type":"insecureAcceptAnything"}]
                },
            "docker":
                {
                    "registry.ocp.rhev.local:5000": [
                      {
                        "type": "signedBy",
                        "keyType": "GPGKeys",
                        "keyPath": "/root/.gnupg/pubring.gpg"
                      }
                    ]
                }
        }
}

// the remote image cannot pull due to not fetching the signature.
# podman pull registry.ocp.rhev.local:5000/test/echo:latest
Trying to pull registry.ocp.rhev.local:5000/test/echo:latest...ERRO[0000] Error pulling image ref //registry.ocp.rhev.local:5000/test/echo:latest: Source image rejected: A signature was required, but no signature exists 
Failed
Error: error pulling image "registry.ocp.rhev.local:5000/test/echo:latest": unable to pull registry.ocp.rhev.local:5000/test/echo:latest: unable to pull image: Source image rejected: A signature was required, but no signature exists

// Configure where the signature can refer from.
# cat /etc/containers/registries.d/private-registry.yaml 
docker:
  registry.ocp.rhev.local:5000:
    sigstore-staging: file:///var/lib/atomic/sigstore
    sigstore: http://registry.ocp.rhev.local:8080/sigstore

// You can pull after that.
# podman pull registry.ocp.rhev.local:5000/test/echo:latest
Trying to pull registry.ocp.rhev.local:5000/test/echo:latest...Getting image source signatures
Checking if image destination supports signatures
Copying blob 01caf8c9797a done
Copying blob 5bd0fbc2f2ce done
Copying blob 43657f040db5 done
Copying blob 150b8bacc7b9 done
Copying config 186db29b42 done
Writing manifest to image destination
Storing signatures
186db29b42166f9e65ef276813e90c28d0efbca320671c41c1ee851694663eac
```

Done.
