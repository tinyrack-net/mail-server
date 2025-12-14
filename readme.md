
```bash
curl -fL https://get.k3s.io | \
sh -s - server \
  --cluster-init \
  --cluster-cidr=10.57.0.0/16 \
  --service-cidr=10.58.0.0/16 \
  --disable traefik \
  --disable servicelb
```

```bash
kubectl create namespace sealed-secrets
kubectl -n sealed-secrets apply -f ./flux/main.key.yaml --force
```

```bash
flux bootstrap gitea \
  --token-auth \
  --hostname=git.winetree94.com \
  --repository=mail-server \
  --branch=main \
  --path=./flux/clusters/production \
  --owner=tinyrack
```

```bash
kubectl create secret generic docmost-secret \
        --namespace docmost-system \
        --dry-run=client \
        --from-literal=SOME_SECRET_KEY=SOME_SECRET_VALUE \
        --from-literal=SOME_SECRET_KEY=SOME_SECRET_VALUE \
        --from-literal=SOME_SECRET_KEY=SOME_SECRET_VALUE -o yaml \
        | kubeseal --cert ./tinyrack-production-key.crt \
        > ./some.secret.yaml
```
