# sealed secrets key gen

Set a reminder to rotate this in 10 years...

```bash
SECRETNAME=secrets-key
NAMESPACE=sealed-secrets
PUBLICKEY=tls.crt
PRIVATEKEY=tls.key
```

```bash
openssl req -x509 -days 3650 -nodes -newkey rsa:4096 -keyout "$PRIVATEKEY" -out "$PUBLICKEY" -subj "/CN=sealed-secret/O=sealed-secret"
```

```
kubectl -n "$NAMESPACE" create secret tls "$SECRETNAME" --cert="$PUBLICKEY" --key="$PRIVATEKEY"
kubectl -n "$NAMESPACE" label secret "$SECRETNAME" sealedsecrets.bitnami.com/sealed-secrets-key=active
```
