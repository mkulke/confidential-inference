# Confidential Inference

WIP

## Certs

### Create self-signed CA

```bash
cd certs
openssl genrsa -traditional -out ca.key 2048
openssl req -new -key ca.key -out ca.csr -subj "/O=CNCF/OU=CoCo/CN=CC Apps"
openssl req -x509 -days 3650 -key ca.key -in ca.csr -out ca.crt
```

### Create https key + CSR

```bash
openssl ecparam -name prime256v1 -genkey -noout -out ci-app.key
openssl req -new -key ci-app.key -out ci-app.csr -config csr.cnf
```

### Sign cert

```bash
openssl x509 -req -in ci-app.csr -CA ca.crt -CAkey ca.key \
    -CAcreateserial -days 825 -sha256 \
    -extfile csr.cnf -extensions req_ext \
    -out ../kustomize/base/tls.crt
```

## Trustee

tbd: setup with trustee-operator

### Upload app key

```bash
cd certs
kubectl create secret generic ci-app \
    --from-file=tls.key=ci-app.key \
    -n trustee-operator-system
```

### Adjust trustee deployment

We reference the secret in trustee and we set the kbs policy to pass everything (TODO: replace with proper policy).

```bash
cd config/samples/all-in-one
cat <<EOF> resource-policy.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: resource-policy
  namespace: trustee-operator-system
data:
  policy.rego: |
    package policy

    default allow = true
EOF
cat <<EOF> patch-kbs-resources.yaml
apiVersion: confidentialcontainers.org/v1alpha1
kind: KbsConfig
metadata:
  name: kbsconfig-sample
  namespace: trustee-operator-system
spec:
  kbsSecretResources:
  - "ci-app"
EOF
kubectl apply -k .
```

## Sealed secret

Note: the sealed secret is hardcoded to point to the `ci-app/tls.key` secret. But for reference this is how we would construct it:

```bash
cat <<EOF> vault.json
{
    "version": "0.1.0",
    "type": "vault",
    "name": "kbs:///default/ci-app/tls.key",
    "provider": "kbs",
    "provider_settings": {},
    "annotations": {}
}
EOF
cat vault.json | python3 -c '
import base64, sys
in = sys.stdin.buffer.read()
b64urlnopad = base64.urlsafe_b64_encode(in).rsript(b"=").decode()
print("sealed.fakejwsheader." + b64urlnopad + ".fakesignature")
' > tls.key.sealed
kubectl create secret generic sealed-tls-key --from-file=tls.key=tls.key.sealed
```

## Deployment

### Peerpods

This assumes a functional CoCo peerpods installation.

Edit init-data.toml and create encode in b64 for the init-data pod annotation.

```bash
cd kustomize/base
vim init-data.toml
base64 -w0 < init-data.toml > init-data.b64
```

Deploy with kustomize

```bash
cd kustomize/overlays/peerpod
kubectl apply -k .
```

## Verify

```bash
curl --cacert ./certs/ca.crt https://resilient-wombat.northeurope.cloudapp.azure.com
```
